---
title: soul学习06——soul插件处理逻辑
date: 2021/1/20 21:16:10
categories:
- soul
tags:
- soul
- 网关
keywords:
- soul , 网关, SoulWebHandler, DefaultSoulPluginChain, SoulPlugin, AbstractSoulPlugin
---

# 概述

本文接着上一篇文章，继续分析soulWebHandler的代码细节，看他是怎么调用插件链的。

<!-- more -->

# DefaultSoulPluginChain

上文粗鲁介绍了`DefaultSoulPluginChain` ，我们在复习一下，并以此为基础接着讲。它的核心代码不多，如下：

```java
// 该代码是soulWebHandler 中 handle方法实际执行的方法
// plugins是构造soulWebHandler时，从spring容器中拿到的所有的soulPlugin实现
@Override
public Mono<Void> execute(final ServerWebExchange exchange) {
  return Mono.defer(() -> {
    // 遍历
    if (this.index < plugins.size()) {
      SoulPlugin plugin = plugins.get(this.index++);
      // 判断plugin是否在该上下文中跳过
      Boolean skip = plugin.skip(exchange);
      if (skip) {
        // 递归调用自己
        return this.execute(exchange);
      }
      // 调用插件的execute，并且将 请求上下文和本chain作为参数出入
      return plugin.execute(exchange, this);
    }
    return Mono.empty();
  });
}

```

逻辑就是挨个调用所有开启插件的execute方法。然后我们来看看soulPlugin的execute执行逻辑，这段逻辑在实现SoulPlugin接口的抽象类`AbstractSoulPlugin`中。

# SoulPlugin

介绍`AbstractSoulPlugin`之前我们先看一眼SoulPlugin接口：

```java
public interface SoulPlugin {

    /**
     * 插件处理逻辑，主要做3件事
       1. 找请求匹配的 选择器和规则
       2. 如果有匹配根据规则、选择器执行插件
       3. 调用插件链的execute继续执行下个插件
     */
    Mono<Void> execute(ServerWebExchange exchange, SoulPluginChain chain);

    /**
      返回插件顺序，用于插件链的排序
     */
    int getOrder();

    /**
     * 返回 插件名字
     */
    default String named() {
        return "";
    }

    /**
     * 判断插件是否需要跳过
     */
    default Boolean skip(ServerWebExchange exchange) {
        return false;
    }

}
```

通过接口我们能很清晰的看出来每个插件需要干的事情。

# AbstractSoulPlugin

所有插件都是集成抽象类`AbstractSoulPlugin`，而它实现了execute方法，同时要求子类自行实现doExecute方法。在execute中主要干的是第一件事——根据当前流量找匹配的选择器和规则的逻辑。核心代码如下：

```java
/**
     * this is Template Method child has Implement your own logic.
     * 插件需要自行实现该方法，查件核心逻辑就在于此。
     * @param exchange 请求上下文
     * @param chain    插件调用链 DefaultSoulPluginChain
     * @param selector 匹配上的选择器
     * @param rule     匹配上的规则
     * @return {@code Mono<Void>} 不返回，插件修改的东西直接存在exchange即可
     */
    protected abstract Mono<Void> doExecute(ServerWebExchange exchange, SoulPluginChain chain, SelectorData selector, RuleData rule);

    /**
     * 调用按顺序执行的execute
     */
    @Override
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        // 取插件名，根据名字从缓存中拿配置
        String pluginName = named();
        // 插件管理页面配置的内容，主要看是否启用，包含多个选择器，每个选择包含多个规则
        final PluginData pluginData = BaseDataCache.getInstance().obtainPluginData(pluginName);
        if (pluginData != null && pluginData.getEnabled()) {
          // 全部选择器
            final Collection<SelectorData> selectors =    BaseDataCache.getInstance().obtainSelectorData(pluginName);
              // 选择器为空
            if (CollectionUtils.isEmpty(selectors)) {
              // 无选择器，如果是waf插件依然要执行
              // 如果是divide、dubbo、spring-cloud插件则会大个错误日志，所以如果不用对应类型的路由插件，请关闭
              // 最后 执行 chain.execute，即插件了继续往后走
                return handleSelectorIsNull(pluginName, exchange, chain);
            }
            // 从配置的 selector中查找是否有匹配此流量的 selector
            final SelectorData selectorData = matchSelector(exchange, selectors);
            if (Objects.isNull(selectorData)) {
                return handleSelectorIsNull(pluginName, exchange, chain);
            }
            //匹配到selector 打印匹配日志
            selectorLog(selectorData, pluginName);
          // 从cache中拿选择器全部规则
            final List<RuleData> rules = BaseDataCache.getInstance().obtainRuleData(selectorData.getId());
          
            if (CollectionUtils.isEmpty(rules)) {
                return handleRuleIsNull(pluginName, exchange, chain);
            }
            RuleData rule;
            // 判断是否是全部流量
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

每个插件通用的handler方法逻辑很简单。首先从cache中拿到插件配置PluginData，这个数据就是我们在soul-admin管理界面，点击"插件管理"-->"xx插件"，右侧配置的全部内容，主要是多个选择器以及每个选择器的多个规则。然后根据上下文exchange判断找出符合的选择器中符合的规则。然后调用各个插件需要实现的doExecute方法，该方法传入了：命中的选择器，命中的规则，请求上下文，以及插件调用链。其他省略的方法大多都是判断选择器、规则是否与当前请求匹配，主要是用java stream实现的。

soul的这部分代码逻辑非常清晰没有什么特别需要说明的。下面我们从一个sign插件的doExecute方法入手，看看各个插件时如何实现该方法的。

# SignPlugin

直接上代码：

```java
public class SignPlugin extends AbstractSoulPlugin {
    private final SignService signService;

    /**
     * 使用signService实例化
     */
    public SignPlugin(final SignService signService) {
        this.signService = signService;
    }
		// 获取差价名称 soulPlugin需要实现
    @Override
    public String named() {
        return PluginEnum.SIGN.getName();
    }
		// 插件顺序
    @Override
    public int getOrder() {
        return PluginEnum.SIGN.getCode();
    }

    @Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
      // 执行校验逻辑
        Pair<Boolean, String> result = signService.signVerify(exchange);
        if (!result.getLeft()) {
            Object error = SoulResultWrap.error(SoulResultEnum.SIGN_IS_NOT_PASS.getCode(), result.getRight(), null);
          // 校验失败，调用链不继续执行，直接返回错误信息
            return WebFluxResultUtils.result(exchange, error);
        }
      // 逻辑执行完，继续调用插件链下一个插件
        return chain.execute(exchange);
    }
}
```

从代码中可以看出，具体插件主要负责干的是后两件事：执行插件实际逻辑以及调用插件链的execute继续执行下个插件。这里插件业务逻辑是封装给SignService执行的，本文主要过逻辑暂不细究插件业务功能的实现。

# 总结

soul中所有流量走一遍插件调用链，就完成路由转发以及网关业务处理。插件调用链的执行方式以及各个插件内的执行逻辑十分清晰。不得不佩服做的深厚的软件开发功底。下一篇我们介绍2个非常重要的插件的实现，1个是globalPlugin以及1个路由插件（spring-cloud），这两个插件是所有访问spring-cloud下游服务的流量必须经过的2个插件。