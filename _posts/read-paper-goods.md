---
title: [论文翻译]Goods: Organizing Google’s Datasets
date: 2022/09/19 19:33:10
categories:
- 论文

tags:
- 论文

keywords:
- 论文, Goods, 元数据管理, 元数据服务, 血缘关系, 数据集搜索
---
# 摘要

google针对公司多源、异构、海量数据集设计了“Goods”系统来管理。Goods收集了每个数据集重要的元信息以及血缘关系，并向公司内工程师提供：查找数据集、监控数据集以及对数据集行注释以使其他人能够更好的使用数据集分析它们之间的关系。论文主要讨论了抓取和推断数十亿数据集的元数据、保持大规模元数据目录的一致性以及向用户公开元数据所必须克服的技术挑战。

<!-- more -->
# 简介

目前现代大型企业通常不限制工程师对数据的使用，这样虽然能促进创新提升竞争力，但也会使数据集爆炸增长。面对海量的数据，企业需要有原则的数据管理手段来避免数据孤岛的产生，拖慢企业的研发效率。

企业数据管理 (EDM)是常见的数据集管理方式但不够灵活。Google提出让企业内完全自由地访问和生成数据集，以事后的方式解决寻找正确数据的理念并设计了Goods系统，它是一个事后（post-hoc）的数据资产管理系统，在不打扰工程师使用数据的前提下，非侵入的收集数据集元数据，并提供更高效的数据管理和查询服务。下图是Goods的基本架构。

![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/read-paper-1-1.png)

首先，Goods使用爬虫不断的从不同的数据集或基础设施中获取数据集诸如所有者、访问时间、内容特征、生产任务等信息。然后将有关联数据集的元数据合并为一个逻辑数据集，组织数据集最终聚合成1个中央数据目录。使用这个数据目录Goods提供了丰富的数据服务，具体如下：

对于数据集所有者提供：

1. 展示团队全部数据集的基本信息，即便在不同存储中
2. 展示这些数据集之间的关联关系
3. 监控数据集的大小、值分布等并可配置报警
4. 展示数据集上下游血缘

对于数据集使用者提供:

1. 公司级数据的搜素引擎，方便外部用户根据条件找到数据集。
2. 为每个数据集提供配置页面，帮助用户了解其架构并创建样板代码来访问或查询数据
3. 数据集详情页会根据该数据集展示相似数据集，这些数据集与本数据集相似或用共同主键可join组合出其他数据。

Goods提供的公共服务：

1. 作为共和交换数据集的信息枢纽。所有者可以添加描述注释，安全员可以标记敏感信息增加使用限制，使用者针对数据集提问。
2. 提供API，用于上传或获取元数据。

# 技术挑战

1. 数据集规模巨大。16年google内部数据集至少有260亿个，在这种规模下google并未选择收集全量的数据集，而是致力于为制定优先级以及优化处理流程。
2. 多样性。系统中的数据集存储格式及方式多样，元数据也都有不同的访问特性。但是抽象统一的元数据为用户隐藏数据集多样性，提供统一的查询方式方式是有必要且富有挑战的。同时，数据集间多样的关系（hive表与PARQUET文件）也会影响元数据的组织方式。
3. 数据集变更。在google每天有约10亿（5%）的数据集被创建或删除。这种情况下需要考虑哪些数据集的元数据值得被收录并计算。Goods并未直接放弃临时数据集的元数据，因为1. 刚创建的临时数据集元数据对用户意义很大；2. 临时数据与非临时数据经常会互相转换。
4. 元数据准确性。由于 Goods 以事后和非侵入性方式明确识别和分析数据集，因此元数据不完全准确。Goods坚持主动收集已经记录在现有基础设施不同角落的数据集元数据，而非改变现有DE的工作流程。
5. 计算数据集重要性。在收集元数据后，推断数据集对用户的重要程度是必要且有挑战的。数据集的重要性判断需要从整体着眼，无法使用网页搜索中对web重要性推断的方式处理数据集。、
6. 理解数据集语义。了解数据集语义对于搜索、排名和描述数据集非常有用，但从原始数据中识别语义十分困难，因为数据中很少有足够的信息来进行推断。

# Goods数据目录

