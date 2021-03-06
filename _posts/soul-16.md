---
title: soul番外1——XSS攻击
date: 2021/2/1 23:44:10
categories:
- soul
tags:
- soul
- 网关
keywords:
- soul , 网关, xss攻击
---
# 概述

网关作为所有服务的大门，应该有一些安全防护的功能，抵御常规的攻击。XSS攻击是非常常见的一种攻击方式。本文将简单介绍XSS攻击并讨论网关应该如何抵御XSS攻击。

<!--more-->

# 是什么

跨站脚本（XSS）是指恶意攻击者往Web页面里插入恶意Script代码，当用户浏览该页之时，嵌入Web里面的恶意代码（JavaScript、html等）会被执行，从而达到恶意攻击用户的目的。其实归根结底，XSS的攻击方式就是想办法“教唆”用户的浏览器去执行一些这个网页中原本不存在的前端代码。攻击者可以利用XSS 达到窃取用户 Cookie，劫持帐户，进行钓鱼攻击、构造XSS蠕虫、修改网页代码、利用网页重定向等目的。XSS攻击一般可以分为：反射型XSS、存储型XSS。

# 反射型XSS

反射型XSS的攻击相对于访问者而言是一次性的，具体表现在恶意脚本通过url的方式传递给了服务器，而服务器则只是不加处理的把脚本“反射”回访问者的浏览器而使访问者的浏览器执行相应的脚本。也就是说想要触发漏洞，需要访问特定的链接才能够实现。

详细来说恶意代码并没有保存在目标网站，而是通过引诱用户点击一个恶意链接来实施攻击。这类恶意链接有哪些特征呢？

1. 恶意脚本附加到 url 中，只有点击此链接才会引起攻击
2. 不具备持久性，即只要不通过这个特定 url 访问，就不会有问题
3. xss漏洞一般发生于与用户交互的地方

举个例子，比如我们在访问一个链接的时候（http://102.3.203.111/Web/reflectedXSS.jsp?param=value...），这个URL中就带了参数（param=value...），如果服务端没有对参数进行必要的校验，直接根据这个请求的参数值构造不同的HTML返回，让value出现在返回的html中（JS,HTML某元素的内容或者属性）并被浏览器解释执行，就可能存在反射型XSS漏洞。

我们用一张图来解释XSS漏洞攻击的原理：

![图1](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/soul15-01.jpeg)

上图中，攻击者（黑衣人）骗取用户信任，构造一个带有跨站脚本的链接，诱骗用户点击（第2、3步），跨站脚本在服务端（WEB应用程序）上没有被过滤，直接返回用户浏览器（第4步），用户浏览器执行恶意脚本（第5步），后面发生的事情就像第6、7步描述的那样。

# 存储型XSS

持久型/存储型XSS：嵌入到web页面的恶意HTML代码被存储到服务器端（数据库），攻击行为将伴随着攻击数据一直存在。用稍微专业的话来说就是：客户端发送请求到服务器端，服务器在没有验证请求中的信息的情况下，就对请求进行了处理，从而导致原本正常的页面被嵌入了恶意HTML代码。之后当其他用户访问该页面时，恶意代码自动执行，给用户造成了危害。如下图

![图1](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/soul15-02.png)

# XSS实际应用

## 劫持访问

劫持访问就是在恶意脚本中插入诸如`<script>window.location.href="http://www.baidu.com";</script>`的代码，那么页面就会跳转到百度首页。劫持访问在持久型和非持久型XSS中都比较常被利用。持久型XSS中劫持访问的危害不用说大家都清楚，但有人会问非持久型XSS中劫持访问有什么作用呢？很简单，试想下像[http://qq.com](https://link.zhihu.com/?target=http%3A//qq.com)，[http://baidu.com](https://link.zhihu.com/?target=http%3A//baidu.com)这样的域名下出现非持久型XSS，那么在发送钓鱼链接时就可以通过[http://qq.com](https://link.zhihu.com/?target=http%3A//qq.com)等域名进行跳转，一般人一看到[http://qq.com](https://link.zhihu.com/?target=http%3A//qq.com)之类的域名警惕性会下降，也就更容易上当了。

## 盗用cookie实现无密码登录

在网页浏览中我们常常涉及到用户登录，登录完毕之后服务端会返回一个cookie值。这个cookie值相当于一个令牌，拿着这张令牌就等同于证明了你是某个用户。如果你的cookie值被窃取，那么攻击者很可能能够直接利用你的这张令牌不用密码就登录你的账户。如果想要通过script脚本获得当前页面的cookie值，通常会用到document.cookie。试想下如果像空间说说中能够写入xss攻击语句，那岂不是看了你说说的人的号你都可以登录（不过某些厂商的cookie有其他验证措施如：Http-Only保证同一cookie不能被滥用）

# XSS防范

在预防之前我们先来聊聊XSS 有哪些注入的方法：

1. 在 HTML 中内嵌的文本中，恶意内容以 script 标签形成注入。
2. 在内联的 JavaScript 中，拼接的数据突破了原本的限制（字符串，变量，方法名等）。
3. 在标签属性中，恶意内容包含引号，从而突破属性值的限制，注入其他属性或者标签。
4. 在标签的 href、src 等属性中，包含 `javascript:` 等可执行代码。
5. 在 onload、onerror、onclick 等事件中，注入不受控制代码。
6. 在 style 属性和标签中，包含类似 `background-image:url("javascript:...");` 的代码（新版本浏览器已经可以防范）。
7. 在 style 属性和标签中，包含类似 `expression(...)` 的 CSS 表达式代码（新版本浏览器已经可以防范）。

那么预防的方案主要有：

1. 在表单提交或者URL参数传递前，对需要的参数进行过滤。
2. 后端过滤用户输入。检查用户输入的内容中是否有非法内容。如<>、”、’、%、; 、() 、&、+等。
3. 严格控制输出。做html实体转义处理：PHP中可以利用下面这些函数对可能出现XSS漏洞的参数进行过滤：htmlspecialchars() 、htmlentities()、strip_tags() 、urlencode()、 intval()等。Java中可以使用全局过滤器对可能出现XSS漏洞的参数进行过滤和转义。（ononloadload双写绕过）

# 网关工作

网关作为请求的输入输出口，可以统一的预防XSS攻击。网关可以统一的对url中的param参数以及可能会保存到数据空中的参数进行校验。检测输入的内容是否包含敏感字符，如果有可以做统一的日志警告或者拒绝请求。具体xss相关知识非常推荐大家阅读[此文](https://tech.meituan.com/2018/09/27/fe-security.html)。