---
title: soul学习12——http长轮询同步数据
date: 2021/1/27 23:46:10
categories:
- soul
tags:
- soul
- 网关
keywords:
- soul , 网关, soul-web, http长轮询, 数据同步, AsyncContext, HttpLongPolling
---
# 概述

前两篇文章以websocket同步方式介绍了整个数据同步的流程，今天我们来看看http长轮询同步数据是如何实现的。使用websocket同步数据时，在soul-admin服务端有负责监听数据变化事件的`HttpLongPollingDataChangedListener`以及负责接收soul网关连接的`WebsocketCollector`，在soul服务端有实现`SyncDataService`的`WebsocketSyncDataService`，负责实现当接收到配置变化。下面我们来对应看看http长轮询这几部分是如何实现的。

<!--more-->



# 网关端

在soul网关端如果要开启http同步数据的方式需要做如下配置：

```yaml
soul:
    file:
      enabled: true
    corss:
      enabled: true
    dubbo :
      parameter: multi
    sync:
    	http:
      	url : http://localhost:9095
```

与websocket类似，他需要配置admin的地址，不过比较奇怪的是他并不需要配路径。通过`HttpSyncDataConfiguration`我们发现soul网关发现`soul.sync.http.url`参数存在时会向spring容器中注册了`HttpSyncDataService`，构造Service需要

`soul.sync.http`下的配置，需要PluginData、MetaData、AuthData这些订阅者，总体上和websocket类似。下面我们详细看下`HttpSyncDataService`的实现：

