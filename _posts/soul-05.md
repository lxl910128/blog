---
title: soul学习——05soul核心处理链
categories:
- soul
tags:
- soul, 网关
keywords:
- soul , 网关, SoulWebHandler, WebHttpHandlerBuilder, HttpWebHandlerAdapter
---

# 概述

本文主要分析soul中`SoulWebHandler`的源码。该类实现了webHandler接口，替代了webFlux默认的`DispatcherHandler`负责全部流量的处理。可以说`SoulWebHandler`是soul处理流量的入口，体现了soul实现网关的核心思路。剧透核心思路就是挨个调用启动的插件，遍历插件配置，看流量是否匹配，如果匹配则根据配置执行插件逻辑。

插件大体可以分为路由插件和业务插件，路由插件负责将请求转发到正确的下游服务上，业务插件负责执行网关的通用业务逻辑，比如限流、鉴权等。流量必须在路由插件中配置才能正确转发，而业务插件不是必配的。每个插件都是单独针对全部流量配置筛选和使用规则，所以配置可以很灵活。
<!-- more -->
# WebHttpHandlerBuilder

WebHandler定义了Web请求必要一些处理行为，官方解释如下：

>Contract to handle a web request.
>
>负责处理web请求
>
>Use HttpWebHandlerAdapter to adapt a WebHandler to an HttpHandler.
>
>使用HttpWebHandlerAdapter将WebHandler装饰为一个HttpHandler
>
> The WebHttpHandlerBuilder provides a convenient way to do that while also optionally configuring one or more filters and/or exception handlers.
>
>WebHttpHandlerBuilder提供了一种方便的方法，将WebHandler装饰为1个HttpHandler，同时还可以选择配置一个或多个过滤器和/或异常处理程序。

可以看出主要是WebHttpHandlerBuilder在使用WebHandler，它的build方法构造出了HttpHandler——用于处理Http请求，需要依托于webHandler。其核心源码如下：

```java
/*
Static factory method to create a new builder instance by detecting beans in an ApplicationContext.
WebHttpHandlerBuilder使用ApplicationContext初始化的静态方法

需要加载如下内容
The following are detected:

根据beanName= webHandler 获取WebHandler
WebHandler [1] -- looked up by the name WEB_HANDLER_BEAN_NAME.

获取WebFilter并排序
WebFilter [0..N] -- detected by type and ordered, see AnnotationAwareOrderComparator.

获取WebExceptionHandler并排序
WebExceptionHandler [0..N] -- detected by type and ordered.

根据beanName=webSessionManager 获取WebSessionManager
WebSessionManager [0..1] -- looked up by the name WEB_SESSION_MANAGER_BEAN_NAME.

根据beanName=serverCodecConfigurer 获取serverCodecConfigurer
ServerCodecConfigurer [0..1] -- looked up by the name SERVER_CODEC_CONFIGURER_BEAN_NAME.

根据beanName=localeContextResolver 获取localeContextResolver
LocaleContextResolver [0..1] -- looked up by the name LOCALE_CONTEXT_RESOLVER_BEAN_NAME.
*/
public static WebHttpHandlerBuilder applicationContext(ApplicationContext context) {
  ……
}
public HttpHandler build() {
    // 使用名称为webHandler的bean和filter构造FilteringWebHandler，本质是构造了DefaultWebFilterChain处理链
		WebHandler decorated = new FilteringWebHandler(this.webHandler, this.filters);
    // 再加上异常处理
		decorated = new ExceptionHandlingWebHandler(decorated,  this.exceptionHandlers);
    // wenHandler 构造HttpWebHandlerAdapter，该装饰器，装饰器用WebHandler实现HttpHandler的功能
		HttpWebHandlerAdapter adapted = new HttpWebHandlerAdapter(decorated);
		……………………
		return adapted;
	}
```

可以看出`SoulWebHandler`在`WebHttpHandlerBuilder` 里和`WebFilter`以及`WebExceptionHandler`构成了1个处理链WebHandler，然后再通过装饰器变成HttpHandler提供http请求处理逻辑。这里取WebHandler是通过bean的名字取的，因此只要设置好名字SoulWebHandler就成功替代了dispatchWebHandler。

