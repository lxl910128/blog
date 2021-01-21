---
title: soul学习——07公共插件逻辑
date: 2021/1/21 21:33:10
categories:
- soul
tags:
- soul
- 网关
keywords:
- soul , 网关,
---

# 概述

本文继续分析soul网关的处理流程。根据上文所有流量必定会走globalPlugin和1个路由插件，本文就来分析下这两个插件，其中路由插件选取spring-cloud插件来分析。

<!-- more -->

# GlobalPlugin

作为插件处理链的第一环且全部流量都要经过的一环，globalPlugin插件实现的逻辑必定十分基础且通用并且关乎于后续插件的处理，废话不多说直接上代码：

```java
public class GlobalPlugin implements SoulPlugin {
    // 将ServerWebExchange转为SoulContext
    private SoulContextBuilder builder;
    
    /**
     * 初始化GlobalPlugin需要一个SoulContextBuilder
     */
    public GlobalPlugin(final SoulContextBuilder builder) {
        this.builder = builder;
    }
    
    @Override
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
      // 上线文中取 request
        final ServerHttpRequest request = exchange.getRequest();
      // 取header 
        final HttpHeaders headers = request.getHeaders();
      // websocker需要的请求头
        final String upgrade = headers.getFirst("Upgrade");
        SoulContext soulContext;
        if (StringUtils.isBlank(upgrade) || !"websocket".equals(upgrade)) {
          // 正常的http请求，将ServerWebExchange转为SoulContext
            soulContext = builder.build(exchange);
        } else {
            final MultiValueMap<String, String> queryParams = request.getQueryParams();
            soulContext = transformMap(queryParams);
        }
      // 将soulContext放到标准上下文ServerWebExchange中
        exchange.getAttributes().put(Constants.CONTEXT, soulContext);
        return chain.execute(exchange);
    }
    // 规定插件执行顺序为0，最小
    @Override
    public int getOrder() {
        return 0;
    }
    // 对于websocker 构建SoulContext的方法
    private SoulContext transformMap(final MultiValueMap<String, String> queryParams) {
        SoulContext soulContext = new SoulContext();
        soulContext.setModule(queryParams.getFirst(Constants.MODULE));
        soulContext.setMethod(queryParams.getFirst(Constants.METHOD));
        soulContext.setRpcType(queryParams.getFirst(Constants.RPC_TYPE));
        return soulContext;
    }
    
    @Override
    public String named() {
        return "global";
    }
}
```

可以看出GlobalPlugin主要干的工作就是借助SoulContextBuilder用ServerWebExchange生成SoulContext，并将其存到ServerWebExchange的`context` key下。

我们纠结`SoulContextBuilder`转化的具体逻辑，但是可以看看`SoulContext`都存了什么，源码如下：

```java
public class SoulContext implements Serializable {

    /**
     * dubbo中的appName，http或spring-cloud中的第一级Path，sofa中的appName，websocket中的module
     */
    private String module;
    /**
     * dubbo中的serviceName，http或spring-cloud中的实际path去掉第一级，sofa中的serviceName，websocket中的module
     */
    private String method;
    /**
     * rpc类型. 可选值： "http","dubbo","springCloud","sofa".
     */
    private String rpcType;
    /**
     * httpMethod："get","post" .
     */
    private String httpMethod;
    /**
     * 业务签名sign
     */
    private String sign;
    /**
     * timestamp 
     */
    private String timestamp;
    /**
     * appKey .在header中
     */
    private String appKey;
    /**
     * path. uri path
     */
    private String path;
    /**
     * the contextPath. dubbo和sofa中从metaData中取，http或spring-cloud中和module一样是”/“+path的第一级
     */
    private String contextPath;
    /**
     * realUrl.  http或spring-cloud中是path去掉第一级
     */
    private String realUrl;
    /**
     *  dubbo params.
     */
    private String dubboParams;

    /**
     * startDateTime.
     */
    private LocalDateTime startDateTime;
}

```

可以看出以上这些变量是soul定义的上下文环境，部分变量根据下游服务不同有特殊处理需要注意，具体处理方式以`DefaultSoulContextBuilder` 的为准。

# spring-cloud Plugin

spring-cloud插件的主要功是当下游服务是spring-cloud体系的服务时，保证请求能正常的请求到下游服务上，源码如下：

