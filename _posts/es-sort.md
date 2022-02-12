---
title: elasticsearch 搜索排序
date: 2022/02/12 17:26:10
categories:
- elasticsearch

tags:
- elasticsearch
- sort

keywords:
- elasticsearch, es, sort, 搜索排序
---

# 概述

搜索结果排序是搜索引擎必备的功能，也是能直接影响用户搜索体验的重要功能。elasticsearch作为一个优秀的分布式搜索引擎，不仅可以针对属性做字典序或数值排序，还可以根据 **实用评分函数（practical scoring function）**计算相似度并排序。本文主要分享elasticsearch提供的搜索排序能力，具体将从以下四点进行介绍：

1. 首先解释关键概念——搜索和过滤，区分它们的差异。
2. 介绍Es排序API，如何通过查询请求中sort字段控制es排序。
3. 介绍es在计算相似度评分以及如何控制相关性。
4. 总结全文并介绍一些本人在搜索排序中的心得体会。

<!--more-->

# 搜索和过滤

在聊elasticsearch的排序前，需要先区分一组重要的概念——**搜索**和**过滤**。

在过滤的语境下，用户明确知道数据具备哪些特征，知道如何使用条件语句准确筛选想要的数据。一般sql中where语句实现的都是过滤，比如过滤出所有的男性、过滤出年龄在19-24的人等。数据在过滤语境中只有匹配或不匹配两种情况。

在搜索的语境下，查询需求往往不能简化True/False的逻辑判断，甚至你都不知道文档有哪些属性，一般使用搜索引擎（baidu、google）都是搜索需求。比如：为什么天是蓝的？这时候搜索引擎会找出与搜索内容相似度最高的N条数据，这N条数据可能是用户需要的也可能不是。

对于过滤结果的排序，一般是对某字段按字母序或数字序排序，比较简单。但是对搜索结果的排序我们通常是根据复杂的算法得出的相似度得分排序，经典相似度算法有TF/IDF，与检索词频和反向文档频率相关。

在研发过程中务必要分清楚需求是过滤还是搜索，如果是过滤使用正确的DSL语句（bool中的filter）es会帮我们进行缓存，加速搜索。

# ES排序API

我们都知道在SQL可以使用order by语句对搜索结果进行排序，而es的排序排序关键字是sort。与SQL语句order by相同，es的排序的基本思路也是对1个或多个字段进行字母序或数值序排序，不过它比SQL多了些配置项。sort数组的值可以是字符串可以是一个对象，具体结构如下：

```json
GET _search  // 1. 搜索app是kuaishou的文档并按price、月份、相似度得分排序
{
  "query": {
    "match": {
      "app": "kuaishou"  
    }
  },
  "sort": [
    {  			// 2. 第一个排序依据，表示按价格升序
      "price": {      
        "order": "asc", // 3. 排序策略，asc正序，desc倒叙
        "mode": "avg",  // 4. 多值时取平局值,支持：min|max|sum|avg|median
        "missing": "_last",  //5. 如果字段为null排最后,`_first`表示排最前
        "unmapped_type": "long" // 6. 指定字段类型，避免字段为null时出错
      }
    },
    {       // 7.  第二个排序依据，使用脚本计算得出
      "_script": {
        "type": "number",   // 8. 脚本返回的数据类型
        "script": {         
          "lang": "painless", // 使用painless语法
          // 脚本正文
          "source": "doc['time'].value.getMonthValue()*params.factor",
          "params": {
            "factor": 10000
          }
        },
        "order": "asc"
      }
    },
    "_score"    // 10  第三个排序依据，使用相似度得分_score
  ]
}
```

elasticsearch除了对常规属性排序外，还可以对内嵌对象（nested）和地理位置GEO类型的属性进行排序。	elasticsearch对内嵌对象中的属性进行排序的示例如下：

