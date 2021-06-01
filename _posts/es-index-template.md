---
title: elasticsearch-Index template 索引模板
date: 2021/05/31 16:26:10
categories:
- elasticsearch

tags:
- elasticsearch
- index

keywords:
- elasticsearch,  index template, 索引模板
---

# 简介

Elasticsearch 本着让用户方便快捷的使用搜索功能的原则，对数据定义（索引定义）做了高度抽象，尽可能得避免了重复性定义工作，使之更加灵活。

Elasticsearch 在这方面做的工作主要体现是索引模板（Index template）和动态映射（Dynamic Mapping）两个功能。索引模板的主要功能，是允许用户在创建索引（index）时，引用已保存的模板来减少配置项。操作的一般过程是先创建索引模板，然后再手动创建索引或保存文档（Document）。而自动创建索引时，索引模板会作为配置的基础作用。对于 X-Pack 的数据流（Data Stream）功能，索引模板用于自动创建后备索引（Backing Indices）。该功能的意义是提供了一种配置复用机制，减少了大量重复作劳动。

目前 Elasticsearch 的索引模板功能以 7.8 版本为界，分为新老两套实现，新老两个版本的主要区别是模板之间复用如何实现。

**老版本：**

使用优先级（order）关键字实现，当创建索引匹配到多个索引模板时，高优先级会继承并铺盖低优先级的模板配置，最终多个模板共同起作用。

**新版本：**

删除了 order 关键字，引入了组件模板 Component template 的概念，是第一段可以复用的配置块。在创建普通模板时可以声明引用多个组件模板，当创建索引匹配到多个新版索引模板时，取用最高权重的那个。

下面将以生命周期（创建、查看、使用、删除）为切入点，分别介绍如何使用 Elasticsearch 新老两个版本的索引模板。

<!--more-->

# 新版索引模板

在使用新版本的索引模板功能前，我们应当确认 Elasticsearch 是否开启了安全设置，如果是，那么我们操作的角色对于集群需要有 manage_index_templates 或 manage 权限才能使用索引模板功能，这个限制对新老版本都适用。

对于新版，Elasticsearch 为了方便用户调试，提供了模拟 API 来帮助用户测试创建索引最终的配置。该模拟 API 主要有 2 个

第一个模拟在现有索引模板下创建 1 个索引，最终使用的模板配置是什么样的。

第二个模拟在指定模板的配置下，最终模板配置是什么样的。

第一种模拟 API 使用范例如下：

```json
# 创建1个组件模板ct1
PUT /_component_template/ct1             # 1
{
  "template": {
    "settings": {
      "index.number_of_shards": 2       
    }
  }
}
# 创建1个组件模板ct2
PUT /_component_template/ct2             # 2
{
  "template": {
    "settings": {
      "index.number_of_replicas": 0     
    },
    "mappings": {
      "properties": {
        "@timestamp": {
          "type": "date"
        }
      }
    }
  }
}
# 创建1个索引模板 final-template
PUT /_index_template/final-template      # 3
{
  "index_patterns": ["my-index-*"],
  "composed_of": ["ct1", "ct2"],         
  "priority": 5,
  "template":{
  "settings": {
      "index.number_of_replicas": 1
    },
    "mappings": {
      "properties": {
        "name": {
          "type": "keyword"
        }
      }
    }
  }
}
# 验证创建名为 my-index-00000 的索引使用的配置是如何
POST /_index_template/_simulate_index/my-index-00000   # 4
#返回
{
  "template" : {    # 引用模板中的配置有settings、mappings、aliase
    "settings" : {
      "index" : {
        "number_of_shards" : "2",
        "number_of_replicas" : "1"
      }
    },
    "mappings" : {
      "properties" : {
        "@timestamp" : {
          "type" : "date"
        },
        "name" : {
          "type" : "keyword"
        }
      }
    },
    "aliases" : { }
   },
   "overlapping" : [  # 5
     {
       "name" : "test-template",
       "index_patterns" : ["my-*"]
    }
   ]
}
```

