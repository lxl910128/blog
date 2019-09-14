---
title: elasticsearch中字符串包含查询
categories:
- elasticsearch
tags:
- 查询分析
---

# 概要
本文从业务需求出发，提出elasticsearch中字符串包含查询的业务场景。然后介绍了解决这一问题的4种方法并提出预测结果。这四种方式是在使用不同分词方式的基础上运行term、queryString、Wildcard、prefix查询实现的。接着通过实验验证预测结果。最后分析得出结论，使用prefix查询并结合本人开发的rockstone分词器的搜索效率最高且数据膨胀率可控。

<!--more-->

# 需求
业务需求是客户希望模糊搜索手机号，即搜索出包含这几位数字全部的手机号。简单的说就是希望实现mysql中`like %xxx%`的效果，或者是java中`Stirng.contains('xxx')`的功能。由于我们数据主要存在elasticsearch中，且电话数据量较大，为了最大化elasticsearch的搜索性能，如何正确的分词使用恰当的搜索方式就十分重要了。

# 定性分析
为了解决这个问题，我们先后使用了如下四种方案。  

|name|分词方式(eg.'abc')|查询方式|term个数(长度为n)|
|:-: | :-: | :-: |:--:|
|001号|[a,b,c]|queryString|n|
|002号|[a,ab,abc,b,bc,c]|term|(1+n)n/2|
|003号|[abc]|wildcard|1|
|004号|[abc,bc,c]|prefix|n|

最最开始的时候，我们对手机字段使用标准分词器，但发现标准分词器对成串的数字并做不分词，因此无法查询数字串中的部分数字。所以首先想到的001号方案是将手机切分成1个1个的数字，使用queryString查询。在查询时将要搜索的数字用""包围，规定结果中的查询数字必须是按序相邻的。该方法可以实现业务需求，但是在使用过程中我们发现查询速度很慢，究其原因是查询时需要花费大量CPU来计算词距。即需要计算在哪些文档中搜索数字是按指定顺序相邻的。  


为了解决搜索慢的问题002号方案应允而生。它的思想是穷举手机所有有序的分词组合，然后使用查询效率最高的term查询实现部分匹配查询。简单的说就是比如手机号是"abcd"，我将它分成[abcd,abc,bcd,ab,bc,cd,a,b,c,d]，这样不管abcd中的哪一部分都被做成了索引词，因此可以直接使用term查询来实现部分匹配，且理论上效率是最高的。  


002号方案看起来十分完美，但是有个重要的问题就是，建立索引要花费大量的时间和空间。1个长度为11位的大陆手机号，使用001号方案需要切出11（n）个词建立倒排索引。而002号方案则需要切出66（(1+n)n/2）个词建立索引。那么是否有更经济的方式呢？答案是肯定的。在深入学习了elasticsearch的文档后，发现了通配符查询(Wildcard query)。它是term-level的查询，就是说它是作用于term上的查询。对应于本次需求，就是手机号不用分词，通过Wildcard查询"\*xxx\*"即可实现业务需求。  


正当我高兴于elasticsearch功能能强大时，我突然发现官方文档中的一行小字。
>  Avoid beginning patterns with * or ?. This can increase the iterations needed to find matching terms and slow search performance.

嗯...大神们果然说的不错，算法无非是空间换时间，时间换空间，想即要时间又要空间是不可能的！那么到此就结束了么？当然不。有没有哪种分词方式可以在wildcard的基础上解决不在查询开头使用通配符的问题呢？这样通过牺牲少量空间，获取更高的时间效率。


其实，答案已经呼之欲出了，就是将手机按照出栈的方式切分。即将"abcd"切分成[abcd,bcd,cd,d]，然后使用前缀匹配(prefix)查询，就可以实现业务所需的查询需求了。使用这种分词方式，term的个数与字符串长度相等，同时由于elasticsearch底层(luence)的支持其查询效率也相当不错的。唯一不足的是这种分词方式无法通过配置实现，需要通过开发es插件实现。


# 定量分析
设计创建4个索引，每个索引只有1个field，每个索引的field分别使用上述4种分词方式。然后向每个索引录入300万个长度为16的随机字符串。最后使用相应查询对每个索引连续查询100次比较总耗时。下面主要展示索引创建，分词验证，查询验证以及实验结果。

## 索引创建

001号方案的索引创建语句
```json
{
    "settings": {
        "index": {
            "number_of_shards": 1,
            "number_of_replicas": 0,
            "analysis": {
                "analyzer": {
                    "unigram": {
                        "type": "custom",
                        "tokenizer": "unigram_tokenizer"
                    }
                },
                "tokenizer": {
                    "unigram_tokenizer": {
                        "token_chars": [
                            "letter",
                            "digit",
                            "punctuation",
                            "symbol"
                        ],
                        "min_gram": "1",
                        "type": "ngram",
                        "max_gram": "1"
                    }
                }
            }
        }
    },
    "mappings": {
        "dynamic": false,
        "properties": {
            "uid": {
                "type": "text",
                "analyzer":"unigram"
            }
        }
    }
}
```


002号方案的索引创建语句

