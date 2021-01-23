---
title: soul学习09——插件相关抽象及设计
date: 2021/1/23 23:33:10
categories:
- soul
tags:
- soul
- 网关
keywords:
- soul , 网关, soul-web, HttpWebHandlerAdapter, SoulConfig, webFilter
---
# 概述

之前一系列文章讲解了soul网关的核心处理流程。未来的一些列文章将主要完成2个事情。1个是soul的配置同步机制另一个是设计完成一个与安全相关的插件。soul的配置同步机制可以说是与核心处理逻辑同等重要的功能，同样也是soul与spring-gateway、zuul核心的区别与特点。而完成一个安全相关的插件并贡献到社区，也算是对soul学的的一个总结。

本文主要再看看插件设计方面的东西，即`soul-plugin-api` 与 `soul-plugin-base`这两个项目， 为后续写插件做准备，同时重点关注数据同步的东西。
<!-- more -->
# soul-plugin-api

这个项目主要规定了插件相关的各个接口以及接口的一些公共实现，下面逐一再梳理下。

首先一个重要概念是`SoulContext`它是soul定义的上线文信息，保存在springFlux定义的http上下文ServerWebExchange中，这里面都是soul在整体流程流转中涉及到的东西。但是挺不理解为什么不直接把值放到ServerWebExchange中，或者把搜索内容都放到SoulContext里，插件只关注SoulContext就可以了。现在的情况是插件运行时有些值从ServerWebExchange中拿，有些值从SoulContext拿。在soul-plugin-api中同样定义了一个接口，用于构造`SoulContext`，即`SoulContextBuilder`，他的默认实现在global-plugin中，从这也可以看出SoulContext的构造是在global插件做的。

其次是`SoulPlugin`接口，它定义了1个插件必须实现的4个功能：execute执行，getOrder插件执行顺序，named插件名称，skip插件是否跳过。soul有一个抽象类实现了soulPlugin接口即`AbstractSoulPlugin`，代码实现soul-plugin-base中，之前文章介绍过，他核心做的一件事就是找出此次的流量与插件的那个选择器，哪规则匹配，然后调用具体插件的执行方法。基于这两个也就引出了soul对插件有2中定义："单一职责插件"和”匹配流量处理插件“，单一职责插件是直接实现soulPlugin接口，这种插件如果启用则对所有流量都处理。而”匹配流量处理插件“是继承`AbstractSoulPlugin`抽象类，只针对符合规则的流量做响应处理。

`SoulPluginChain`定义了插件链需要实现的方法也就是`execute`，执行下一个插件，目前也仅有1个具体实现`DefaultSoulPluginChain`在`SoulWebHandler`中。`SignService`定义了签名验证方法signVerify，由于api层面的验签各公司实现都不太相同，所有在实际使用时自己实现才是比较现实的做法。方式是实现`SignService`接口，并将其注册成bean就OK了。`SoulResult`定义了生成soul返回body的接口，接口有2个方法1个是获取成功的返回结果另一个是获取失败的，他的默认实现是`DefaultSoulResult`，返回的body是`DefaultSoulEntity`结构的，由code:int、message:string、data:object 组成。和返回相关的还有个`SoulResultWrap`，在soul-base-plugin中，它对获取返回又做了1层封装主要是提供了静态方法，方法的实现是从spring中拿到`DefaultSoulResult`然后再调对应的success或error方法，以及一个`SoulResultEnum`枚举，它定义了各种各种code的含义以及说明。基本和httpCode吻合但也有soul自定义的，这些code通常为负数。

# soul-plugin-base

这里面主要是所有插件都会用到的一些工具类以及soul-plugin-api一些接口的实现。

首先是`cache` package中的`BaseDataCache`和`CommonPluginDataSubscriber` 。BaseDataCache用ConcurrentMap保存了插件的配置，主要有：plugin、selector和rule。`CommonPluginDataSubscriber`则是soul的另一大核心，动态更新配置的消费端，这部分会接收到soul-admin发来的数据，并更新`BaseDataCache`中核心的3个map，后续会着重介绍着一块。

`condition`包中的方主要是用来用selector和rule的流量匹配的，下属`judge`包中可以看出匹配的主要模式有eq（全等），like（包含），match（url的通配符匹配）和RegEx（正则）。以上这些都是最细粒度的匹配方法，`strategy`中增加了逻辑关系and和or，当着组合的还是这4中判断元语。在匹配的过程中我们可以对请求的header、uri、UrlQueryParam、host、post使用匹配方法并用and、or连接。选择器selector和规则rule中的匹配都是这一套判断逻辑。比较奇怪的是soul没有实现not这个逻辑计算。

`PluginDataHandler`是各个插件将插件配置`PluginData`转为自己使用的配置的方法，这个方法主要是soul-admin告诉各个插件配置更改时调用hadler相应方法，保证配置正确被更新，这块在数据同步时会再深入了解。

最后是`utils`包中的各种工具方法。比较有意思的有`WebFluxResultUtils`，webFlux写返回的工具。`SpringBeanUtils`用户获取spring中的bean，他的spring上下文使用过`ApplicationContextAware`的方式获得的。此方法可以用于其他项目！源码入下：

```java
//@Configuration 使用spring.factories配置成Configuration
public class SpringExtConfiguration {

    /**
     * Application context aware application context aware.
     * spring会在何时的实际调用ApplicationContextAware的setApplicationContext方法，并传入向下文
     * 这样我们就可在非spring管理的对象中使用spring上下文了
     * @return the application context aware
     */
    @Bean
    public ApplicationContextAware applicationContextAware() {
        return new SoulApplicationContextAware();
    }

    /**
     * The type Soul application context aware.
     */
    public static class SoulApplicationContextAware implements ApplicationContextAware {

        @Override
        public void setApplicationContext(final ApplicationContext applicationContext) throws BeansException {
   // ApplicationContextAware接口的这个方法可以拿到spring向下文，再将其给个单例，这样非spring的代码就可以那spring中的bean了
          SpringBeanUtils.getInstance().setCfgContext((ConfigurableApplicationContext) applicationContext);
        }
    }
}
```

其他utils等具体使用时再介绍。

# 总结

后续看配置同步可以以`CommonPluginDataSubscriber`为切入点。

