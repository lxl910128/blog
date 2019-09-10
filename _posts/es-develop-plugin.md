---
title: elasticsearch解析器插件开发介绍
categories:
- elasticsearch
tags:
- plugin
---

# 概要
本文主要介绍如何开发ES解析器插件。开发的解析插件的核心功能是可以将词"abcd"分为\["abcd","bcd","cd","d"\]。此次本作者尝试以官方文档、源码及注释为主要学习材料，而不是学习他人总结的博客。使用此模式可以对开发插件的各步骤有更详细的理解，建议大家可以这样尝试一下。  

在开始前还需要说明的是，根据高人指点，上篇[文章](http://blog.gaiaproject.club/es-contains-search/)中提四种分词方案即将"abcd"分词为\["abcd","bcd","cd","d"\]，其实使用ES自带的`edge_ngram` tokenizer加`reverse` token filter就可实现相似的功能。即将"abcd"分词为\["a","ba","cba","dcba"\]，搜索时借助`prefix`查询就可以完成字符串包涵的搜索需求。这里还有一个需要注意的点是，使用`prefix`查询时需要自行将查询词逆序。当然也可以使用`match_phrase_prefix`使用`analyzer`配置只带`reverse` token filter的解析器对查询词进行反转。使用`match_phrase_prefix`还有个好出会自行根据出现频率排序。

# 预备知识点
1. 在es中一串字符串如果要被索引，需要经过对应的解析器`Analyzers`将其转化出`terms`和`tokens`。
2. `term`是搜索的最小单元
3. `token`是一种在对文本进行分词时产生的对象，它包括term，term在文本中的位置，term长度等信息。
4. es解析器在分词时有3个步骤。`character filters`、`tokenizer`和`token filters`
5. `character filters`将原始文本作为字符流接收，可以对字符流进行增删改，最后将新的流输出。比如无差别将大写字母变小写并删除符号：'HELLO WORD! LOL' -> 'hello word lol'
6. `tokenizer`接收`character filters`转化后的字符流然后将它切分成词输出`token`流。对于英文一般采用空格切分'hello word' -> \['hello','word','lol'\]
7. `token filter`接收`token`流，并增删改tokens。\['hello','word','lol'\] -> \['hello','word','smile'\]

举个例子，使用standard Analyzer对"The 2 QUICK Brown-Foxes jumped over the lazy dog's bone."进行解析时，首先会用Standard Tokenizer切分句子(该analyzer默认没有Character Filters)，

# 插件介绍
ES插件主要是用来自定义增强ES核心功能的。主要可以扩展的功能包括：
* `ScriptPlugin`脚本插件，ES默认的脚本语言是Painless，可自定义其它脚本语言，java、js等。
* `AnalysisPlugin`解析器插件，可以扩展character filters、tokenizer和token filter。
* `DiscoveryPlugin`发现插件，使集群可以发现节点，如使建立在AWS上的集群可以发现节点。
* `ClusterPlugin`集群插件，增强对集群的管理，如控制分片位置。
* `IngestPlugin`摄取插件，增强节点的ingest功能，例如可以在ingest中通过tika解析ppt、pdf内容。
* `MapperPlugin`映射插件，可添加新的字段类型。
* `SearchPlugin`搜索插件，扩展搜索功能，如添加新的搜索类型，高亮类型等。
* `RepositoryPlugin`存储库插件，可添加新的存储库，如S3，Hadoop HDFS等。
* `ActionPlugin`API扩展插件，可扩展Http的API接口。
* `NetworkPlugin`网络插件，扩展ES底层的网络传输功能。
* `PersistentTaskPlugin`持续任务插件，用于注册持续任务的执行器。
* `EnginePlugin`实体插件，创建index时，每个enginePlugin会被运行，引擎插件可以检查索引设置以确定是否为给定索引提供engine factory。

