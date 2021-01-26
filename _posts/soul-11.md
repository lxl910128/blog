---
title: soul学习10——数据同步主流程(下)
date: 2021/1/26 23:43:10
categories:
- soul
tags:
- soul
- 网关
keywords:
- soul , 网关, soul-web, websocket, 数据同步, ApplicationListener
---
# 概述

从今天admin端观察一下数据同步。主要看admin是在pluginData数据变更时是如何通知soul网关更新pluginData的数据。

<!-- more -->

# WebsocketCollector

上一篇文章我们聊到如果使用websocket同步数据的方式，soul网关会在启动时连接soul-admin的"ws://adminIP:9095/websocket"。跟踪看到了soul-admin的`WebsocketCollector`，从名字大胆猜测是websocket相关的`collector`层，源码如下：

```java
// 地址与服务端注册地址相吻合
@ServerEndpoint("/websocket")
public class WebsocketCollector {
  // 将session保存在内存中
    private static final Set<Session> SESSION_SET = new CopyOnWriteArraySet<>();
    private static final String SESSION_KEY = "sessionKey";
    // 在建立连接时执行
    @OnOpen
    public void onOpen(final Session session) {
      // 打日志 保存session到set中
        log.info("websocket on open successful....");
        SESSION_SET.add(session);
    }
    // 当收到信息是的逻辑
    @OnMessage
    public void onMessage(final String message, final Session session) {
      // 收到自己发给自己的消息？
        if (message.equals(DataEventTypeEnum.MYSELF.name())) {
            try {// 在threadLocal保存 session 键值为sessionKey
                ThreadLocalUtil.put(SESSION_KEY, session);
                SpringBeanUtils.getInstance().getBean(SyncDataService.class).syncAll(DataEventTypeEnum.MYSELF);
            } finally {// 清理threadlocal
                ThreadLocalUtil.clear();
            }
        }
    }
    //当某个连接关闭时，将session从内存中删除
    @OnClose
    public void onClose(final Session session) {
        SESSION_SET.remove(session);
        ThreadLocalUtil.clear();
    }
    //当发生错误时也清理session并打日志
    @OnError
    public void onError(final Session session, final Throwable error) {
        SESSION_SET.remove(session);
        ThreadLocalUtil.clear();
        log.error("websocket collection error: ", error);
    }
    //主动发送数据给连接者，该方法是静态方法，admin主要靠它将数据发送给各个链接者
    public static void send(final String message, final DataEventTypeEnum type) {
        if (StringUtils.isNotBlank(message)) {
          // 如果消息类型是MYSELF
            if (DataEventTypeEnum.MYSELF == type) {
              // 从thread中拿到session
                Session session = (Session) ThreadLocalUtil.get(SESSION_KEY);
                if (session != null) {// 向session中发送message信息
                    sendMessageBySession(session, message);
                }
            } else {//除了myself类型，遍历所有session并发送数据
                SESSION_SET.forEach(session -> sendMessageBySession(session, message));
            }
        }
    }
    // 向session发送数据
    private static void sendMessageBySession(final Session session, final String message) {
        try {
            session.getBasicRemote().sendText(message);
        } catch (IOException e) {
            log.error("websocket send result is exception: ", e);
        }
    }
}
```

通过上述代码我们可以看出，通常是admin向连接的soul网关发送数据，发送数据的时机是静态send方法被调用时，发送的方式是遍历内存中的所有的session将消息发送他们，每个session都代表1个存活的soul网关。下面我们来着重研究下admin在什么时候会调用send方法以及MYSELF这个type代表的意义。

# DataChangedListener

首先看send方法调用的地方，非常的集中，是在一个叫`WebsocketDataChangedListener`的类中，该类继承了`DataChangedListener`，该接口主要的方式从名字上看，意思是当Plugin、selector、rule、AppAuth、MetaData相关数据发生变化时触发发送数据的逻辑而WebsocketDataChangedListener是使用websocket发送数据的实现。在该实现中最后都是调用的`WebsocketCollector::send`方法。那么又是什么时候`WebsocketDataChangedListener`被调用的呢？

```java
public class WebsocketDataChangedListener implements DataChangedListener {
    @Override
    public void onPluginChanged(final List<PluginData> pluginDataList, final DataEventTypeEnum eventType) {
        WebsocketData<PluginData> websocketData =
                new WebsocketData<>(ConfigGroupEnum.PLUGIN.name(), eventType.name(), pluginDataList);
        WebsocketCollector.send(GsonUtils.getInstance().toJson(websocketData), eventType);
    }
    @Override
    public void onSelectorChanged(final List<SelectorData> selectorDataList, final DataEventTypeEnum eventType)			{....} 
    @Override
    public void onRuleChanged(final List<RuleData> ruleDataList, final DataEventTypeEnum eventType) { ...
    }
    @Override
    public void onAppAuthChanged(final List<AppAuthData> appAuthDataList, final DataEventTypeEnum eventType) {..
    }
    @Override
    public void onMetaDataChanged(final List<MetaData> metaDataList, final DataEventTypeEnum eventType) {...
    }
}
```