```json
POST /_search // 1.搜索男性，按照大学平均分降序
{
   "query" : {
      "term" : { "sex" : "man" }
   },
   "sort" : [
       {	// 2.用内嵌对象school中的得分字段排序，注意字段名是全路径
          "school.score" : { 
             "mode" :  "avg",
             "order" : "desc",
             "nested": { //3. 用内嵌对象的属性必须新增该 对象
                "path": "offer", // 4. 父级属性名，也需要是全路径
                "filter": { // 5. 可指定一个过滤筛选
                   "term" : { "school.type" : "tertiary" }
                }
             }
          }
       }
    ]
}
```

elasticsearch同样支持对地理位置类型(geo)的属性排序。基本逻辑就是按照到指定坐标的距离作为排序依据。

```json
GET _search  // 1. 按照到lat=32,lon=-94由近到远排序
{
  "query": {
    "match": {
      "message": "chrome"
    }
  },
  "sort": [
    {
      "_geo_distance": {
        "pin.location": {  // 源点坐标
          "lat": 32,
          "lon": -97
        },
        "distance_type":"arc",  //计算距离方式:arc(默认)\plane(快但不准)
        "ignore_unmapped": true , // 可有接受没有pin.location属性
        "order": "asc", // 正序
        "unit": "m" // 计算距离的单位是米
      }
    }
  ]
}
```

如果不配置sort，es会用相似度得分_score进行排序，相似度表征了该文档与搜索条件的相关程度，是特定算法计算生成的。如果配置了sort就不会再计算相似度了。但有些情况相似度会是我们计算排序的依据之一，这时就需要在searchRequestBody中与sort平级的位置配置"track_scores":true。提供相似度排序是ES有别与其他数据库实现的一大亮点，下面一小节我们将着重介绍相似度的计算。

# **Es控制相关性**

控制相关性是指调整命中文档相似度得分的计算方式，使结果更符合搜索预期。在搜索语境下是十分重要的话题。个人认为，es搜索的本质分为2步：

- 第一步，使用布尔模型（BIR）来查找匹配的文档，布尔模型组织的条件本质都是看属性是否含有某词(term)。
- 第二步，根据所有命中词计算每个文档的相似度得分，并根据评分排序返回。

​	在上述过程中，控制相关性的手段主要有：

1. 单个命中词的相似度得分公式，搜索前配置。
2. 多个得分如何生成这个文档的最终得分，搜索时用查询语句控制。

​	本小节将从这两个方面来介绍如何控制结果相关性。

## **相似度得分公式**

在搜索语境下复杂搜索都可以划分成多个基础搜索的组合，基础搜索都是看1个词在文档某属性内是否命中。当命中时，就根据该属性评分公式计算这个子查询的评分。属性的评分公式可以在创建mapping时设置。es提供了BM25、classic和bool3种计算方式，默认是BM25，当然也可以自定义。配置方式如下。

```json
PUT my_index
{
  "mappings": {
    "_doc": {
      "properties": {
        "default_field": {
          "type": "text"    // 1 默认BM25相似度算法
        },
        "classic_sim_field": {
          "type": "text",   // 2 TF/IDF相似度算法
          "similarity": "classic"
        },
        "boolean_sim_field": {
          "type": "text",   // 3 bool相似度算法
          "similarity": "boolean"
        },
         "custom_sim_field": {
          "type": "text",   // 4 自定义相似度算法
          "similarity": "my_similarity"
        }
      }
    }
  },
  "settings": {
    "index": {
      "similarity": {        // 5 自定义相速度算法配置
        "my_similarity": {
          "type": "DFR",     // 相似度计算模型
          "basic_model": "g",
          "after_effect": "l",
          "normalization": "h2",
          "normalization.h2.c": "3.0"
        }
      }
    }
  }
}
```

其中BM25和classic的计算公式大体都是：

![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/es-score.png)


- **idf**：逆向文档频率，表示词在整个文档集合里出现的频率，频次越高，得分**越低**。
- **tf**：词频，表示词在文档属性中出现的频度是多少，频度越高，得分**越高** 。
- **norm**：命中属性的长度归一值，词（term）长度约越短，得分**越高** 。
- **boost**：提权，查询时人为指定的提升权重。

BM25和classic的主要不同就是idf，tf，norm的计算方式的不同，BM25的具体如下：

