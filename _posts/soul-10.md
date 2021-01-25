---
title: soul学习10——数据同步主流程(上)
date: 2021/1/25 22:13:10
categories:
- soul
tags:
- soul
- 网关
keywords:
- soul , 网关, soul-web, websocket, 数据同步, data Subscriber
---
# 概述

从今天起开始研究soul数据同步的内容。soul区别于其它网关的一大特点就是动态配置更新功能。所有网关配置在admin上更新后会自动同步到soul网关节点。主要需要同时的数据有3类：auteData、metaData、PluginData，更新方式也有websock、zookeeper、http长轮询、nacos4种。本文旨在梳理数据同步的核心流程，后续着重研究websock、zookeeper和http长轮询的实现细节。

<!-- more -->

# soul-sync-data-api

我们从soul-sync-data-api入手，该项目定义了同步数据主要用的接口。该项目下主要有以下四个接口：

1. AuthDataSubscriber，权限数据订阅接口
2. MetaDataSubscriber，meta数据订阅接口
3. PluginDataSubscriber，插件数据订阅接口
4. SyncDataService，数据同步服务接口

这些接口大致可以分为2类：XXXDataSubscriber和SyncDataService。其中XXXDataSubscriber接口是让各订阅者实现在接到admin发来的更新、删除相关数据后需要执行的具体代码逻辑。从接口数量中我们可以发现soul在执行过程中，主要有3类数据需要在admin和soul网关服务之间同。他们分别是：

1. auth数据，sign插件在验证api层的签名是需要app公钥，app私钥等信息，这些信息在admin生成需要传输给soul网关
2. meta数据，服务的基本信息，这些信息是admin管理页面"系统管理"->"元数据管理"中的内容，这些信息表示了有哪些服务注册到了我们soul网关中
3. plugin数据，插件配置数据，插件配置相关的数据，admin管理页面"插件列表"中配置的数据，这些数据关系到soul网关中插件运行。这些数据细化有：
   1. pluginData，插件配置
   2. selectorData，插件选择器配置
   3. ruleData，插件规则配置

SyncDataService仅仅是为了标记各个数据同步服务，因为该接口没有定义任何放。进一步研究发现实现SyncDataService接口的有4个，分别是：`HttpSyncDataService`、 `NacosSyncDataService`、 `ZookeeperSyncDataService`、 `WebsocketSyncDataService`，即soul实现的4种同步数据的方式。下面我们以官方默认的websocket同步插件配置数据入手梳理数据同步的流程。

# PluginDataSubscriber

我们先看看在soul网关订阅者接收到时间通知都做了哪些主要操作。PluginDataSubscriber主要有以下方法：

1.  void onSubscribe(PluginData pluginData)，更新PluginData的逻辑
2.  void unSubscribe(PluginData pluginData)，删除PluginData的逻辑
3. void refreshPluginDataAll()，刷新全部插件数据
4. void refreshPluginDataSelf(List<PluginData> pluginDataList)，批量删除插件数据
5. void onSelectorSubscribe(SelectorData selectorData) ，更新选择器数据的逻辑
6. void unSelectorSubscribe(SelectorData selectorData)，删除指定选择器的逻辑
7. void refreshSelectorDataAll() ，删除全部选择器数据的逻辑
8. void refreshSelectorDataSelf(List<SelectorData> selectorDataList)，批量删除选择器数据
9. void onRuleSubscribe(SelectorData selectorData) ，更新指定规则数据的逻辑
10. void unRuleSubscribe(SelectorData selectorData)，删除指定规则的逻辑
11. void refreshRuleDataAll() ，删除全部规则数据的逻辑
12. void refreshRuleDataSelf(List<SelectorData> selectorDataList)，批量规则选择器数据

该接口的实现类只有1个，那就是plugin-base中的`CommonPluginDataSubscriber`。直接上代码，逻辑比较简单

```java
public class CommonPluginDataSubscriber implements PluginDataSubscriber {
    
  	// 插件名 ==》 各插件的配置更新逻辑
    private final Map<String, PluginDataHandler> handlerMap;
		// 插件更新逻辑作为初始化的传参
    public CommonPluginDataSubscriber(final List<PluginDataHandler> pluginDataHandlerList) {
        this.handlerMap = pluginDataHandlerList.stream().collect(Collectors.toConcurrentMap(PluginDataHandler::pluginNamed, e -> e));
    }
    
    @Override
    public void onSubscribe(final PluginData pluginData) {
      // 所有操作最终调的都是subscribeDataHandler方法
        subscribeDataHandler(pluginData, DataEventTypeEnum.UPDATE);
    }
    …………………………
      // 传参：更新的数据：PluginData、selectorData、ruleData，操作：update、create、delete、refresh、myself
    private <T> void subscribeDataHandler(final T classData, final DataEventTypeEnum dataType) {
      // 判空
        Optional.ofNullable(classData).ifPresent(data -> {
            if (data instanceof PluginData) {
              // 操作 PluginData
                PluginData pluginData = (PluginData) data;
              // 如果是更新操作
                if (dataType == DataEventTypeEnum.UPDATE) {
                  // 更新内存（ConcurrentMap）中对应的pluginData
                    BaseDataCache.getInstance().cachePluginData(pluginData);
                  // 找到对应插件的配置数据操作类，并调用更新方法
                    Optional.ofNullable(handlerMap.get(pluginData.getName())).ifPresent(handler -> handler.handlerPlugin(pluginData));
                } else if (dataType == DataEventTypeEnum.DELETE) {
                  // 删除内存中的pluginData
                    BaseDataCache.getInstance().removePluginData(pluginData);
                   // 找到对应插件的配置数据操作类，并调用其删除方法
                    Optional.ofNullable(handlerMap.get(pluginData.getName())).ifPresent(handler -> handler.removePlugin(pluginData));
                }
            } else if (data instanceof SelectorData) {
              // 同PluginData
                ………………
            } else if (data instanceof RuleData) {
              // 同PluginData
                ………………
        });
    }
}

```