```java
public class HttpSyncDataService implements SyncDataService, AutoCloseable {
    private static final AtomicBoolean RUNNING = new AtomicBoolean(false);//标记是否启动
    private static final Gson GSON = new Gson();
    private Duration connectionTimeout = Duration.ofSeconds(10); // 连接超时时间
    private RestTemplate httpClient;// 长连接的httpClient
    private ExecutorService executor;// 线程池
    private HttpConfig httpConfig;
    private List<String> serverList;
    private DataRefreshFactory factory;

    public HttpSyncDataService(final HttpConfig httpConfig, final PluginDataSubscriber pluginDataSubscriber,
                               final List<MetaDataSubscriber> metaDataSubscribers, final List<AuthDataSubscriber> authDataSubscribers) {
        this.factory = new DataRefreshFactory(pluginDataSubscriber, metaDataSubscribers, authDataSubscribers);
        this.httpConfig = httpConfig;
        this.serverList = Lists.newArrayList(Splitter.on(",").split(httpConfig.getUrl()));// 配置的admin的地址
        this.httpClient = createRestTemplate(); // 创建httpClient
        this.start();// 初始化
    }
    // 构建同步httpClient
    private RestTemplate createRestTemplate() {
        OkHttp3ClientHttpRequestFactory factory = new OkHttp3ClientHttpRequestFactory();
        factory.setConnectTimeout((int) this.connectionTimeout.toMillis());// 10秒连接超时
        factory.setReadTimeout((int) HttpConstants.CLIENT_POLLING_READ_TIMEOUT);// 90秒读数据超时
        return new RestTemplate(factory);// spring 提供的封装
    }
		//初始化时调用，启动
    private void start() {
        // 可能初始化多次，如果初始化了就不启动了
        if (RUNNING.compareAndSet(false, true)) {
            // ConfigGroupEnum枚举的是数据更新的类型：APP_AUTH\PLUGIN\RULE\SELECTOR\META_DATA
            // 该方法是拉取各个配置的，具体实现见下
            this.fetchGroupConfig(ConfigGroupEnum.values());
            int threadSize = serverList.size();
            //创建线程池 个数为soul-admin的个数
            this.executor = new ThreadPoolExecutor(threadSize, threadSize, 60L, TimeUnit.SECONDS,
                    new LinkedBlockingQueue<>(),
                    SoulThreadFactory.create("http-long-polling", true));
            // 启动http长连接每个soul-admin创建一个线程用来监听变化
            this.serverList.forEach(server -> this.executor.execute(new HttpLongPollingTask(server)));
        } else {
            log.info("soul http long polling was started, executor=[{}]", executor);
        }
    }
    private void fetchGroupConfig(final ConfigGroupEnum... groups) throws SoulException {
        for (int index = 0; index < this.serverList.size(); index++) {
            String server = serverList.get(index);
            try {		// 每个soul-admin都调用此方法
                this.doFetchGroupConfig(server, groups);
                break;
            } catch (SoulException e) {
                // no available server, throw exception.
                if (index >= serverList.size() - 1) {
                    throw e;
                }
                log.warn("fetch config fail, try another one: {}", serverList.get(index + 1));
            }
        }
    }
    private void doFetchGroupConfig(final String server, final ConfigGroupEnum... groups) {
        StringBuilder params = new StringBuilder();
        for (ConfigGroupEnum groupKey : groups) {
            params.append("groupKeys").append("=").append(groupKey.name()).append("&");
        }
      // 拼url       
//{adminIP}:{adminport}/configs/fetch?groupKeys=APP_AUTH&groupKeys=PLUGIN&groupKeys=RULE&groupKeys=SELECTOR &groupKeys=META_DATA
        String url = server + "/configs/fetch?" + StringUtils.removeEnd(params.toString(), "&");
        log.info("request configs: [{}]", url);
        String json = null;
        try {//请求获取响应配置
            json = this.httpClient.getForObject(url, String.class);
        } catch (RestClientException e) {
            String message = String.format("fetch config fail from server[%s], %s", url, e.getMessage());
            log.warn(message);
            throw new SoulException(message, e);
        }
        // 更新配置
        boolean updated = this.updateCacheWithJson(json);
        if (updated) {
            log.info("get latest configs: [{}]", json);
            return;
        }
        // 如果无更新，则睡30秒，因为可能admin还没启动成功
        log.info("The config of the server[{}] has not been updated or is out of date. Wait for 30s to listen for changes again.", server);
        ThreadUtils.sleep(TimeUnit.SECONDS, 30); // 睡30秒
    }
		// 更新配置
    private boolean updateCacheWithJson(final String json) {
        JsonObject jsonObject = GSON.fromJson(json, JsonObject.class);
        JsonObject data = jsonObject.getAsJsonObject("data");
        // if the config cache will be updated?
        return factory.executor(data);
    }
  // 线程池
		class HttpLongPollingTask implements Runnable {
        private String server;// 1个soul-admin的地址
        private final int retryTimes = 3;
        HttpLongPollingTask(final String server) {
            this.server = server;
        }
        @Override
        public void run() {
            while (RUNNING.get()) {// 如果是启动状态则无限循环
                for (int time = 1; time <= retryTimes; time++) { // 单此请求如果有错重试3次
                    try {// 核心逻辑在doLongPolling
                        doLongPolling(server);
                    } catch (Exception e) {
                        // print warnning log.
                        if (time < retryTimes) {
               log.warn("Long polling failed, tried {} times, {} times left, will be suspended for a while! {}",
                                    time, retryTimes - time, e.getMessage());
                            ThreadUtils.sleep(TimeUnit.SECONDS, 5); // 3次内睡5秒再试
                            continue;
                        }
                        // 3次后5分钟再试
                        log.error("Long polling failed, try again after 5 minutes!", e);
                        ThreadUtils.sleep(TimeUnit.MINUTES, 5);
                    }
                }
            }
            log.warn("Stop http long polling.");
        }
    }
    @SuppressWarnings("unchecked")// 长连接 无限循环时方法
    private void doLongPolling(final String server) {
        MultiValueMap<String, String> params = new LinkedMultiValueMap<>(8);
      // 构造参数，是个map key为APP_AUTH\PLUGIN\RULE\SELECTOR\META_DATA value为对应值的md5和最后修改时间
        for (ConfigGroupEnum group : ConfigGroupEnum.values()) {
            ConfigData<?> cacheConfig = factory.cacheConfigData(group);
            String value = String.join(",", cacheConfig.getMd5(),        String.valueOf(cacheConfig.getLastModifyTime()));
            params.put(group.name(), Lists.newArrayList(value));
        }
      // 构造请求头
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
        HttpEntity httpEntity = new HttpEntity(params, headers);
        String listenerUrl = server + "/configs/listener"; // 请求adminIP:port/configs/listener
        log.debug("request listener configs: [{}]", listenerUrl);
        JsonArray groupJson = null;
        try {// 发送请求 ，验证哪些配置更改了
            String json = this.httpClient.postForEntity(listenerUrl, httpEntity, String.class).getBody();
            log.debug("listener result: [{}]", json);
            groupJson = GSON.fromJson(json, JsonObject.class).getAsJsonArray("data");
        } catch (RestClientException e) {
            String message = String.format("listener configs fail, server:[%s], %s", server, e.getMessage());
            throw new SoulException(message, e);
        }
        if (groupJson != null) {
            ConfigGroupEnum[] changedGroups = GSON.fromJson(groupJson, ConfigGroupEnum[].class);
            if (ArrayUtils.isNotEmpty(changedGroups)) {
                log.info("Group config changed: {}", Arrays.toString(changedGroups));
              // 全量获取变更的数据
                this.doFetchGroupConfig(server, changedGroups);
            }
        }
    }

    @Override
    public void close() throws Exception {
        RUNNING.set(false);
        if (executor != null) {
            executor.shutdownNow();
            // help gc
            executor = null;
        }
    }
}
```

