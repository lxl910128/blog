---
title: elasticsearch-Search template 模板搜索
date: 2021/05/31 18:26:10
categories:
- elasticsearch

tags:
- elasticsearch
- search

keywords:
- elasticsearch,  search template, 模板搜索
---

# 概述

Elasticsearch 允许使用模板语言 mustache 来预设搜索逻辑，在实际搜索时，通过参数中的键值，对来替换模板中的占位符，最终完成搜索。该方式将搜索逻辑封闭在 Elasticsearch 中，可以使下游服务，在不知道具体搜索逻辑的情况下完成数据检索。我们以 Kibana 自带的航班数据`kibana_sample_data_flights`为基础，以按航班号搜索为例，简单介绍搜索模板的使用。

第一步，创建 ID 为 testSearchTemplate 的搜索模板，语句如下

```json
POST _scripts/testSearchTemplate
{ 
  "script": {
    "lang": "mustache",   #使用 mustache 模板语言
    "source": {   # 脚本内容
      "query": {    # 搜索逻辑
        "term": {
          "FlightNum": {
            "value": "{{FlightNum}}"  # 占位符 FlightNum
          }
        }
      }
    }
  }
}
```

第二步，传参搜索数据，语句如下

```json
GET kibana_sample_data_flights/_search/template
{
  "id": "testSearchTemplate",     # 使用的模板ID
  "params": {
    "FlightNum": "9HY9SWR"    # 占位符替换的值
  }
}
```

以上两步就是使用模板搜索数据，该逻辑等同于下面这个搜索

```json
GET kibana_sample_data_flights/_search
{
  "query": {
    "term": {
      "FlightNum": {
        "value": "9HY9SWR"
      }
    }
  }
}
```

<!--more-->

# API介绍

下面我们从搜索模板的生命周期：创建、查看、使用、删除来展开介绍模板搜索相关 API。

## 准备

在正式介绍之前，我们先来说一说关于模板搜索的几个预备知识。

首先，如果使用的 Elasticsearch 集群开启了安全功能，那么角色对操作的索引必须要有`read`权限。

其次，搜索模板使用的语法是`Mustache`