从本章开始介绍Goods的实现细节。首先是系统核心——数据目录（catalog），它保存了从不同数据系统获取的数据集信息，构建全公司可用数据集的统一视图。对于具有相似元数据的多个数据（eg.周期产生新版本数据），Goods使用1个逻辑数据集统一表示，这样不但能方便用户认知还能优化计算。接下来两小节，首先描述每个数据集有哪些元数据，然后介绍逻辑数据集的元数据提取机制。

### 数据集元数据

数据集元数据中一小部分（数据集的大小、所有者、读者和访问权限）从存储系统（GoogleFS、Bigtable、数据库服务器）中爬取。而绝大多数元数据存储系统不关注，比如生成数据集的作业、数据集访问日志、Schema等。这些元数据从客户端日志、数据集内容或者数据库编码中分析得出。数据集具体元数据内容如下：

![img](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/read-paper-1-2.png)

* Basic Metadata ：基础元数据有时间戳、文件格式、所有者、权限等。这些元数据直接来源于存储系统，其他元数据的生成大多会依赖基础元数据。
* Provenance：血缘元数据维护了数据集的生产方式、使用方式、该数据集依赖于哪些数据集以及其他哪些数据集依赖于该数据集。这些信息来源于生产日志，可以帮助我们更好的理解数据集，了解数据在公司内如何流转。Goods在构建血缘时会注意关系生效的起止时间。针对海量的访问日志选择抽样处理并且仅计算上下有限跳数的血缘而非全部血缘。
* Schema：Schema是描述结构化数据的重要元数据。Google结构化数据大多不是自描述的，而是使用protocol buffers（PB）编码[1]存储，需要推断出将数据集与PB的对应关系。
* Content summary：Goods可以访问扫描所有数据集，因此也会分析数据摘要信息。主要分析内容如下：
  * 分析数据标识，使用HyperLogLog算法判断1个或多个属性的基数是否与行数是否一致，来推断寻找数据标识。
  * 使用LSH和checksums算法分析数据指纹，用于找到相似的数据集。
* User provided annotations ：所有者可以为数据集增加注释，注释可影响数据集搜索排名。
* Semantics：对于PB数据Goods通过分析源码注释丰富数据语义。或者用数据内容与Google知识图谱进行比对获取如地址、公司等实体。

除了上述元数据Goods还收集数据集的团队标识；所属项目；元数据修改记录。最后，Goods允许其他团队添加他们自己定义的元数据。


### 使用集群组织数据集

Goods数据目录中的25B数据并不是独立存在的。如果可以识别数据集所属的自然集群，那么不仅可以为用户提供逻辑抽象，还能加速元数据的收集，虽然可能略微损失元数据精度。例如可以将同一作业每天生成的数据集、相同数据集的不同版本、不同集群中的相同数据集作为一个集群。Goods仅收集集群中部分数据集的元数据，其他数据集共用这些元数据。如果用户对集群增加描述，它通常适用于集群中各个版本的数据集。

数据集合并为集群的方式不应该太复杂，不然会抵消掉使用集群减少而减少重复计算的收益。Google依据数据集路径划分集群，因为数据集的路径通常会嵌入的时间戳、版本等聚类标识符。例如对于每日生成的数据几，使用'/dataset/2015-10-10/daily_scan '作为路径。

Google用数据不同维度将数据集组织成半网格结构（emi-lattice structure），下图展示用产生日期和数据版本2个维度组织数据集。

![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/read-paper-1-3.png)

下表列出了Google组织数据集为半网格结构的所有维度。最终形成的路径的每个非叶子节点都可以作为1个集群。虽然Goods可以通过算法选择维度优化集群划分，但频繁的集群变动会使用户造成困惑，所以Goods仅在根目录创建1个实体保存在数据目录中。

![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/read-paper-1-4.png)

在确定集群后其元数据由个别下属数据集聚合形成，然后再传播给集群中每个数据集。数据集是否继承使用集群的元数据需要根据实际情况判断。下图是向集群传播owner属性的例子。

![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/read-paper-1-5.png)

# 后端实现


本小节主要介绍构建和维护数据目录（catalog）的细节，主要有目录数据的存储方式，相关批处理任务的实现，数据一致性以及容错性。

## 数据目录存储

