---
title: Elasticsearch字符串包含查询优化
date: 2022/10/07 09:20:00

categories:
- elasticsearch

  tags:
- 搜索优化

  keywords:
- elasticsearch, 包含查询, wildcard, 优化
---
# 摘要

本文提出一种新的分词方式，配合前缀搜索可实现es中字符串包含查询，并与三种常用的包含查询实现进行对比，得出新方案在索引数据不过度膨胀的情况下，搜索耗时比现有方案减少80%。

<!-- more -->

# 背景

数据地图是快手内部的数据资产查找工具，帮助用户查询数据资产。其中**联想搜索**是重要的找数方式这一，该方式根据用户实时输入查找资产名称包含该关键字的数据资源。简单来说就是对数据资源名称字段执行如mysql中 like %keyword% 的搜索。比如下图就是在数据地图中输入"offline_attribution_content"联想到的数据资源。

![img](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/es-optimize-like-filter-01.png)


数据地图使用Elasticsearch作为搜索引擎，目前es中索引的数据资产有近百万，每周增涨近万文档，单次联想查询在300ms左右。随着es中索引文档的增加，联想搜索的耗时也在慢慢变长，越来越不能支持用户输入1个字母后就立马出结果的业务需求，需要探索新的搜索思路来提升搜索效率。

# 业界调研

Elasticsearch实现搜索的基本思路是**倒排索引** ，即对索引字段先进行分词形成多个词元（Term），然后建立词元到文档的映射关系。在这种基本模式下elasticsearch实现字符串包含搜索的方式有以下三种：![img](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/es-optimize-like-filter-02.png)

第一种思路是最容易考虑到的方案，也是数据地图当前使用的搜索方式。在该方案中表名new_offline_attribution_content_func在录入索引时不会被分词，搜索offline_attribution_content时会被转为表达式*offline_attribution_content*搜索。

第二种思路在new_offline_attribution_content_func录入索引时使用simple分词变为["new", "offline", "attribution", "content" ,"func"]，使用queryString搜索"offline_attribution_content"时，请求会被转为搜索"offline"，“attribution”，"content"并要求原文中这3个词是连续出现的。

但第二种方案有漏洞，当用户1个词没输入完时如offline_attribution_cont，请求实际是搜索 "offline"，“attribution”，"cont"且要求连续，由于"offline_attribution_content"并没有切分出"cont"这个词而导致搜索错误。如果想继续使用"queryString"搜索，可以将simple分词方式调整为按字符分词，将new_offline_attribution_content_func分词为['n','e','w','_', 'o', ......  ,'f','u','n', 'c']，搜索offline_attribution_content变为查找存在连续匹配['o','f','f','l', ......  ,'e','n','t']字符的文档。

第三种思路是使用es N-Gram分词器穷举索引词所有可能出现的分词组合，对于搜索关键词不做分词，使用等值查询term直接搜索。该方式会将索引词new_offline_attribution_content_func分为(1+36)*36/2 = 666个词元。

下表和下图总结了上述三种思路分词方式（以分'abcd'为例），分词后词元数与原词长度的关系，实现字符串包含搜索使用的查询方式以及构建的倒排索引的简略结构：

![img](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/es-optimize-like-filter-03.png)

![img](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/es-optimize-like-filter-04.png)


# 新的搜索方式

## Es倒排索引实现

Elasticsearch倒排索引底层由两部分组成分别是用FST字典树实现的“Term Index”和用SkipList实现的“Term Dictionary”如图。

![img](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/es-optimize-like-filter-05.png)


从底层索引出发我们再来分析上述三种思路。对于第一种思路索引词不做任何分词直接构成字典树，但是在搜索时由于搜索词头部也有通配符，所以字典树的索引并未产生效果，实质上搜索Term Index的过程就是遍历的过程。当匹配到关键词后直接将skipList中全部关联文档作为结果返回即可。由于Term Index并未发挥索引效果，那么在字典树节点增多的情况下必然导致效率下滑。

对于第二种思路是将索引词切成1个字符且每一个字符都做索引，所以Term Index仅有1层，由26个字母、数字以及一些特殊符号构成，且每个字符对应的skipLink会异常的长，比如字母e的skipLink会包含几乎所有的文档ID。最终结果还需要处理字符在相对位置，保持连续匹配的逻辑。因此该方案搜索效率必然欠佳。

