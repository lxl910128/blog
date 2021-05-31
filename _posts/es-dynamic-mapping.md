---
title: elasticsearch-dynamic mapping 动态映射
date: 2021/05/31 16:26:10
categories:
- elasticsearch

tags:
- elasticsearch
- mapping

keywords:
- elasticsearch, dynamic mapping, 动态映射
---
# 概述
通常来说，搜索数据一般需要经过 3 个步骤：

- 定义数据（建表建索引）

- 录入数据

- 搜索数据

在现实使用中，定义数据往往是比较繁琐，并且有大量的重复操作。

Elasticsearch 本着让用户使用更方便快捷的原则，针对这个问题做了很多工作，使定义数据的方式更加抽象灵活，多个雷同的字段可使用 1 个配置完成。

比较有代表性的2个功能分别是：

- 索引模板（index template）：可以根据规则自动创建索引。

- 动态映射（dynamic mapping）：自动将新字段添加到映射中。

本小节我们着重介绍动态映射（dynamic mapping）

> 根据官方的定义动态映射可以自动检测和添加新字段（field）到映射（mapping）中。动态映射可以通过基础属性自动发现（Dynamic field mappings）以及复杂属性动态生成（Dynamic templates）2个方式实现此功能。

<!--more-->

# 动态字段映射（Dynamic field mappings）

在默认情况下，当索引一个文档时有字段是在映射中没有配置的，那么 Elasticsearch 将会根据该属性的类型，自动将其增加到映射中。该功能可以通过配置`dynamic`来控制打开。

该配置可以接受以下 3 种选择：

1. ture：默认配置，新字段将会自动加入映射中，并自动推断字段的类型。

1. false：新字段不会增加到映射中，因此不能被搜索，但是内容依然会保存在`_source`中。如无特殊需要建议都配置为 false，这样可以避免写入流程经过 master 节点，从而提高性能。

1. strict：索引文档时如果发现有新字段则报错，整个文档都不会被索引。

该配置可以在创建 mapping 时在根层配置，表示对所有属性适用。也可以每个内嵌对象（inner object）中配置，表示仅对该对象适用。

示例如下

```json
# 创建 test-dynamic-mapping
PUT test-dynamic-mapping
{
  "mappings": {
    "dynamic": false, # 1
    "properties": {
      "person":{
        "dynamic": true, # 2
        "properties": {
          "name":{
            "type":"keyword"
          }
        }
      },
      "company":{
        "dynamic": "strict",  # 3
        "properties": {
          "company_id":{
            "type":"keyword"
          }
        }
      }
    }
  }
}
```

1. #1 处的配置索引`test-dynamic-mapping`整体是不自动增加字段的

1. #2 处对于内嵌对象`person`我们设置它可以自动发现字段

1. #3 处对于内嵌对象`company`我们设置它发现新字段会报错

```json
# 插入文档
PUT test-dynamic-mapping/_doc/1
{
  "school":"test school", # 1
  "person":{
    "name":"tom",
    "age":"12" # 2
  },
  "company":{
    "company_id":"c001"
  }
}
```

1. 传入文档的根层有个未定义的`school`字段

1. 在 person 对象中增加 age 字段

```json
# 再次查看索引mapping
GET test-dynamic-mapping
{
  "test-dynamic-mapping" : {
    "mappings" : {
      "dynamic" : "false",
      "properties" : { # 1
        "company" : {
          "dynamic" : "strict",
          "properties" : {
            "company_id" : {
              "type" : "keyword"
            }
          }
        },
        "person" : {
          "dynamic" : "true",
          "properties" : {
            "name" : {
              "type" : "keyword"
            },
            "age" : { # 2
              "type" : "text",
              "fields" : {
                "keyword" : {
                  "type" : "keyword",
                  "ignore_above" : 256
                }
              }
            }
          }
        }
      }
    }
    ………………
  }
}
```

1. #1 处由于我们对整个 mapping 设置了`dynamic:false`，所以`school`属性没有自动创建

1. 由于内嵌对象`person`的`dynamic:true`，因此自动增加了`sex`属性，该属性派生出 2 个字段索引`person.age`其字段类型是text以及`person.age.keyword`其字段类型是keyword

