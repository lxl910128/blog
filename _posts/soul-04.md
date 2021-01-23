---
title: soul学习04——网关启动
date: 2021/1/18 11:06:10
categories:
- soul
tags:
- soul
- 网关
keywords:
- soul , 网关, SoulWebHandler, DispatcherHandler, DefaultWebFilterChain
---
# 概述

本文主要从源码的角度分析soul-bootstrap的启动。

<!-- more -->

# pom.xml

从soul-bootstrap的pom文件中我们可以发现，我们的网关核心服务主要引用的包有：`soul-spring-boot-starter-gateway`和`soul-spring-boot-starter-plugin-XXX`，推断前者是soul网关服务的核心业务，后者是各个插件的实现。其中`soul-spring-boot-starter-gateway`项目实质只是对`soul-web`项目进行了一层spring引用的封装。而各个插件项目也是引用`soul-plugin-xxx`项目并进行了spring封装，即实例化相关`SoulPlugin`让spring容器接管。方式是通过`spring.factories`文件配置一个被`@Configuration`注解的配置类。下面以RateLimiter插件为例看主要代码：

## spring.factories

```java
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.dromara.soul.springboot.starter.plugin.ratelimiter.RateLimiterPluginConfiguration
```

## RateLimiterPluginConfiguration

```java
@Configuration
public class RateLimiterPluginConfiguration {
    
    /**
     * RateLimiter plugin.
     * RateLimiter插件处理逻辑的实现类
     * @return the soul plugin
     */
    @Bean
    public SoulPlugin rateLimiterPlugin() {
        return new RateLimiterPlugin(new RedisRateLimiter());
    }
    
    /**
     * Rate limiter plugin data handler .
     * 处理soul-admin发布的最新插件配置
     * @return the plugin data handler
     */
    @Bean
    public PluginDataHandler rateLimiterPluginDataHandler() {
        return new RateLimiterPluginDataHandler();
    }
}
```

从这部分代码可以发现，如果想自定义插件，大体也至少需要这两部分，一个是插件处理类，一个是处理动态配置的类。

# soul-web

从soul-bootstrap中我们猜测soul-web实现了soul的核心逻辑，下面我们来重点深入了解一下。首先还是从`spring.factories`入手看看都加载了哪些类到spring容器中。`spring.factories` 内容如下：

```yml
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.dromara.soul.web.configuration.SoulConfiguration
org.springframework.context.ApplicationListener=\
org.dromara.soul.web.logo.SoulLogo
```

## SoulLogo

`SoulLogo`实现spirng的`ApplcationListener`接口监听`ApplicationEnvironmentPreparedEvent`事件，即在SpringApplication启动时环境刚准备好时触发。结合类名可以推断出实现的功能是在项目启动时打印logo，经测试，这种配置方式打印的logo比spring的还靠前。发散一下，该种类可以用于启动时的环境的检测或修改一些配置。

## SoulConfiguration

今天分析的重点，源码如下，注释大多为初步猜测可能不对：

