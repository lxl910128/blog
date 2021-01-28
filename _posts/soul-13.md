---
title: soul学习13——zookeeper同步数据
date: 2021/1/28 23:39:10
categories:
- soul
tags:
- soul
- 网关
keywords:
- soul , 网关, soul-web, zookeeper 数据同步
---
# 概述

前几篇文章介绍了suol用websocket、http长轮询的方式同步数据，今天我们来看看zookeeper同步数据是如何实现的。使用websocket同步数据时，在soul-admin服务端有负责监听数据变化事件的`HttpLongPollingDataChangedListener`以及负责接收soul网关连接的`WebsocketCollector`，在soul服务端有实现`SyncDataService`的`WebsocketSyncDataService`，负责实现当接收到配置变化。下面我们来对应看看zookeeper这几部分是如何实现的。

<!--more-->

# 网关端

在soul网关端如果要开启zookeeper同步数据的方式需要做如下配置：

```yaml
soul:
    file:
      enabled: true
    corss:
      enabled: true
    dubbo :
      parameter: multi
    sync:
      zookeeper:
				url: localhost:2181
				sessionTimeout: 5000
				connectionTimeout: 2000
```

与websocket和http长轮询不同的是，它需要配置一个和admin公用的zookeeper集群地址。网关端的`ZookeeperSyncDataService`代码如下，代码值选取了pluginData和selectorData的更新，其他的appAuth、rule、metaData数据的更新大同小异略过

