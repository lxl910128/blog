---
title: soul学习——03性能测试
categories:
- soul
tags:
- soul, 网关
keywords:
- soul , 网关, 插件, 高性能, 可配置, wrk, 性能测试, QPS
---

# 概述

本问对soul进行压测，了解soul的性能。

<!-- more -->

# 测试环境

1. 使用上一篇使用的环境，1个eureka注册中心，1个测试服务，1个soul-admin，1个soul网关
2. 使用全部业务插件配置full选择器，但不配置规则，按照实际情况配置spring-cloud插件，保证正常业务转发
3. 使用VisualVM监控soul网关的性能变化
4. 使用wrk进行压测

# 第一次测试

配置6个线程压测5分钟`http://localhost:9195/test-api/demo/test`，结果如下

![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/soul03-01.jpg)

可以发现QPS只有2194！感觉很低，赶紧关注下VisualVM，结果如下

![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/soul03-02.jpeg)

![](/Users/luoxiaolong/Documents/梦码/01作业/soul03-03.jpeg)

![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/soul03-04.jpeg)

从VisualVM的日志来看`logback` 消耗的时间比较多，忽然意识到，自己soul 服务使用`java -jar `直接启动的，在标准控制台里打了大量`  xxx selector success match , selector name :xxx` 的日志，猜测可能是日志打印消耗了性能，决定改进。

# 第二次测试

本次测试我将`soul-bootstrap.jar`中的日志级别调整为`WARN` ，并且在启动时使用`nohup java -jar soul-bootstrap.jar > /dev/null 2>&1 &` ，不将任何日志打印出来以减少消耗。

再使用相同的压测方式测试，结果如下：

![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/soul03-05.jpeg)

可以看出在6线程10连接的情况下QPS变为9052，提升了不少。但是经过和群里的小伙伴讨论认为还是下手轻了6个线程怎么能体现出我大soul的性能优势呢？所有决定再次测试加大并发。

# 第三次测试

本次测试我将`soul-bootstrap.jar`中的日志级别调整为`OFF` ，设置wrk参数为32个线程，维持100个连接，压测5分钟，具体测试参数如下图所示：

![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/soul03-06.jpeg)

可以发现QPS这次变到了1.2W可以说是相当厉害了。和群里的小伙伴沟通，他们压测的结果也是1W左右的并发。关注下VisualVM，结果如下:

![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/soul03-07.jpeg)

发现CPU核心耗时变成了netty或响应式框架种，感觉这样是比较正常的了。

# 总结

1. 在仅转发，无其他插件的情况下soul的QPS可以达到1.2W左右
2. 有时候不正确的打印日志也会降低程序的性能
3. soul-admin配置页面能直接关闭日志输出，这是我后续才发现的，十分方便
4. 在做性能测试时，有时候结果不高，很可能是因为并发不高，没有测到边界