```java
@Configuration
@ComponentScan("org.dromara.soul") // 扫码org.dromara.soul 包下所有配置类
@Import(value = {
  ErrorHandlerConfiguration.class,  //spring 错误处理的配置
  SoulExtConfiguration.class,  // 获取 soul默认的success/error 返回结果
  SpringExtConfiguration.class // 创建 SoulApplicationContextAware 
    })
@Slf4j
public class SoulConfiguration {
    
    /**
     * Init SoulWebHandler.
     *	初始化SoulWebHandler，该类实现WebHandler 非常重要
     *  初始化时需要容器中所有soulPlugin作为参数
     * @param plugins this plugins is All impl SoulPlugin.
     * @return {@linkplain SoulWebHandler}
     */
  // SoulWebHandler 命名为webHandler
    @Bean("webHandler")
    public SoulWebHandler soulWebHandler(final ObjectProvider<List<SoulPlugin>> plugins) {...}
    
    /**
     * init dispatch handler.
     * 初始化DispatcherHandler，该类十分重要
     * @return {@link DispatcherHandler}.
     */
  // dispatcherHandler 改名字了！
    @Bean("dispatcherHandler")
    public DispatcherHandler dispatcherHandler() {...}
    
    /**
     * Plugin data subscriber plugin data subscriber.
     * 订阅插件配置数据
     * @param pluginDataHandlerList the plugin data handler list
     * @return the plugin data subscriber
     */
    @Bean
    public PluginDataSubscriber pluginDataSubscriber(final ObjectProvider<List<PluginDataHandler>> pluginDataHandlerList) {...}
    
    /**
     * Generic param resolve service generic param resolve service.
     * 暂时没猜到，dubbo相关，应该不重要
     * @return the generic param resolve service
     */
    @Bean
    @ConditionalOnProperty(name = "soul.dubbo.parameter", havingValue = "multi")
    public DubboParamResolveService dubboMultiParameterResolveServiceImpl() {...}
    
    /**
     * Generic param resolve service dubbo param resolve service.
     * 暂时没猜到，dubbo相关，应该不重要
     * @return the dubbo param resolve service
     */
    @Bean
    @ConditionalOnMissingBean(value = DubboParamResolveService.class, search = SearchStrategy.ALL)
    public DubboParamResolveService defaultDubboParamResolveService() {
        return new DefaultDubboParamResolveService();
    }
    
    /**
     * Remote address resolver remote address resolver.
     *
     * @return the remote address resolver
     */
    @Bean
    @ConditionalOnMissingBean(RemoteAddressResolver.class)
    public RemoteAddressResolver remoteAddressResolver() {...}
    
    /**
     * Cross filter web filter.
     * if you application has cross-domain.
     * this is demo.
     * 1. Customize webflux's cross-domain requests.
     * 2. Spring bean Sort is greater than -1.
     *
     * 跨域处理
     * @return the web filter
     */
    @Bean
    @Order(-100)
    @ConditionalOnProperty(name = "soul.cross.enabled", havingValue = "true")
    public WebFilter crossFilter() {...}
    
    /**
     * Body web filter web filter.
     * 
     * 打开时，如果form表单提，修改header的CONTENT_LENGTH?
     * @param soulConfig the soul config
     * @return the web filter
     */
    @Bean
    @Order(-10)
    @ConditionalOnProperty(name = "soul.file.enabled", havingValue = "true")
    public WebFilter fileSizeFilter(final SoulConfig soulConfig) {...}
    
    /**
     * Soul config soul config.
     * soul配置
     * @return the soul config
     */
    @Bean
    @ConfigurationProperties(prefix = "soul")
    public SoulConfig soulConfig() {...}
    
    /**
     * Init time web filter.
     * 
     * 打开filterTimeEnable 对拒绝服务返回408？
     * @param soulConfig the soul config
     * @return {@linkplain TimeWebFilter}
     */
    @Bean
    @Order(30)
    @ConditionalOnProperty(name = "soul.filterTimeEnable")
    public WebFilter timeWebFilter(final SoulConfig soulConfig) {...}
    
    /**
     * Web socket web filter web filter.
     *
     * webSocket filter，如果请求头websocket=Upgrade，验证module、method、rpcType必须在header里
     * @return the web filter
     */
    @Bean
    @Order(4)
    public WebFilter webSocketWebFilter() {...}
}
```

总得来说，最重要的config里实例化了`DispatcherHandler`,`SoulWebHandler`以及很多spring的webFilter实现一些特定过滤逻辑。下面说下`DispatcherHandler`以及`Webhandler`。

## DispatcherHandler

通过资料查询，发现DispatcherHandler是spring webFlux的核心，他负责处理全部请求，主要分3步，1.找到负责该请求的handler；2.执行handler得到result，3.处理result。官方注解如下：