其实逻辑很简单，就是根据方法定义修改soul网关`BaseDataCache`中存储的配置信息。BaseDataCache存储配置信息的方式是用ConcurrentMap。各个插件具体执行时也是用BaseDataCache中拿到响应配置的。同时，有些插件的有些配置改变时，处理内存中的配置一定要更新外，还需要一些的别的操作，这是就需要插件自行实现`PluginDataHandler`，实现响应方法然后将其交给spring来管理，这样，在更新配置后就自然会被调用到了。目前插件可以定义plugin、selector、rule发生更新或删除时特殊的业务操作。比如在Monitor监控插件中，除了要在BaseDataCache更新配合，还需要更新`MetricsConfig`的配置并让Metrics组件重启。

# WebsocketSyncDataService

WebsocketSyncDataService的代码是写在`soul-sync-data-websocket`中，凡是引用`soul-spring-boot-starter-sync-data-websocket`的项目都会构造WebsocketSyncDataService并交给spring管理，目前使用WebsocketSyncDataService的项目就是soul网关（就是soul-web）。源码如下：

```java
public class WebsocketSyncDataService implements SyncDataService, AutoCloseable {

    private final List<WebSocketClient> clients = new ArrayList<>();

    private final ScheduledThreadPoolExecutor executor;
    
    /**
     * Instantiates a new Websocket sync cache.
     *
     * @param websocketConfig      webSocket配置，只有1个url参数，指向admin，如果有多个admin配置多个
     * @param pluginDataSubscriber soul网关中的pluginData订阅者CommonPluginDataSubscriber,负责实现插件数据更新逻辑
     * @param metaDataSubscribers  网关中的metaData订阅者，dubbo、sofa、tars等RPC框架实现的下游服务有meta信息网关需要关注
     * @param authDataSubscribers  auth信息订阅者，主要是sign插件关注
     */
    public WebsocketSyncDataService(final WebsocketConfig websocketConfig,
                                    final PluginDataSubscriber pluginDataSubscriber,
                                    final List<MetaDataSubscriber> metaDataSubscribers,
                                    final List<AuthDataSubscriber> authDataSubscribers) {
        String[] urls = StringUtils.split(websocketConfig.getUrls(), ",");
      // 创建 定时任务的线程池
        executor = new ScheduledThreadPoolExecutor(urls.length, SoulThreadFactory.create("websocket-connect", true));
        for (String url : urls) {
            try {
              // 根据URL创建SoulWebsocketClient，主要有webSocketClient和调用各个Subscriber的WebsocketDataHandler
              // webSocketClient可以实现当连接（http->tcp）中有message是触发WebsocketDataHandler执行订阅中心方法
                clients.add(new SoulWebsocketClient(new URI(url), Objects.requireNonNull(pluginDataSubscriber), metaDataSubscribers, authDataSubscribers));
            } catch (URISyntaxException e) {
                log.error("websocket url({}) is error", url, e);
            }
        }
        try {
            for (WebSocketClient client : clients) {
              // 尝试连接 admin对应websoucket接口，3000超时，也就是向websocket注册自己
                boolean success = client.connectBlocking(3000, TimeUnit.MILLISECONDS);
                if (success) {
                    log.info("websocket connection is successful.....");
                } else {
                    log.error("websocket connection is error.....");
                }
              // 每个websocket创建1个 每个隔30秒就执行的任务
                executor.scheduleAtFixedRate(() -> {
                  // 验证客户端是不是关闭了，如果关闭则从立案
                    try {
                        if (client.isClosed()) {
                            boolean reconnectSuccess = client.reconnectBlocking();
                            if (reconnectSuccess) {
                                log.info("websocket reconnect is successful.....");
                            } else {
                                log.error("websocket reconnection is error.....");
                            }
                        }
                    } catch (InterruptedException e) {
                        log.error("websocket connect is error :{}", e.getMessage());
                    }
                }, 10, 30, TimeUnit.SECONDS);
            }
            /* client.setProxy(new Proxy(Proxy.Type.HTTP, new InetSocketAddress("proxyaddress", 80)));*/
        } catch (InterruptedException e) {
            log.info("websocket connection...exception....", e);
        }

    }
    
    @Override
    public void close() {
        for (WebSocketClient client : clients) {
          // 关闭所有 websocketClient
            if (!client.isClosed()) {
                client.close();
            }
        }
        if (Objects.nonNull(executor)) {
            executor.shutdown();
        }
    }
}
```

代码其实也不负责，核心意思就是在soul网关启动后，根据websocket配置的url，连接soul-admin，当有数据发送过来时就调用相应的数据订阅者，处理对应配置更新或删除的逻辑。通知每个websocket都有1个定时线程在监控websocketClient的连接状态，如果关闭则重连。

# 总结

今天主要从soul网关这一侧了解了数据同步的相关内容，总体来说，就是soul网关实现了4类数据的订阅者，当数据同步方式发现数据变化事件时就调用响应订阅者执行数据更新逻辑。其中如果是websocke同步方式，会在网关启动时主动连接admin对应接口，接收发送来的数据，同时会有定时任务检查websocket是否断开，如果断开则重连。