整体代码逻辑是其实比较明显，soul网关启动时请求每个soul-admin的`/configs/fetch`接口获取全量配置信息。同时会启动soul-admin数量的线程调用admin的`/configs/listener`来校验配置是否变更，如果变更再具体调用`/configs/fetch`来全量更新数据。在本类中所有请求都是借助RestTemplate发送，设置了readTimeOut是90秒，而线程轮询的时间间隔其实是30秒，这样就应该达到了TCP连接不断的效果，也就是所谓的长轮询。

# admin端

soul-admin中没有websocket的Collector，因为http长轮询访问的接口是都是http的所有都在`org.dromara.soul.admin.controller`下。那么我们看看`HttpLongPollingDataChangedListener`的实现。这个类不是直接实现`DataChangedListener`，而是继承抽象类`AbstractDataChangedListener`。首先来看`AbstractDataChangedListener`，代码如下：

```java
public abstract class AbstractDataChangedListener implements DataChangedListener, InitializingBean {
  // 缓存所有配置
   protected static final ConcurrentMap<String, ConfigDataCache> CACHE = new ConcurrentHashMap<>();
    @Resource
    private PluginService pluginService;
    @Override
    public final void afterPropertiesSet() {
        updateAppAuthCache();
        updatePluginCache();
        updateRuleCache();
        updateSelectorCache();
        updateMetaDataCache();
        afterInitialize();
    }
    protected abstract void afterInitialize();
  // 子类需要实现
 		protected void afterAppAuthChanged(final List<AppAuthData> changed, final DataEventTypeEnum eventType) {
    }
  // 实现 plugin数据更新是的操作
    @Override
    public void onPluginChanged(final List<PluginData> changed, final DataEventTypeEnum eventType) {
        if (CollectionUtils.isEmpty(changed)) {
            return;
        }
      // 更新cache
        this.updatePluginCache();
      // 调用子类更新配置的实现
        this.afterPluginChanged(changed, eventType);
    }
  // 更新cache，更新的数据是从pluginservice中拿出的全部数据
  	protected void updatePluginCache() {
        this.updateCache(ConfigGroupEnum.PLUGIN, pluginService.listAll());
    }
  // 就是跟新本类 ConcurrentMap 的数据
   protected <T> void updateCache(final ConfigGroupEnum group, final List<T> data) {
        String json = GsonUtils.getInstance().toJson(data);
        ConfigDataCache newVal = new ConfigDataCache(group.name(), json, Md5Utils.md5(json), System.currentTimeMillis());
        ConfigDataCache oldVal = CACHE.put(newVal.getGroup(), newVal);
        log.info("update config cache[{}], old: {}, updated: {}", group, oldVal, newVal);
    }
   // 初始化时就更新内存中的数据
    protected void updateMetaDataCache() {
        this.updateCache(ConfigGroupEnum.META_DATA, metaDataService.listAll());
    }
}
```

上述是pluginData相关的实现，其他数据类似，之前我们聊到过有PluginData更新时间发生时，会调用`onPluginChanged`，在这里面主要是先更新本类内存中的配置，然后再调用子类实现是的`afterPluginChanged`。同时在相关实现类构造时，调用了`afterPropertiesSet`它主要是先把所有配置加载到内存中，然后调用子类的 `afterInitialize`方法。 下面我们看看http长轮询继承这个抽象类又做了哪些逻辑。代码如下：