1. 在 #1 处创建了 1 个名为`cr1`的组件模板，该模板设置了索引分片数为2
2. 在 #2 处创建了 1 个名为`cr2`的组件模板，该模板设置了索引由1个`@timestamp`属性，并且副本数是0
3. 在 #3 处创建了1个名为`final-template`的索引模板，它适用于所有以`my-index-`开头的索引
4. 在 #4 处向`/_index_template/_simulate_index/my-index-00000`发送 POST 请求，测试创建名为`my-index-00000`的索引，下面的 JSON 是使用索引模板的配置，可以看出 template 字段是组件 cr1、cr2 以及索引模板 final-template 全部配置的聚合。
5. #5处的`overlapping`字段表示忽略了名为`test-template`模板的配置。

第二种模拟 API 使用范例如下：

```json
# 默认组件模板 ct1、crt 已创建，内容如前
# 验证按照该模板创建的索引时会使用的配置
POST /_index_template/_simulate/<index-template> #1
{  # 2
  "index_patterns":["test-*"],
  "composed_of": ["ct1","ct2"],
  "priority": 5,
  "template": {
    "settings": {
      "index.number_of_replicas": 1
    },
    "mappings": {
      "properties": {
        "name": {
          "type": "keyword"
        }
      }
    }
  }
}
# 返回
{    # 3
  "template" : {
    "settings" : {
      "index" : {
        "number_of_shards" : "2",
        "number_of_replicas" : "1"
      }
    },
    "mappings" : {
      "properties" : {
        "@timestamp" : {
          "type" : "date"
        },
        "name" : {
          "type" : "keyword"
        }
      }
    },
    "aliases" : { }
  },
  "overlapping" : [
    {
     "name" : "test-template",
     "index_patterns" : ["my-*"]
    }
  ]
}
```

1. #1 向`/_index_template/_simulate/<index-template>`发送 POST 请求，其中`<index-template>`为自定义索引模板的名词，该模板并不会实际创建
2. #2 处是请求的 body，与创建一个新版索引模板个格式完全一样，后续详细介绍，目前只需要知道它引用了 cr1 和 cr2 两个组件模
3. #3 处为请求返回的 body，可以看出内容是组件模板 cr1、cr2 以及请求 body3 个配置的组合。

## 创建

新版本索引自动配置功能，主要依托组件模板（component template）和索引模板（index template）2 个概念来完成。下面依次来介绍他们是如何实现的。

### 组件模板的创建

组件模板是用来构建索引模板的特殊模板，它的内容一般是多个索引模板的公有配置，索引模板在创建时，可以声明引用多个组件模板。

在组件模板中可配置的索引内容有：别名`aliases`、配置`settings`、映射`mappings`3 个，具体配置方式与创建索引时一致。组件模板只有在被索引模板引用时，才会发挥作用。当我们需要创建或更新一个组件模板时，向`/_component_template`路径发送 PUT 请求即可。

具体示例如下：

```json
# 创建组件模板
PUT /_component_template/template_1?create=true&master_timeout=30s   #1
{
  "template": {  # 2
    "settings": {
      "number_of_shards": 1
    },
    "mappings": {
      "_source": {
        "enabled": false
      },
      "properties": {
        "name": {
          "type": "keyword"
        }
      }
    },
    "aliases": {
      "test-index": {}
    }
  },
  "version": 1,    #3
  "_meta": {       #4
    "description1": "for test",
    "description2": "create by phoenix"
  }
}
```

1. #1 处向`/_component_template/template_1`发送 PUT 请求创建一个名为`template_1`的组件模板，模板名可以任意替换，需要注意的是，Elasticsearch 中预置有6个组件模板：`logs-mappings`、`logs-settings`、`metrics-mappings`、`metrics-settings`、`synthetics-mapping`、`synthetics-settings`，建议不要覆盖。同时 url 中还包含2个可选的查询参数`create`和`master_timeout`
   1. `create`，表示此次请求是否是创建请求，如果为 true 则系统中如果已有同名的会报错，默认为 false，表示请求可以是创建也可能是更新请求
   2. `master_timeout`，表示可以容忍的连接 Elasticsearch 主节点的时间，默认是30s，如果超时则请求报错