```json
# 再次查看索引 mapping
GET test-dynamic-mapping
{
  "test-dynamic-mapping" : {
    "mappings" : {
      "dynamic" : "false",
      "properties" : { # 1
        "company" : {
          "dynamic" : "strict",
          "properties" : {
            "company_id" : {
              "type" : "keyword"
            }
          }
        },
        "person" : {
          "dynamic" : "true",
          "properties" : {
            "name" : {
              "type" : "keyword"
            },
            "age" : { # 2
              "type" : "text",
              "fields" : {
                "keyword" : {
                  "type" : "keyword",
                  "ignore_above" : 256
                }
              }
            }
          }
        }
      }
    }
    ………………
  }
}
```

1. 索引新文档时增加`company.company_name`字段

1. 由于`company`对象`dynamic:strict`，所以创建文档的请求返回了 1 个`strict_dynamic_mapping_exception`错误

对于JSON 中的字段  遵循以下映射方式发现新属性。

| JSON data type | Elasticsearch data type                                                             |
| -------------- | ----------------------------------------------------------------------------------- |
| null           | 不添加                                                                                 |
| true 或 false   | boolean 类型                                                                          |
| 带小数的数字，如1.1    | float 类型                                                                            |
| 整数，如 3         | long 类型                                                                             |
| 数组             | ES 不特殊处理数组类型                                                                        |
| 字符串            | 如果配置了自动识别且通过则可被识别为 date、float、long 类型如果未配置则会识别为 text 类型且增加 keyword 子属性使用 keyword 类型 |

对于 JSON 中的字符串字段，我们可以通过配置`date_detection: true`和`numeric_detection: true`尝试将它们转化成数值类型或时间类型，`date_detection`默认为 true，`numeric_detection`默认为 false。

在识别数字时，所有整型字符串会识别成long型，带小数的字符串会识别成float类型。默认情况下`yyyy/MM/dd HH:mm:ss`、`yyyy/MM/dd`、`epoch_millis`格式的字符串会识别成date类型。

```json
# 创建测试索引
PUT test-dynamic-mapping
{
  "mappings": {
    "dynamic": true,
    "numeric_detection": true, # 1
    "properties": {
      "field1":{
        "type": "keyword"
      }
    }
  }
}
# 插入数据
PUT test-dynamic-mapping/_doc/1
{
  "date":"2021/05/01", # 2
  "float":"1.1", # 3
  "long":"1" # 4
}
# 查看 mapping 变化
GET test-dynamic-mapping
{
  "test-dynamic-mapping" : {
     "mappings" : {
      "dynamic" : "true",
      "numeric_detection" : true,
      "properties" : {
        "date" : {    # 5
          "type" : "date",
          "format" : "yyyy/MM/dd HH:mm:ss||yyyy/MM/dd||epoch_millis"
        },
        "field1" : {
          "type" : "keyword"
        },
        "float" : {   #6
          "type" : "float"
        },
        "long" : {   # 7
          "type" : "long"
        }
      }
    }
  }
}
```

1. #1 处在创建索引时设置字符串可以自动识别为数值类型

1. #2 处 date 字段条是符合时间格式的字符串

1. #3 处 float 字段是符合小数格式的字符串

1. #4 处 long 字段是符合整型格式的字符串

1. #5 处 date 字段加入 mapping 并被自动识别成了 date 类型

1. #6 处 float 字段加入 mapping 并被自动识别成了 float 类型

1. #7 处 long 字段加入 mapping 并被自动识别成了 long 类型

Elasticsearch 识别日期字符串的格式，是可以通过`dynamic_date_formats`来配置。该字段支持使用`y`、`m`、`d`、`h`等字符自定义格式，具体方式与 Java 中`DateTimeFormatter`对象实现的规则相同。

同时还能配置大量国际标准的时间格式比如：epoch_millis、basic_date、basic_date_time、strict_date_optional_time_nanos 等，

