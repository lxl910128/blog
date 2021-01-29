---
title: soul学习14——Divide插件学习
date: 2021/1/29 16:39:10
categories:
- soul
tags:
- soul
- 网关
keywords:
- soul , 网关, 插件, divide
---
# 概述

前面的一些列文章介绍了soul网关的核心流程以及数据同步这两大最重要的功能逻辑。鉴于后续希望为soul贡献1个插件，所以接下来再学习学习soul中其它核心插件的写法，今天先看看divide插件

<!--more-->

# Divide plugin使用

divide-plugin是和spring-clould-plugin同类型的接入类插件，可以让任意http服务都接入到soul中，并让请求正确路由到下游服务。使用divide时首先需要在soul网关中必须引入`soul-spring-boot-starter-plugin-divide`项目。而对于下游服务有3种接入方式，第一种，对于`spring-boot`项目需要做以下步骤

1. 项目以引入`soul-spring-boot-starter-client-springmvc`包

2. 配置文件中增加如下内容

   ```yaml
      soul:
        http:
          adminUrl: http://localhost:9095
          port: 你本项目的启动端口
          contextPath: /http
          appName: http
          full: false  
      # adminUrl: 为你启动的soul-admin 项目的ip + 端口，注意要加http://
      # port: 你本项目的启动端口
      # contextPath: 为你的这个mvc项目在soul网关的路由前缀，这个你应该懂意思把？ 比如/order ，/product 等等，网关会根据你的这个前缀来进行路由.
      # appName：你的应用名称，不配置的话，会默认取 `spring.application.name` 的值
      # full: 设置true 代表代理你的整个服务，false表示代理你其中某几个controller
   ```

3. 在你的 `controller` 的接口上加上 `@SoulSpringMvcClient` 注解。

第二种对于`spring-mvc`项目需要做以下步骤

1. 项目以引入`soul-client-springmvc`包

2. spring的xml文件增加如下配置

   ```xml
       <bean id ="springMvcClientBeanPostProcessor" class = "org.dromara.soul.client.springmvc.init.SpringMvcClientBeanPostProcessor">
            <constructor-arg  ref="soulSpringMvcConfig"/>
       </bean>
           
       <bean id="soulSpringMvcConfig" class="org.dromara.soul.client.springmvc.config.SoulSpringMvcConfig">
            <property name="adminUrl" value="http://localhost:9095"/>
            <property name="port" value="你的端口"/>
            <property name="contextPath" value="/你的contextPath"/>
            <property name="appName" value="你的名字"/>
            <property name="full" value="false"/>
       </bean>
   ```

3. 在你的 `controller` 的接口上加上 `@SoulSpringMvcClient` 注解。

`spring-boot`或`spring-mvc`项目完成以上2步在项目启动时admin就会自动感知到服务的启动并在metadata中保存该服务可以提供的接口。第三种如果你是非常非常常规的web服务没有用spring框架，或者你是spring项目但是你不想因为soul而改变你的代码，那么就需要手动配置，让soul知道有新的下游服务接入。这样的话如何配置呢？很简单，直接配置divide插件的选择器selector和规则rule。

# soul-plugin-divide

下面我们看看引入soul网关的divide插件是如何实现的。还是从实现了`AbstractSoulPlugin`的DividePlugin开始