```java
Central dispatcher for HTTP request handlers/controllers. 
// http请求的核心调度员
Dispatches to registered handlers for processing a request, providing convenient mapping facilities.
// 向要处理的request请求分配注册的handler
DispatcherHandler discovers the delegate components it needs from Spring configuration. 
// dispatcherHandler从spring容器中获取所需类的组件
It detects the following in the application context:
// 它关注的类型主要有：
HandlerMapping -- map requests to handler objects
// 根据 request 找对应handler
HandlerAdapter -- for using any handler interface
// 调用具体的方法对用户发来的请求来进行处理
HandlerResultHandler -- process handler return values
// 处理返回值
DispatcherHandler is also designed to be a Spring bean itself and implements ApplicationContextAware for access to the context it runs in. 
// DispatcherHandler同样是bean并且实现了ApplicationContextAware，为了方便使用spring上下文。
If DispatcherHandler is declared with the bean name "webHandler" it is discovered by WebHttpHandlerBuilder.applicationContext which creates a processing chain together with WebFilter, WebExceptionHandler and others.
// 如果DispatcherHandler被命名为webHandler则会被` WebHttpHandlerBuilder.applicationContext`发现，然后与`WebFilter`、`WebExceptionHandler`以及其他一起创建请求处理链条
A DispatcherHandler bean declaration is included in @EnableWebFlux configuration.
// 使用注解@EnableWebFlux就可以创建DispatcherHandler
```

DisDispatcherHandler 重要处理方法——handle。

```java

@Override
//ServerWebExchange 存放着重要的请求-响应属性、请求实例和响应实例等等，有点像Context的角色。
public Mono<Void> handle(ServerWebExchange exchange) {
   if (this.handlerMappings == null) {
      return createNotFoundError();
   }
   // 遍历handlerMapping
   return Flux.fromIterable(this.handlerMappings)
         // 保证 逻辑执行完 顺序还是原来的顺序
         // 执行handlerMapping的getHandler方法(上文提到重点重新的)
         .concatMap(mapping -> mapping.getHandler(exchange))
         // 取Flux的一个变为Momo
         .next()
         // 空报错
         .switchIfEmpty(createNotFoundError())
         // 调用invokeHandler,使用mappinghandler和exchange得到一个result
         .flatMap(handler -> invokeHandler(exchange, handler))
         // 调用handleResult 处理result 无返回
         .flatMap(result -> handleResult(exchange, result));
}
private <R> Mono<R> createNotFoundError() {
   return Mono.defer(() -> {
      Exception ex = new ResponseStatusException(HttpStatus.NOT_FOUND, "No matching handler");
      return Mono.error(ex);
   });
}
// 找一个支持这个handler的handlerAdapter，并调用其handler方法，得到result
private Mono<HandlerResult> invokeHandler(ServerWebExchange exchange, Object handler) {
   if (this.handlerAdapters != null) {
      for (HandlerAdapter handlerAdapter : this.handlerAdapters) {
         if (handlerAdapter.supports(handler)) {
            return handlerAdapter.handle(exchange, handler);
         }
      }
   }
   return Mono.error(new IllegalStateException("No HandlerAdapter: " + handler));
}
private Mono<Void> handleResult(ServerWebExchange exchange, HandlerResult result) {
//调用匹配的HandlerResultHandler处理result
   return getResultHandler(result).handleResult(exchange, result)
         .checkpoint("Handler " + result.getHandler() + " [DispatcherHandler]")
         .onErrorResume(ex ->
               result.applyExceptionHandler(ex).flatMap(exResult -> {
                  String text = "Exception handler " + exResult.getHandler() +
                        ", error=\"" + ex.getMessage() + "\" [DispatcherHandler]";
                  return getResultHandler(exResult).handleResult(exchange, exResult).checkpoint(text);
               }));
}
// 找能处理result的resultHandler
private HandlerResultHandler getResultHandler(HandlerResult handlerResult) {
   if (this.resultHandlers != null) {
      for (HandlerResultHandler resultHandler : this.resultHandlers) {
         if (resultHandler.supports(handlerResult)) {
            return resultHandler;
         }
      }
   }
   throw new IllegalStateException("No HandlerResultHandler for " + handlerResult.getReturnValue());
}
```

但是请注意！！！由于在配置文件中，DispatcherHandler改了bean的名字，将soulWebHandler命名为webHandler，所以实际上webFlux没有用默认的DispatcherHandler，而是用的soulWebHandler！！！。这一点可以通过调用请求时debug `DefaultWebFilterChain`，可以发现它持有的webHandler不是DispatcherHandler而是soulWebHandler。也是因为这个操作soul接管了所有请求流量。

![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/soul04-01.png)

所以SoulWebHandler是等同于DispatcherHandler一样重要，负责请求处理的核心代码！

## SoulWebHandler

非常重要下次再讲。