```java
public class ZookeeperSyncDataService implements SyncDataService, AutoCloseable {
    private final ZkClient zkClient;// zk客户端
    // 各种订阅者 负责处理 各种各类配置数据变化的逻辑
    private final PluginDataSubscriber pluginDataSubscriber;
    private final List<MetaDataSubscriber> metaDataSubscribers;
    private final List<AuthDataSubscriber> authDataSubscribers;

    /**
     * 初始化zookeeper异步数据接收处理服务
     */
    public ZookeeperSyncDataService(final ZkClient zkClient, final PluginDataSubscriber pluginDataSubscriber,
                                    final List<MetaDataSubscriber> metaDataSubscribers, final List<AuthDataSubscriber> authDataSubscribers) {
        this.zkClient = zkClient;
        this.pluginDataSubscriber = pluginDataSubscriber;
        this.metaDataSubscribers = metaDataSubscribers;
        this.authDataSubscribers = authDataSubscribers;
        // 观察各种数据的变化
        watcherData();
        watchAppAuth();
        watchMetaData();
    }
    // 观察 plugin、selector、rule的数据的变化
    private void watcherData() {
        final String pluginParent = ZkPathConstants.PLUGIN_PARENT;
        //  /soul/plugin 下所有的子地址，该地址下是全部在运行的插件的名称
        List<String> pluginZKs = zkClientGetChildren(pluginParent);
        for (String pluginName : pluginZKs) {
            // 观测某个插件的 plugin、selector、rule的数据
            watcherAll(pluginName);
        }
        // 观测 /soul/plugin/ 下所有的子目录，当有有变化时再调用 watcherAll
        // 此种情况发生在 新开启或关闭插件时触发
        // 此时需要把这些插件的plugin、selector、rule数据监控起来
        zkClient.subscribeChildChanges(pluginParent, (parentPath, currentChildren) -> {
            if (CollectionUtils.isNotEmpty(currentChildren)) {
                for (String pluginName : currentChildren) {
                    watcherAll(pluginName);
                }
            }
        });
    }
    // 观测某插件的 plugin、selector、rule的数据
    private void watcherAll(final String pluginName) {
        watcherPlugin(pluginName);
        watcherSelector(pluginName);
        watcherRule(pluginName);
    }
    // 对于某个插件的plugin的数据进行观测
    private void watcherPlugin(final String pluginName) {
        // 地址是 /soul/plugin/${pluginName}
        String pluginPath = ZkPathConstants.buildPluginPath(pluginName);
        // 地址不存在则创建
        if (!zkClient.exists(pluginPath)) {
            zkClient.createPersistent(pluginPath, true);
        }
        // 向pluginData的订阅者发送从该地址拿到的数据
        cachePluginData(zkClient.readData(pluginPath));
        // 开始观测这个地址，如果 有更新或删除操作则 再向pluginData的订阅者发送数据
        subscribePluginDataChanges(pluginPath, pluginName);
    }
    // 观测某插件的 selector数据
    private void watcherSelector(final String pluginName) {
        String selectorParentPath = ZkPathConstants.buildSelectorParentPath(pluginName);
        List<String> childrenList = zkClientGetChildren(selectorParentPath);
        if (CollectionUtils.isNotEmpty(childrenList)) {
            childrenList.forEach(children -> {
                String realPath = buildRealPath(selectorParentPath, children);
                // 缓存selector 具体某个配置
                cacheSelectorData(zkClient.readData(realPath));
                // 并监控他的变更 ，主要是更改和删除
                subscribeSelectorDataChanges(realPath);
            });
        }
        // 观测 /soul/selector/${pluginName} 所有子目录
        // 主要处理selector增加的情况
        subscribeChildChanges(ConfigGroupEnum.SELECTOR, selectorParentPath, childrenList);
    }

    // 观测目录下子目录的变更，主要针对 各个配置的增删的情况
    private void subscribeChildChanges(final ConfigGroupEnum groupKey, final String groupParentPath, final List<String> childrenList) {
        switch (groupKey) {
            case SELECTOR:
                // 当 /soul/selector 下文件夹发生变化时
                zkClient.subscribeChildChanges(groupParentPath, (parentPath, currentChildren) -> {
                    if (CollectionUtils.isNotEmpty(currentChildren)) {
                        // 新老子目录取交，找出增加的目录
                        List<String> addSubscribePath = addSubscribePath(childrenList, currentChildren);
                        addSubscribePath.stream().map(addPath -> {
                            String realPath = buildRealPath(parentPath, addPath);
                            // 缓存新目录的数据
                            cacheSelectorData(zkClient.readData(realPath));
                            return realPath;
                            // 监控新目录的的变更
                        }).forEach(this::subscribeSelectorDataChanges);

                    }
                });
                break;
            default:
                throw new IllegalStateException("Unexpected groupKey: " + groupKey);
        }
    }
  // 当某个pluginData数据变增加或删除了
    private void subscribePluginDataChanges(final String pluginPath, final String pluginName) {
        zkClient.subscribeDataChanges(pluginPath, new IZkDataListener() {

            @Override // /soul/plugin/${plugin} 被变更时触发，向订阅者发送新数据
            public void handleDataChange(final String dataPath, final Object data) {
                Optional.ofNullable(data)
                        .ifPresent(d -> Optional.ofNullable(pluginDataSubscriber).ifPresent(e -> e.onSubscribe((PluginData) d)));
            }

            @Override // /soul/plugin/${plugin} 被删除时触发，向订阅者发送移除数据的时间
            public void handleDataDeleted(final String dataPath) {
                final PluginData data = new PluginData();
                data.setName(pluginName);
                Optional.ofNullable(pluginDataSubscriber).ifPresent(e -> e.unSubscribe(data));
            }
        });
    }
   // 当 selectorData数据变更或删除了
    private void subscribeSelectorDataChanges(final String path) {
        zkClient.subscribeDataChanges(path, new IZkDataListener() {
            @Override 
            public void handleDataChange(final String dataPath, final Object data) {
                cacheSelectorData((SelectorData) data);
            }

            @Override
            public void handleDataDeleted(final String dataPath) {
                unCacheSelectorData(dataPath);
            }
        });
    }
    // 向pluginData订阅者发送数据
    private void cachePluginData(final PluginData pluginData) {
        Optional.ofNullable(pluginData).flatMap(data -> Optional.ofNullable(pluginDataSubscriber)).ifPresent(e -> e.onSubscribe(pluginData));
    }
   // 向SelectorData订阅者发送数据
    private void cacheSelectorData(final SelectorData selectorData) {
        Optional.ofNullable(selectorData)
                .ifPresent(data -> Optional.ofNullable(pluginDataSubscriber).ifPresent(e -> e.onSelectorSubscribe(data)));
    }
	// 取消某些selectorData
    private void unCacheSelectorData(final String dataPath) {
        SelectorData selectorData = new SelectorData();
        final String selectorId = dataPath.substring(dataPath.lastIndexOf("/") + 1);
        final String str = dataPath.substring(ZkPathConstants.SELECTOR_PARENT.length());
        final String pluginName = str.substring(1, str.length() - selectorId.length() - 1);
        selectorData.setPluginName(pluginName);
        selectorData.setId(selectorId);
        Optional.ofNullable(pluginDataSubscriber).ifPresent(e -> e.unSelectorSubscribe(selectorData));
    }
    // 找出currentChildren 中 alreadyChildren没有的
    private List<String> addSubscribePath(final List<String> alreadyChildren, final List<String> currentChildren) {
        if (CollectionUtils.isEmpty(alreadyChildren)) {
            return currentChildren;
        }
        return currentChildren.stream().filter(current -> alreadyChildren.stream().noneMatch(current::equals)).collect(Collectors.toList());
    }
  // 拿某个地址的所有子目录，如果这个目录不存在则创建
    private List<String> zkClientGetChildren(final String parent) {
        if (!zkClient.exists(parent)) {
            zkClient.createPersistent(parent, true);
        }
        return zkClient.getChildren(parent);
    }
    @Override
    public void close() {
        if (null != zkClient) {
            zkClient.close();
        }
    }
}
```

