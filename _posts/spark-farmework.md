---
title: spark学习————spark架构
categories:
- spark
tags:
- spark架构
---
# 概述

本文首先从spark对MapReduce的改进开始介绍spark的主要特点。然后罗列了spark的重要概念。接着从集群结构，编程模型以及计算模型概述spark。最后列举spark代码实现的核心模块及其功能概述。本文参考的spark版本为v2.1.0。
<!--more-->

# spark与MapReduce
首先spark依旧是MapReduce模式下的分布式计算框架。其主要是在实现细节上对MR进行了丰富与改进。
## MR的局限
1、频繁I/O引发的性能瓶颈。由于MapReduce对HDFS的频繁操作（包括计算结构持久化、数据备份、资源下载），导致磁盘I/O成为系统性的能主要瓶颈。  

2、MR过于底层，模型通用但单一。虽然MR一招可以应对绝大多数分布式计算，但是这也造成了对应常见场景已经实现困难的问题，比如一个简单的查询也需要组织编写Map和reduce函数。

3、部分新场景不能很好的支持。比如以及机器学习为代表的迭代计算和流式计算。

## Spark的改进
1、减少磁盘I/O。spark一方面运行在进行map计算时将中间输出和结构保存在内存总，另一方面在进行reduce拉取中间结果时避免了大量的磁盘I/O操作。

2、增加并行度。在有向无环图DAG的帮助下spark将任务切分成多个stage，stage间即可以并行计算也可以串行计算。

3、避免重新计算。当Stage中某个partition的task失败时，会重新对这个stage进行调度，但在重新计算这个stage时成功的partition将不再运行。同时spark维护了RDD之间的血缘关系，一旦某个RDD失败则可以由父RDD重建。虽然可以重建，但如果血缘过长恢复将非常耗时。因此spark支持设置检查点，将计算的中间结果保存，如果RDD失败可以不用从头计算，而是从检查点开始计算。

4、灵活的内存管理策略。Spark将内存分为堆上的存储内存、堆外的存储内存、堆上的执行内存以及堆外的执行内存。执行内存和存储内存边间边界不定，可根据情况互相借用内存。由于Spark是基于内存的，所以对内存资源管理的好坏直接决定了spark的性能。为此spark的内存管理器借助Tungsten实现了一种与操作系统的内存Page非常相似的数据结构。用于直接控制操作系统内存，节省了创建JAVA对象在堆中占用的内存，使得spark对内存的使用更接近硬件。同时，spark对内存的管理是task级别的，即为每个task都分配1个任务内存管理器。

5、易于使用。spark在基本的RDD计算模型上又进行了封装。使其可以支持SQL计算、流式计算、迭代计算等。

# spark基础知识
## 基本概念
**RDD(resillient distributed dataset)**:弹性分布式数据集。用户操作的数据集在spark中被抽象为RDD。Spark应用程序通过使用transform API，可以将RDD封装为一系列具有血缘关系的RDD即DAG。然后通过spark的action API将RDD及DAG提交到DAGScheduler实际执行全部计算。 

**DAG(Directed Acycle graph)**:有向无环图。根据图论定义如果一个有向图无法从某个顶点出发经过若干条边回到该点，则这个图是一个有向无环图。spark根据RDD的有向无环图构建它们之间的血缘关系。

**Partition**:数据分片。RDD(逻辑上的一组数据)实际上被划分成了多少个分片。spark会根据partition的数量来确定每个stage需要有多少个task。

**NarrowDependency**:窄依赖，即子RDD依赖于服RDD中固定的Partiton。窄依赖分为OneToOneDependency和RangeDependency。

**ShuffleDependency**:shuffle依赖，也称宽依赖，即子RDD对父RDD中的所有partition都可能产生依赖。这是子RDD对父RDD各个partition的依赖取决于分区计算器的算法。

**Job**: 由多个task组成的并行计算，是用户提交的作业。一个Job一般由多个transform操作和1个action操作(位于最后)组成。transform操作不会实际运算主要是构建RDD之间的血缘，action操作则会实际在集群中执行计算并产生结果。

**stage**: Job的执行阶段。stage的创建是DAGScheduler根据shuffle依赖破坏DAG图来实现的。按照shuffle依赖来划分stage依赖代表，我们必须等待上一个stage对于全部partition都计算完成才能进行下一个stage。Spark将stage划分为2种类型：ResultStage，用于执行action操作的最终的stage。ShuffleMapStage，会由shuffler操作输出映射文件。如果job重复使用相同的RDD，则stage会在多个job之间共享。

**task**: 具体发送给executor执行的任务。一个job在每个stage内都会按照RDD的Partition数量创建多个task。Task分为ShuffleMapTask和ResultTask两种，分别是ShuffleMapStage和ResultStage产生的task。

**shuffle**: Shuffle操作是所有MapReduce计算框架的核心部分。Shuffle用于连接Map任务的输出和Reduce任务的输入。在spark中，shuffle操作（e.g groupByKey）可以理解为，被分成了由上游stage占有的map任务，和由下游stage含有的reduce任务。Map任务的输出结果按照指定分片策略将数据分区，并告诉driver自己的数据有哪些key(向driver发送MapStatus)，reduce任务会向driver询问自己所需key的数据在哪(自己所需key的MapStatus)，然后想相应的Map任务拿对应的数据。

# 集群结构
**Application**: 用户编写的spark应用程序，最终一般以jar包的相识提交给spark集群。

**driver进程**: 执行Application的mian方法的进程，并且负责初始化SparkContext。创建SparkContext就是初始化spark任务运行的集群环境。最主要是向clusterMananger申请计算资源，创建管理计算资源的对象。