- idf：
  
  ![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/es-bm25-idf.png)
  
  
  - docCount：文本总数
  - docFreq：包含该词的文档数
  
- tfnorm：
  
  ![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/es-bm25-tfnorm.png)
  
  
  - BM25中norm和tf在一个公式中
  - k1：可配置的参数，控制着词频对tf上升速度的影响。默认值为 1.2 。值越小词频增加对tf的影响越快，值越大变化越慢。
  - b：可配置参数，控制着字段长归一值所起的作用， 0.0 会禁用归一化， 1.0 会启用完全归一化。默认值为 0.75 。
  - freq：词频，查询词在该字段出现的次数
  - fieldLength：是满足查询条件的doc的filed的term个数
  - avgFieldLength：是满足查询条件的所有doc的filed的平均term个数

classic(TF/IDF)相关参数的参数的计算方式：

- idf：﻿
  
  ![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/es-classic-idf.png)
  
- tf：
  
  ![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/es-classic-tf.png)
  
- norm：
  
  ![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/es-classic-norm.png)
  
  

BM25作为是classic的升级版。classic默认高频词，如：the、and、这、那、的等停用词在建立索引时已被剔除，就没用考虑词频上线问题。具体而言就是classic会使高频词的得分非常高。究其原因是由于tf是关于出现次数的线性函数，如果这个词出现10次那么根据公式就会乘10。而BM25使词频为5比词频为1的得分有更显著提升，但是词频是20次与词频是1000的得分几乎相同。为什么BM25会有如此效果？主要由于BM25的tfNorm的导数为减函数且无限趋近于0。BM25和classic因词频对得分的影响如下图所示：

![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/es-sort-bm25.jpg)

除了默认的BM25和老版本默认的classic(TF/IDF)，es还可配置为bool类型，这种类型命中就返回1没命中就返回0，具体得分根据查询时boost设置。

如果默认提供的3种方式都不能满足业务需求，还可以自定义得分公式。根据上文的展示，首先要选取一个计算模型，用type配置，ES提供的计算模型除了BM25、classic还有DFR(divergence from randomness)、DFI(divergence from independence)、IB(Information based model )、LMDirichlet(LM Dirichlet similarity)、LMJelinekMercer(LM Jelinek Mercer similarity)。每种计算模型都有各自可以细化的参数配置，比如上文示例中DFR就有参数basic_model、after_effect、normalization、normalization.h2.c，而我们使用BM25时则可以细化配置k1和b，具体模型说明请参见[文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.6/index-modules-similarity.html)

