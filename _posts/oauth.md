---
title: OAuth 2.0介绍及实战
categories:
- OAuth
---
# 前言
起因时最近在调整博客的评论系统，该是使用到了OAuth2.0协议，就顺便学习了一下并在这里给大家分享下心得体会。本文会首先介绍OAuth2.0的相关知识，然后简单分析gitment评论系统是如何通过OAuth从git获取用户信息的。  
本博客的评论系统使用的是gitment，通过配置就可实现评论功能，不用关心评论保存以及维护用户。该工具是基于github实现，操作时需要获取用户github的资源及权限，这就涉及到github API中的授权机制————OAuth协议。  
<!--more-->
维基百科告诉我们，**开放授权（OAuth）是一个开放标准，允许用户让第三方应用访问该用户在某一网站上存储的私密的资源（如照片，视频，联系人列表），而无需将用户名和密码提供给第三方应用。** 
# 需求介绍
以博客评论系统为例子，本博客使用的gitment评论系统本质上是将用户的评论保存到我在github上预先建立的issues中，从而达到博客不需要后端存储评论信息的效果。用户评论博文时必须让gitment获取到用户github的用户信息（头像，名字等），代替用户将他的评论发到我的issues下。如下图分别是博客中的评论系统以及实际issues中存储的用户评论。  
![博客中的评论](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/140737.png)  
![issues的评论](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/140923.png)  

问题是用户如何授权gitment让其可以从github上获取自己的用户信息以及代替自己在issues下写评论呢？  
比较粗暴的方式是，用户告诉gitment自己github的帐号密码，这样gitment就可以作为用户登陆git获取信息发送评论了。但是这样做有以下几个严重缺点。
1. gitment为了服务需要存储用户的帐号密码，这十分部安全；
2. 用户无法限制gitment在github中的权限，比如权限范围，有效时间；
3. 用户只能修改密码才能收回权限，但是修改密码会导致其它第三方应用的权限都被收回；
4. 如果有一个第三方应用被攻破，用户的帐号密码就会被泄露。

为了解决这个需求OAuth诞生了，它可以在不给第三帐号密码的情况下，为第三方授权。同时，用户可以控制授权范围以及过期时间。

# 角色介绍
整个OAuth协议涉及到的一下几个角色：
* 第三方应用（Third-party application），即上述例子中gitment，它需要获取用户的权限、资源，后文简称为客户端；
* 资源所有者（HTTP service），即上例中的github，它拥有用户的资源；一般由认证服务和资源服务两部分组成；
  * 认证服务（Authorization server），github专门用于做权限认证的服务；
  * 资源服务（Resource server），github专门存储用户资源服务；
* 用户（Resource Owner），即上例中要发表评论的用户，资源的所有者。
* 代理（User Agent），一般是客户端可访问的一个浏览器。

# OAuth的思路
OAuth的整体思路是在客户端与资源所有者之间设置一个授权层。客户端不能以用户的身份值接访问资源所有者，需要用户在授权服务那登记客户端有用访问权限，然后客户端再去授权服务处领取token，客户端拿着token才能从资源服务那获取用户资源。这个token有明确的权限范围和有效期。

# 运行流程
OAuth2.0的运行流程大体如下：
```sequence
participant A as 客户端
participant B as 用户
participant C as 认证服务
participant D as 资源服务

A -->> B:1 Authorization Request
B -->> A:2 Authorization Grant
A -->> C:3 Authorization Grant
C -->> A:4 Access Token
A -->> D:5 Access Token
D -->> A:6 Protected Resource
```
上图描述了OAuth2.0中4个角色之间的交互。
1. 客户端向用户申请授权。授权申请可以直接发给用户(如图所示)，但推荐将认证服务作为中介。
2. 客户端收到授权凭证。
3. 客户端使用授权凭证向认证服务申请token。
4. 认证服务确认无误发送token。
5. 客户端使用token向资源服务申请资源。
6. 资源服务确认零盘无误向客户端发放资源。

不难看出，2和3两步十分重要，即用户如何给客户端授权以及客户端如何从认证服务处获取token。  
OAuth 2.0提供了4种方式完成用户授权客户端获取token。它们分别是：
1. 授权码授权（Authorization Code Grant）
2. 简单授权（Implicit Grant）
3. 用户密码授权（Resource Owner Password Credentials Grant）
4. 客户端证书授权（Client Credentials Grant）

