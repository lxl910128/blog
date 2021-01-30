---
title: soul学习15——sign插件学习
date: 2021/1/30 23:44:10
categories:
- soul
tags:
- soul
- 网关
keywords:
- soul , 网关, 插件, 验签, sign, https, 数字签名
---
# 概述

本文我们只要看下sign插件——业务端验签插件的原理和实现。并讨论下数字签名以及业务层验签。

<!--more-->

# sign plugin使用

要使用sign-plugin需要以下几步：

1. soul网关引入了`soul-spring-boot-starter-plugin-sign`，
2. 在soul-admin中开启sign-plugin插件
3. 在soul-admin –> 认证管理中，点击新增，新增一条 AK/SK。添加时会要求你选择在soul网关注册的下游服务及路径。
4. 在soul-admin –> 插件列表-> sign，配置监控路径，注意这里监控的路径在不同位需要置配2遍
5. 请求网关时需要在header中带上4个参数
   1. timestamp，请求的时间戳
   2. appKey，此请求的AK
   3. version，写死`1.0.0`
   4. sign，签名
6. 正常请求网关，如果验证失败会返回401信息提示验签失败。

sign生成方式需要请求的path、header中的timestamp、以及sk。生成方式是按照path{path}timestamp{timestamp}version{1.0.0}{sk}的方式形成字符串。如果path为`/api/service/abc`，header中的timestamp是`1571711067186`,sk是`506EEB535CF740D7A755CB4B9F4A1536`，那么sign= `A021BF82BE342668B78CD9ADE593D683`=`MD5("path/api/service/abctimestamp1571711067186version1.0.0506EEB535CF740D7A755CB4B9F4A1536").toUpperCase()`。请求header中应该有如下4个信息才能过验证：

1. timestamp: 1571711067186
2. appKey: 1TEST123456781
3. sign: A90E66763793BDBC817CF3B52AAAC041
4. version: 1.0.0

# 实现逻辑

`SignPlugin`实现非常简单，调用SignService的`signVerify`方法验证`ServerWebExchange`上下文，通过调用下一层插件，不通过则直接回写response。下面我们看看SignService::signVerify的实现。

```java
@Override
    public Pair<Boolean, String> signVerify(final ServerWebExchange exchange) {
      // 缓存中拿插件配置 为空或enable则跳过
        PluginData signData = BaseDataCache.getInstance().obtainPluginData(PluginEnum.SIGN.getName());
        if (signData != null && signData.getEnabled()) {
            final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
            assert soulContext != null;
          // soul的上下文SoulContext不能为空，验证
            return verify(soulContext, exchange);
        }
        return Pair.of(Boolean.TRUE, "");
    }
    
    private Pair<Boolean, String> verify(final SoulContext soulContext, final ServerWebExchange exchange) {
      // http header中的appKey、sign、timestamp不能为空
        if (StringUtils.isBlank(soulContext.getAppKey())
                || StringUtils.isBlank(soulContext.getSign())
                || StringUtils.isBlank(soulContext.getTimestamp())) {
            log.error("sign parameters are incomplete,{}", soulContext);
            return Pair.of(Boolean.FALSE, Constants.SIGN_PARAMS_ERROR);
        }
      // timestap不能比现在早过5分钟
        final LocalDateTime start = DateUtils.formatLocalDateTimeFromTimestampBySystemTimezone(Long.parseLong(soulContext.getTimestamp()));
        final LocalDateTime now = LocalDateTime.now();
        final long between = DateUtils.acquireMinutesBetween(start, now);
        if (between > delay) {
            return Pair.of(Boolean.FALSE, String.format(SoulResultEnum.SING_TIME_IS_TIMEOUT.getMsg(), delay));
        }
      // 验签
        return sign(soulContext, exchange);
    }
    
    private Pair<Boolean, String> sign(final SoulContext soulContext, final ServerWebExchange exchange) {
      // appKey是"认证管理"页面分发出去的，否者验签失败
        final AppAuthData appAuthData = SignAuthDataCache.getInstance().obtainAuthData(soulContext.getAppKey());
        if (Objects.isNull(appAuthData) || !appAuthData.getEnabled()) {
            log.error("sign APP_kEY does not exist or has been disabled,{}", soulContext.getAppKey());
            return Pair.of(Boolean.FALSE, Constants.SIGN_APP_KEY_IS_NOT_EXIST);
        }// 该ak配置的地址不能为空，否者验签失败
        List<AuthPathData> pathDataList = appAuthData.getPathDataList();
        if (CollectionUtils.isEmpty(pathDataList)) {
            log.error("You have not configured the sign path:{}", soulContext.getAppKey());
            return Pair.of(Boolean.FALSE, Constants.SIGN_PATH_NOT_EXIST);
        }
   // ak配置的地址与本次请求的地址需要能match上，否者验签失败
        boolean match = pathDataList.stream().filter(AuthPathData::getEnabled)
                .anyMatch(e -> PathMatchUtils.match(e.getPath(), soulContext.getPath()));
        if (!match) {
            log.error("You have not configured the sign path:{},{}", soulContext.getAppKey(), soulContext.getRealUrl());
            return Pair.of(Boolean.FALSE, Constants.SIGN_PATH_NOT_EXIST);
        }
      // 根据之前介绍的sign生成逻辑再生成一遍签名sign
        String sigKey = SignUtils.generateSign(appAuthData.getAppSecret(), buildParamsMap(soulContext));
      // 验证 刚生成的和http头中的sign
        boolean result = Objects.equals(sigKey, soulContext.getSign());
        if (!result) {
          // 不同验签失败
            log.error("the SignUtils generated signature value is:{},the accepted value is:{}", sigKey, soulContext.getSign());
            return Pair.of(Boolean.FALSE, Constants.SIGN_VALUE_IS_ERROR);
        } else {
          // 形同则验签成功
            List<AuthParamData> paramDataList = appAuthData.getParamDataList();
            if (CollectionUtils.isEmpty(paramDataList)) {
                return Pair.of(Boolean.TRUE, "");
            }
          // 如果auth配置了往下带的参数，则在向下请求的header中的appParam字段加上配置的参数
            paramDataList.stream().filter(p ->
                    ("/" + p.getAppName()).equals(soulContext.getContextPath()))
                    .map(AuthParamData::getAppParam)
                    .filter(StringUtils::isNoneBlank).findFirst()
                    .ifPresent(param -> exchange.getRequest().mutate().headers(httpHeaders -> httpHeaders.set(Constants.APP_PARAM, param)).build()
            );
        }
        return Pair.of(Boolean.TRUE, "");
    }
```

