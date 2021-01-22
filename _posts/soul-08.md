---
Readertitle: soul学习——08soul-web拾遗
date: 2021/1/22 20:33:10
categories:
- soul
tags:
- soul
- 网关
keywords:
- soul , 网关, soul-web, HttpWebHandlerAdapter, SoulConfig, webFilter
---

# 概述

通过之前几篇文章的梳理，soul网关的主线逻辑已经清晰了，核心的功能逻辑在各个plugin中，流程逻辑在soul-web中。soul-web就是网关业务的实际执行者，下面我们来看看该项目中还有哪些我们没注意到的功能。

<!-- more -->

# SoulConfig

根据soul-web的包，逐一往下看，首先看到的就是SoulConfig

```java
@Data
public class SoulConfig {
    private Integer filterTime = 10;
    private Boolean filterTimeEnable = false;
    private Integer upstreamScheduledTime = 30;
    private Integer fileMaxSize = 10;
}
```

有4个变量，首先是第一组filterTimeEnable和filterTime，这个两个变量共同控制的是一个TimeWebFilter，可以从soulConfiguration中看出来。

```java
 public class SoulConfiguration {   
		@Bean
    @Order(30)
    @ConditionalOnProperty(name = "soul.filterTimeEnable")
    public WebFilter timeWebFilter(final SoulConfig soulConfig) {
        return new TimeWebFilter(soulConfig);
    }
 }
```

而timeWebFilter是一个已经删除的filter，原来好像控制的是请求时间不能和当前时间差太多。

```java
@Deprecated
public class TimeWebFilter extends AbstractWebFilter {

    private SoulConfig soulConfig;

    public TimeWebFilter(final SoulConfig soulConfig) {
        this.soulConfig = soulConfig;
    }
    @Override
    protected Mono<Boolean> doFilter(final ServerWebExchange exchange, final WebFilterChain chain) {
       /* final LocalDateTime start = requestDTO.getStartDateTime();
        final LocalDateTime now = LocalDateTime.now();
        final long between = DateUtils.acquireMinutesBetween(start, now);
        if (between < soulConfig.getFilterTime()) {
            return Mono.just(true);
        }*/
        return Mono.just(true);
    }
		// doFilter 为false才执行
    @Override
    protected Mono<Void> doDenyResponse(final ServerWebExchange exchange) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(HttpStatus.REQUEST_TIMEOUT);
        Object error = SoulResultWrap.error(SoulResultEnum.TIME_ERROR.getCode(), SoulResultEnum.TIME_ERROR.getMsg(), null);
        return WebFluxResultUtils.result(exchange, error);
    }
}
```

下一个变量是fileMaxSize，这个变量是与`soul.file.enabled`共同起作用的，当`soul.file.enabled`为true时表示支持传输文件，而在支持文件的时候文件大小的上限是${fileMaxSize}mb。这些部分逻辑也是用webFilter实现的。并且也是在SoulConfiguration中注册的。

```java
public class FileSizeFilter implements WebFilter {
 @Override
  public Mono<Void> filter(@NonNull final ServerWebExchange exchange, @NonNull final WebFilterChain chain) {
    // 从request头中拿到mediaType
    MediaType mediaType = exchange.getRequest().getHeaders().getContentType();

    if (MediaType.MULTIPART_FORM_DATA.isCompatibleWith(mediaType)) {
      // 如果是multipart/form-data 则执行逻辑
      ServerRequest serverRequest = ServerRequest.create(exchange, messageReaders);
      return serverRequest.bodyToMono(DataBuffer.class)
        .flatMap(size -> {
          if (size.capacity() > BYTES_PER_MB * fileMaxSize) {
            // body 大于预设值 默认10MB则报错，直接返回
            ServerHttpResponse response = exchange.getResponse();
            response.setStatusCode(HttpStatus.BAD_REQUEST);
            Object error = SoulResultWrap.error(SoulResultEnum.PAYLOAD_TOO_LARGE.getCode(), SoulResultEnum.PAYLOAD_TOO_LARGE.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
          }
          BodyInserter<Mono<DataBuffer>, ReactiveHttpOutputMessage> bodyInsert = BodyInserters.fromPublisher(Mono.just(size), DataBuffer.class);
          HttpHeaders headers = new HttpHeaders();
          headers.putAll(exchange.getRequest().getHeaders());
          // 删除 request的 Content-Length
          headers.remove(HttpHeaders.CONTENT_LENGTH);
          // 参考 spring gateway 对body进行了改造使其可以被读取？
          CachedBodyOutputMessage outputMessage = new CachedBodyOutputMessage(exchange, headers);
          return bodyInsert.insert(outputMessage, new BodyInserterContext())
            .then(Mono.defer(() -> {
              ServerHttpRequest decorator = decorate(exchange, outputMessage);
              // 重新构造ServerWebExchange 继续调用webFilter链
              return chain.filter(exchange.mutate().request(decorator).build());
            }));
        });
    }
    return chain.filter(exchange);
  }
}
```