```json
{
    "settings": {
        "index": {
            "number_of_shards": 1,
            "number_of_replicas": 0,
            "max_ngram_diff": 32,
            "analysis": {
                "analyzer": {
                    "unigram": {
                        "type": "custom",
                        "tokenizer": "unigram_tokenizer"
                    }
                },
                "tokenizer": {
                    "unigram_tokenizer": {
                        "token_chars": [
                            "letter",
                            "digit",
                            "punctuation",
                            "symbol"
                        ],
                        "min_gram": "1",
                        "type": "ngram",
                        "max_gram": "32"
                    }
                }
            }
        }
    },
    "mappings": {
        "dynamic": false,
        "properties": {
            "uid": {
                "type": "text",
                "analyzer":"unigram"
            }
        }
    }
}
```


003号方案的索引创建语句

```json
{
    "settings": {
        "index": {
            "number_of_shards": 1,
            "number_of_replicas": 0
        }
    },
    "mappings": {
        "dynamic": false,
        "properties": {
            "uid": {
                "type": "keyword"
                
            }
        }
    }
}
```


004号方案的索引创建语句

```json
{
    "settings": {
        "index": {
            "number_of_shards": 1,
            "number_of_replicas": 0
        }
    },
    "mappings": {
        "dynamic": false,
        "properties": {
            "uid": {
                "type": "text",
                "analyzer":"rockstone"
            }
        }
    }
}
```

## 分词验证
本部分主要展示使用`curl -XGET localhost:9200/{index}/_analyze -d {"field":"uid","text":"abcd"}`来检验各个分词器的效果。  


001号方案的分词效果
```json
{
    "tokens": [
        {
            "token": "a",
            "start_offset": 0,
            "end_offset": 1,
            "type": "word",
            "position": 0
        },
        {
            "token": "b",
            "start_offset": 1,
            "end_offset": 2,
            "type": "word",
            "position": 1
        },
        {
            "token": "c",
            "start_offset": 2,
            "end_offset": 3,
            "type": "word",
            "position": 2
        }
    ]
}
``` 

002号方案的分词效果
```json
{
    "tokens": [
        {
            "token": "a",
            "start_offset": 0,
            "end_offset": 1,
            "type": "word",
            "position": 0
        },
        {
            "token": "ab",
            "start_offset": 0,
            "end_offset": 2,
            "type": "word",
            "position": 1
        },
        {
            "token": "abc",
            "start_offset": 0,
            "end_offset": 3,
            "type": "word",
            "position": 2
        },
        {
            "token": "b",
            "start_offset": 1,
            "end_offset": 2,
            "type": "word",
            "position": 3
        },
        {
            "token": "bc",
            "start_offset": 1,
            "end_offset": 3,
            "type": "word",
            "position": 4
        },
        {
            "token": "c",
            "start_offset": 2,
            "end_offset": 3,
            "type": "word",
            "position": 5
        }
    ]
}
```

003号方案的分词效果
```json
{
    "tokens": [
        {
            "token": "abc",
            "start_offset": 0,
            "end_offset": 3,
            "type": "word",
            "position": 0
        }
    ]
}
```

004号方案的分词效果
```json
{
    "tokens": [
        {
            "token": "abc",
            "start_offset": 0,
            "end_offset": 3,
            "type": "word",
            "position": 0
        },
        {
            "token": "bc",
            "start_offset": 1,
            "end_offset": 3,
            "type": "word",
            "position": 1
        },
        {
            "token": "c",
            "start_offset": 2,
            "end_offset": 3,
            "type": "word",
            "position": 2
        }
    ]
}
```
## 查询验证
本部分将使用对应的查询方式查询各个索引，验证查询方式的正确性。

001号方案的queryString查询。
![queryString查询](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/queryStringSearch.png)

002号方案的term查询。
![term查询](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/termSearch.png)

003号方案的Wildcard查询。
![Wildcard查询](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/WildcardSearch.png)

004号方案的prefix查询。
![prefix查询](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/rockstoneSearch.png)

## 实验结果
本章将列出连续查询各个索引100次的耗时对比。每个索引都保存了300万个只含有一个uid属性的文档。uid属性保存的是长度为16位的随机字符串。每次查询的内容是随机字符。在各个索引没有副本的情况下，其占用空间如下图所示。

![索引占用空间](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/diskCost.png)


实验结果：

|序号|查询字符串长度|001号方案|002号方案|003号方案|004号方案|
|:-:|:-:|:-:|:-:|:-:|:-:|
|1|3|12652 ms|11841 ms|13144 ms|3181 ms|
|2|3|12171 ms|9883 ms|13156 ms|3506 ms|
|3|4|12300 ms|16128 ms|13799 ms|2908 ms|
|4|4|12418 ms|13674 ms|11938 ms|2301 ms|
|5|5|14467 ms|14309 ms|17096 ms|5825 ms|
|6|5|13738 ms|13327 ms|14942 ms|4652 ms|

从实验结果上来看term查询并没有比其他查询效率高，反而和queryString、queryString、Wildcard效率差不多。思考原因应该是该分词方式切出的term数太多了。不过如预期使用rockstone分词方式配合prefix查询的效率最好。


# 总结
1. rockstone分词方式配合prefix查询在完成字符串包含查询的效率最好并且数据膨胀率不是很夸张。
2. 算法上时间换空间，空间换时间的基本思路是正确的。
3. 根据具体业务情况设计出更有效的分词方式使用契合的查询方式才能使搜索效率最高。
4. 有机会我将再介绍下如何编写elasticsearch解析插件以及rockstone插件的实现。