# HttpWebHandlerAdapter

`HttpWebHandlerAdapter` 主要功能是使用WebHandler实现HttpHandler接口。核心处理逻辑简短说就是把HttpHandler接口入参`request`和`response`封装成WebHandler用ServerWebExchange，然后调WebHandler的处理方法。具体如下：

```java
	@Override
	public Mono<Void> handle(ServerHttpRequest request, ServerHttpResponse response) {
		if (this.forwardedHeaderTransformer != null) {
			request = this.forwardedHeaderTransformer.apply(request);
		}
    //  request+response->ServerWebExchange
		ServerWebExchange exchange = createExchange(request, response);

		LogFormatUtils.traceDebug(logger, traceOn ->
				exchange.getLogPrefix() + formatRequest(exchange.getRequest()) +
						(traceOn ? ", headers=" + formatHeaders(exchange.getRequest().getHeaders()) : ""));
		// 拿webHandler 该handler核心处理逻辑是soulWebHandler实现，以及附带了各种webFilter、异常处理等
    // 调用webHandler的handler方法处理ServerWebExchange
		return getDelegate().handle(exchange)
				.doOnSuccess(aVoid -> logResponse(exchange))
				.onErrorResume(ex -> handleUnresolvedError(exchange, ex))
				.then(Mono.defer(response::setComplete));
	}
```

# SoulWebHandler

直接上核心代码：

```java
public final class SoulWebHandler implements WebHandler {
		// 构造函数传入的 容器中所有的soulPlugin
    private List<SoulPlugin> plugins;
  	// 调度器
 		private Scheduler scheduler;

    /**
     * Instantiates a new Soul web handler.
     *
     * @param plugins the plugins
     */
    public SoulWebHandler(final List<SoulPlugin> plugins) {
        this.plugins = plugins;
        String schedulerType = System.getProperty("soul.scheduler.type", "fixed");
      // 构造线程调度器 newParallel 或 elastic
        if (Objects.equals(schedulerType, "fixed")) {
            int threads = Integer.parseInt(System.getProperty(
                    "soul.work.threads", "" + Math.max((Runtime.getRuntime().availableProcessors() << 1) + 1, 16)));
            scheduler = Schedulers.newParallel("soul-work-threads", threads);
        } else {
            scheduler = Schedulers.elastic();
        }
    }

 @Override
    public Mono<Void> handle(@NonNull final ServerWebExchange exchange) {
        // 性能监听相关？
        MetricsTrackerFacade.getInstance().counterInc(MetricsLabelEnum.REQUEST_TOTAL.getName());
        Optional<HistogramMetricsTrackerDelegate> startTimer = MetricsTrackerFacade.getInstance().histogramStartTimer(MetricsLabelEnum.REQUEST_LATENCY.getName());
       // 所有插件构造插件处理链
        return new DefaultSoulPluginChain(plugins)
               // 执行
                .execute(exchange)
               // 分配线程
                .subscribeOn(scheduler)
               // 成功时计算性能？
                .doOnSuccess(t -> startTimer.ifPresent(time -> MetricsTrackerFacade.getInstance().histogramObserveDuration(time)));
    }
		// 插件处理链
    private static class DefaultSoulPluginChain implements SoulPluginChain {

        private int index;

        private final List<SoulPlugin> plugins;

        DefaultSoulPluginChain(final List<SoulPlugin> plugins) {
            this.plugins = plugins;
        }
        @Override
        // 挨个调用插件
        public Mono<Void> execute(final ServerWebExchange exchange) {
            return Mono.defer(() -> {
                if (this.index < plugins.size()) {
                    SoulPlugin plugin = plugins.get(this.index++);
                    Boolean skip = plugin.skip(exchange);
                    if (skip) {
                        return this.execute(exchange);
                    }
                    return plugin.execute(exchange, this);
                }
                return Mono.empty();
            });
        }
    }
}

```

soulWebFilter的逻辑很简单，核心就是请求走一遍插件调用链，即调用每个插件的`execute`方法，该方法需要`ServerWebExchange`作为传参，这个参数就是WebFilter::handler的入参数。