> 更多的关于该种脚本语言的介绍以及功能请查看其官方[文档](https://mustache.github.io/mustache.5.html)

最后，模板搜索属于 Elasticsearch 中 Script 功能的扩展， Script 的限定及用法基本都适用于模板搜索。比如，集群关于 Script 的配置也会影响模板搜索，配置项`script.allowed_types`可规范模板搜索接受的类型（ inline / stored / both ），`script.allowed_contexts`也会限制模板搜索可进行的操作。

## 创建

搜索模板的创建与 Elasticsearch 其它脚本的创建一样，都是发送 1 个`POST`请求即可。

如下所示：

```json
POST _scripts/<templateId>  # 1
{
  "script": {
    "lang": "mustache", # 2
    "source": {   # 3
      "query": {
          "term": {
            "FlightNum": {
              "value": "{{FlightNum}}"  # 可变参数 FlightNum
            }
          }
        }
    }
  }
}
```

1. 向`_scripts/<templateId>`发送 POST 请求来创建搜索模板，其中`<templateId>`是你为该模板设置的 ID，搜索时会用到该 ID

1. lang 参数配置的是搜索模板使用的脚本语言为`mustache`

1. source 参数配置的是搜索模板的具体内容，该部分的格式参照 Elasticsearch 搜索的请求 body，需要搜索时填充的值使用`mustache`语法，配置占位符即可，比如本例中的占位符就是`{{FlightNum}}`

## 查看

当我们想查看之前创建的模板内容，或者验证某个 ID 的模板是否存在时，可以向`_scripts/<templateId>`发送 GET 请求来获取模板的具体内容。

示例如下：

```json
GET _scripts/<templateId>  # 1
{
  "_id" : "testSearchTemplate",  # 2
  "found" : true,  # 3
  "script" : {   # 4
    "lang" : "mustache", 
    "source" : """{"query":{"term":{"FlightNum":{"value":"{{FlightNum}}"}}}}""", 
    "options" : {  # 5
      "content_type" : "application/json; charset=UTF-8"
    }
  }
}
```

1. 请求的 path 为`_scripts/<templateId>`，其中`<templateId>`为你要查询的模板 Id，请求类型为 GET

1. 返回的 body 中，_id 属性再次表明此次查询的模板 ID，本示例查询的是之前创建的`testSearchTemplate`模板

1. found 属性表明此次查询是否查到结果，如果模板 ID 存在则此值为 true，反之为 false

1. script 就是该搜索模板的具体内容与保存时相同。核心有 lang 属性表示脚本语法，source 属性存放脚本具体内容

1. script 属性中的 Options 属性是非必要其它脚本属性，默认会有 content_type 属性，该属性保存查询时 http 请求的`content-type` ，默认为`application/json; charset=UTF-8`

## 删除

在一个搜索模板完成了它的使命后，我们需要及时删除它，因为 Elasticsearch 默认缓存脚本的数据量是有上限的，删除的方式很简单，发送一个`DELETE`请求即可。

示例如下：

```json
DELETE _scripts/<templateId>   #1
```

`<templateId>`为要删除的搜索模板的 ID，比如`_scripts/testSearchTemplate` 表示的就是删除 ID 为`testSearchTemplate`的搜索模板。

## 使用

搜索模板的使用就是在搜索时，直接发送占位符的值，即可执行预设搜索语句。由于还是在搜索的范畴，所以发送请求的 path 是`_search/template`。

下面是关于使用搜索模板进行查询的示例：

```json
GET <index>/_search/template?<query_parameters> #1
{
  "source": """{"query": {"term": {"FlightNum": {"value": "{{FlightNum}}"}}}}""",  #2
  "id": "testSearchTemplate", # 3
  "params": {  # 4
    "FlightNum": "9HY9SWR"
  },
  "profile": true, # 5
  "explain": true  # 6
}
```

模板搜索发送的地址为`<index>/_search/template`，与搜索一样`<index>`处为选填参数，你可以指定搜索的索引，不指定则表示搜索全部索引。

因为本质上还是属于搜索的范畴，所以一些搜索参数在模板搜索是也可以使用，比如：

- scroll（可选，时长）：表示本搜索需要支持游标搜索，游标过期时间为配置值

- ccs_minimize_roundtrips(可选，布尔值)：如果为 true 则在跨集群搜索时最小化集群间交互。默认为 true

- expand_wildcards（可选，字符串）：表示索引通配符作用的范围，可配置为全部（all）、打开索引（open）、关闭索引（closed）、隐藏索引（hidden，需要与open或closed结合使用）、不允许通配符（none）

- explain（可选，布尔值）：表示返回结果是否带计算得分的详细信息，默认是false

- ignore_throttled（可选，布尔值）：如果为 true 则表示查询忽略被限制的索引，被限制的索引一般指被冻结（freeze）的索引，该值默认是 true

- ignore_unavailable（可选，布尔值）：如果为 true 则表示关闭的索引不在搜索范围内，默认值为 true

- preference（可选，字符串）：指定执行该操作的节点或分片，默认是随机的

- rest_total_hits_as_int（可选，布尔值）：如果为 true 则 hits.total 将会是个数值而非一个对象，默认为 false

- routing(可选, 字符串)：配置搜索执行的路由

- search_type（可选，字符串）：这是搜索的类型，可选值有：query_then_fetch、dfs_query_then_fetch

- source 字段：用于配置搜索模板，该字段与 ID 字段冲突只能二选一，使用 source 表示不使用保存的模板而使用本模板

- id 字段：表示本次查询使用的搜索模板 ID，该字段与 source 字 段冲突只能二选一

- params 字段：配置的 key-value 值将替换模板中的占位符执行搜索

- profile 字段：是可选字段，表示返回结果中是否有 Elasticsearch 执行搜索的一些元信息

- explain 字段：是可选字段，与 http 中搜索参数配置的 explain 含义一样，表示结果是否带计算得分的详细信息

上述搜索返回结果如下：

```json
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 9.071844,
    "hits" : [
      {
        "_shard" : "[kibana_sample_data_flights][0]",
        "_node" : "ydZx8i8HQBe69T4vbYm30g",
        "_index" : "kibana_sample_data_flights",
        "_type" : "_doc",
        "_id" : "KPRFDHkB9LctWlE3WLqj",
        "_score" : 9.071844,
        "_source" : {
          "FlightNum" : "9HY9SWR",
          "DestCountry" : "AU",
          "OriginWeather" : "Sunny"
        },
        "_explanation" : {}  # 计算得分的逻辑
      }
    ]
  },
  "profile" : {} # 搜索细节信息
}
```



# 其它

本部分将介绍关于模板搜索的一些小技巧。通常情况下我们写的搜索模板，往往是很难一次就配置正确的，因此需要频繁的测试我们写的模板，与参数结合后是否是我们预期的搜索语句，这时我们就可以使用以下这个请求，来校验模板使用是否正确。

```json
GET _render/template # 1
{
  "source": """{"query": {"term": {"FlightNum": {"value": "{{FlightNum}}"}}}}""" ,# 2
  "params": { # 3
    "FlightNum": "9HY9SWR"
  }
}

{  # 4
  "template_output" : {
    "query" : {
      "term" : {
        "FlightNum" : {
          "value" : "9HY9SWR"
        }
      }
    }
  }
}
```

1. 向`_render/template`发送 GET 请求来验证模板是否正确

1. source 字段为要验证的搜索模板，该字段可以省略，如果省略需要在 path 处指定模板iID，比如`_render/template/testSearchTemplate`

1. params 字段为模板使用的参数

1. 此 JSON 就是该请求的返回，`template_output`字段就是在使用此`params`下搜索模板生成的查询语句

模板语言 mustache 有许多功能，这里再介绍几个比较常见的。

比如我们使用占位符替换的不是一个字符串，而是一个对象或数组对象，那么我们可以用`{{#toJson}}{{/toJson}}`来实现，

具体如下：

```json
GET _render/template
{
  "source": """{"query": {"term": {"FlightNum": {{#toJson}}FlightNum{{/toJson}}  }}}""", # 1
  "params": { # 2
    "FlightNum": {
      "value":"9HY9SWR"
    }
  }
}

{ # 3
  "template_output" : {
    "query" : {
      "term" : {
        "FlightNum" : {
          "value" : "9HY9SWR"
        }
      }
    }
  }
}
```

在配置模板时，我们将`FlightNum`的 value 配置为`{{#toJson}}FlightNum{{/toJson}}`，即表示占位符`FlightNum`是一个对象

在配置 params 时，我们将 FlightNum 的值设置为一个 JSON 对象`{ "value":"9HY9SWR"}`

通过校验请求的返回，可以看到`{{#toJson}}FlightNum{{/toJson}}`被替换为对象`{ "value":"9HY9SWR"}`

Mustache 还能在将变量套入模板时做一些处理，比如将数组变量组合成字符串放入模板、设置占位符的默认值，以及对 URL 转码。

示例如下

```json
GET _render/template
{
  "source": {
    "query": {
      "term": {
        "FlightNum": "{{#join delimiter='||'}}FlightNums{{/join delimiter='||'}}", #1
        "DestCountry":"{{DestCountry}}{{^DestCountry}}AU{{/DestCountry}}",#2
        "Dest": "{{#url}}{{Dest}}{{/url}}"#3
      }
    }
  },
  "params": {
    "FlightNums": [
      "9HY9SWR",
      "adf2c1"
    ],
    "Dest":"http://www.baidu.com"
  }
}

{
  "template_output" : {
    "query" : {
      "term" : {
        "FlightNum" : "9HY9SWR||adf2c1", # 4
        "DestCountry" : "AU", #5
        "Dest" : "http%3A%2F%2Fwww.baidu.com" # 6
      }
    }
  }
}
```

第一个模板使用`{{#join delimiter='||'}}{{/join delimiter='||'}}`设置了数组合并的分割字符为 "||"，传参时`FlightNums`配置的为`["9HY9SWR","adf2c1"]`，而生成的则是 #4 处的`9HY9SWR||adf2c1`

第二个模板使用`{{^DestCountry}}AU{{/DestCountry}}`设置了占位符 DestCountry 的默认值为 AU，这样我们在params中并未配置 DestCountry 的值，但生成的 #5 处自动用 AU 替换了占位符

第三个模板我们用`{{#url}}{{/url}}`声明了此处是一个 URL，需要进行转义，则在 #6 处配置的`http://www.baidu.com`变为了`http%3A%2F%2Fwww.baidu.com`

# 鸣谢

本文收录至《Elastic Stack 实战手册》，欢迎和我一起解锁开发者共创书籍，系统学习 Elasticsearch。感谢阿里组织ES百人大战，感谢各位老师及小伙伴的帮助。https://developer.aliyun.com/topic/elasticstack/playbook

![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/elastic-stack-logo.png)