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
3. `token`是一种在对文本进行分词时产生的对象，它包括term，term在文本中的位置，term起止位置等信息。
4. es解析器在分词时有3个步骤。`character filters`、`tokenizer`和`token filters`
5. `character filters`将原始文本作为字符流接收，可以对字符流进行增删改，最后将新的流输出。比如无差别将大写字母变小写并删除符号：'HELLO WORD! LOL' -> 'hello word lol'
6. `tokenizer`接收`character filters`转化后的字符流然后将它切分成词输出`token`流。对于英文一般采用空格切分'hello word' -> \['hello','word','lol'\]
7. `token filter`接收`token`流，并增删改tokens。\['hello','word','lol'\] -> \['hello','word','smile'\]

举个例子，使用standard Analyzer对"The 2 QUICK Brown-Foxes jumped over the lazy dog's bone."进行解析时，首先会用Standard Tokenizer切分句子(该analyzer默认没有Character Filters)，然后使用(Lower Case Token Filter)将所有字符转化为小写。最后会生成如下这些Token。
```json
{
    "tokens": [
        {
            "token": "the",
            "start_offset": 0, //开始位置
            "end_offset": 3,  //结束位置
            "type": "<ALPHANUM>", //词类型
            "position": 0 //偏移
        },
        {
            "token": "2",
            "start_offset": 4,
            "end_offset": 5,
            "type": "<NUM>",
            "position": 1
        },
        {
            "token": "quick",
            "start_offset": 6,
            "end_offset": 11,
            "type": "<ALPHANUM>",
            "position": 2
        },
        {
            "token": "brown",
            "start_offset": 12,
            "end_offset": 17,
            "type": "<ALPHANUM>",
            "position": 3
        },
        {
            "token": "foxes",
            "start_offset": 18,
            "end_offset": 23,
            "type": "<ALPHANUM>",
            "position": 4
        },
        {
            "token": "jumped",
            "start_offset": 24,
            "end_offset": 30,
            "type": "<ALPHANUM>",
            "position": 5
        },
        {
            "token": "over",
            "start_offset": 31,
            "end_offset": 35,
            "type": "<ALPHANUM>",
            "position": 6
        },
        {
            "token": "the",
            "start_offset": 36,
            "end_offset": 39,
            "type": "<ALPHANUM>",
            "position": 7
        },
        {
            "token": "lazy",
            "start_offset": 40,
            "end_offset": 44,
            "type": "<ALPHANUM>",
            "position": 8
        },
        {
            "token": "dog's",
            "start_offset": 45,
            "end_offset": 50,
            "type": "<ALPHANUM>",
            "position": 9
        },
        {
            "token": "bone",
            "start_offset": 51,
            "end_offset": 55,
            "type": "<ALPHANUM>",
            "position": 10
        }
    ]
}
```

这些tokens包括主要包含以下这些Terms: [the, 2, quick, brown, foxes, jumped, over, the, lazy, dog's, bone ]

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

# 插件开发
## 官方教程
[官方教程](https://www.elastic.co/guide/en/elasticsearch/plugins/current/plugin-authors.html)对插件开发介绍的比较少。主要是告诉我们我们开发完成的插件应该以zip包的形式存在。在zip包的根目录种中最起码要包含我们开放的插件jar包以插件配置文件`plugin-descriptor.properties`。es是从配置文件认识自定义插件的。如果插件需要依赖其它jar包，则将其页放在zip根目录下即可。


在插件配置文件`plugin-descriptor.properties`我们至少应该配置以下变量:
* **description**：插件介绍
* **version**：插件版本
* **name**：插件名称
* **classname**：要加载的插件类的全名。这个类需要继承`Plugin`类并实现插件类型接口，比如`ActionPlugin`、`AnalysisPlugin`等
* **java.version**：插件适用的java版本
* **elasticsearch.version**：插件适用的es版本


在插件成功编译成zip包后，我们可以适用`bin/elasticsearch-plugin install file:///path/to/your/plugin`命令来安装插件，或这将zip包直接放在es的`plugins/`目录下。

开发ES插件**总结**来说需要以下几步：
1. 需要编写一个继承`Plugin`实现插件类型的类。
2. 需要编写插件配置文件`plugin-descriptor.properties`。
3. 将这配置文件和jar包打成1个zip包。

## 项目初始化
根据官方教程可知，最终我们需要得到一个还有配置文件和jar包的zip包。这里我们借助`maven`来实现插件的打包工作。