其中第一种是最完善也时最官方推荐的方式，本文将着重介绍。

# 授权模式
## 授权码授权（Authorization Code Grant）
授权码模式可以获取访问token和刷新token。由于这是基于HTTP重定向的流，因此客户端必须能够与资源所有者的用户代理（通常是Web浏览器）进行交互，并且能够从授权服务器接收HTTP请求（通过重定向）。  
授权码模式是功能最完整、流程最严密的授权模式。它的特点就是通过客户端的后台服务器与认证服务器进行互动。整体流程如下图：

```sequence
participant A as 客户端
participant B as 用户代理
participant C as 认证服务
participant D as 用户


A -->> B:1 Client Identifier & Redirection URI
B -->> C:1 Client Identifier & Redirection URI
B -->> D:2 User authenticates
B -->> C:2 User authenticates
C -->> B:3 Authorization Code
B -->> A:3 Authorization Code
A -->> C:4 Authorization Code & Redirection URI
C -->> A:5 Access Token Optional Refresh Token
```
注意，图中1，2，3的先被分成2部分是因为它们通过了用户代理。
1. 用户访问客户端，后者将前者导向认证服务器。
2. 用户在代理上选是否通过授权，一般情况下此时会出现A申请的资源。
3. 假设用户授予访问权限，授权服务器使用先前提供的重定向URI（在请求1中或在客户端注册期间配置）将用户代理重定向回客户端。 重定向URI包括授权码和客户端先前提供的任何本地状态。
4. 客户端使用上一步获取授权码以及获取授权码时使用的重定向URI，向认证服务器申请令牌。这一步是在客户端的后台的服务器上完成的，对用户不可见。
5. 授权服务器对客户端进行身份验证，验证授权代码，并确保收到的重定向URI与步骤3中用于重定向客户端的URI相匹配。 如果有效，授权服务器返回访问token和刷新token。


刷新token的作用是客户的端访问token过期时，客户端可以用刷新token直接向认证服务换取新的访问token。