Goods使用可伸缩的Key-value数据库——BigTable存储数据目录。每一行存储1个集群或1个数据集的数据并用路径作为key。BigTable提供行级事务，与业务操作多以数据集为粒度相符，可满足绝大多数一致性需求。

在物理层面上，1个元数据记录由多个列族构成。一般用于批处理计算的数据和用于前端业务展示的数据会存在不同的列族中，比如用于计算数据集血缘的原始数据，由于数据量巨大，会压缩后存储在单独列族中。

在BigTable里每行数据有2类元数据：上一节介绍的数据集元数据和状态元数据。状态元数据保存各种元数据生成任务的状态信息。任务状态信息包括时间戳、运行状态、错误信息等。这些信息可用于协调任务的执行以及检测任务执行的情况（已处理多少数据？执行错误有哪些？）。使用Bigtable的Temporal数据模型，可以保留历史的状态元数据，有助于了解模型过去执行过什么，方便debug。

## 批处理任务的性能及调度

Goods系统主要有2类服务：大量的批处理任务；少量支持前端或API的任务。Goods的分析模型，血缘产出任务，元数据收集任务都设计的易于扩展。这些批处理任务有的执行仅需几小时，而有的则需要几天，对于耗时长的批处理任务，会选择数据所在区域执行任务。

Goods不限制任务的运行顺序，可以随时停止下线。每个任务都包含1个或多个模块，模块间有依赖关系。比如列指纹模块的运行依赖于元数据采集模块的完成。Goods使用上文提到的标注在行粒度上的状态元数据协调各模块间运行。比如模块B执行依赖于A模块，那么B模块处理1行时会查看状态元数据关于A任务的记录，然后再决定是处理、跳过还是重复处理该行。模块也可以根据自己的状态元数据来避免在指定时间窗口内重复执行处理。

Goods中的任务大多数是每天执行且执行时间不会超过1天，如果有大型任务会用增加并发或任务拆分的方式降低执行时间。如果出现大量新数据集接入时，重量级的服务（ Schema分析任务）可能需要几周才能准备完成。对于这种情况Goods会开辟独立的队列执行高优数据集的任务，以确保任务依然在24小时内完成。数据集的优先级取决于源于用户标注或引用度。

Goods中的元数据爬虫任务使用盲写(blind write)的方式向数据目录添加数据，不区分新增还是更新。这种无指令(no-op)方式比读取老数据与新数据融合后再写入的反连接(anti-join)方式更高效。但需要注意，某些情况下无指令方式会造成任务重复运行或阻塞垃圾回收。

## 容错

Goods系统需要处理大量数据难免出现错误。当任务模块处理独立的数据集时，错误信息会记录在状态元数据中，错误数达到阈值前会自动重试。任务模块处理互相依赖的数据集是，错误信息会记录在任务(job)的状态元数据上。比只有当血缘解析任务比数据集产出任务完成时间更晚时，才会将关系纳入血缘图谱中。该方法较为保守，可能由于任务模块部分失败导致数据重新写入BigTable，但是该方法能够保证血缘图谱的正确性，BigTable写入幂等性保证重复写入不会造成太大影响。同时我们允许在血缘信息上标记“已消费”，方便后续垃圾回收。

一些检测数据集内容的解析任务模块会使用特殊的库，这些库不可避免的会出现崩溃或死循环的情况。然而系统不允许分析任务长期处于崩溃或挂起中，因此Goods用沙箱运行危险任务，并使用看门狗进程将死循环任务转变失败，以确保其他任务继续执行。

为了增加鲁棒性，数据目录会在不同的地理位置冗余数据。当数据写入master节点系统会在后台异步写入到其他集群。

## 垃圾回收

Goods系统每天会接受并生产大量瞬态(transient)数据。只要数据已被用于构建血缘关系相关元数据用于更新数据目录后，系统就会删除目录上对应数据集已经被删除的条目。系统初期使用简单且保守的回收方式，当记录1周内未更新时才会删除，但这使系统变得臃肿。最后Goods系统改用主动且激进的清理策略，避免系统必须停止所有工作专门进行垃圾回收。

已下是Goods系统在垃圾回收方面的一些经验总结：

