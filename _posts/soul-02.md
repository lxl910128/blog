---
title: soul学习02——插件测试
date: 2021/1/16 1:01:10
categories:
- soul
tags:
- soul
- 网关
keywords:
- soul , 网关, 插件, 高性能, 可配置, 熔断, 限流
---
# 前言
本文主要介绍soul的精华——插件部分。并从个人的角度对比soul与spring gateway的差别。

<!-- more -->

# 插件概述

插件是soul可以实现功能的体现。目前soul有12种插件，我认为大概可以分为2类，接入类插件和功能类差价。接入类插件用来让各种框架下的服务可以正确接入soul实现路由转发功能，功能类插件实现网关的业务功能。可以不配置业务插件，但是至少得配置1个接入类插件。

接入类插件主要有：

1. divide，该插件支持spring-boot项目或其他提供http或websocket接口的服务接入soul实现基础的路由转发功能。
2. dubbo，该插件支持dubbo框架的项目快速接入soul实现基础的路由转发功能
3. spring cloud，该插件制止spring cloud体系的项目快速接入soul网关，主要是可以兼容eureka或nacos
4. sofa，该插件可以支持提供sofa-rpc的服务接入soul网关，而调用者可以依然使用http协议通过soul访问sofa服务

业务类插件有：

1. hystrix，该插件提供了熔断功能
2. sign，提供了签名验证功能
3. waf，提供了防火墙功能
4. rewrite，提供了重写访问路径的功能
5. rate-limiter，提供了限流功能
6. monitor，提供了监控自身运行状态，请求响应延迟，QPS,TPS等相关metrics的功能
7. sentinel，提供了熔断限流的能力
8. resilience4j，同样也实现了熔断限流的能力

下面逐一了解分析。

# 核心概念

在使用soul网关时有2个核心概念需要介绍，1个是**选择器（selector）**另一个是**规则（rule）**。选择器是用来筛选出1些请求，规则是对指定流量根据插件配置处理规则。关系可以理解为，1个插件可以配置多个选择器，每个选择器可以配置多个具体处理的规则逻辑。

需要注意的是选择器有筛选流量的作用，同时规则除了配置处理逻辑也能配置流量筛选逻辑。这样做可以更精细化的配置处理策略。选择器和规则在配置筛选条件的方式是相同的，可以通过：uri的path，url的query参数，http的header，目标服务ip，请求host5个纬度来筛选，并且可以通过and/or2中逻辑连接符来合并多个筛选条件。

在经过选择器和规则的双重过滤后，如果流量被匹配到，那么则表示该流量需要执行该插件的逻辑，具体执行方法根据规则的配置进行。

# 执行顺序

基于上一篇搭建的环境，来看看soul是如何执行他们的。首先我们为每个业务插件配置1个全局的选择器，但不配置具体的规则。比如对sentinel插件做如下配置：

![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/soul02-01.jpg)

访问昨天配置的`/test-api/demo/test`服务，可以发现如下日志

![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/soul02-02.jpg)

从日志中我们可以发现整个服务匹配了我们设置的多个业务插件，同时每次匹配到插件时`AbstractSoulPlugin`都会打日志。进入该类我们发现此类是每个插件执行的通用逻辑，代码注释如下:

```java
// 依照处理chain处理http请求
public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        // 取插件名
        String pluginName = named();
        // 插件管理页面配置的插件bean，主要看是否启用
        final PluginData pluginData = BaseDataCache.getInstance().obtainPluginData(pluginName);
        if (pluginData != null && pluginData.getEnabled()) {
            final Collection<SelectorData> selectors = BaseDataCache.getInstance().obtainSelectorData(pluginName);
            if (CollectionUtils.isEmpty(selectors)) {
                return handleSelectorIsNull(pluginName, exchange, chain);
            }
            // 从配置的 selector中查找是否有匹配此流量的 selector
            final SelectorData selectorData = matchSelector(exchange, selectors);
            if (Objects.isNull(selectorData)) {
                return handleSelectorIsNull(pluginName, exchange, chain);
            }
            //匹配到selector 打印匹配日志
            selectorLog(selectorData, pluginName);
            final List<RuleData> rules = BaseDataCache.getInstance().obtainRuleData(selectorData.getId());
          
            if (CollectionUtils.isEmpty(rules)) {
                return handleRuleIsNull(pluginName, exchange, chain);
            }
            RuleData rule;
            if (selectorData.getType() == SelectorTypeEnum.FULL_FLOW.getCode()) {
                //get last
                rule = rules.get(rules.size() - 1);
            } else {
                // 查找匹配插件
                rule = matchRule(exchange, rules);
            }
            if (Objects.isNull(rule)) {
                return handleRuleIsNull(pluginName, exchange, chain);
            }
            //匹配到rule 打印匹配日志
            ruleLog(rule, pluginName);
            // 执行插件逻辑，传入执行chain是为了 执行后继续执行后续插件
            return doExecute(exchange, chain, selectorData, rule);
        }
        return chain.execute(exchange);
    }
```

而调用该方法的类是`SoulWebHandler` ，此处我们发现了由插件组成的调用链条，如下

![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/soul02-03.jpg)

可以看出插件的调用链条依次为：

1. globalPlugin
2. signPlugin
3. wafPlugin
4. rateLimitePlugin
5. hystrixPlugin
6. resilience4jPlugin
7. springCloudPlugin
8. webClientPlugin
9. monitorPlugin
10. webClientResponsePlugin

可以发现业务插件的调用属性和日志中显示是大体一样的。其中没有配置但出现在调用链中的globalPlugin我们大胆的猜测，该插件封装的是通用处理，webClientPlugin是实际请求下游服务的插件，webClientRespousePlugin是想服务调用者回写响应的插件。

# 与pring gateway比较

1. SGW的配置都是围绕路由来进行的，一个路由有一组匹配规则以及一组处理filter，而soul是以插件为基础，每个插件配置一直选择器和1组规则。配置的着眼点不太一样。
2. SGW动态更新配置支持的不太好而soul非常方便
3. spring gateway没有可视化管理界面
4. spring gateway支持的熔断限流方式单一。soul提供多种可供选择。
5. spring gateway无法支持非http服务接入，而soul可以
6. spring gateway只能手动配置路由，soul可以自己发现基础路由。
7. soul不支持以时间作为流量过滤条件
8. soul没有插件能支持校验body的内容