```java
public class HttpLongPollingDataChangedListener extends AbstractDataChangedListener {
    private static final String X_REAL_IP = "X-Real-IP";
    private static final String X_FORWARDED_FOR = "X-Forwarded-For";
    private static final String X_FORWARDED_FOR_SPLIT_SYMBOL = ",";
    private static final ReentrantLock LOCK = new ReentrantLock();// 修改缓存中数据时要上锁
    private final BlockingQueue<LongPollingClient> clients;// 等待超时或被阻塞的soul网关的/configs/listener请求
    private final ScheduledExecutorService scheduler;// 带计时器的线程池
    private final HttpSyncProperties httpSyncProperties;
    /**
     * 初始化，该类监听更种配置数据的变化，并使用长轮询方式通知soul网关
     */
    public HttpLongPollingDataChangedListener(final HttpSyncProperties httpSyncProperties) {
        this.clients = new ArrayBlockingQueue<>(1024);
        this.scheduler = new ScheduledThreadPoolExecutor(1,
                SoulThreadFactory.create("long-polling", true));
        this.httpSyncProperties = httpSyncProperties;
    }
    @Override // 初始化时调用 
    protected void afterInitialize() {
        long syncInterval = httpSyncProperties.getRefreshInterval().toMillis();
        // 每五分钟刷AbstractDataChangedListener中缓存的配置
        scheduler.scheduleWithFixedDelay(() -> {
            log.info("http sync strategy refresh config start.");
            try {
                this.refreshLocalCache();
                log.info("http sync strategy refresh config success.");
            } catch (Exception e) {
                log.error("http sync strategy refresh config error!", e);
            }
        }, syncInterval, syncInterval, TimeUnit.MILLISECONDS);
        log.info("http sync strategy refresh interval: {}ms", syncInterval);
    }
    // 调用父类，更新所有配置 到内存中
    private void refreshLocalCache() {
        this.updateAppAuthCache();
        this.updatePluginCache();
        this.updateRuleCache();
        this.updateSelectorCache();
        this.updateMetaDataCache();
    }
  //---------------------------------soul网关请求souladmin的处理逻辑------------------------------------
    /**  
     *  /configs/listener 接口会实际调用此方法，也就是说soul网关周期调用admin，admin实际的处理逻辑在这
     *  根据官方解释，如果配置有变化请求会立马返回，如果配置一直没变化则会被阻塞，阻塞60秒再返回！
     */
    public void doLongPolling(final HttpServletRequest request, final HttpServletResponse response) {
        // 对比发送来的配置的MD5查看 与内存中是否一样
        List<ConfigGroupEnum> changedGroup = compareChangedGroup(request);
        // 那实际请求iP
        String clientIp = getRemoteIp(request);
        // 有不同，立马返回
        if (CollectionUtils.isNotEmpty(changedGroup)) {
            this.generateResponse(response, changedGroup);
            log.info("send response with the changed group, ip={}, group={}", clientIp, changedGroup);
            return;
        }
        // AsyncContext可以从当前线程传给另外的线程
        // 并在新的线程中完成对请求的处理并返回结果给客户端，初始线程便可以还回给容器线程池以处理更多的请求。
        // 解决每个请求需要1个Servlet的线程从头到尾负责处理
        final AsyncContext asyncContext = request.startAsync();
        // AsyncContext timeout 不准, 需要自己控制
        asyncContext.setTimeout(0L);
        // 交给线程池处理，Servlet可以释放处理其他请求，该线程池与刷新内存中配置的线程池是同一个
        // 构造LongPollingClient任务，任务内又创建了个任务60秒后执行，
        scheduler.execute(new LongPollingClient(asyncContext, clientIp, HttpConstants.SERVER_MAX_HOLD_TIMEOUT));
    }
     /**
     * 如果在超时时间内即60秒内还是没有数据跟心，则返回空，如果60秒内有数据更新则会触发DataChangeTask任务将变更数据返回
     */
    class LongPollingClient implements Runnable {
        // 请求上下文，此对象解放了 Servlet线程
        private final AsyncContext asyncContext;
        // 实际请求者IP
        private final String ip;
        // 超时时间
        private final long timeoutTime;
        // 超时future
        private Future<?> asyncTimeoutFuture;
        // 构造LongPollingClient
        LongPollingClient(final AsyncContext ac, final String ip, final long timeoutTime) {
            this.asyncContext = ac;
            this.ip = ip;
            this.timeoutTime = timeoutTime;
        }
        @Override
        public void run() {
            // 线程池中增加60秒后执行的任务
            this.asyncTimeoutFuture = scheduler.schedule(() -> {
                // 这个任务主要工作是，在被阻塞的请求队列中移除自己
                clients.remove(LongPollingClient.this);
                // 再次比较有没有差异数据
                List<ConfigGroupEnum> changedGroups = compareChangedGroup((HttpServletRequest) asyncContext.getRequest());
                // 发送请求
                sendResponse(changedGroups);
            }, timeoutTime, TimeUnit.MILLISECONDS);
            // 将自己加到被阻塞的请求队列中
            clients.add(this);
        }
        // 放response
        void sendResponse(final List<ConfigGroupEnum> changedGroups) {
            // 首先取消这个等待60秒后执行的任务
            // 可能是数据有更新提前取消
            if (null != asyncTimeoutFuture) {
                asyncTimeoutFuture.cancel(false);
            }
            // 发送数据
            generateResponse((HttpServletResponse) asyncContext.getResponse(), changedGroups);
            // 请求完成
            asyncContext.complete();
        }
    }
  //--------------------------发生配置更改事件时的处理逻辑------------------------------------------------
   @Override  // plugin数据更新时调用
    protected void afterPluginChanged(final List<PluginData> changed, final DataEventTypeEnum eventType) {
       // 在线程池中开一个DataChangeTask的线程处理
        scheduler.execute(new DataChangeTask(ConfigGroupEnum.PLUGIN));
    }
    // 当有数据更新时 找到hold的请求立马返回新的数据
    class DataChangeTask implements Runnable {
        // 更新的数据类型
        private final ConfigGroupEnum groupKey;
        // 变更时间
        private final long changeTime = System.currentTimeMillis();
        // 初始化
        DataChangeTask(final ConfigGroupEnum groupKey) {
            this.groupKey = groupKey;
        }
        @Override
        public void run() {
          // 遍历被hold的请求
            for (Iterator<LongPollingClient> iter = clients.iterator(); iter.hasNext();) {
                LongPollingClient client = iter.next();
                iter.remove();
                // 发送数据，注意这调的是LongPollingClient中的sendResponse
                // 请求内会取消LongPollingClient中设置的60秒后执行的任务
                client.sendResponse(Collections.singletonList(groupKey));
                log.info("send response with the changed group,ip={}, group={}, changeTime={}", client.ip, groupKey, changeTime);
            }
        }
    }
    
}
```