> 所有可选项可以参照[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/7.10/mapping-date-format.html):[__https://www.elastic.co/guide/en/elasticsearch/reference/7.10/mapping-date-format.html__](https://www.elastic.co/guide/en/elasticsearch/reference/7.10/mapping-date-format.html)

```json
# 创建测试索引
PUT test-dynamic-mapping
{
  "mappings": {
    "dynamic": true,
    "dynamic_date_formats": ["MM/dd/yyyy"]  # 识别MM/dd/yyyy格式的时间
    "properties": {
      "field1":{
        "type": "keyword"
      }
    }
  }
}
# 插入数据
PUT test-dynamic-mapping/_doc/1
{
  "date":"09/25/2015",
  "date1":"2015/09/25"
}
# 查看mapping变化
{
  "test-dynamic-mapping" : {
    "mappings" : {
      "dynamic" : "true",
      "dynamic_date_formats" : [
        "MM/dd/yyyy"
      ],
      "properties" : {
        "date" : {          # 符合MM/dd/yyyy格式的字符串识别为了date类型
          "type" : "date",
          "format" : "MM/dd/yyyy"
        },
        "date1" : {        # 符合默认yyyy/MM/dd 格式的字符串识未正确识别
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "field1" : {
          "type" : "keyword"
        }
      }
    }
  }
}
```

# 动态模板（Dynamic templates）

Elasticsearch 的动态字段映射（Dynamic field mappings）虽然使用简单，但往往不满足现实的业务场景，比如对于整型字段，往往用不着 long 类型，使用 integer 类型就足够了；对于字符串类型的字段，我们希望细化分词方式，而不是使用默认分词，以及对于不同字段采用不同的分词方式等。这时可以使用动态模板（Dynamic templates）功能来实现上述需求。

动态模板允许你在创建 mapping 时，设置自定义规则。当新字段满足该规则时，则按照预先的配置来创建字段。

Elasticsearch 允许用户通过3个角度来定义规则：新字段的数据类型，属性名和路径。创建 mapping 时可以通过`dynamic_templates`字段配置多个动态模板。

模板的整体结构如下：

```json
{
  "mappings":{
    "dynamic_templates": [
      {
        "templateName":{   #1
          ……匹配规则……        # 2
          "mapping": { ... } #3      
        }
      }
     ]
  }
}
```

1. #1 处定义了动态模板的名称，每个动态模板都需要配置名字，本例中配置的模板名称为`templateName`

1. #2  处可以使用`match_mapping_type`、`match` 、 `unmatch` 、 `match_pattern`、`path_match`、`path_unmatch`来配置该模板的匹配规则，规则可以是多个，规则之间是`与`的关系

1. #3 处配置的是符合该规则的字段使用的 mapping 配置，此处与正常创建字段相同，主要需要配置`type`，`analyzer`等

下面我们通过几个例子来说明一下匹配规则中的各个关键字如何使用。

## match_mapping_type

`match_mapping_type`用于按照数据类型匹配，当用户想对 JSON 中具有某种数据类型的字段设置做特殊配置时，可以用此种匹配方式。该字段可配置的数据类型有如下几种：

- boolean，匹配值是 true 或 false 的字段。

- date，当字符串开启了时间类型识别且字符串符合预设日期格式则会被匹配

- double，匹配含有小数的字段

- long，匹配值是整型的字段

- object，匹配值是对象的字段

- string，匹配值是字符串的字段

- *，表示所有数据类型即匹配所有字段

之前我们提到 Elasticsearch 会自动将整型字段自动创建为long型，如果我们知道文档中所有数值都不会超过int范围，那么我们可以用如下配置，让所以非小数的数值字段自动创建为integer类型。

```json
PUT test-dynamic-mapping
{
  "mappings": {
    "dynamic_templates": [
      {
        "test_float": {
          "match_mapping_type": "long", # 值是整型的字段会被匹配
          "mapping": {
            "type": "integer"  # 字段 type 统一设为 integer
          }
        }
      }
    ]
  }
}
```

## match 、unmatch

在生产使用中最多的场景，是根据字段的名称进行匹配。这时就可以用`match`和`unmatch`这两种匹配方式。`match`匹配的是符合设置的所有字段，`unmatch`匹配的是不符合某种配置的所有字段。在设置匹配规则时可以使用`*`表 0 个或多个字符。

比如下面这个模板就表示所有属性名以`long_`开头且不以`_text`结尾的字段配置其`type`为long。

```json
PUT test-dynamic-mapping
{
  "mappings": {
    "dynamic_templates": [
      {
        "test_float": {
          "match": "long_*", # 属性名以 long_ 开头
          "unmatch": "*_test", # 属性名不以 _test 结尾
          "mapping": {
            "type": "long"  # 字段 type 设为 long
          }
        }
      }
    ]
  }
}
```

## match_pattern

仅仅使用通配符，可能不能满足我们多变的匹配需求，那么我们可以将`match_pattern`设为`regex`，这时`match`字段就可以用正则表达式了。

比如下面这个模板就表示所有以`profit_`开头，后跟至少 1 位数字的属性，将它们的`type`设为`keyword`。

```json
PUT test-dynamic-mapping
{
  "mappings": {
    "dynamic_templates": [
      {
        "test_float": {
          "match_pattern": "regex",  # match 使用正则表达式
          "match": "^profit_\d+$"  # 标准正则
          "mapping": {
            "type": "keyword"  # 字段 type 设为 keyword
          }
        }
      }
    ]
  }
}
```

## path_match 、 path_unmatch

在 Elasticsearch 中存储的文档允许有内嵌对象，当还有多层内嵌对象时，属性一般有路径的概念。属性的路径也可以作为匹配的条件。这个配置的用法与`match`和`unmatch`雷同，但需要注意的是`match`和`unmatch`仅作用于最后一级的属性名。

如下模板表示设置`person`内嵌对象除了`age`外其它所有以`long_`开头的字段新增时类型设为text。

```json
PUT test-dynamic-mapping
{
  "mappings": {
    "dynamic_templates": [
      {
        "test_float": {
          "match_pattern": "long_",  # 以 long_ 开头
          "path_match": "person.*",  # 内嵌对象 person 所有字段
          "path_unmatch": "*.age"    # 排除 age 字段
          "mapping": {
            "type": "text"  # 字段 type 设为 text
          }
        }
      }
    ]
  }
}
```



# 其它技巧及注意事项

在日常生产中难免有这样的需求，字段是什么类型就将类型设为什么，字段名是什么就用什么解析器。对于这种需求我们在配置动态模板的`mapping`时，可以使用占位符`{name}` 表示字段名，用 `{dynamic_type}`表示识别出的字段类型 。

比如下面 2 个模板一起表示的意思是，所有新增字符串类型字段，其解析器是字段的名称，所有其他类型字段新增时，类型就设为识别的字段类型，但是`doc_value`设为 false.

```json
PUT test-dynamic-mapping
{
  "mappings": {
    "dynamic_templates": [
      {
        "named_analyzers": {  # 字段名即是该字段的解析器名称
          "match_mapping_type": "string", # 匹配所有 string 类型
          "match": "*",  # 匹配任意属性名
          "mapping": {
            "type": "text",
            "analyzer": "{name}"  # 解析器是字段名
          }
        }
      },
      {
        "no_doc_values": {  
		 # 匹配所有类型，但匹配string的在前,所以实际匹配除string的其他所有字段
          "match_mapping_type":"*", 
          "mapping": {
            "type": "{dynamic_type}", # 类型直接作为type
            "doc_values": false
  
          }
        }
      }
    ]
  }
}
​
PUT test-dynamic-mapping/_doc/1
{
  "english": "Some English text",  # 该字段是新字段，会在mapping中新增会用english解析器
  "count":   5  # 该字段的类型会是 long
, doc_values为false
}
```

在使用动态模板时，还有以下几点需要注意。

1. 所有`null`值及空数组属于无效值，不会被任何动态模板匹配，在索引文档时，只有字段第一次有有效值时，才会与各动态模板匹配，找出匹配的模板创建新字段。

1. 规则匹配时，按照动态模板配置的顺序依次对比，使用最先匹配成功的模板，这就意味着如果有字段同时符合 2 个动态模板，那么会使用在`dynamic_templates`数组中靠前的那个。每个动态模板的匹配方式至少应包含`match`、`path_match` 、 `match_mapping_type`中的一个，`unmatch`和`path_unmatch`不能单独使用。

1. `mapping`的`dynamic_templates`字段是可以在运行时修改的，每次修改会整体替换`dynamic_templates`的所有值而非追加。

比如下面的请求就是将映射`test-dynamic-mapping`原来的动态模板配置删除，并配一个名为`newTemplate`的动态模板。

```json
PUT test-dynamic-mapping/_mapping
{
  "dynamic_templates": [
    {
      "newTemplate": {
        "match": "abc*",
        "mapping": {
          "type": "keyword"
        }
      }
    }
  ]
}
```

# 鸣谢

本文收录至《Elastic Stack 实战手册》，欢迎和我一起解锁开发者共创书籍，系统学习 Elasticsearch。感谢阿里组织ES百人大战，感谢各位老师及小伙伴的帮助。https://developer.aliyun.com/topic/elasticstack/playbook

![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/elastic-stack-logo.png)