对于第三种思路，1个索引词会分成(1+n)n/2个词元来参与构建字典树，最终Term Index的膨胀率会是十分惊人的。不过查询时搜索词通过字典树能非常快速的找到一个skipLink数组，该数组即是搜索结果。该方案随着数据的增多Term Index会十分复杂，写入会非常耗时，搜索效率也有一定衰弱。

## 新方案设计

通过分析，当前的三种方案在数据量增长的情况下都会对搜索效率及存储产生影响。但也可以看出Term Index使用字典树的索引方式对前缀匹配有先天优势的。那我们是否可以通过一种特殊的分词方式，将包含匹配转换为前缀匹配来提升整体的搜索效率呢？

基于上述分析本文提出一种名为rockStone的分词方式，该方式从左至右依次移除索引词的1个字符后作为1个新的词元来构建Term Index。如new_offline_attribution_content_func会被分词为：[‘new_offline_attribution_content_func’, 'ew_offline_attribution_content_func', 'w_offline_attribution_content_func', '_offline_attribution_content_func', 'offline_attribution_content_func', .... 'func', 'unc', 'nc',  'c']。当包含搜索'offline_attribution_content'会被转为前缀搜索'offline_attribution_content*'，从而命中词元'offline_attribution_content_func'，继而达到查询到new_offline_attribution_content_func的效果。该方案一个索引词会分成词长度个词元，索引膨胀率不会太高。使用前缀匹配搜Term Index的时间复杂度是O(n)，匹配结果是多个skipLink数组，而最终结果就是这些数组的并集，整体搜素效率也不会太差。同样以分词'abcd'查询bc为例，新方案构建的倒排索引结构如下图所示。

![img](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/es-optimize-like-filter-06.png)


# 实验对比

上一小节我们定性的分析了新设计的分词方式配合前缀搜索的搜索效率及索引占用空间会明显由于其他三种索引。本小节我们将设计实验来验证我们的猜想。

本实验创建4个索引，名称为idx_wildcard、idx_query_string、idx_term、idx_prefix分别表示第一方案、第二方案、第三方案及新方案。每个索引都有300万个文档，每个文档仅有1个属性uid，该属性保存的是长度为16位由数字及小写字母组成的随机字符串。在各个索引没有副本的情况下，其占用空间如下表所示。

![img](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/es-optimize-like-filter-07.png)

本次实验使用MacBook Pro借助docker启动v6.6.7版本elasticsearch，docker容器内存上限为1G，es进程Xms与Xmx皆为512m。实验对比了4个方案随着查询字符串的变长，单次查询耗时的变化。其中单次耗时是查询100次取平均值，查询字符串同样是随机生成。实验结果入下图所示。

![img](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/es-optimize-like-filter-08.png)


从测试结果来看目前使用的wildcard查询效率最低，单次查询在约900ms，第二种方案耗时约150ms左右，由于需要计算连续匹配，该方案随着查询词变长耗时逐渐增加。第三种方案耗使用es效率最高的term查询，耗时约在30ms左右，但由于分出的词元实在过于庞大，耗时依然高于新方案。本次提出的新方案在查询词较短的情况下与第三种方案耗时基本持平，当查询词稍长后耗时小于第三个方案，平均耗时在20ms左右。总体而言可以产出新方案的查询效率是明显优于其他三种方案的，耗时仅为当前数据地图使用方案的2%。

最后需要注意的是查询词和索引词都使用随机字符串，当查询词长度较小的时候是可以文档搜索到结果的，而长度多大基本是必定搜索不到结果的，该因素可能对查询耗时有所影响。同时新方案主要使用的是前缀搜索，如果搜索关键词仅为1-2个字符，那么会命中大量文档从而失去搜索意义，所以在使用该方案时建议自动略过短字符的情况。

# 总结

本文从elasticsearch索引方式出发，针对字符串包含查询提出一种新的一套分词及查询方式，并于常规3种实现进行实验对比，最后得出新方案在索引膨胀可控的情况下可极大的提升搜索效率，并且查询词越长搜索效率越高。有类似需求的小伙伴可尝试使用。最后欢迎大家关注使用数据地图，我们凭借丰富的元数据及血缘信息，使用全文检索和图查询技术，致力于提高用户找数效率，提高数据协作开发时的体验。最后，欢迎大家关注快手大数据平台。