继续跟踪发现soul-admin在项目启动时由`DataSyncConfiguration`负责将`WebsocketDataChangedListener`初始化并把它注册到spring容器中。那么又是哪个类会使用到DataChangedListener呢？根据我全目录下搜索DataChangedListener字符串，发言使用DataChangedListener主要是`DataChangedEventDispatcher`这个类。

# DataChangedEventDispatcher

这个类先大胆猜测这个类的主要功能是当有配置数据变化事件发生时的调度程序。直接上源码：

```java
@Component
public class DataChangedEventDispatcher implements ApplicationListener<DataChangedEvent>, InitializingBean {
    private ApplicationContext applicationContext;// spring上下文
    private List<DataChangedListener> listeners;// 全部DataChangedListener实现
    public DataChangedEventDispatcher(final ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }
    @Override
    @SuppressWarnings("unchecked")
    public void onApplicationEvent(final DataChangedEvent event) {
      // 变量所有DataChangedListener实现，发送数据
        for (DataChangedListener listener : listeners) {
            switch (event.getGroupKey()) {
                case APP_AUTH:
                    listener.onAppAuthChanged((List<AppAuthData>) event.getSource(), event.getEventType());
                    break;
                case PLUGIN:
                    listener.onPluginChanged((List<PluginData>) event.getSource(), event.getEventType());
                    break;
                case RULE:
                    listener.onRuleChanged((List<RuleData>) event.getSource(), event.getEventType());
                    break;
                case SELECTOR:
                    listener.onSelectorChanged((List<SelectorData>) event.getSource(), event.getEventType());
                    break;
                case META_DATA:
                    listener.onMetaDataChanged((List<MetaData>) event.getSource(), event.getEventType());
                    break;
                default:
                    throw new IllegalStateException("Unexpected value: " + event.getGroupKey());
            }
        }
    }
		// 在property，也就是application设置完后，从上下文中取出所有DataChangedListener的实现，复制给属性
    @Override
    public void afterPropertiesSet() {
        Collection<DataChangedListener> listenerBeans = applicationContext.getBeansOfType(DataChangedListener.class).values();
        this.listeners = Collections.unmodifiableList(new ArrayList<>(listenerBeans));
    }
}
```

很明显`ApplicationListener`是spring某种事件监听的实现，当关注的事件发生时，就会调用`DataChangedEventDispatcher`的onApplicationEvent方法，而该方法就是调用所有`DataChangedListener`的相关方法，实现数据发送到soul网关的逻辑。、

# ApplicationListener

通过翻阅资料我们父发现，ApplicationListener是Spring事件机制的一部分，与抽象类ApplicationEvent类配合来完成ApplicationContext的事件机制。如果容器中存在ApplicationListener的Bean，当ApplicationContext调用publishEvent方法时，对应的Bean会被触发。这一过程是典型的观察者模式的实现。在soul中`DataChangedEventDispatcher`就是`DataChangedEvent`事件的观察，`DataChangedEvent`不出所料的继承了`ApplicationEvent`。

再接着看就很简单了，以`metaDataService`中的`delete`方法来举例，该方法在接收到`POST  /meta-data/batchDeleted`时被调用，源码如下：

```java
@Service("metaDataService")
public class MetaDataServiceImpl implements MetaDataService {
		// mybatis操作metaData数据表
    private final MetaDataMapper metaDataMapper;
   // spring 事件发布器
    private final ApplicationEventPublisher eventPublisher;
    @Autowired(required = false)
    public MetaDataServiceImpl(final MetaDataMapper metaDataMapper, final ApplicationEventPublisher eventPublisher) {
        this.metaDataMapper = metaDataMapper;
        this.eventPublisher = eventPublisher;
    }
    @Override
    @Transactional//事务 所有id要不全删除，要不全部删
    public int delete(final List<String> ids) {
        int count = 0;
        List<MetaData> metaDataList = Lists.newArrayList();
        for (String id : ids) {
            MetaDataDO metaDataDO = metaDataMapper.selectById(id);
            count += metaDataMapper.delete(id);
            // p构造发布数据
            metaDataList.add(MetaDataTransfer.INSTANCE.mapToData(metaDataDO));
        }
      // 通过spring的事件发布器来发布删除事件，最后会发送个各个soul网关
        eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.META_DATA, DataEventTypeEnum.DELETE, metaDataList));
        return count;
    }
}
```

# 总结

soul-admin中使用spring的事件机制来发布各配置数据的变更事件，由`DataChangedEventDispatcher`来统一处理这些事件，处理方式是调用所有`DataChangedListener`的实现，因为可能存在有的soul网关是通过http长轮询连接到soul-admin而有些是通过websocket。这些DataChangedListener负责使用对应的方法将数据发送给使用该方式同步数据的soul网关。