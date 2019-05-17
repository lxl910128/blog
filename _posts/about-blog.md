---
title: 关于这个博客
categories:
- 随笔
---

# 为什么写博客
最近在查关于图数据库的资料，忽然发现同事写的一篇博客。哈?！他竟然在写博客，哇！他竟然坚持了3年多了，嗯...是时候开始写博客，把看到的东西总结消化成纹章了。所以这个博客就出现了，正好使闲置的云服务器和域名有了用处。

<!--more-->
# 如何优雅的写博客
经过调研决定使用hexo生成博客，用nginx发布博客，最终实现通过域名可以访问博客的效果。关于怎么使用nginx+hexo构建自己的博客就不再赘述了，现在聊聊我做的一些其它工作。由于我有自己的域名，所以没有将博客deploy到git上，但这就造成我每次写完文章需要先上传到服务器，然后使用hexo生成静态文件，最后将静态文件部署到nginx上。这很不方便，关键是还不能对文章进行版本管理。联想到git有hook功能，决定将文章用git管理起来，每次推文件时让git hook触发服务上部署的自动部署程序。这样就实现了只要有git就能写博客更新博客的效果。这里有几点需要说明：
* 我仅将hexo的source文件夹放到了git上，这样可以避免git上的文将量
* 在服务器上部署了一个名为[hooker](https://github.com/lxl910128/hooker)(我知道他啥意思..)的web应用，在接到git的回调时运行shell命令实现。

# 为什么叫盖娅计划
博主粗浅的认为人类大部分苦难来源于彼此不能互相理解。在阿西莫夫的银河帝国中，描述了一个盖娅星系，在这个星系中所有生物在拥有共有意识的同时还保有个体意识，我希望地球也能成为这样的世界。当然我知道这是不可能实现的或是在我有生之年是见不到的。但有位名叫nakedMan（出自HIMYM）的“哲人”曾说过“从今天起定一个目标，往后每次做选择都要想着这个目标，生活将会更加精彩（大概是这个意思）。”我的目标是盖娅星系，那么这个博客就叫盖娅计划吧。  
私以为数据是人或物在网络中的延伸，人与人、人与物可以借由网络通过数据建立联系。只有建立了紧密的联系才可以谈进一步的相互理解。由此看来学好大数据、物联网等计算技术是目前切实可行的实现盖娅星系的方向。因此本博客主要用于记录总结博主在学习计算机技术的心得以及一些小脑洞。