这块比较复杂，但是要把握住核心思想，即当soul网关请求  /configs/listener  时需要实现的逻辑是：

1. 如果配置有变则立马返回
2. 如果配置在60秒内有边也立马返回
3. 如果60秒内配置无更新则也返回

那么是怎么实现的呢？是有3个线程在scheduler线程池里互相配合的结果。首先，如果请求发过来第一遍验证发现配置有改变则直接返回。如果没有更新则创建一个LongPollingClient任务，该任务有创建了1个60秒后执行回写响应的任务asyncTimeoutFuture，LongPollingClient任务放在clients这个变量里，asyncTimeoutFuture任务保存在LongPollingClient中。如果60秒内都没有配置更新则asyncTimeoutFuture会执行回写响应。如果60秒内有配置更新则会开启DataChangeTask任务，他会从clients拿到所有的LongPollingClient任务，依次执行取消asyncTimeoutFuture任务，然后回写response。

# 总结

soul的http长轮询设计的十分巧妙，值得深入学习。总得来说长轮询的机制时应对频繁查询数据是否变更的场景，实现方式是首先是http的read的超时必须够长，然后是http服务端要阻塞请求，但阻塞的时间要短于read超时时间。服务端只有在有数据变更或超时时才返回数据。这样做即可以保证数据更新立马被请求端感知，又能减少TCP连接。实现时注意要使用`AsyncContext`将请求移交给别的线程，不要让Servlet阻塞。