```java
public class SpringCloudPlugin extends AbstractSoulPlugin {
    private final LoadBalancerClient loadBalancer;

    /**
    * 初试化是需要传入一个 spring的 loadBalance
     */
    public SpringCloudPlugin(final LoadBalancerClient loadBalancer) {
        this.loadBalancer = loadBalancer;
    }

    @Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        if (Objects.isNull(rule)) {
            return Mono.empty();
        }
        // soul 的上下文 ，不能为空
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        // 从rule 的handler中构造出 SpringCloudRuleHandle 仅 path和timeout2个属性
        final SpringCloudRuleHandle ruleHandle = GsonUtils.getInstance().fromJson(rule.getHandle(), SpringCloudRuleHandle.class);
        // 从Selector 的handler中构造出 SpringCloudSelectorHandle 仅 serviceId 1个属性
        final SpringCloudSelectorHandle selectorHandle = GsonUtils.getInstance().fromJson(selector.getHandle(), SpringCloudSelectorHandle.class);
        // selectorHandle中serviceId, ruleHandle中path不能为空
        if (StringUtils.isBlank(selectorHandle.getServiceId()) || StringUtils.isBlank(ruleHandle.getPath())) {
            Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_CONFIG_SPRINGCLOUD_SERVICEID.getCode(), SoulResultEnum.CANNOT_CONFIG_SPRINGCLOUD_SERVICEID.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        // loadBalancer 根据serviceId拿到实际请求地址，比如eureka为注册中心，会请求eureka注册中心
        final ServiceInstance serviceInstance = loadBalancer.choose(selectorHandle.getServiceId());
        if (Objects.isNull(serviceInstance)) {
            Object error = SoulResultWrap.error(SoulResultEnum.SPRINGCLOUD_SERVICEID_IS_ERROR.getCode(), SoulResultEnum.SPRINGCLOUD_SERVICEID_IS_ERROR.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        // 拿到实际URI
        final URI uri = loadBalancer.reconstructURI(serviceInstance, URI.create(soulContext.getRealUrl()));
        // 拼出实际URL
        String realURL = buildRealURL(uri.toASCIIString(), soulContext.getHttpMethod(), exchange.getRequest().getURI().getQuery());
        // 放到 上下文的 HTTP URL中
        exchange.getAttributes().put(Constants.HTTP_URL, realURL);
        //set time out.
        exchange.getAttributes().put(Constants.HTTP_TIME_OUT, ruleHandle.getTimeout());
        return chain.execute(exchange);
    }
		// 顺序 50
    @Override
    public int getOrder() { return PluginEnum.SPRING_CLOUD.getCode();}
    @Override // name  springCloud
    public String named() { return PluginEnum.SPRING_CLOUD.getName();}

    /**
     * plugin is execute.
     * 如果RPCtype 不是spring-cloud 则在插件链中 无脑跳过
     */
    @Override
    public Boolean skip(final ServerWebExchange exchange) {
        final SoulContext body = exchange.getAttribute(Constants.CONTEXT);
        return !Objects.equals(Objects.requireNonNull(body).getRpcType(), RpcTypeEnum.SPRING_CLOUD.getName());
    }

    private String buildRealURL(final String url, final String httpMethod, final String query) {
       ……
    }
}
```

可以看出spring-cloud插件也没做什么特殊的事情，主要就是用从注册中心找到合适的下游服务的URI地址，然后拼上其他信息形成实际要访问的URL，放在请求上下文的httpUrl key中。

和想象中的不太一样，我以为在此差价中就要访问后端服务了，我们继续找一下实际请求下游服务的地方

# WebClientPlugin

看了下插件调用链，发现有个WebClientPlugin，大胆猜测这就是个webclient，用于请求实际服务。下面我们来看看源码：

```java
public class WebClientPlugin implements SoulPlugin {
		// 
    private final WebClient webClient;

    /**
     * 初试化时 传入的 webClient
     */
    public WebClientPlugin(final WebClient webClient) {
        this.webClient = webClient;
    }

    @Override
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
      // 拿上下文
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
      // 拿真是url, 就是我们在spring-cloud插件中保存的
        String urlPath = exchange.getAttribute(Constants.HTTP_URL);
        if (StringUtils.isEmpty(urlPath)) {
            Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_FIND_URL.getCode(), SoulResultEnum.CANNOT_FIND_URL.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
      // 超时时间 默认3秒？
        long timeout = (long) Optional.ofNullable(exchange.getAttribute(Constants.HTTP_TIME_OUT)).orElse(3000L);
      // 重试次数 默认0
        int retryTimes = (int) Optional.ofNullable(exchange.getAttribute(Constants.HTTP_RETRY)).orElse(0);
        log.info("you request,The resulting urlPath is :{}, retryTimes: {}", urlPath, retryTimes);
      // 还是从上下文中取 method
        HttpMethod method = HttpMethod.valueOf(exchange.getRequest().getMethodValue());
      // 响应式的webclient 设置 method 以及 urlPath
        WebClient.RequestBodySpec requestBodySpec = webClient.method(method).uri(urlPath);
      // 调用下面的 handleRequestBody
        return handleRequestBody(requestBodySpec, exchange, timeout, retryTimes, chain);
    }

    @Override // 51 
    public int getOrder() {
        return PluginEnum.DIVIDE.getCode() + 1;
    }

    @Override
    public String named() {
        return "webClient";
    }

    @Override // rpc 是 http 或 rpc是 SPRING_CLOUD 时 不跳过
    public Boolean skip(final ServerWebExchange exchange) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        return !Objects.equals(RpcTypeEnum.HTTP.getName(), soulContext.getRpcType())
                && !Objects.equals(RpcTypeEnum.SPRING_CLOUD.getName(), soulContext.getRpcType());
    }

// 接着execute 继续看
    private Mono<Void> handleRequestBody(final WebClient.RequestBodySpec requestBodySpec,
                                         final ServerWebExchange exchange,
                                         final long timeout,
                                         final int retryTimes,
                                         final SoulPluginChain chain) {
        return requestBodySpec
          // 设置header
          .headers(httpHeaders -> {
            httpHeaders.addAll(exchange.getRequest().getHeaders());
            httpHeaders.remove(HttpHeaders.HOST);
        })			// 使用 header中的content-type或使用默认的application/json
                .contentType(buildMediaType(exchange))
          			// 设置 body
                .body(BodyInserters.fromDataBuffers(exchange.getRequest().getBody()))
          			// 执行请求，返回 Mono<ClientResponse>
                .exchange()
                // 如果请求失败 
                .doOnError(e -> log.error(e.getMessage()))
         				 // 设置超时 
                .timeout(Duration.ofMillis(timeout))
          			// 当超时时 重试 
                .retryWhen(Retry.onlyIf(x -> x.exception() instanceof ConnectTimeoutException)
                    .retryMax(retryTimes)
                    .backoff(Backoff.exponential(Duration.ofMillis(200), Duration.ofSeconds(20), 2, true)))
          			// 正常是调用 doNext()
                .flatMap(e -> doNext(e, exchange, chain));

    }
		// 接handleRequestBody
    private Mono<Void> doNext(final ClientResponse res, final ServerWebExchange exchange, final SoulPluginChain chain) {
      // 如果成功/失败 设置上下文
        if (res.statusCode().is2xxSuccessful()) {
            exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.SUCCESS.getName());
        } else {
            exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.ERROR.getName());
        }
      // 将response保存在上下文中
        exchange.getAttributes().put(Constants.CLIENT_RESPONSE_ATTR, res);
       // 继续调用 插件链
        return chain.execute(exchange);
    }
}
```