```java
public class DividePlugin extends AbstractSoulPlugin {
    @Override// 此时请求已经通过了selector和rule的筛选需要执行具体divide规则
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
      // 将rule配置转为DivideRuleHandle对象
        final DivideRuleHandle ruleHandle = GsonUtils.getInstance().fromJson(rule.getHandle(), DivideRuleHandle.class);
      // 根据selectorId从缓存中拿到所有的可以请求的下游服务
        final List<DivideUpstream> upstreamList = UpstreamCacheManager.getInstance().findUpstreamListBySelectorId(selector.getId());
        if (CollectionUtils.isEmpty(upstreamList)) {// 没有可用下游服务报错
            log.error("divide upstream configuration error： {}", rule.toString());
            Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_FIND_URL.getCode(), SoulResultEnum.CANNOT_FIND_URL.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }// 请求者IP
        final String ip = Objects.requireNonNull(exchange.getRequest().getRemoteAddress()).getAddress().getHostAddress();
      // 根据负责均衡算法拿到这个IP 适合访问的1个下游服务
      // 通过配置可使用选择 HASH、RANDOM\roudRobin 3中负载均衡的任意一种
        DivideUpstream divideUpstream = LoadBalanceUtils.selector(upstreamList, ruleHandle.getLoadBalance(), ip);  // 没选出来，报错直接返回
        if (Objects.isNull(divideUpstream)) {
            log.error("divide has no upstream");
            Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_FIND_URL.getCode(), SoulResultEnum.CANNOT_FIND_URL.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        // set the http url
        String domain = buildDomain(divideUpstream);
        String realURL = buildRealURL(domain, soulContext, exchange);
        // 配出选出服务的实际请求URL放在请求上线文中
        exchange.getAttributes().put(Constants.HTTP_URL, realURL);
        // 设置超时时间
        exchange.getAttributes().put(Constants.HTTP_TIME_OUT, ruleHandle.getTimeout());
        exchange.getAttributes().put(Constants.HTTP_RETRY, ruleHandle.getRetry());
        return chain.execute(exchange);
    }
		// 插件名字 divide
    @Override
    public String named() {
        return PluginEnum.DIVIDE.getName();
    }
		// rpcType非http跳过本插件
    @Override
    public Boolean skip(final ServerWebExchange exchange) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        return !Objects.equals(Objects.requireNonNull(soulContext).getRpcType(), RpcTypeEnum.HTTP.getName());
    }
		// 顺序50
    @Override
    public int getOrder() {
        return PluginEnum.DIVIDE.getCode();
    }
}
```

整体来说divide插件的实现逻辑和之前我们介绍的spring-cloud-plugin核心逻辑相同，都是构造实际请求了url放在请求上下文中。不同的是spring-cloud插件是借助注册中心如eureka拿到合适的1个下游服务。而divide是从内存`UpstreamCacheManager`中拿到的所有服务，然后根据插件配置执行自己实现的负责均衡算法选择出合适的1个下游服务。可配的3种负载策略大概的含义如下：

1. HASH，根据请求者IP取1个下游服务，如果下游服务不变相同IP请求的永远是同一个服务
2. Random，每次请求都从所有下游服务中随机选一个
3. RoundRobin，所有服务轮询被请求

# UpstreamCacheManager

刚才我们提到插件是从UpstreamCacheManager中根据selectorId选择出全部可用下游服务的。那么为什么UpstreamCacheManager会知道所有的下游服务呢？它是如何验证下游服务是否可用呢？我们来具体看下。