## 简单授权（Implicit Grant）
简化模式（implicit grant type）不通过第三方应用程序的服务器，直接在用户代理（浏览器）中向认证服务器申请令牌，跳过了"授权码"这个步骤，因此得名。所有步骤在浏览器中完成，令牌对访问者是可见的，且客户端不需要认证。流程如下图：
![Implicit Grant](http://www.ruanyifeng.com/blogimg/asset/2014/bg2014051205.png)

* （A）客户端将用户导向认证服务器。
* （B）用户决定是否给于客户端授权。
* （C）假设用户给予授权，认证服务器将用户导向客户端指定的"重定向URI"，并在URI的Hash部分包含了访问令牌。
* （D）浏览器向资源服务器发出请求，其中不包括上一步收到的Hash值。
* （E）资源服务器返回一个脚本，其中包含的代码可以获取Hash值中的令牌。
* （F）浏览器执行上一步获得的脚本，提取出令牌。
* （G）浏览器将令牌发给客户端。

##  用户密码授权（Resource Owner Password Credentials Grant）
密码模式中，用户向客户端提供自己的用户名和密码。客户端使用这些信息，向资源所有者索要token。  
在这种模式中，用户必须把自己的密码给客户端，但是客户端不得储存密码。这通常用在用户对客户端高度信任的情况下，比如客户端是操作系统的一部分，或者由一个著名公司出品。而认证服务器只有在其他授权模式无法执行的情况下，才能考虑使用这种模式。

## 客户端证书授权（Client Credentials Grant）
客户端证书授权指客户端以自己的名义，而不是以用户的名义，向"服务提供商"进行认证。严格地说，客户端模式并不属于OAuth框架所要解决的问题。在这种模式中，用户直接向客户端注册，客户端以自己的名义要求"服务提供商"提供服务，其实不存在授权问题。

# 实战
下面通过介绍gitmet评论系统如何通过OAuth获取用户在github上的资源权限，来体验下OAuth的整个流程。  
gitmet的使用流程可以概述为1、在自己的账号上注册一个OAuthApp；2、将OAuthApp中的Client ID和Client Secret配置到next主题的配置文件中。3、博客页面使用评论功能。

## 1、注册Git OAuth
在GitHub页面点击自己的头像打开菜单，点选“Settings”，然后在设置页面依次选择“Developer settings”--> “OAuth Apps” --> “New OAuth App”，进入创建OAuthApp的界面。
![注册](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/20190123213125.png)
其中需要特别注意的是Authorization callback URL，这个就是“授权码模式”中1、3步中提到的重定向URL。Github接受的是客户端注册时预留重定向URL的方式。这个URL需要设为博客网页的域名，要不然会验证不通过。  
注册完成会有如下图所示：
![完成注册](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/20190123214856.png)
这一步主要目的就是告诉github我们要构建个第三方应用，这个应用会申请获取用户在github上存的资料。

## 2、配置gitment
我是用的hexo主题是next，所以在next的_config.yml中配置gitment.enable，gitment.github_user，gitment.github_repo，gitment.client_id，gitment.client_secret。如图
![配置](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/20190123220237.png)
其中gitment.github_user是你github的用户名，gitment.github_repo需要是你git上存在repo，博客文章的评论都会存在这个repo的issues中。  
然后重新生成一遍hexo就可以使用评论功能了。目前可以说你的博客或者博客中嵌入的gitment代码就组成了OAuth流程中第三方应用。 

## 3、评论博客
之前全都是前期准备，现在才到真正走OAuth验证流程的时候。岔一句，根据官方文档，github使用的是授权码授权模式。 
现在登录博客，每篇文章中后面都会有评论的区域。点击登录会弹出git的认证页面，点击确定后完成验证返回微博页面。如下图
![未登录](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/%E5%9B%BE20190123223943.png)
![验证](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/20190123223902.png)
![登录后](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/20190123224145.png)
其实，上述一通操作已经完整走完了一遍OAuth验证。因为从用户的角度来看在授权码授权模式中你只需要在用户代理上确定授权就行了，其实就是图二中你在github认证页面（用户代理）输入账号密码的过程。剩下的过程对于用户来说都是透明的。  
拿到授权后客户端（博客页面）就可以向git索要你的用户信息了，直观体现就是博客获取到了你github的头像，如图3所示。同时它也权限代替你在git上评论我的issues了。  

## 4、客户端角度分析
本小结我们从客户端的角度仔细分析下授权码模式的处理过程。  
首先回顾上一节，其实在博客中点击登录时，就触发了整个流程的第一步客户端代理（博客页面）向认证服务器发送认证请求。可能有人会问，为什么页面也能当客户端呢？其实可以这么理解，博客页面中javaScript代码有实现客户端的逻辑，再加上浏览器这个代码运行容器就构成了客户端。  
那么客户端向认证服务器发送了什么请求呢？我们可以用chrome开发者工具查看页面“登录”二字的逻辑得知。其页面逻辑如下图
![登录](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/20190123230652.png)
不难看出登录二字其实是个a标签，它向git的认证服务器发送了一个请求，这对应于授权码模式的第一步中用户代理向认证服务器发送请求。从图中可以看出请求的主要参数有：
* scope OAuth规范中的字段。表示第三方应用申请的用户资源范围，本次请求申请的范围是public_repo。github可申请的范围可以在[官方文档](https://developer.github.com/apps/building-oauth-apps/understanding-scopes-for-oauth-apps/)查看。
* client_id OAuth规范中的字段。表示本应用在github注册的ID。
* redirect_uri OAuth规范中的字段。表示用户验证通过后的回调URI，同时git还会验证该uri与我们注册OAuthApp时填的回调URI域名是否匹配。同时后续客户端申请访问token是也要与这个URI相匹配。
* client_secret github认证独有。三方应用的秘钥，根据实验此值不填也行。

在OAuth协议中，除了上述这些参数第一步还有以下几个参数：
* response_type 认证类型，必填，授权码模式则此值应为“code”，我猜测由于github默认只支持授权码模式所以省略此参数。
* state 随机字符串，可选，用于防止跨站点请求伪造攻击。github也是支持这个字段的只不过gitment没有使用这个参数。

下面我们发一下这个请求，为了方便后续演示我们将回调URI改成localhost:2222/_getCode，并在本地启动相应的web服务接收git回调。同时将OAuthApp的回调URL修改为localhost:2222。修改后的URL如下所示：
```
https://github.com/login/oauth/authorize?scope=public_repo&redirect_uri=http%3A%2F%2Flocalhost:2222/_getCode&client_id=XXXXX&state=xyz
```




# 参考
http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html
https://www.barretlee.com/blog/2016/01/10/oauth2-introduce/
http://www.rfcreader.com/#rfc6749  
https://developer.github.com/apps/building-oauth-apps/authorizing-oauth-apps/