在`WebClientPlugin`旁边还有个`NettyHttpClientPlugin`，使用netty实现的http client，猜测可以通过配置可以启用netty的client。既然请求下游服务的插件找到了，那么顺便看看如何向上游服务回写结果的插件吧。

# WebClientResponsePlugin

看了下插件调用链，发现有个WebClientResponsePlugin，大胆猜测这就是回写结果给请求方的插件，用于请求实际服务。下面我们来看源码：

```java
public class WebClientResponsePlugin implements SoulPlugin {

     @Override
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        // 注意 处是先调用插件链的后续，然后在执行这些代码，这样保证了次插件逻辑是最后执行的
        return chain.execute(exchange).then(Mono.defer(() -> {
            // 从上下文中拿出 返回  这个是之前提到的spring HttpHandler 的response
            // 结果靠它写回给用户
            ServerHttpResponse response = exchange.getResponse();
            // 下游服务返回的response
            ClientResponse clientResponse = exchange.getAttribute(Constants.CLIENT_RESPONSE_ATTR);
            // 为null 返回错误
            if (Objects.isNull(clientResponse)
                    || response.getStatusCode() == HttpStatus.BAD_GATEWAY
                    || response.getStatusCode() == HttpStatus.INTERNAL_SERVER_ERROR) {
                Object error = SoulResultWrap.error(SoulResultEnum.SERVICE_RESULT_ERROR.getCode(), SoulResultEnum.SERVICE_RESULT_ERROR.getMsg(), null);
                return WebFluxResultUtils.result(exchange, error);
            }
            // 超时的返回
            if (response.getStatusCode() == HttpStatus.GATEWAY_TIMEOUT) {
                Object error = SoulResultWrap.error(SoulResultEnum.SERVICE_TIMEOUT.getCode(), SoulResultEnum.SERVICE_TIMEOUT.getMsg(), null);
                return WebFluxResultUtils.result(exchange, error);
            }
            // 用原response的 statusCode、cookie、header设置给上游的response
            response.setStatusCode(clientResponse.statusCode());
            response.getCookies().putAll(clientResponse.cookies());
            response.getHeaders().putAll(clientResponse.headers().asHttpHeaders());
            // 写 body，也就是给客户返回response
            return response.writeWith(clientResponse.body(BodyExtractors.toDataBuffers()));
        }));
    }

    @Override // 100 理论最大的
    public int getOrder() {
        return PluginEnum.RESPONSE.getCode();
    }

    @Override // rpc 是 http 或 rpc是 SPRING_CLOUD 时 不跳过
    public Boolean skip(final ServerWebExchange exchange) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        return !Objects.equals(RpcTypeEnum.HTTP.getName(), soulContext.getRpcType())
                && !Objects.equals(RpcTypeEnum.SPRING_CLOUD.getName(), soulContext.getRpcType());
    }
}

```

在`WebClientResponsePlugin`旁边同样也有个`NettyClientResponsePlugin`，应该是和`NettyHttpClientPlugin`成套配置的。核心逻辑基本相同，只不过组件从webflux的DefaultWebClient换成了reactor-netty。

# 总结

本文介绍了soul网关实现基础路由转发功能的全部插件，他们是：GlobalPlugin--> spring-cloud Plugin ---> WebClientPlugin --->  WebClientResponsePlugin ，且调用顺序如上。如果不考虑其他网关业务仅考虑路由转发功能的话，这些插件就足够了，这也是所有请求必走的主线流程。