整体验签逻辑非常简单，就是首先用时间看请求是否在效期内，用约定的签名方式同样生成出1个签名，如果和请求带的签名相同则表明验签通过。这套逻辑在sk\ak不丢失的情况下保证了请求这是合法有效的。因为他有我们约定的ak\sk以及知道我们的签名方法并且这个签名就是访问该path的（path是签名内容）。但如果伪造者盗用了我们的http header，在5分钟内请求同样的接口，同时修改请求参数（body，url的param）则是可以通过我们的校验的。当然这是soul官方提供的SignService，使用者应该自己实现自己的SignService。

# 应用层签名

# 数字签名

数字签名在ISO7498—2标准中定义为：“附加在数据单元上的一些数据，或是对数据单元所作的密码变换，这种数据和变换允许数据单元的接收者用以确认数据单元来源和数据单元的完整性，达到保护数据，防止被人(例如接收者、中间人)伪造”。简单来说数据签名是为了达到以下效果：

1. 确认信息是由签名者发送的;
2. 确认信息自签名后到收到为止，未被修改过;
3. 签名者无法否认信息是由自己发送的。

数字签名实现一般如下：

1. 发送方确定要签名的内容，soul中是时间、版本、path，其他场景多为报文的MD5值
2. 对签名内容使用某种加密算法进行加密产生加密签名，该步骤是为了实现签名者无法否认信息是由自己发送的。通常情况是用RSA算法用私钥加密（因为只有你有你的私钥），而soul中是MD5，但由于签名内容还附加了你的sk所以也不能抵赖签名内容确实是你想发的。
3. 将加密签名和签名内容一起发给接收方。通过是将其和报文一起发送给接收方而soul中是随请求发给soul
4. 接收方使用同样色思路生成加密签名如何和发送来的一样则验证通过。

通常情况是接收这再次计算报文MD5，然后RSA公钥解密加密签名比较2个MD5。如果信息不是由签名者发的则公钥无法解密，如果内容被修改则MD5不正确。因为是你的私钥签的MD5所有必定是你发的内容。在soul中因为有同样的MD5所以时间、版本、path、sk必定是你想要告诉我的通知，那么用时间验证请求是否过期就能成立，能用path验证请求是否合理的，用sk以及header中的appKey是否成对验证了请求发送人是否可靠。

##  HTTPS中的S

http中的s即LS/SSL主要用的就是数字签名技术实现的。https在建立连接时会发送给接收这1个自己的证书，这个这书是由权威机构（RSA公钥是公开的）签发的。证书由核心内容+签名构成的。运用数字签名思想我们能确保1、核心内容是全文机构颁发的；2、核心内容没被改过；3、签名者不能否认核心内容。那么核心内容是什么呢？1、授予的域名；2、有效期；3、域名的公钥。意思就是你和这个网站用在有效期内用次公钥。有了这个公钥你和服务器就能交换1个对称加密算法的秘钥，在往后的通信中用此加密方式加密通信内容。如果没有中间机构确保网站公钥的正确性，则可能受到中间人攻击。如果直接用公钥加密传输内容则效率远没有对称加密效率。

## 应用层签名

其实soul-sign插件实现的就是应用层签名，该方法保证了请求者的身份，请求的时效性。但其实内容有没有被修改这一点，只能说可以保证header中的时间戳、请求path、版本是没有被改过的，因为他是签名的主体。那么可能有人要问了应用层签名保证了请求者没有被伪造，那么对于请求这来说有没有方式保证服务者没有被伪造呢？其实是有的，https从某种程度上保证了这一点，因为他提供了合法的证书。同时还提供了数据传输的保密性。