1. 删除记录最好有明确的条件，条件谓词最好是会更新该记录的任务模块的状态或元数据。例如，当数据集已从存储系统删除且血缘任务已完成时Goods会删除数据集元信息。
2. 由于Bigtable不区分插入和更新，从数据目录中删除一条记录时，必须保证其它正在运行的模块不会将这条数据的部分信息写入目录，这种情况被称为"dangling rows"。
3. 所有其他模块不能依赖于垃圾回收模块，并且能和它同时运行。

由于BigTable支持"conditional mutations"，即支持当条件符合时按照事务原则删除或更新1行记录。任务模块更新BigTable时依赖记录未被删除会导致开销巨大，因为"conditional mutations"机制带来了极大的日志结构（log-structured）读取成本。GooDs允许垃圾回收以外的任务模块进行非事务(non-transactional)的更新。

垃圾回收删除数据分以下两步进行：

1. 通过条件判断出需要删除的数据，此时数据并没有真正删除，而是标记了一个墓碑(tombstone)。
2. 24小时之后，如果该行数据仍然符合删除标准，则真正删除，否则将墓碑移除。

其他模块遵从下列规则修改数据：

1. 可以非事务更新数据
2. 更新时忽略标记了墓碑的数据
3. 模块任务的单次执行不能超过24小时

# 前端功能

## 数据集详情页

详情页用于展示数据集的元数据信息。用户可以通过输入数据集或数据集集群的路径直接查看之前介绍的大部分元数据信息，同时也可以直接编辑部分元数据。详情页的元数据信息必须兼顾数据量和完整性。比如，为了不让用户被信息淹没，详情页会只展示压缩过得血缘信息或仅展示最新的血缘信息。

数据集详情页会插入其他工具的外链，比如详情页的血缘信息会链接到任务中心(job-centric tools)的任务详情页。同样，其他工具比如代码管理工具业务外链到数据集详情页。详情页还支持生成各种代码（C++、Java、SQL）片段，这些代码片段带有数据集schema等元数据信息，复制开发环境后就可轻松访问数据集内容。

总体而言，详情页目的是提供一站式商店，用户可以查看有关数据集的信息并了解可以在生产中使用数据集的上下文。

## 数据集搜索

搜索功能允许用户通过简单的关键词搜到数据集。该功能依赖常规的倒排索引实现文件检索。构建索引时，每个数据集会被当成一个”文档“，从元数据中生成索引词（index token），不同元数据生成文档不同属性的索引词。下图表示在不同属性搜索”a“词的含义。

![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/read-paper-1-6.png)

不同元数据在构建索引是需要考虑其业务场景。比如对于路径属性有部分匹配的查询需求。用户搜索'x/y'时希望'a/x/y/b'的被命中，而'a/y/x/b'的不被命中。为此Goods使用路径分割符切分路径属性并记录分割后关键词的顺序，查询时会以相同的方式解析部分路径，用连续匹配的方式查询路径索引。

用关键字找到数据集仅是搜索第一步，第二步是用打分函数为匹配的数据集排序，将用户可能需要的数据集排在前面。以下是设计打分函数的一些经验：

1. 数据集的重要性依赖于它的类型。比如Dremel数据集比文件数据集重要，因为Dremel表需要用户注册，使其对更多用户可见。
2. 不同属性排序权重不同。比如关键字命中路径属性的数据集应比命中生产任务的数据集排序靠前。
3. 可用血缘信息判断数据集重要性。通常情况下游任务越多的数据集越重要，但有些原始数据表（eg 爬虫数据）被大量引用，但对普通用户价值不高。
4. 有所有者描述信息的数据集重要性更高。

搜索优化是一个持续的过程。优化过程中理解影响数据集重要性的因素，有助于我们调整元数据爬取任务的优先级。除了关键字搜索外，Goods还支持按负责人、数据集类型搜索，用户可以基于上述结果再深入搜索。

## 团队看板

Goods看板用于可视化展示团队数据集中有趣的数据。比如，健康指标，存储引擎是否在线等。当更新仪表盘元数据时会自动更新仪表盘内容。用户可以轻松地将仪表板页面嵌入到其他文档中，并与他人共享仪表板。Goods 仪表板还能监控数据属性，如果超过预期还能发送报警（eg. 分片行数，列的值分布）。除了监控报警外，Goods系统还能够通过趋势分析，自动监控数据集的一些公共属性。比如基于数据集历史增长率，监控数据集大小是否符合预期。

# 经验总结