2. #2 处`template`的内容是对索引的设置，主要有别名，映射和配置，在本例中配置了索引分片数是 1，`_source`不用保存，索引有名为`name`的属性，类型是 keyword，同时索引别名有`test-index`。
3. #3 处是用户指定的组件模板的版本号，为了方便外部管理，此为可选项，默认不会为组件模板增加版本号
4. #4 处是用户为组件模板设置的 meta 信息，该对象字段可随意配置，但不要设置的过大。该字段也是可选的

### 索引模板的创建

创建或更新一个索引模板的方式都是向`/_index_template`发送 1 个 PUT 请求。索引模板可以配置的内容主要有 3 类，分别是别名`aliases`、配置`settings`、映射`mappings`。这 3 部分的配置方式，与创建索引时的设置完全一样这里不再赘述。

创建索引模板的示例：

```json
PUT /_index_template/test_template?create=false&master_timeout=30s #1
{
  "index_patterns" : ["te*"],    #2
  "priority" : 1,   #3 
  "composed_of": ["template_1"], #4
  "template": {     #5
    "settings" : {
      "number_of_shards" : 2
    }
  },
  "version": 2,   #6
  "_meta": {      #7
    "user": "phoenix",
    "time": "2021/05/06"
  }
}
# 测试模板
POST /_index_template/_simulate_index/test    #8
{
  "template" : {
    "settings" : {
      "index" : {
        "number_of_shards" : "2"  # 索引模板覆盖了组件模板的配置
      }
    },
    "mappings" : {
      "_source" : {             # 使用组件模板的配置
        "enabled" : false
      },
      "properties" : {
        "name" : {        # 使用组件模板的配置
          "type" : "keyword"
        }
      }
    },
    "aliases" : {
      "test-index" : { }     # 使用组件模板的配置
    }
  }
}
```

1. #1 处向`/_index_template/test_template`发送 PUT 请求创建索引模板，模板名称为`test_template`，名称可任意填写，与组件模板相同，有2个可选的查询参数：
   1. `create`，表示此次请求是否是创建请求，如果为 true 则系统中如果已有同名模板则会报错，默认为 false，表示请求可以是创建也可能是更新请求
   2. `master_timeout`，表示可以容忍的连接 Elasticsearch 主节点的时间，默认是30s，如果超时则请求报错
2. #2 处`index_patterns`字段用于设置匹配索引的规则，目前仅支持使用索引名称匹配，支持`*`号作为通配符，该字段是必填字段。需要注意 Elasticsearch 自带了3种匹配规则的索引模板：`logs-*-*`、`metrics-*-*`、`synthetics-*-*`，建议不要做相同配置。
3. #3 处`priority`字段配置的是模板权重，当 1 个索引名符合多个模板的匹配规则时， 会使用该值最大的做为最终使用的模板，此值默认为 0，Elasticsearch 自带的模板此值是100。
4. #4 处`composed_of`字段用于配置索引模板引用的组件模板，此处引用的组件模板配置自动会增加到该模板中。此例我们引用了之前创建的`template_1`组件。
5. #5 处`template`的内容是对索引的设置。
6. #6 处是用户指定的索引模板的版本号，为了方便外部管理，此为可选项，默认不会为组件模板增加版本号
7. #7 处是用户为索引模板设置的 Meta 信息，该对象字段可随意配置，但不要设置的过大。该字段也是可选的
8. #8 处用模拟 API 创建名为`test`的索引，我们发现返回的模板包含了模板`test_template`和组件`template_1`的所有配置

## 查看

创建完索引模板或组件模板后，我们可以使用 GET 请求再次查看其内容，请求是可以指定名称表示精准找，也可以使用通配符`*`进行范围查找，如果不指定名称则会返回全部。

