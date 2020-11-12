```
---
title: elasticsearch学习——节点角色  

categories:
- elasticsearch  

tags:
- elasticsearch,node  

keywords:
- elasticsearch, master eligible node, data node, ingest node, coordinationg node
---
```



# 概述

你启动的elasticsearch实例就是一个node。在es中node是有角色区分的，正确分配node角色可以有效提升es效率及稳定性，同时对后续学习es增删改机制也有重大帮助。

本文将主要介绍node各节点的功能，配置节点角色的最佳实践（仅供探讨）。

# ES节点角色

在es中我们可以通过配置使一个节点有以下一个或多个角色：

1. master备选节点（master-eligible node）
2. 数据节点（data node）
3. 预处理节点（ingest node）
4. 机器学习节点（machine learning node）
5. 协调节点（coordinating node）

在es中所有节点都知道集群中其它节点的情况，每个节点会将请求转发到合适的节点上。在默认情况下每个节点是具有上述所有角色（初机器学习节点外）的。即默认情况下，所有节点都是master的备选节点，都用于存储数据，都可以做预处理操作，如果有机器学习模块，所有节点都可以做机器学习的处理。下面详细介绍每种角色。

# master-eligible node

master-eligible节点表示该节点有权参选成为master节点。通过`node.master`配置，默认配置是`true`。master对集群健康起到至关重要的作用。它主要负责集群轻量级操作，比如：创建\删除索引，判定那些节点属于本集群、分片该分配到那个节点上。在最新版的配置中，可以将节点配置为voting-only master-eligible节点，即仅参与master节点的选举，但是不能被选为master，会做与master eligible类似的事，但永远不会成为master。关于选举master的机制，后续后时间在深入介绍。不过值得注意的是，7.x和6.x相比选举模式有调整。

注意如果node.master=false , node.data=false，其在数据目录下发现任何索引原信息则节点拒绝启动。如果一个节点，你想将node.data node.master都设为false，最好的方式是先用allocation filter 将shard数据移走，再指定一个新的空的数据目录。

# data node

data节点主要工作有：维护shard，执行数据的CRUD，搜索，聚合操作。对I/O、CPU、内存消耗都比较高。在7.X的高版本中data节点还可以继续划分为一下角色：`data_content`、`data_hot`、`data_warm`、`data_cloud`。某个节点可以配置多个细分角色，但如果配置了data的细分角色就不能再配置为普通的data角色。

**Content data node**：存储普通数据，支持CRUD、search或聚合操作。与存储时序数据不同，存储在该节点的数据价值在一段时间内保持相对恒定，因此随着时间的流逝，将其转移到具有不同性能特征的层中是没有意义的。

**hot data node**：时序类的数据会首先进入该类节点，这些节点需要更好的read、write性能，需要更高的硬件配置。

**warm data node**: 存储的索引不再定期更新，但仍在查询中。 查询量通常比索引处于热层时的频率低。 性能较低的硬件通常可用于此层中的节点。

**cold node**：冷数据节点存储只读索引，这些索引的访问频率较低。 该层使用性能较低的硬件，并且可能利用快照支持的索引来最大程度地减少所需的资源。

需要注意的是：如果`node.data=false`，且在数据目录下找到任何分片数据，则节点拒绝启动。如果想将`data node`的配置改为`node.data =false`，需要使用allocation filter 安全的将shard数据转移到其他节点

# ingest node

ingest节点用于进行pipeline操作，pipeline中可以配置多个处理环节。比如入数据时，统一对数据进行处理。首先不应该让es在入数据前做太多预处理操作，入ES的数据应该是已经处理好的。如果因为环境问题必须使用es进行大量pipeline操作，那么建议预处理节点不再担任其他职务，仅做预处理。作为ingest node还是需要维护一些数据的，比如每个索引元数据、集群元数据。

# coordinating node

协调节点（coordinating node）仅做转发请求，将搜索结果聚合并返回给用户端以及分发批量的索引操作。比如一般搜索操作分为2个阶段。第一阶段客户端将请求发送到集群中某个协调节点上，协调节点将请求转发到保存数据的数据节点上。每个数据节点在本地执行该请求，并将其结果返回给协调节点。在第二阶段，协调节点将每个数据节点产生的结果缩减为单个全局结果集。

每个节点都隐式的是一个协调节点。这意味着将`node.master`、`node.data`、`node.ingest`都设置为`false`时则该节点仅充当协调节点。设置仅用作协调的节点，可以大大减轻master和data节点的负担。但是配置过多的仅有协调功能的节点，也会增加master节点的负担，因为它需要同步每个节点的状态。一般`data node`会承担协调的职责。

# machine learning node

首先需要将`xpack.ml.enabled`和`node.ml`都设为`true`。机器学习是`x-pack`附加功能，如果要使用相关功能，必须至少设置一个机器学习节点。

# 其他

不同版本对es节点的划分有所不同。但都会有master-eligible node、data node、ingest node和coordinating node。在6.x的版本中会有`Tribe node`使用`tribe.*`来配置。它是种特殊的协调节点，用户连接2个不同的ES集群，实现跨集群查询。在7.x中设置Remote clusters相关配置即可实现跨集群搜索。在7.X高版本中（7.7+）还有`Transform node`，该类节点与是为了配合Transforming data使用，该功能是`x-pack`的特殊功能。

# 节点配置最佳实践

1. 小集群（10台以下），没有预处理操作的情况下，所有节点默认配置即可。
2. 如果集群中索引分片数据大，搜索时必须深分页，聚合为了保证正确性会取top N+X时，协调节点压力增大，建议设置仅做协调的节点。
3. 对于大集群可以设置至少3个节点仅作为`master-eligible node`以保证master节点稳定工作
4. 如果有大量pipeline操作，则应该设置节点仅有`ingest node`角色，单独处理pipeline操作。
5. 大量节点，一般情况下`data node`同时也当做`coordinating node`