1. 随着使用不断演进(Evolve as you go)，起初Goods的主要目标是数据集发现，随着用户不断使用也发现以下需求：

   1. 安全审计。某些数据包含个人身份信息，使用时需要严格遵守访问策略。使用Goods可以轻松找到敏感数据集，并在违反策略时提醒数据所有者。
   2. 重新查到数据集(re-find datasets) ，工程师难免会忘记不常用数据集的路径，通过Goods则可以用简单的关键字轻松找到。
   3. 读懂老代码(understand legacy code) ，老代码因缺乏最新的文档而难读，工程师可以通过Goods血缘图谱提供的历史输入输出数据集反推老代码的逻辑。
   4. 数据集书签（Bookmark datasets），数据集详情页能一站式分享数据集信息，用户可以轻松访问并将数据集分享给其他人。
   5. 注释数据集（Annotate datasets），Goods可充当可跨团队共享的数据集注释的中心。比如标记隐私等级，告诫工程师合规使用。
2. 排序中使用领域信息（Use domain-specific signals for ranking）。通过之前的介绍不难发现，数据集搜索与网页搜索有很大区别。根据Good统的经验，数据集的血缘关系为排序提供了领域信息。比如，一些团队会基于主数据生成不同版本的非规范数据以便应对不同场景。非规范数据与主数据关键词相似，但搜索时主数据应排名更高。如果数据集的产出依赖于另一个团队的数据集，那么外团队数据集会更加重要。
3. 预见并处理特殊数据集（Expect and handle unusual datasets），由于数据目录中数据量巨大，早期遇到过不少意想不到的情况。Goods系统处理非常规数据集的策略：首先提供最简单的特定问题解决方案，接着在必要时归纳出同一类问题的总体解决方案。
4. 按需存储数据（Export data as required），起初Goods使用key-value数据库存储数据目录，倒排索引支持搜索服务，为了支持血缘展示及搜索，Goods又引入了图数据库。如果现有存储不能支持用户需求，Goods的经验是引入新的合适的存储引擎。
5. 确保可恢复（Ensure recoverability ），提取数以十亿计数据集的元数据计算成本非常高，稳定状态下Goods每天只处理一天量的新数据集。丢失或损毁数据目录的重要部分可能需要数周恢复甚至无法恢复。为了保证可恢复，Goods不但设置Bigtable滚动快照保留期为几天，也制定了如下专门的数据恢复方案。1. 为高优数据创建单独的快照；2. 维持备用数据目录服务，数据是主服务的子集以预防主服务掉线；3. 数据目录使用数据集监控服务，保证尽早检测到数据损毁或丢失。

# 相关研究

Goods系统可以看做一个数据湖，一种用于存储海量数据且访问便捷的仓库，数据在生成时无需预先分类。Goods可以看做Google关于数据空间【1】的具体实现。其他公司也有类似实现，比如IBM的数据湖管理系统【2】、云上数据湖服务【3】。

DataHub【4，5，6】是数据集的版本控制系统类似GIT，许多用户能够在集中式存储库中分析、协作、修改和共享数据集。CKAN【7】, Quandl 【8】和 Microsoft Azure Marketplace【9】可管理多个数据源的数据且有组织和分发的功能。这些系统需要所有者主动上报数据并贡献标注信息。Goods系统采用了post-hoc方式生成目录，工程师在生成和维护数据集的时候无需考虑GooDs系统的存在。

有一些系统【10，11，12】可以从html页面中提取结构化数据并提供搜索服务。这些系统主要工作在解析网页数据并且没有血缘、所有者等信息，而Goods系统的侧重点则正好相反。

Spyglass【13】和Propeller【14】作为存储系统并提供了丰富且高效的文件搜索能力。由于Goods中数据集来源于不同的存储系统现成元数据有限，搜索也面临巨大挑战。但Goods的定位并不仅仅是搜索系统，而是数据集管理系统。历史研究【15，16】讨论过大规模结构化数据的索引和搜索问题，但无法较好解决技术挑战提到的问题。

文献【17】对血缘管理有大量研究。PASS和Trio等系统也会维护血缘关系，但是系统假设血缘信息是已知的或者能够从访问数据的过程中分析出。而Goods系统需要通过信息分析出来，并且还要分析出关系的强弱。

# 总结和展望