```json
GET /_component_template/template_1?local=false&master_timeout=30s #1
GET /_component_template/template_* #2
GET /_component_template #3
GET /_index_template/test_template #4
GET /_index_template/test_* #5
GET /_index_template #6
```

1. #1 是查看名为`template_1`的组件模板
2. #2 是查看名字以`template_`开头的所有组件模板
3. #3 是查看所有组件模板，通过该请求我们可以发现 Elasticsearch 默认创建了很多组件模板，使用时应尽量避免冲突
4. #4 是查看名字为`test_template`的索引模板
5. #5 是查看名字以`test_`开头的所有索引模板
6. #6 是查看所有索引模板，通过该请求我们可以发现 Elasticsearch 默认创建了很多索引模板，使用时应尽量避免冲突

如 #1 所示，上述所有请求都可以增加 2 个可选的查询参数：

1. `local`，如果为`true`模板的配置仅从本地节点中获取，默认为`false`表示此次查询结果是`master`节点返回的
2. `master_timeout`，表示可以容忍的连接elasticsearch主节点的时间，默认是30s，如果超时则请求报错

## 使用

关于组件模板如何使用我们在创建索引模板时已经提及，增加`composed_of`字段声明索引模板引用的组件模板，这样组件模板的配置，就自动增加到了索引模板中。

索引模板的使用主要发生在创建索引的时候，如果创建的索引名与索引模板的`index_patterns`配置相匹配，那么该索引将会在此模板的基础上创建。

示例如下：

```json
# 创建名为test-my-template的索引
PUT /test-my-template
{
  "settings": {
    "index": {
      "number_of_replicas": 3
    }
  },
  "mappings": {
    "properties": {
      "desc":{
        "type": "keyword"
      }
    }
  }
}
# 查看索引信息
GET /test-my-template
# 返回
{
  "test-my-template" : {
    "aliases" : {          # 组件模板 template_1 的配置
      "test-index" : { }
    },
    "mappings" : {
      "_source" : {         # 组件模板 template_1 的配置
        "enabled" : false
      },
      "properties" : {
        "desc" : {          # 创建索引时的配置
          "type" : "keyword"
        }, 
        "name" : {          # 组件模板 template_1 的配置
          "type" : "keyword"
        }
      }
    },
    "settings" : {
      "index" : {
        "routing" : {
          "allocation" : {
            "include" : {
              "_tier_preference" : "data_content"
            }
          }
        },
        "number_of_shards" : "3",   # 覆盖了 索引模板 number_of_shards=2 的配置
        "provided_name" : "test-my-template",
        "creation_date" : "1620550940095",
        "number_of_replicas" : "1",
        "uuid" : "7UZKxK56SZ-bAwBeDL-Jvg",
        "version" : {
          "created" : "7100099"
        }
      }
    }
  }
}
```

本例我们创建了名为`test-my-template`的索引，其索引名以`te`开头，匹配索引模板`test_template`的筛选规则。我们在创建时仅指定了分片数和一个`desc`属性，而在我们使用 GET 请求查看索引信息时可以看出，索引模板中的别名配置、`_source`配置和`name`属性的配置被使用了。但是关于分片的设置，由于创建索引时明确指出了，所以模板中的配置`number_of_shards=2`被覆盖为了 3。

由于在向 一个不存在的索引中插入数据时，Elasticsearch 会自动创建索引，如果新的索引名匹配某个索引模板的话，也会以模板配置来创建索引。在使用该功能前，请确保 Elasticsearch 集群的`action.auto_create_index`配置是允许你自动创建目标索引的。

示例如下：