最后1个参数是upstreamScheduledTime，猜测和定时更新配置有关，但是没到找调用该函数的地方，后面再补充。

# HttpWebHandlerAdapter

之前我们提到过，在spring-webFlux中处理请求有一层是webServer将请求交个httpHandler，它接收接收ServerHttpRequest和ServerHttpResponse，然后构造成ServerWebExchange，交给webHandler来处理。在webFlux中，将httpHandler和webHandler串起来的是`HttpWebHandlerAdapter`，它即同时实现了这两个接口，在httpHandler的逻辑中直接调用webHandler的handler。而他是由由`WebHttpHandlerBuilder`构造的。`HttpWebHandlerAdapter`实现了处理请求的全部外围逻辑。它包含了实际执行业务的soulWebHandler，所有的WebFilter，所有WebExceptionHandler（统一异常处理）以及可选的WebSessionManager（session管理器），ServerCodecConfigurer（在服务端扩展http读写的CodecConfigurer），LocaleContextResolver（提供国际化支持，通过设置系统的环境，根据运行环境使用不同的语言显示）。既然`HttpWebHandlerAdapter`提供了这么多可扩展点，那么soul有没有使用呢？答案当然是有的，就在soul-web中。

## webHandler

这里特别要注意webHandler和webFilter会一起构成`HttpWebHandlerAdapter`，左右webFilter在soul中是可以使用的，并且soul也用了，刚才TimeWebFilter和FileSizeFilter就是很好的例子。那么soul还使用了哪些webFilter呢？在`org.dromara.soul.web.filter`下我们发现还有跨域相关的`CrossFilter`和webSocker参数处理相关的`WebSocketParamFilter`。

CrossFilter

```java
// 跨域支持，就是加各种header
public class CrossFilter implements WebFilter {
    private static final String ALLOWED_HEADERS = "x-requested-with, authorization, Content-Type, Authorization, credential, X-XSRF-TOKEN,token,username,client";
    private static final String ALLOWED_METHODS = "*";
    private static final String ALLOWED_ORIGIN = "*";
    private static final String ALLOWED_EXPOSE = "*";
    private static final String MAX_AGE = "18000";
    @Override
    @SuppressWarnings("all")
    public Mono<Void> filter(final ServerWebExchange exchange, final WebFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        if (CorsUtils.isCorsRequest(request)) {
           // 是跨域请求
            ServerHttpResponse response = exchange.getResponse();
            HttpHeaders headers = response.getHeaders();
            // 增加以下header
            headers.add("Access-Control-Allow-Origin", ALLOWED_ORIGIN);
            headers.add("Access-Control-Allow-Methods", ALLOWED_METHODS);
            headers.add("Access-Control-Max-Age", MAX_AGE);
            headers.add("Access-Control-Allow-Headers", ALLOWED_HEADERS);
            headers.add("Access-Control-Expose-Headers", ALLOWED_EXPOSE);
            headers.add("Access-Control-Allow-Credentials", "true");
            if (request.getMethod() == HttpMethod.OPTIONS) {
                response.setStatusCode(HttpStatus.OK);
                return Mono.empty();
            }
        }
        return chain.filter(exchange);
    }
}
```

WebSocketParamFilter