**clusterManager服务**: clusterManager是用于管理集群计算资源的外部服务，常见的有:Spark standalone manager, Mesos, YARN，这也是spark集群常用的部署方式。

**worker节点**: 在集群中任何可以跑Application代码的节点都可以算作worker节点，是被clusterManager管理的计算机。

**executor进程**: 在worker节点上启动的用于执行Application的进程，该进程运行任务并将数据保存在内存和磁盘中。每个Application都有自己的executors。Executor需要维护一个线程池来运行tasks 

![spark集群简图](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/spark%E9%9B%86%E7%BE%A4%E9%97%B4%E5%9B%BE.jpg)

# 核心编程模型
首先用户需要使用SparkContext提供的API编写spark应用程序(Application)。经过spark近些年的发展，spark还提供了SparkSession、DataFrame、SQLContext、HiveContext以及StreamingContext。这些都是对sparkContext的封装，并提供了更加丰富的功能。  

用户使用`spark-submit`将spark应用程序提交给driver，并由driver使用反射机制来执行。spark应用程序首先通常会进行sparkContext的初始化。初始化的主要任务是通过根据sparkConf向clusterManager注册Application，并向其申请计算资源。clusterManager根据Application的需求，给application分配Executor资源，即在worker节点上启动CoarseGrainedExecutorBackend进程。该进程内部会创建Executor，在创建Executor的过程中会向driver进行反向注册，driver中的taskScheduler将维护分配到的executor的信息。  

在初始化完运行环境后，sparkContext会根据用户使用的各种transform算子构建RDD之间的血缘关系和DAG，当使用到action算子时，DAGScheduler将会把构建的DAG创建成JOB，并根据RDD的依赖关系将DAG划分为不同的stage。DAGScheduler根据stage内RDD中partition的数量创建多个task，并批量的交给TaskScheduler，尤其根据FIFO或FAIR调度算法将task分配给不同的executor。由executor结合其所有的partition执行task从而完成分布式计算。  

另外为了任务能更好的运行，sparkContext在RDD转换开始前会用BlockManager和BroadcastManager将重要的变量进行广播，比如hadoop集群配置。同时sparkContext还可以对任意RDD进行持久化，以减少计算失败时重新计算的时间。目前spark支持使用HDFS、Amazon S3等作为存储。

# RDD计算模型
RDD是spark定义的数据抽象。Spark的计算过程主要是RDD的迭代计算的过程。用户使用RDD感觉上是一个完整的集合，但这个数据集合其实是分散在spark集群中的，数据分散的最小单位是partition，每个partition数据只会在一个task中计算。下图展示了1个spark应用中数据大致的流转情况。

![spark集群简图](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/spark%E8%AE%A1%E7%AE%97%E6%A8%A1%E5%9E%8B.png) 

在上图中有7个RDD分别是A-G。每个RDD中的小方框代表了1个partition。根据stage划分算法，他们被分成了3个stage。因为groupBy和join算子是存在宽依赖的算子。其中stage 1、stage 2是ShuffleMapStage，最后join并将数据保存到HDFS的stage 3是ResultStage。

# Spark核心模块

**SparkEnv**: sparkEnv根据sparkConf构建的上spark执行环境，它包含了spark task运行的各个组件。它内部封装了RPC环境（RpcEnv），序列化管理器，广播管理器（BroadcastManager），map任务输出跟踪器（MapOutputTracker），存储体系，度量体系（MetricsSystem）等。SparkEnv是在SparkContext初始化时构建的，并会被传输到各个task中。


**RPC框架**: spark内置的RPC框架使用netty实现，有同步和异步的多种实现，Spark各组件间的通信都依赖RPC框架。


**事件总线**: ListenerBus是sparkContext内部各组件间使用事件——监听器模式异步调用的实现。


**度量系统**: MetricsSystem由spark中的多种度量源（source）和多种度量输出（sink）构成，完成对spark集群中各个组件运行期状态的监控。  


**Web UI**: 在spark初始化时启动的用于监控应用执行状态的可视化界面。它的数据来源主要是度量系统。  


**存储体系**: 为了减少磁盘I/O，spark优先使用内存作存储，如果内存不够会是有磁盘。但是由于有些task是存储密集型有些是计算密集型，所以存储体系需要灵活界定内存存储空间与执行存储空间之间的边界，使资源高校利用。此外spark借助Tungsten实现了直接操作操作系统内存，省去了在JVM 堆内分配java对象。  


**调度系统**: 调度系统主要有DAGScheduler和TaskScheduler组成。DAGScheduler负责创建JOB，将DGA中的RDD划分成不同的stage，给stage创建对应的task，提交task给TaskScheduler。TaskScheduler负责按照FIFO或FAIR等调度策略对Task进行调度，将Task发送到应用所拥有的executor上。  


**计算引擎**: 计算引擎Executor主要由管理内存的内存管理器（MemoryManager）、任务内存管理器（TaskMemoryManager），管理shuffle的外部排序器（ExternalSorter）、Shuffle管理器（ShuffleManager）以及其他一些组件组成。MemoryManager主要是对存储体系中的存储内存和执行内存提供支持及管理。TaskMemoryManager是对分配给单个task的内存资源进行更细粒度的管理和控制。ExternalSorter用在shuffle操作的map和reduce端，对shuffleMapTask计算得到的中间结果进行排序、聚合等操作从而达到优化shuffle的目的。ShuffleManager用于将各个partition对应的shuffleMapTask产生的中间结果持久化到磁盘，并在reduce端按照partition远程拉取shuffleMapTask产生的中间结果。  

# 结束语
作者计划阅读spark各个核心模块的代码，并作简要介绍，敬请期待。本文主要参考《spark内核设计的艺术》。