本位主要介绍了Google数据管理系统，它可访问企业内部数以十亿计的数据集的元数据。依然有需要挑战有待解决：

1. 如何做数据集搜素排序？如何鉴别重要数据集？
2. 补充完善数据集元数据信息。考虑使用知识图片技术，将用代码、团队信息等完善元数据。
3. 理解数据集业务语义信息，帮助用户搜索数据集
4. 调整Goods存储系统，使用主动上报和被动发结合的方式收集元数据。

最后希望类似Goods数据管理系统能够帮助企业实现数据驱动，将数据当作企业核心资产，发展出类似"代码规约"的"数据规约(data discipline)"。

# 引用文献

【1】M. Franklin, A. Halevy, and D. Maier. **From databases to dataspaces: A new abstraction for information management**. SIGMOD Rec., 34(4):27–33, Dec. 2005.

【2】I. Terrizzano, P. M. Schwarz, M. Roth, and J. E. Colino. **Data wrangling: The challenging journey from the wild to the lake**. In CIDR 2015, Seventh Biennial Conference on Innovative Data Systems Research, Asilomar, CA, USA, 2015

【3】Azure data lake. https://azure.microsoft.com/en-us/solutions/data-lake/.

【4】Data lakes and the promise of unsiloed data. http://www.pwc.com/us/en/technology-forecast/2014/cloud-computing/features/data-lakes.html.

【5】Quandl. https://www.quandl.com.

【6】A universally unique identifier (uuid) urn namespace. https://www.ietf.org/rfc/rfc4122.txt

【7】CKAN. http://ckan.org.

【8】Quandl. https://www.quandl.com.

【9】Azure marketplace. http://datamarket.azure.com/browse/data.

【10】M. J. Cafarella, A. Y. Halevy, D. Z. Wang, E. Wu, and Y. Zhang. **Webtables: exploring the power of tables on the web**. PVLDB, 1(1):538–549, 2008.

【11】M. Yakout, K. Ganjam, K. Chakrabarti, and S. Chaudhuri. **InfoGather: entity augmentation and attribute discovery by holistic matching with web tables**. In K. S. Candan, Y. Chen, R. T. Snodgrass, L. Gravano, and A. Fuxman, editors, SIGMOD Conference, pages 97–108. ACM, 2012.

【12】S. Balakrishnan, A. Y. Halevy, B. Harb, H. Lee,J. Madhavan, A. Rostamizadeh, W. Shen, K. Wilder, F. Wu, and C. Yu. **Applying webtables in practice**. In CIDR 2015, Seventh Biennial Conference on Innovative Data Systems Research, Asilomar, CA, USA, 2015.

【13】A. W. Leung, M. Shao, T. Bisson, S. Pasupathy, and E. L. Miller. **Spyglass: Fast, scalable metadata search for large-scale storage systems**. In M. I. Seltzer and R. Wheeler, editors, FAST, pages 153–166. USENIX, 2009.

【14】L. Xu, H. Jiang, X. Liu, L. Tian, Y. Hua, and J. Hu. **Propeller: A scalable metadata organization for a versatile searchable file system**. Technical Report 119, Department of Computer Science and Engineering, University of Nebraska-Lincoln, 2011.

【15】P. Rao and B. Moon. **An internet-scale service for publishing and locating xml documents**. In Proceedings of the 2009 Int’l Conference on Data Engineering (ICDE), pages 1459–1462, 2009.

【16】I. Konstantinou, E. Angelou, D. Tsoumakos, and N. Koziris. **Distributed indexing of web scale datasets for the cloud**. In Proceedings of the 2010 Workshop on Massive Data Analytics on the Cloud, MDAC ’10, pages 1:1–1:6, 2010.

【17】J. Cheney, L. Chiticariu, and W.-C. Tan. **Provenance in databases: Why, how, and where. Found**. Trends databases, 1(4):379–474, Apr. 2009.

【18】K.-K. Muniswamy-Reddy, D. A. Holland, U. Braun, and M. Seltzer. **Provenance-aware storage systems**. In Proceedings of the Annual Conference on USENIX ’06 Annual Technical Conference, pages 43–56, 2006.

【19】J. Widom. **Trio: A system for integrated management of data, accuracy, and lineage**. In CIDR, pages 262–276, 2005.