```java
// 验证 webSocket请求中httpheader是否完备
public class WebSocketParamFilter extends AbstractWebFilter {
    @Override
    protected Mono<Boolean> doFilter(final ServerWebExchange exchange, final WebFilterChain chain) {
        final ServerHttpRequest request = exchange.getRequest();
        final HttpHeaders headers = request.getHeaders();
        final String upgrade = headers.getFirst("Upgrade");
      // hedaer中upgrade key不能为空且值为websocket
        if (StringUtils.isNoneBlank(upgrade) && RpcTypeEnum.WEB_SOCKET.getName().equals(upgrade)) {
          // header中module、method、rpcType不能为空
            return Mono.just(verify(request.getQueryParams()));
        }
        return Mono.just(true);
    }
		// 如果filter没通过则走以下函数
    @Override
    protected Mono<Void> doDenyResponse(final ServerWebExchange exchange) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
        Object error = SoulResultWrap.error(SoulResultEnum.PARAM_ERROR.getCode(), SoulResultEnum.PARAM_ERROR.getMsg(), null);
      // 回写错误信息
        return WebFluxResultUtils.result(exchange, error);
    }
    private Boolean verify(final MultiValueMap<String, String> queryParams) {
        return !StringUtils.isBlank(queryParams.getFirst(Constants.MODULE))
                && !StringUtils.isBlank(queryParams.getFirst(Constants.METHOD))
                && !StringUtils.isBlank(queryParams.getFirst(Constants.RPC_TYPE));
    }
}
```

## WebExceptionHandler

soul对WebExceptionHandler的实现是`GlobalErrorHandler`，源码如下：

```java
// 这种错都是比较严重的，没有被捕获的意外错误
public class GlobalErrorHandler extends DefaultErrorWebExceptionHandler {
    /**
     * Instantiates a new Global error handler.
     *
     * @param errorAttributes    the error attributes
     * @param resourceProperties the resource properties
     * @param errorProperties    the error properties
     * @param applicationContext the application context
     */
    public GlobalErrorHandler(final ErrorAttributes errorAttributes,
                              final ResourceProperties resourceProperties,
                              final ErrorProperties errorProperties,
                              final ApplicationContext applicationContext) {
        super(errorAttributes, resourceProperties, errorProperties, applicationContext);
    }

    @Override
    protected Map<String, Object> getErrorAttributes(final ServerRequest request, final boolean includeStackTrace) {
      // 根据request打错误日志
        logError(request);
      // 形成错误返回的body
        return response(request);
    }

    @Override
    protected RouterFunction<ServerResponse> getRoutingFunction(final ErrorAttributes errorAttributes) {
        return RouterFunctions.route(RequestPredicates.all(), this::renderErrorResponse);
    }

    @Override
    protected int getHttpStatus(final Map<String, Object> errorAttributes) {
        return HttpStatus.INTERNAL_SERVER_ERROR.value();
    }

    private Map<String, Object> response(final ServerRequest request) {
        Throwable ex = getError(request);
        Object error = SoulResultWrap.error(HttpStatus.INTERNAL_SERVER_ERROR.value(), HttpStatus.INTERNAL_SERVER_ERROR.getReasonPhrase(), ex.getMessage());
      // 错误都是500的错误，错误信息看ex 的message
        return GsonUtils.getInstance().toObjectMap(GsonUtils.getInstance().toJson(error));
    }
    private void logError(final ServerRequest request) {
        Throwable ex = getError(request);
        log.error(request.exchange().getLogPrefix() + formatError(ex, request));
    }
    private String formatError(final Throwable ex, final ServerRequest request) {
        String reason = ex.getClass().getSimpleName() + ": " + ex.getMessage();
        return "Resolved [" + reason + "] for HTTP " + request.methodName() + " " + request.path();
    }
}
```

# ForwardedRemoteAddressResolver

剩下soul-web中唯一看起来比较重要的就是实现了RemoteAddressResolver的ForwardedRemoteAddressResolver。他同样在SoulConfiguration中被实例化并委托给spring管理。功能是尝试从header中的 X-Forwarded-For 拿客户端的地址。

# 总结

总体来看soul的扩展都是围绕HttpWebHandlerAdapter提供的可扩展点展开的，而核心实现逻辑是使用换名字的小技巧将HttpWebHandlerAdapter默认使用的DispatcherHandler换为soulWebHandler来接管全部流量的。这也启发我们在扩展soul时不光可以写plugin来扩展，还可直接写webFilter等。