总体来说soul在zookeeper中主要的目录结构是：/soul/plugin/\${pluginName}、  /soul/selector/\${pluginName}/xxx、 /soul/rule/\${pluginName}/xxx、 /soul/selector/\${pluginName}/xxx、 /soul/auth/xxx、 /soul/metaData/xxx。启动时都会先从对应目录下找数据，然后发给对应数据的订阅者。然后soul主要关注最底层目录数据的变更和删除事件，在父级目录关注子目录变更达到关注新数据增加事件。当触发相应事件后借助对应数据的订阅者触发配置数据更新操作。

# admin端

admin端主要是将变更的数据更新到zookeeper对应的目录下。它主要有2个类同样一个是实现`DataChangedListener`的`ZookeeperDataChangedListener`以及在启动时运行的初始化数据的`ZookeeperDataInit`。首先说ZookeeperDataInit它继承了CommandLineRunner，在spring boot应用中，我们可以在程序启动之前spring容器启动好之后用它执行任务执行任何任务。源码如下

```java
public class ZookeeperDataInit implements CommandLineRunner {
    private final ZkClient zkClient;// 从spring中拿到 zkClient
    private final SyncDataService syncDataService;// 同步数据的服务
    public ZookeeperDataInit(final ZkClient zkClient, final SyncDataService syncDataService) {
        this.zkClient = zkClient;
        this.syncDataService = syncDataService;
    }

    @Override
    public void run(final String... args) {
      // 核心路径
        String pluginPath = ZkPathConstants.PLUGIN_PARENT;
        String authPath = ZkPathConstants.APP_AUTH_PARENT;
        String metaDataPath = ZkPathConstants.META_DATA;
      // 如果都不存在，如果第一次用的话肯定不存在
        if (!zkClient.exists(pluginPath) && !zkClient.exists(authPath) && !zkClient.exists(metaDataPath)) {
          // 数据同步服务发送一次同步所有数据的事件，这是ZookeeperDataChangedListener就会接到所有数据的更新事件
            syncDataService.syncAll(DataEventTypeEnum.REFRESH);
        }
    }
}
```

ZookeeperDataInit干的一件事就是启动时看看zookeeper里面有没有历史数据，如果没有数据则发送全量数据刷新事件，这是`ZookeeperDataChangedListener`就会接到所有类型数据的刷新时间并作出响应，下面我们看下它的源码，由于大同小异所以以Plugin数据为例做介绍：

```java
public class ZookeeperDataChangedListener implements DataChangedListener {
    private final ZkClient zkClient;// spring容器中拿到
    public ZookeeperDataChangedListener(final ZkClient zkClient) {
        this.zkClient = zkClient;
    }
    @Override
    public void onPluginChanged(final List<PluginData> changed, final DataEventTypeEnum eventType) {
      // plugin数据变化时
        for (PluginData data : changed) {
          // /soul/plugin/${pluginName}
            String pluginPath = ZkPathConstants.buildPluginPath(data.getName());
            // 删除数据做特殊处理
            if (eventType == DataEventTypeEnum.DELETE) {
              // 删除zk对应目录
                deleteZkPathRecursive(pluginPath);
              // 删除zk对应插件selector数据
                String selectorParentPath = ZkPathConstants.buildSelectorParentPath(data.getName());
                deleteZkPathRecursive(selectorParentPath);
               // 删除zk对应插件rule数据
                String ruleParentPath = ZkPathConstants.buildRuleParentPath(data.getName());
                deleteZkPathRecursive(ruleParentPath);
                continue;
            }
            //更新和创建 走一个逻辑就是想 zk节点加数据
            insertZkNode(pluginPath, data);
        }
    }
    // 先尝试创建zk节点 再添加数据
    private void insertZkNode(final String path, final Object data) {
        createZkNode(path);
        zkClient.writeData(path, data);
    }
    // 创建节点
    private void createZkNode(final String path) {
        if (!zkClient.exists(path)) {
            zkClient.createPersistent(path, true);
        }
    }
    // 删除节点
    private void deleteZkPath(final String path) {
        if (zkClient.exists(path)) {
            zkClient.delete(path);
        }
    }
    // 递归删目录及其子目录
    private void deleteZkPathRecursive(final String path) { 
        if (zkClient.exists(path)) {
            zkClient.deleteRecursive(path);
        }
    }
}
```

admin主要负责数据变更时向zk写或删数据，逻辑简单清晰。

# 总结

整体老说zk同步数据逻辑非常清晰，这也可以看出zk非常适合干这样的工作的，在集群环境下同步meta信息。从中我学到了如果向监控数据新增可以在父节点关注子节点的变化，其他情况可以关注某个目录下数据的修改或删除。