如果这些计算模型还是不能满足需求，那么可将type设为scripted用脚本自行设计相似度公式。在该脚本的上下文中，es为用户提供了诸如词频（doc.freq）、属性term个数（doc.length）等常用数据可直接使用，具体上下文环境参见[文档](https://www.elastic.co/guide/en/elasticsearch/painless/6.6/painless-similarity-context.html)。如下配置就是classic用脚本的实现：

```json
PUT /index
{
  "settings": {
    "number_of_shards": 1,
    "similarity": {
      "scripted_tfidf": {
        "type": "scripted",
        "script": {  //   同 classic
          "source": "
  double tf = Math.sqrt(doc.freq); 
  double idf = Math.log((field.docCount+1.0)/(term.docFreq+1.0)) + 1.0; 
  double norm = 1/Math.sqrt(doc.length); 
  return query.boost * tf * idf * norm;"
        }
      }
    }
  },
  "mappings": {
    "_doc": {
      "properties": {
        "field": {
          "type": "text",
          "similarity": "scripted_tfidf"
        }
      }
    }
  }
}
```

## **查询语句影响相关性**

对属性相似度公式的设置只是影响评分的基础，更多评分的调整一般通过查询语句实现。查询语句对评分的影响最值接的方式是子查询设置boost。在上一小节我们提到boost会与默认得分相乘。这个参数我们在match、query string、term等查询中都能设置，直接影响相似度结果。我们不光可以在查询时设置boost，还可以在创建mapping时就为1个属性指定boost，但十分不建议这么做，boost最好只在查询时设置。

之前我们讲的都是1个词在1个属性中命中后的相似度计算方式，正常业务中会用bool组织多个查询，那么这个文档的最终得分是如何通过多个得分计算出的来的？答案是，bool查询遵循越多命中越好原则，文档最终得分是should和must中每个子句的得分累加。注意，filter和must_not属于过滤查询不贡献得分。

显然bool查询这种粗暴的累加方式并不灵活。比如我们在复杂查询时文档最终得分我们希望是子查询中得分最高的那个，此时我们就可以使用DisjunctionMaxQuery(dis_max)查询，使文档最终得分等于子查询最高得分。当然一刀切也是不合适的，我们可以通过tie_breaker=[0,1]来设置其余非max得分语句的权重。

也有的时候我们的评分根本用不到那么复杂的计算模型，当子句命中就用默认得分即可，这种情况下可以使用constant_score查询，该查询只用设置过滤查询和匹配后得分——boost即可，由于不计算相似度评分可提升搜索效率。

还有需求会希望查询子句提供负相关性，而非像must_not一样直接过滤掉，此时可使用boosting查询，让某些条件体统负相关性。

如果以上这些还不能满足需求，则可以选用自由度最高的function_score查询。它允许对每个匹配的文档应用1个或多个加强函数修改原始评分。只有一个增强函数时，查询语句大致形态如下所示：

```json
{
    "query": {
        "function_score": {
            "query": { "match_all": {} }, // 主查询
            "random_score": {},   // 评分增强函数
            "boost_mode":"multiply"  // 其他辅助配置
        }
    }
}
```

当需要多个增强函数共同作用时，其形态大致如下：

```json
{
    "query": {
        "function_score": {
          "query": { "match_all": {} },  // 主查询
          "functions": [  // 多个评分增强函数
              {
                  // 此评分增强函数作用的范围
                  "filter": { "match": { "test": "bar" } },  
                  "random_score": {},   // 评分增强函数1
                  "weight": 23					// 评分增强函数2
              },
              {
                  "filter": { "match": { "test": "cat" } },	
                  "weight": 42					// 评分增强函数1
              }
          ],
          "max_boost": 42,    				  // 一些列其他辅助配置
          "score_mode": "max",
          "boost_mode": "multiply",
          "min_score" : 42
        }
    }
}
```

首先，辅助配置有如下3个：

- **boost_mode**:决定old_score和加强score如何合并，可配置：
  - multiply：new_score = old_score * 加强score，默认
  - sum：new_score = old_score + 加强score
  - min：new_score= min(old_score, 加强score)
  - max：new_score= max(old_score, 加强score)
  - avg:  new_score= (old_score+加强score)/2
  - replace：new_score = 加强score，直接替换old_score
- **score_mode**：决定functions里面的加强score怎么合并（多个增强函数的filter命中了同1个文档），会先合并加强score成一个总加强score，再使用总加强score区和old_score做合并，换言之就是**先执行score_mode，再执行boost_mode**。其可配置项有：multiply、sum、avg、first、max、min。
- **max_boost**：限制加强函数的最大效果，就是限制加强score最大能多少，但是不会限制old_score。如果加强score超过了max_boost的限制值，会把加强score的值设置成max_boost。

增强函数官方提供了以下5种：

- **weight**：为每个文档应用一个简单而不被规范化的权重提升值：当 weight 为 2 时，最终结果为 2 * _score 。
- **field_value_factor**：使用1个属性值来修改 _score ，如将属性 votes （点赞数）作为评分因素。
- **random_score**：为每个用户都使用一个不同的随机评分对结果排序，但对某一具体用户来说，看到的顺序始终是一致的。
- **衰减函数——** linear **、** exp **、** gauss。将1个浮动值结合到评分 _score 中，例如结合 publish_date 获得最近发布的文档，结合 geo_location 获得更接近某个具体经纬度（lat/lon）地点的文档，结合 price 获得更接近某个特定价格的文档。
- **script_score**：如果需求超出以上范围时，用自定义脚本完全控制评分计算，实现所需逻辑。

下面通过3个实例介绍这部分内容。第一个例子是文章点赞数对搜索结果的影响。设想有个网站供用户发布博客并且可以让他们为自己喜欢的博客点赞，我们希望将更受欢迎的博客放在搜索结果列表中相对较上的位置，同时全文搜索的评分仍然作为相关度的主要排序依据，可以简单的通过存储每个博客的点赞数来实现它。

```json
GET /blogposts/post/_search
{
  "query": {
    "function_score": {
      "query": {  // 主查询命中 title和content包含 popularity的文章
        "multi_match": {
          "query":    "popularity",
          "fields": [ "title", "content" ]
        }
      },
      "field_value_factor": {  
        "field":    "votes",  // 属性votes参与评分计算
        "modifier": "log1p",  // votes的修饰方式
        "factor":   0.1   // 影响因子
      },
      "boost_mode": "sum",  // 与老评分的组合方式
      "max_boost":  1.5   // 增强分最大为1.5
    }
  }
}
```

这个例子中我们用votes属性修改最终得分，使用log1p函数修饰votes，并设置影响因子为0.1，根据上述查询最终评分计算公式为：

![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/es-last-score.png)

在field_value_factor中，修饰方式可选的有：none, log, log1p, log2p, ln, ln1p, ln2p, square, sqrt, or reciprocal. 默认是 none即不修改。

第二个例子作为民宿网站的所有者，总会希望让所有商品有曝光的机会。在当前查询下，有相同评分 _score 的文档会每次都以相同次序出现，为了提高展现率，在此引入一些随机性就能保证有相同评分的文档都能有均等相似的展现机率。

```json
{
  "query": {
    "function_score": {
      "filter": {
        "term": { "city": "Barcelona" }  // 查询巴塞罗那的民宿
      },
      "functions": [
        { 
          // 有wifi的权重为1（主查询是过滤，old score都是1）
          "filter": { "term": { "features": "wifi" }}, 
          "weight": 1
        },
        {
          // 有院子的权重为1（主查询是过滤，old score都是1）
          "filter": { "term": { "features": "garden" }},
          "weight": 1
        },
        { 
          // 有游泳池的权重为2（主查询是过滤，old score都是1）
          "filter": { "term": { "features": "pool" }},
          "weight": 2
        },
        {  
          // 函数会输出一个 0 到 1 之间的数，sead相同，输出值相同
          "random_score": {   
             // 使用用户ID保证每个用户每次看到的排序是相同的
            "seed":  "{usersId}" 
          }
        }
      ],
      "score_mode": "sum"
    }
  }
}
```

第三个例子假如你还是1个民宿往老板，用户对于民宿的选择，也许用户希望离市中心近点，但如果价格足够便宜，也有可能选择一个更远的住处，也有可能反过来是正确的：愿意为最好的位置付更多的价钱。如果我们添加过滤器排除所有市中心方圆 1 千米以外的民宿，或排除所有每晚价格超过 1000元的，我们可能会将用户愿意考虑妥协的那些选择排除在外。

function_score 查询会提供一组 **衰减函数（decay functions）**，让得分不再是有或没有，而是一个渐变的过程。ES提供了三种衰减函数—— linear(匀速) 、 exp(先快后慢) 和 gauss(先慢后快) （线性、指数和高斯函数），它们可以操作数值、时间以及经纬度地理坐标点这样的字段。所有三个函数都能接受以下参数：

- **origin**：中心点， 或字段可能的最佳值，落在原点 origin 上的文档评分 _score 为满分 1.0 。
- **scale**：衰减率，即一个文档从原点 origin 下落时，评分 _score 改变的速度。（例如，每 100块钱或每 100 米）。
- **decay**：从原点 origin 衰减到 scale 所得的评分 _score ，默认值为 0.5 。
- **offset**：以原点 origin 为中心点，为其设置一个非零的偏移量 offset 覆盖一个范围，而不只是单个原点。在范围 -offset <= origin <= +offset 内的所有评分 _score 都是 1.0 。

这三个函数的唯一区别就是它们衰减曲线的形状，如下图所示：

![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/es-sort-decay.png)

那么用户希望租一个离北京市中心近（ { "lat": 116, "lon": 40} ）且每晚不超过 1000月的度假屋，而且与距离相比，我们的用户对价格更为敏感，这样查询可以写成：

```json
{
  "query": {
    "function_score": {
      "functions": [
        {
          "gauss": {  // 使用高斯函数 先缓慢衰减后快速衰减 
            "location": { 
              "origin": { "lat": 116, "lon": 40 }, // 北京位置
              "offset": "2km", // 2公里内不衰减
              "scale":  "3km" // 衰减率  3公里 得分减半
            }
          }
        },
        {
          "gauss": {
            "price": {    // 价格
              "origin": "500",  // 中心值
              "offset": "500", //  300-700 评分不衰减
              "scale":  "200"  //  衰减率200
            }
          },
          "weight": 2 // 价格权重是位置得分的2倍
        }
      ],
     "score_mode": "avg",  // 增强得分为价格和位置得分平均值
     "boost_mode": "multiply", // 最后得分是增强分与原始评分相乘
    }
  }
}
```

# **心得体会**

elasticsearch以非常优秀的横向扩展能和丰富的查询支持赢得了许多人的喜爱，以至于大家会把ES当MySQL来使用。像使用关系型数据库一样，属性的值通常是1个词时，此时使用BM25计算相似度评分时，词频只会是1，属性的长度(fieldLength)或平均长度(avgFieldLength)也大概率都是1，词频(idf)也会因为属性是枚举值而没有区分度。相似度评分的最大影响因素就是文档命中查询词的次数，虽然这种方式能解决80%的问题。说到底，由于相似度计算的方式，我认为es的属性应该存的是较长的文本，1句话或1篇文章，这样才能发挥相似度查询的优势。所以较好的处理方式是将所有可能被查属性的值拼成1个字符串数组，放在1个属性里，人为增加属性长度。同时要着重理解需求是过滤还是搜索。

在实践中，简单的查询组合就能提供很好的搜索结果，但是为了获得**具有成效**的搜索结果，就必须反复推敲修改相似度评分计算方式。可以通过在请求体中增加"explain": true来要求es返回每个命中文档的得分计算公式，从而帮助我们深入理解为什么这个文档会被排在了前面。通常对策略字段应用权重提升就足以获得良好的结果，但更为正确的做法是要**监控搜索结果**。比如，监控用户点击TOP N结果的频次；用户不查看首次搜索的结果而直接执行第二次查询的频次；用户来回点击并查看搜索结果的频次等等，诸如此类的信息。这些都是用来评价搜索结果与用户之间相关程度的指标。一旦有了这些监控手段，想要调试查询就并不复杂。本文介绍的只是工具，要想物尽其用并将搜索结果提高到**极高的**水平，唯一途径就是需要具备能评价度量用户行为的强大能力。

# **总结**

本文介绍了es为排序提供的相关能力。首先，本文解释了搜索和过滤的区别，我们在使用关系型数据库中所有搜索都是过滤，而es主要提供的能力是搜索，在搜索语境下我们排序主要依照文档与搜索关键字的相似度。然后，介绍了我们可以通过在请求Body中增加sort对象来控制返回结果按照什么排序，默认是根据相似度评分降序。接着，本文介绍了如何控制相似度评分，主要有2个维度，1、搜索前对属性评分公式的设置，一般使用BM25；2、搜索时用DSL语句对命中属性后得到的多个评分进行加工得出最终得分。需要着重了解2点：最常用的bool查询是简单的将多个得分累加得出文档最终得分；搜索时调整得分的最终武器是Funciton Score查询。

# **参考文献**

[function-score查询](https://www.elastic.co/guide/en/elasticsearch/reference/6.6/query-dsl-function-score-query.html)

[search-request-sort](https://www.elastic.co/guide/en/elasticsearch/reference/6.6/search-request-sort.html)

[ES控制相关性](https://www.elastic.co/guide/cn/elasticsearch/guide/current/controlling-relevance.html)