```json
# 向不存在的 test 索引中保存1条数据
PUT /test/_doc/1
{
  "name":"tom",
  "age": 18
}

# 查看 test 索引信息
GET /test
{
  "test" : {
    "aliases" : {         # 组件模板 template_1 的配置
      "test-index" : { }
    },
    "mappings" : {
      "_source" : {       # 组件模板 template_1 的配置
        "enabled" : false 
      },
      "properties" : {
        "age" : {         # ES 自动推断创建的属性
          "type" : "long"
        },
        "name" : {       # 组件模板 template_1 的配置
          "type" : "keyword"
        }
      }
    },
    "settings" : {
      "index" : {
        "routing" : {
          "allocation" : {
            "include" : {
              "_tier_preference" : "data_content"
            }
          }
        },
        "number_of_shards" : "2",     # 使用索引模板的配置
        "provided_name" : "test",
        "creation_date" : "1620551933756",
        "number_of_replicas" : "1",
        "uuid" : "Oop40MPySjKbSKvs0OQwTA",
        "version" : {
          "created" : "7100099"
        }
      }
    }
  }
}
```

如上例所示，我们向不存在的`test`索引中保存 1 条数据，该索引名与索引模板`test_template`的规则相匹配。因此索引的配置与模板配置完全契合，除了模板中没有配置的`age`字段，是通过 Elasticsearch 属性自动创建功能生成的。

## 删除

当索引模板或组件模板完成了它们的使命，我们应该及时将其删除，避免因遗忘导致创建出的索引出现了预料之外的配置。删除操作非常简单，发送 DELETE 请求即可，删除同样可以使用通配符`*`一次删除多个，示例如下：

```json
DELETE  /_component_template/template_1?master_timeout=30s&timeout=30s  #1
DELETE  /_component_template/template_*  #2
DELETE  /_index_template/test_template  #3
DELETE  /_index_template/test_*  #4
```

1. #1 表示删除名为`template_1`的组件模板
2. #2 表示删除名字以`template_`开头的组件模板
3. #3 表示删除名为`test_template`的索引模板
4. #3 表示删除名字以`test_`开头的索引模板

如 #1 所示，上述所有请求都可以增加 2 个可选的查询参数：

1. `timeout`，表示可以容忍的等待响应时间，默认是 30s，如果超时则请求报错
2. `master_timeout`，表示可以容忍的连接 Elasticsearch 主节点的时间，默认是 30s，如果超时则请求报错

# 老版索引模板

Elasticsearch 7.8 版本之前的索引模板功能，与新版本基本相同，唯一的区别就是在模板的复用方式上。老版本允许在创建索引时，匹配到多个模板，多个模板间根据`order`配置的优先级从低到高依次覆盖。这种方式会造成用户在创建索引时，不能明确知道自己到底用了多少模板，索引配置在继承覆盖的过程中非常容易出错。

下面依然从模板的生命周期出发，介绍如何使用。

## 创建

与新版本一样，创建或更新一个老版索引模板，只需要向`/_template`发送PUT请求即可，通过索引模板可配置的字段依然是：别名`aliases`、配置`settings`、映射`mappings`3个。

具体示例如下：

```json
# 创建或更新老模板
PUT /_template/old_template?order=1&create=false&master_timeout=30s  # 1
{
  "index_patterns": ["te*", "bar*"],      #2
  "settings": {             #3
    "number_of_shards": 1
  },
  "aliases": {              #4
    "old-template-index": {}
  }, 
  "mappings": {             #5
    "_source": {
      "enabled": false
    },
    "properties": {
      "host_name": {
        "type": "keyword"
      }
    }
  },
  "version": 0              #6
}
```

1. #1 处向`/_template/old_template`发送PUT请求创建或索引模板，模板名称为`old_template`，名称可任意填写，该请求有 4 个可选的查询参数：
    1. `create`，表示此次请求是否是创建请求，如果为 true 则系统中如果已有同名模板会报错，默认为 false，表示请求可以是创建也可能是更新请求
    2. `master_timeout`，表示可以容忍的连接 Elasticsearch 主节点的时间，默认是30s，如果超时则请求报错
    3. `order`，该变量接受一个整数，表示模板的优先级，数字越大优先级越高，相关配置越可能被实际使用，强烈建议每个模板都根据实际情况配置该值，不要使用默认值。