```java
public final class UpstreamCacheManager {
    private static final UpstreamCacheManager INSTANCE = new UpstreamCacheManager();
   // 配置的下游服务map key selectorId，value是所有下游实例
    private static final Map<String, List<DivideUpstream>> UPSTREAM_MAP = Maps.newConcurrentMap();
   // 存活的下游服务map key selectorId，value是所有下游实例
    private static final Map<String, List<DivideUpstream>> UPSTREAM_MAP_TEMP = Maps.newConcurrentMap();
    // 单例 饿汉，构建时 创建了1个周期执行的任务
    private UpstreamCacheManager() {
      // soul.upstream.check 为true开启了 upstream
        boolean check = Boolean.parseBoolean(System.getProperty("soul.upstream.check", "false"));
        if (check) {
          // 完成上一次任务30秒后执行1次
          // 执行的逻辑是 scheduled方法
            new ScheduledThreadPoolExecutor(1, SoulThreadFactory.create("scheduled-upstream-task", false))
                    .scheduleWithFixedDelay(this::scheduled,
                            30, Integer.parseInt(System.getProperty("soul.upstream.scheduledTime", "30")), TimeUnit.SECONDS);
        }
    }
    // 单例获取实例
    public static UpstreamCacheManager getInstance() {
        return INSTANCE;
    }
    // 根据selectorID找可用下游服务
    public List<DivideUpstream> findUpstreamListBySelectorId(final String selectorId) {
        return UPSTREAM_MAP_TEMP.get(selectorId);
    }
    // 根据selector 移除相关下游服务
    public void removeByKey(final String key) {
        UPSTREAM_MAP_TEMP.remove(key);
    }
    // 如果divide的selector数据改变并被soul网关端感知到时，会触发这个方法
    // 运行在soul 网关端
    public void submit(final SelectorData selectorData) {
      //拿到selector 对应的现有所有下游服务
        final List<DivideUpstream> upstreamList = GsonUtils.getInstance().fromList(selectorData.getHandle(), DivideUpstream.class);
      // 如果新数据有配置怎更新老的配置，如果没有实际配置则删除老的配置
        if (null != upstreamList && upstreamList.size() > 0) {
            UPSTREAM_MAP.put(selectorData.getId(), upstreamList);
            UPSTREAM_MAP_TEMP.put(selectorData.getId(), upstreamList);
        } else {
            UPSTREAM_MAP.remove(selectorData.getId());
            UPSTREAM_MAP_TEMP.remove(selectorData.getId());
        }
    }
		// 周期性任务
    // 遍历UPSTREAM_MAP各个selector的所有下游服务
    // 执行check方法验证下游服务是否都存活
    // 筛选一遍后将存活的下游服务放到UPSTREAM_MAP_TEMP中
    private void scheduled() {
        if (UPSTREAM_MAP.size() > 0) {
            UPSTREAM_MAP.forEach((k, v) -> {
                List<DivideUpstream> result = check(v);
                if (result.size() > 0) {
                    UPSTREAM_MAP_TEMP.put(k, result);
                } else {
                    UPSTREAM_MAP_TEMP.remove(k);
                }
            });
        }
    }
   // 验证服务是否存活
    private List<DivideUpstream> check(final List<DivideUpstream> upstreamList) {
        List<DivideUpstream> resultList = Lists.newArrayListWithCapacity(upstreamList.size());
        for (DivideUpstream divideUpstream : upstreamList) {
          // 根据IP端口看服务是否通，但是没有测试具体PTAH是否可用
            final boolean pass = UpstreamCheckUtils.checkUrl(divideUpstream.getUpstreamUrl());
            if (pass) {
                if (!divideUpstream.isStatus()) {
                  // 上一次探活没通，此次通了，则将可用状态改为true
                    divideUpstream.setTimestamp(System.currentTimeMillis());
                    divideUpstream.setStatus(true);
                    log.info("UpstreamCacheManager detect success the url: {}, host: {} ", divideUpstream.getUpstreamUrl(), divideUpstream.getUpstreamHost());
                }// 加入ret中
                resultList.add(divideUpstream);
            } else {
              // 探活失败
                divideUpstream.setStatus(false);
                log.error("check the url={} is fail ", divideUpstream.getUpstreamUrl());
            }
        }
        return resultList;

    }
}
```

通过代码可以看出在所有的soul网关中都会运行这样一个upstream的cache，关注divide插件的selector变化，如果有更新就同步更新jvm内存中的配置(UPSTREAM_MAP,UPSTREAM_MAP_TEMP)。同时还运行的有探活程序周期性验证selector配置的下游服务是否存活，但是不会验证接口级别，只会验证IP端口是否通。

# 总结

divide插件可以让任何http服务都在无侵入的情况下接入到soul，soul也支持对他们进行探活。从逻辑上讲每个网关会探每个selector配置的所有的下游服务的存活情况，因为selector可以配到接口级别，如果1类用户服务有3个实例，10个接口并且配置了10个selector，soul网关启动了5个，那么30秒内关于探活用户服务的请求在网络中就有5\*10\*3=150个，每个网关都会发10\*3=30个。同时这个`UpstreamCacheManager`在下游服务是websocket服务时也会用到。