2. #2 处`index_patterns`字段用于配置匹配索引的规则，目前仅支持使用索引名称匹配，支持`*`号作为通配符，该字段是必填字段，可配置多个值表示”或“的关系。
3. #3 处`settings`字段用于配置索引属性。
> 具体规则详见文档:[__https://www.elastic.co/guide/en/elasticsearch/reference/7.10/index-modules.html#index-modules-settings__](https://www.elastic.co/guide/en/elasticsearch/reference/7.10/index-modules.html#index-modules-settings)
4. #4 处`aliases`字段用于配置索引的别名。
> 具体规则详见文档:[__https://www.elastic.co/guide/en/elasticsearch/reference/7.10/indices-aliases.html__](https://www.elastic.co/guide/en/elasticsearch/reference/7.10/indices-aliases.html)
5. #5 处`mappings`字段用于配置索引的映射。
> 具体规则详见文档：[__https://www.elastic.co/guide/en/elasticsearch/reference/7.10/mapping.html__](https://www.elastic.co/guide/en/elasticsearch/reference/7.10/mapping.html)
6. #6 处`version`字段是用户指定的索引模板的版本号，为了方便外部管理，此为可选项，Elasticsearch 默认不会为组件模板增加版本号。

## 查看

我们可以使用 GET 请求，查看其模板内容，同样可以使用名称精确查找，也可以使用通配符部分搜索以及搜索全部。老版本还支持使用`HEAD`请求快速验证 1 个模板是否存在

```text
GET /_template/old_template?local=false&master_timeout=30s&flat_settings=false #1
GET /_template/old_* #2
GET /_template #3
HEAD /_template/old_template  #4
```

1. #1 是查看名为`old_template`的索引模板
2. #2 是查看名字以`old_`开头的所有索引模板
3. #3 是查看所有索引模板，通过该请求我们可以发现 Elasticsearch 默认创建了很多组件模板，使用时应尽量避免冲突
4. #4 是验证名为`old_template`的索引模板是否存在，与上面 3 个请求返回模板内容不同，本请求不返回模板内容，以状态码为 200 表示存在，404 表示不存在

如#1所示，上述所有请求都可以增加 3 个可选的查询参数：

1. `local`，如果为`true`组件模板的配置仅从本地节点中获取，默认为`false`表示此次查询结果是`master`节点返回的
2. `master_timeout`，表示可以容忍的连接 Elasticsearch 主节点的时间，默认是30s，如果超时则请求报错
3. `flat_settings`，表示返回的配置中关于`settings`字段如何展示，如果为 false 则使用标准 JSON 格式展示，默认为 true 表示多级属性会压缩成 1 级属性名，以点分格式展示，比如关于索引分片的设置，如果该变量为 true 则返回为`index.number_of_shards:1`

## 使用

索引模板的使用，主要发生在创建索引的时候，如果创建的索引名与索引模板相匹配，那么该索引将会在此模板的基础上创建。需要注意的是，如果同时匹配到新老两个版本的模板，那么默认使用新版本。如果仅匹配到多个老版模板则根据`order`字段依次覆盖。

```json
# 创建索引
PUT /bar-test-old
{
  "settings": {
    "number_of_replicas": 2
  },
  "mappings": {
    "_source": {
      "enabled": true
    },
    "properties": {
      "ip":{
        "type": "ip"
      }
    }
  }
}
# 查看索引
GET /bar-test-old
# 返回
{
  "bar-test-old" : {
    "aliases" : {
      "old-template-index" : { }   # old_template模板中的配置
    },
    "mappings" : {
      "_source": {
        "enabled": true   # 创建时的配置覆盖模板的配置
      },
      "properties" : {
        "host_name" : {    #  old_template模板中的配置
          "type" : "keyword"
        },
        "ip" : {
          "type" : "ip"
        }
      }
    },
    "settings" : {
      "index" : {
        "routing" : {
          "allocation" : {
            "include" : {
              "_tier_preference" : "data_content"
            }
          }
        },
        "number_of_shards" : "1",    # old_template模板中的配置
        "provided_name" : "bar-test-old",
        "creation_date" : "1620615937000",
        "number_of_replicas" : "2",
        "uuid" : "jG3xiiB_S-iFrlwZ7df56g",
        "version" : {
          "created" : "7100099"
        }
      }
    }
  }
}
```

本例我们创建了名为`bar-test-old`的索引，其索引名以`bar`开头，匹配索引模板`old_template`的筛选规则，我们在创建时仅指定了副本数和 1 个`ip`属性，而使用 GET 请求查看索引信息时可以看出，索引模板中的别名配置、`host_name`属性和分片数的配置被使用了。但是关于`_source`的设置，由于创建索引时明确指出了，所以模板中的`false`配置被覆盖为了`true`。

当匹配到多个老版索引模板时，最终配置为多个模板的组合，当不同模板配置了相同字段时，那么以`order`高的模板配置为准。示例如下：

```json
# 创建新版索引模板   
PUT /_index_template/new_template   #1
{
   "index_patterns": ["template*"],
   "priority" : 100, 
   "template": {
    "mappings": {
      "dynamic": "true"
    },
    "aliases": {
      "new-template": {}
    }
  }
}
# 创建老版索引模板，不配置 order
PUT /_template/old_template_1     #2
{
  "index_patterns": ["temp*"],
   "mappings": {
      "dynamic": "false"
    },
  "aliases": {
      "old_template_1": {}
    }
}
# 创建老版本索引模板，配置高 order
PUT /_template/old_template_2?order=10  #3
{
  "index_patterns": ["temp-*"],
   "mappings": {
      "dynamic": "strict"
    },
  "aliases": {
      "old_template_2": {}
    }
}
# 创建索引，名称会匹配 new_template 模板和 old_template_1 模板
PUT /template-test    #4
{
  "settings": {
    "number_of_shards": 2
  },
  "mappings": {
    "properties": {
      "ip":{
        "type": "ip"
      }
    }
  }
}
# 查看索引配置
GET  /template-test  #5
# 返回
{
  "template-test" : {
    "aliases" : {
      "new-template" : { }  #  仅有新版本模板的别名设置
    },
    "mappings" : {
      "dynamic" : "true",
      "properties" : {
        "ip" : {
          "type" : "ip"
        }
      }
    },
    "settings" : {
      "index" : {
        "routing" : {
          "allocation" : {
            "include" : {
              "_tier_preference" : "data_content"
            }
          }
        },
        "number_of_shards" : "2",
        "provided_name" : "template-test",
        "creation_date" : "1620617781794",
        "number_of_replicas" : "1",
        "uuid" : "0PKYANozRya9zG3LdMG1UA",
        "version" : {
          "created" : "7100099"
        }
      }
    }
  }
}
# 创建索引，名称会匹配 old_template_1 模板和 old_template_1 模板
PUT /temp-test #6
{
  "settings": {
    "number_of_shards": 2
  },
  "mappings": {
    "properties": {
      "ip":{
        "type": "ip"
      }
    }
  }
}
# 查看索引配置
GET /temp-test
# 返回
{
  "temp-test" : {
    "aliases" : {
      "old_template_1" : { },  # old_template_1 的配置
      "old_template_2" : { }    # old_template_2 的配置
    },
    "mappings" : {
      "dynamic" : "strict",  # old_template_2 的配置
      "properties" : {
        "ip" : {
          "type" : "ip"
        }
      }
    },
    "settings" : {
        "refresh_interval" : "10s",
        "number_of_shards" : "2",
        "provided_name" : "temp-test",
        "creation_date" : "1620617979932",
        "number_of_replicas" : "1",
        "uuid" : "RwvrdGziT7iVmVutXYPwqA",
        "version" : {
          "created" : "7100099"
        }
      }
    }
  }
}
```

1. #1 处我们创建了 1 个新版本的索引模板匹配名字，以`template`开头的索引，设置可以动态增加属性并设置别名`new-template`
2. #2 处我们创建了 1 个老版本的索引模板匹配名字，以`temp`开头的索引，设置不动态增加属性并设置别名`old_template_1`
3. #3 处我们创建了 1 个老版本的索引模板匹配名字，以`temp-`开头的索引，设置有新增属性时报错并设置别名`old_template_2`，同时配置`order`为10
4. #4 处我们创建了 1 个名为`template-test`的索引
5. #5 处查看索引`template-test`的配置，该名称会匹配新版`new_template`模板，和老版`old_template_1`模板，根据索引的别名信息，只有新版模板配置的别名可以看出，该索引仅仅应用了`new_template`模板
6. #6 处创建了 1 个名为`temp-test`的索引
7. #7 处查看索引`temp-test`的配置，该名称会匹配老版`old_template_1`和`old_template_2`模板，根据索引的别名信息有`old_template_1`和`old_template_1`可以看出，2 个模板的配置都应用了。通过`dynamic`字段为`strict`我们可以判断该字段使用了`order`较高的模板`old_template_2`的配置

## 删除

老版本的索引模板完成使命后，应该及时将其删除，避免创建索引时匹配到不必要的模板，导致最终创建的索引与预期不符。删除操作非常简单，发送 delete 请求即可，删除同样可以使用通配符`*`一次删除多个，示例如下：

```text
DELETE  /_template/old_template_1?master_timeout=30s&timeout=30s  #1
DELETE  /_template/old_template*  #2
```

1. #1 表示删除名为`old_template_1`的索引模板
2. #2 表示删除名字以`old_template`开头的索引模板

如 #1 所示，上述 2 个请求都可以增加 2 个可选的查询参数

1. `timeout`，表示可以容忍的等待响应时间，默认是 30s，如果超时则请求报错
2. `master_timeout`，表示可以容忍的连接elasticsearch主节点的时间，默认是 30s，如果超时则请求报错

# 注意事项及技巧

目前新老版本的索引模板在7.10版本都可以使用，并且都是通过名称匹配的方式，来决定最后使用的配置，那么在使用的过程中难免会遇到，匹配到多个模板且配置有冲突的情况。下面总结几点规则来介绍，创建索引的最终配置是如何决定的。

1. 如果同时匹配到新老 2 个版本的索引模板，那么使用新版模板。
2. 如果仅匹配到多个新版模板，那么使用`priority`值最高的索引模板。
3. 如果新版模板中配置了多个组件模板，且组件中有配置冲突，那么使用`composed_of`数组中靠后的组件模板的配置。
4. 如果组件模板和索引模板有字段冲突，那么使用索引模板中的配置。
5. 如果仅匹配到多个老版模板，那么最终配置由多个模板共同构成，如果有配置冲突，使用`order`值高的模板的配置。
6. 如果创建索引语句中的配置与索引模板（不管新老版本）冲突，那么使用创建语句中的配置。

索引模板一般和动态映射结合使用，这样可以大大减少创建索引的语句，缩减索引创建频次。配置方式是在`mappings`字段中配置`dynamic_templates`的相关内容。

这个功能一般用于创建时序类数据的索引，比如日志数据，每天都会有新数据进入索引，数据量会持续增加，只用一个索引肯定不合适，需要按日或按月创建。

这种情况约定录入数据的索引名称与日期相关，再创建索引模板，这样数据持续录入时，索引也会按需增加，且不用人工干预，后期对不同索引还能做冷热处理。

# 鸣谢

本文收录至《Elastic Stack 实战手册》，欢迎和我一起解锁开发者共创书籍，系统学习 Elasticsearch。感谢阿里组织ES百人大战，感谢各位老师及小伙伴的帮助。https://developer.aliyun.com/topic/elasticstack/playbook

![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/elastic-stack-logo.png)