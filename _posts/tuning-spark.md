---
title: Spark调优(v1.6.0)
categories:
  - spark
tags:
  - 翻译
---
# 前言
最近在学习Spark调优，发现官方文档写的很详细，但网上的译文多有不准，因此本人对Spark 1.6.0版本（工作使用的是1.6版本，现在spark已出到V2.3.0）的文档进行了翻译。一来方便各位自己查阅，二来通过翻译加强理解。关于翻译，部分地方没有逐字翻译而是采用了意译，文中斜体部分是本人额外添加内容。本人水平有限如有差错还望各位批评指正。
为了方便伸手党，现将所有要点总结如下：
1. spark是基于内存的分布式计算框架。比MR快但是不如MR稳定。为了其稳定且更快需要更高效的使用内存资源，官方建议从数据序列化和内存调优入手。
2. 网络传输数据，shuffle操作，持久化等都会触发数据序列化行为。
3. 在SparkConf将序列化方式调为kryo，即设`spark.serializer`为`org.apache.spark.serializer.KryoSerializer`
4. 在使用kryo时自定义类需要提前注册。`conf.registerKryoClasses(Array(classOf[MyClass1]))`
5. 尽量使用原始类型变量少用引用类型变量，如用数组代替List，用JSON代替Map，使用fastutil提供的容器。
6. 在 spark-env.sh增加JVM参数`-XX:+UseCompressedOops` 进行指针压缩。
7. 对于1.6.0之前的版本减少存储内存区域的大小（`spark.storage.memoryFraction`）增大计算内存（从而可以增大GC使用的空间）。1.6.0之后版本降低`spark.memory.storageFraction`来实现同样的效果。
8. 对重复使用的RDD做好持久化，要使用带序列化的持久化级别，特别重要的RDD不但要持久化还要设置`checkpoint`。
9. 在使用checkpoint前必须要持久化（考虑使用`MEMORY_AND_DISK_SER`）避免重复计算RDD。
10. 对于巨大的RDD使用`MEMORY_ONLY_SER`级别的持久化。
11. 增加`-verbose:gc -XX:+PrintGCDetails -XX:+PrintGCTimeStamps`参数关注GC情况或在SparkUI中观察。
12. 特别注意java GC（minor gc）时会阻塞所有task，如果频繁GC会导致shuffle读操作一直失败，出现shuffle临时文件找不到的情况。
13. spark关于GC的调优核心思路是不让临时的RDD进入老年代。
14. 频繁minor GC但没发生多少次major GC时为Eden分配更多内存。
15. 通过`-XX：+ UseG1GC`配置使用G1 GC。
16. 适当增大的并行度，推荐每个CPU核上应并性运行2-3个task。
17. 将大变量（如配置文件）通过广播的方式共享，这样同一个executor只会保留1份数据否则每个task都会保存一份数据。
18. 通过`spark.locality.wait`适当增加task分配时的超时时间，以获取更高的本地级别。
<!--more-->
# 正文及翻译
## 前言
> Because of the in-memory nature of most Spark computations, Spark programs can be bottlenecked by any resource in the cluster: CPU, network bandwidth, or memory. Most often, if the data fits in memory, the bottleneck is network bandwidth, but sometimes, you also need to do some tuning, such as storing RDDs in serialized form, to decrease memory usage. This guide will cover two main topics: data serialization, which is crucial for good network performance and can also reduce memory use, and memory tuning. We also sketch several smaller topics.

由于Spark大多数计算是基于内存的性质，Spark程序在集群中可能因为如下原因而产生瓶颈：CPU，网络带宽，内存。通常，数据存储在内存，那么网络带宽是瓶颈，但是，有时候，你还是需要做一些优化，比如为了减少内存使用量而序列化存储RDD。本值南将涵盖两大主题：数据序列化，内存调优。数据序列化可以有效提高网络性能并且减少内存使用量。对于调优我们还提出了几个较小的主题。
## 数据序列化
>Serialization plays an important role in the performance of any distributed application. Formats that are slow to serialize objects into, or consume a large number of bytes, will greatly slow down the computation. Often, this will be the first thing you should tune to optimize a Spark application. Spark aims to strike a balance between convenience (allowing you to work with any Java type in your operations) and performance. It provides two serialization libraries:
- Java serialization: By default, Spark serializes objects using Java’s ObjectOutputStream framework, and can work with any class you create that implements java.io.Serializable. You can also control the performance of your serialization more closely by extending java.io.Externalizable. Java serialization is flexible but often quite slow, and leads to large serialized formats for many classes.
- Kryo serialization: Spark can also use the Kryo library (version 2) to serialize objects more quickly. Kryo is significantly faster and more compact than Java serialization (often as much as 10x), but does not support all Serializable types and requires you to register the classes you’ll use in the program in advance for best performance.


对于任何分布式应用性能的提高，Serialization都起这至关重要的作用。使用低效率的对象系列化方式，或者消耗大量的字节，都会大大降低计算速度。通常，这是优化Spark需要干的第一件事。Spark旨在在便利性（在spark中使用任意JAVA类型）和性能之间取得平衡。它提供了两个序列化库：
- Java Serialization: 默认使用，Spark序列化对象使用java 的ObjectOutputStream框架，它可序列化任何自定义的类，只要这个类实现了java.io.Serializable 。你还可以通过扩展java.io.Externalizable来更加有效的控制序列化的性能。java 序列化很灵活，但是经常很慢并且很多类序列化后的格式很大。
- Kryo Serialization: Spark还可以使用更快速的Kryo库（Version 2）来序列化对象。Kryo比Java serialization更快（快10倍），(序列化后的对象)更紧凑，但是不能序列化所有类型，并且为了更高的性能你需要把将要使用的类型提前向Kryo注册。

> You can switch to using Kryo by initializing your job with a SparkConf and calling conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer"). This setting configures the serializer used for not only shuffling data between worker nodes but also when serializing RDDs to disk. The only reason Kryo is not the default is because of the custom registration requirement, but we recommend trying it in any network-intensive application. Since Spark 2.0.0, we internally use Kryo serializer when shuffling RDDs with simple types, arrays of simple types, or string type.

> Spark automatically includes Kryo serializers for the many commonly-used core Scala classes covered in the AllScalaRegistrar from the Twitter chill library.

> To register your own custom classes with Kryo, use the registerKryoClasses method.

在使用SparkConf初始化你的job时你可以将序列化方式切换为Kryo，只需在SparkConf对象中增加`conf.set("spark.serializer","org.apache.spark.serializer.KryoSerializer")`。这种序列化对象的方式不仅会用在两个work节点间传输shuffle数据，同时还会用在RDD数据需要持久化到硬盘时。Kryo不是默认设置的原因仅是因为自定义类需要自己注册，但我们建议在任何网络密集型应用程序中都尝试它。从Spark 2.0.0开始我们在shuffle RDD（该RDD使用的是简单数据结构，简单的数组，或是字符串）时尝试在内部使用Kryo的序列化方式。
Spark会自动包含许多常用Scala核心类的Kryo序列化器，这些类包括Twitter chill 库的AllScalaRegistrar。
使用registerKryoClasses方法向Kryo注册你的自定义类。
```scala
val conf = new SparkConf().setMaster(...).setAppName(...)
				.set("spark.serializer","org.apache.spark.serializer.KryoSerializer")
conf.registerKryoClasses(Array(classOf[MyClass1], classOf[MyClass2]))
val sc = new SparkContext(conf)
```
> The Kryo documentation describes more advanced registration options, such as adding custom serialization code.

> If your objects are large, you may also need to increase the spark.kryoserializer.buffer config. This value needs to be large enough to hold the largest object you will serialize.

> Finally, if you don’t register your custom classes, Kryo will still work, but it will have to store the full class name with each object, which is wasteful.

[Kryo文档](https://github.com/EsotericSoftware/kryo)介绍了更多高级注册选项，例如添加自定义序列化代码。
如果你的对象很大，你可能还需要增加spark.kryoserializer.buffer配置。该值需要足够大以容纳您要序列化的最大对象。
最后，如果你没有注册你的自定义类，Kryo仍然可以工作，但它将不得不为每个对象存储完整的类名，这是十分浪费资源的。

## 内存优化
> There are three considerations in tuning memory usage: the amount of memory used by your objects (you may want your entire dataset to fit in memory), the cost of accessing those objects, and the overhead of garbage collection (if you have high turnover in terms of objects).

> By default, Java objects are fast to access, but can easily consume a factor of 2-5x more space than the “raw” data inside their fields. This is due to several reasons:
> - Each distinct Java object has an “object header”, which is about 16 bytes and contains information such as a pointer to its class. For an object with very little data in it (say one Int field), this can be bigger than the data.
> - Java Strings have about 40 bytes of overhead over the raw string data (since they store it in an array of Chars and keep extra data such as the length), and store each character as two bytes due to String’s internal usage of UTF-16 encoding. Thus a 10-character string can easily consume 60 bytes.
> - Common collection classes, such as HashMap and LinkedList, use linked data structures, where there is a “wrapper” object for each entry (e.g. Map.Entry). This object not only has a header, but also pointers (typically 8 bytes each) to the next object in the list.
> - Collections of primitive types often store them as “boxed” objects such as java.lang.Integer. 

> This section will start with an overview of memory management in Spark, then discuss specific strategies the user can take to make more efficient use of memory in his/her application. In particular, we will describe how to determine the memory usage of your objects, and how to improve it – either by changing your data structures, or by storing data in a serialized format. We will then cover tuning Spark’s cache size and the Java garbage collector.

这里有三条内存调优的注意事项：对象使用的内存量（您可能希望整个数据集放入内存），访问这些对象的成本以及垃圾回收的开销（if you have high turnover in terms of objects）。
默认情况下，Java对象访问速度很快，但是相较于原数据类型其消耗的空间要多2-5倍。这是由于一下几个原因：
- 每个不同的Java对象都有个"对象头"，它约有16byte（32-bit JVM 上占用 64bit， 在 64-bit JVM 上占用 128bit 即 16 bytes不考虑开启压缩指针的场景）并且包含例如指针的信息（文末有对象头的结构）。
- Java字符串在原始字符串数据上有大约40个bytes的开销（因为它们将它存储在一个Chars数组中，并保留额外的数据，如长度），并且由于String的内部使用UTF-16编码，将每个字符存储为两个字节。 因此一个10个字符的字符串可以很容易地消耗60 bytes。
- 常见集合类（如HashMap和LinkedList）使用链表数据结构，其中每个条目都有一个“wrapper”对象（例如Map.Entry）。 这个对象不仅有一个头，而且还有指向列表中下一个对象的指针（通常每个8个字节）。
- 基本类型的集合通常将源数据存储为“boxed”对象，如int一般会转为java.lang.Integer

本小节首先概述spark的内存管理，接着讨论用户可以在应用程序中更有效地使用内存的具体策略。具体的，我们将介绍如何确定对象的内存使用情况，以及如何改进它——可以通过改变对象的数据结构，或使用序列化格式存储数据。接下来我们将介绍调整Spark的缓存大小和Java垃圾回收器。
### 内存管理概述
> Memory usage in Spark largely falls under one of two categories: execution and storage. Execution memory refers to that used for computation in shuffles, joins, sorts and aggregations, while storage memory refers to that used for caching and propagating internal data across the cluster. In Spark, execution and storage share a unified region (M). When no execution memory is used, storage can acquire all the available memory and vice versa. Execution may evict storage if necessary, but only until total storage memory usage falls under a certain threshold (R). In other words, R describes a subregion within M where cached blocks are never evicted. Storage may not evict execution due to complexities in implementation.

> This design ensures several desirable properties. First, applications that do not use caching can use the entire space for execution, obviating unnecessary disk spills. Second, applications that do use caching can reserve a minimum storage space (R) where their data blocks are immune to being evicted. Lastly, this approach provides reasonable out-of-the-box performance for a variety of workloads without requiring user expertise of how memory is divided internally.

> Although there are two relevant configurations, the typical user should not need to adjust them as the default values are applicable to most workloads:
> - `spark.memory.fraction` expresses the size of M as a fraction of the (JVM heap space - 300MB) (default 0.75). The rest of the space (25%) is reserved for user data structures, internal metadata in Spark, and safeguarding against OOM errors in the case of sparse and unusually large records.

> - `spark.memory.storageFraction` expresses the size of R as a fraction of M (default 0.5). R is the storage space within M where cached blocks immune to being evicted by execution.

> The value of spark.memory.fraction should be set in order to fit this amount of heap space comfortably within the JVM’s old or “tenured” generation. See the discussion of advanced GC tuning below for details.

Spark中内存的使用大致分为两类：execution(执行)和存储(storage)。execution内存指的是shuffles、joins、sorts和aggregations计算时使用的内存*（个人理解该内存存放了运行时动态创建的对象）*。storage内存则是指用于跨集群缓存和传播内部数据的内存*（个人理解该部分内存主要放的是RDD持久化中需要放入内存的那部分数据）*。在Spark中，execution和storage共享统一区域（M）。当execution内存没有被使用时，storage可以全部可用内存，反之亦然。如果有必要execution可能会回收(evict) storage, 但是仅限于总storage内存使用量低于某个阈值（R）。另一方面，R描述了M中的一个子区域，其中缓存的block不会被回收。由于执行的复杂性，storage可能不会回收execution。
*(Spark从1.6开始才开始模糊execution和storage内存，之前的版本需要明确的划分各部分占用的比例，默认shuffle.memory 0.2 storage.memory 0.6 storage.unroll 0.2)*
这种设计确保了几种理想的性能。首先，不使用缓存*（RDD持久化）*的应用executor可以占用全部内存空间，避免不必要的磁盘泄露。其次，使用缓存的应用可以保留最低限度的storage（R），storage中的数据block不会被回收。最后，这种方法为各种工作负载提供了合理的开箱即用性能，而不需要用户在内部划分内存的专业知识。
虽然有两种相关配置，但用户一般不需要调整它们，因为默认值适用于大多数工作负载：
- `spark.memory.fraction` 用分数表示execution和storage占用内存的比例，默认0.75（JVM 堆内存 -300M）。其余空间（25%）是为用户数据结构，Spark中内部元数据以及作为一个预防因超大对象而导致OOM的缓冲而预留的。
- `spark.memory.storageFraction` 用M的一个比例表示R的大小，默认是0.5。R的大小即是storage空间保持最小的不会被抢占的阈值。

### 确定内存消耗
> The best way to size the amount of memory consumption a dataset will require is to create an RDD, put it into cache, and look at the “Storage” page in the web UI. The page will tell you how much memory the RDD is occupying.

> To estimate the memory consumption of a particular object, use SizeEstimator’s estimate method This is useful for experimenting with different data layouts to trim memory usage, as well as determining the amount of space a broadcast variable will occupy on each executor heap.

最好的测量数据集消耗内存大小的方式是，创建一个RDD将他持久化到内存中，通过sparkUI的“Storage”页面查看。这个页面会告诉你这个RDD的缓存占用了多少内存。
要估计特定对象的内存消耗，请使用SizeEstimator的估计方法。这是个有效地估计不同数据结构并优化内存的方法，以及确定广播变量将占用每个executor堆内存大小。

### 优化数据结构
> The first way to reduce memory consumption is to avoid the Java features that add overhead, such as pointer-based data structures and wrapper objects. There are several ways to do this:

> 1. Design your data structures to prefer arrays of objects, and primitive types, instead of the standard Java or Scala collection classes (e.g. HashMap). The fastutil library provides convenient collection classes for primitive types that are compatible with the Java standard library.
> 2. Avoid nested structures with a lot of small objects and pointers when possible.
> 3. Consider using numeric IDs or enumeration objects instead of strings for keys.
> 4. If you have less than 32 GB of RAM, set the JVM flag `-XX:+UseCompressedOops` to make pointers be four bytes instead of eight. You can add these options in spark-env.sh.

减少内存消耗的第一种方法是避免使用增加Java开销的功能，例如引用类型的对象和wrapper对象。 做这件事有很多种方法：
1. 优先使用原始数据类型以及数组，避免使用java或scala的集合类（比如hashMap）。fastutil库为原始数据类型(兼容java标准库)供了方便的的集合类。
2. 避免嵌套结构
3. 使用数字ID或枚举类型而不是用字符串做key
4. 对于RAM在32gb以下的机器，添加JVM参数 `-XX:+UseCompressedOops` 进行指针压缩，把8-byte指针压缩成 4-byte。可在spark-env.sh中配置。

### 序列化RDD存储
> When your objects are still too large to efficiently store despite this tuning, a much simpler way to reduce memory usage is to store them in serialized form, using the serialized StorageLevels in the RDD persistence API, such as `MEMORY_ONLY_SER`. Spark will then store each RDD partition as one large byte array. The only downside of storing data in serialized form is slower access times, due to having to deserialize each object on the fly. We highly recommend using Kryo if you want to cache data in serialized form, as it leads to much smaller sizes than Java serialization (and certainly than raw Java objects).

当你的对象在做了优化后依然很大无法有效存储，一个减少内存的简单方法是使用序列化的格式存储，即缓存这个对象，并且持久化级别设置为`MEMORY_ONLY_SER`*（_ser表示要序列化）*。Spark将会把RDD的每个partition存储为一个长byte数据组。以序列化方式存储数据的唯一缺点是访问时较慢，这是由于必须反序列化每个对象。我们强烈建议你使用kryo序列化你要缓存的数据，因为它比java序列化产生的数据要小的多（比java原始类型也要小）。

### 垃圾回收调优
> JVM garbage collection can be a problem when you have large “churn” in terms of the RDDs stored by your program. (It is usually not a problem in programs that just read an RDD once and then run many operations on it.) When Java needs to evict old objects to make room for new ones, it will need to trace through all your Java objects and find the unused ones. The main point to remember here is that the cost of garbage collection is proportional to the number of Java objects, so using data structures with fewer objects (e.g. an array of Ints instead of a LinkedList) greatly lowers this cost. An even better method is to persist objects in serialized form, as described above: now there will be only one object (a byte array) per RDD partition. Before trying other techniques, the first thing to try if GC is a problem is to use serialized caching.

> GC can also be a problem due to interference between your tasks’ working memory (the amount of space needed to run the task) and the RDDs cached on your nodes. We will discuss how to control the space allocated to the RDD cache to mitigate this.

垃圾回收对于程序中存在频繁创建临时对象的情况是个大问题。(在只读取一次RDD然后对其执行许多操作的程序中，垃圾回收通常不是问题。)当java需要回收旧对象并为新对象腾空间时，它会跟踪你所有的的对象查出不再使用的。需要注意的是垃圾收集的成本与Java对象的数量成正比，所以使用更少对象的数据结构（例如使用Ints数组而不是LinkedList）可大大降低了成本。更好的方法是以序列化的形式保存对象，如上所述：序列化后的RDD partition只有一个对象（一个字节数组）。 如果GC是个问题，那么首先应尝试使用序列化缓存来解决。
由于任务的工作内存（运行task所需要的内存）和节点上RDD缓存之间的干扰也可能造成GC问题。我们将会讨论如何控制分配给RDD缓存的空间来缓解这种情况。
#### 测试GC的影响
> The first step in GC tuning is to collect statistics on how frequently garbage collection occurs and the amount of time spent GC. This can be done by adding `-verbose:gc -XX:+PrintGCDetails -XX:+PrintGCTimeStamps` to the Java options. (See the configuration guide for info on passing Java options to Spark jobs.) Next time your Spark job is run, you will see messages printed in the worker’s logs each time a garbage collection occurs. Note these logs will be on your cluster’s worker nodes (in the stdout files in their work directories), not on your driver program.

GC调优的第一步是收集垃圾回收频率和GC时间的统计数据。这个可以通过在Java选项中增加`-verbose:gc -XX:+PrintGCDetails -XX:+PrintGCTimeStamps`参数来实现（详细参数请参照有关将Java选项传递给Spark作业的信息的[指南](http://spark.apache.org/docs/2.3.0/configuration.html#dynamically-loading-spark-properties)）。下次运行Spark作业时，每次发生垃圾回收时都会在worker的日志中看到消息。请注意GC信息会在集群的各个worker节点输出（在work目录下的stdout文件中），而不是在driver上输出。
#### 高级GC优化
> To further tune garbage collection, we first need to understand some basic information about memory management in the JVM:
> - Java Heap space is divided in to two regions Young and Old. The Young generation is meant to hold short-lived objects while the Old generation is intended for objects with longer lifetimes.
> - The Young generation is further divided into three regions [Eden, Survivor1, Survivor2].
> - A simplified description of the garbage collection procedure: When Eden is full, a minor GC is run on Eden and objects that are alive from Eden and Survivor1 are copied to Survivor2. The Survivor regions are swapped. If an object is old enough or Survivor2 is full, it is moved to Old. Finally when Old is close to full, a full GC is invoked.

> The goal of GC tuning in Spark is to ensure that only long-lived RDDs are stored in the Old generation and that the Young generation is sufficiently sized to store short-lived objects. This will help avoid full GCs to collect temporary objects created during task execution. Some steps which may be useful are:
> - Check if there are too many garbage collections by collecting GC stats. If a full GC is invoked multiple times for before a task completes, it means that there isn’t enough memory available for executing tasks.
> - In the GC stats that are printed, if the OldGen is close to being full, reduce the amount of memory used for caching by lowering `spark.memory.storageFraction`; it is better to cache fewer objects than to slow down task execution!
> - If there are too many minor collections but not many major GCs, allocating more memory for Eden would help. You can set the size of the Eden to be an over-estimate of how much memory each task will need. If the size of Eden is determined to be E, then you can set the size of the Young generation using the option -Xmn=4/3*E. (The scaling up by 4/3 is to account for space used by survivor regions as well.)
> - As an example, if your task is reading data from HDFS, the amount of memory used by the task can be estimated using the size of the data block read from HDFS. Note that the size of a decompressed block is often 2 or 3 times the size of the block. So if we wish to have 3 or 4 tasks’ worth of working space, and the HDFS block size is 64 MB, we can estimate size of Eden to be 4*3*64MB.
> - Monitor how the frequency and time taken by garbage collection changes with the new settings.
> - Try the G1GC garbage collector with -XX:+UseG1GC. It can improve performance in some situations where garbage collection is a bottleneck. Note that with large executor heap sizes, it may be important to increase the G1 region size with -XX:G1HeapRegionSize（v2.3.0）

> Our experience suggests that the effect of GC tuning depends on your application and the amount of memory available. There are many more tuning options described online, but at a high level, managing how frequently full GC takes place can help in reducing the overhead.

为了进一步调整垃圾收集，我们首先需要了解JVM中有关内存管理的一些基本信息：
- Java堆内存分为年轻代和老年代。年轻代保存生命周期短的对象，老年代保存生命周期长的对象。
- 新生代可以进一步分为三个区域[Eden，Survivor1，Survivor2]
- **垃圾回收的简要过程：当Eden已满时，在Eden上执行minor-GC，在Eden和Survivor1中存活的对象会复制到Survivor2中。Survivor区域交换（Survivor1变为Survivor2，Survivor2变为Survivor1）。如果对象足够老(经过多次Survivor区域交换依然在存活)或Survivor2去面满了，那么会被移到老年代。最后如果老年代快要满了，就有执行full-gc。**

Spark关于GC调优的整体思路是只有长生命周期的RDDs才存储在老年代并且年轻带有足够的大小存放短生命周期的对象。这样可以避免在task运行时触发full-gc。以下几步对优化GC有一定帮助：
- 通过收集GC统计信息来检查是否有太多垃圾回收。如果在任务完成之前多次调用full-GC，则意味着没有足够的内存可用于执行任务。
- 在打印的GC统计信息中，如果老年代接近满，则通过降低`spark.memory.storageFraction`来减少用于缓存的内存量; 缓存更少的对象比减慢任务执行更好！
- 如果发生了多次minor collections（minor GC？年轻代的GC）但没发生多少次major gc（老年代的GC）*（年轻代频繁GC说明Eden区域老是满）*，此时为Eden分配更多内存则会比较有效。您可以将Eden的大小设置的高于task所需内存。如果Eden的大小被定义为E，你可以使用`-Xmn = 4/3E`为新生代设置大小。（扩大到4/3是为了更好的使用survivor空间）。
- 如果你的task是从HDFS中读取数据，可以使用HDFS block的大小来估计task使用的内存量。请注意block正常大小一般是压缩block的2-3倍。如果一个executor进程有3-4个task线程并且HDFS block的大小为64MB，那么我们可以估计Eden的大小应为4*3*64MB。
- 使用新的配置后，应继续监视GC的耗时及频率。
- 通过`-XX：+ UseG1GC`配置使用G1 GC。在垃圾回收出现瓶颈时它可也以提升性能。请注意，对于executor堆内存很大的情况，需要使用`-XX：G1HeapRegionSize`增大G1大小。(V2.0.0新增)

根据经验高效的GC需要 依赖于你的应用及可用内存。[有许多配置可以调优GC。](http://www.oracle.com/technetwork/java/javase/gc-tuning-6-140523.html)但是核心思路是尽量减少full gc。
## 其它优化

### 并行度
> Clusters will not be fully utilized unless you set the level of parallelism for each operation high enough. Spark automatically sets the number of “map” tasks to run on each file according to its size (though you can control it through optional parameters to SparkContext.textFile, etc), and for distributed “reduce” operations, such as groupByKey and reduceByKey, it uses the largest parent RDD’s number of partitions. You can pass the level of parallelism as a second argument (see the spark.PairRDDFunctions documentation), or set the config property spark.default.parallelism to change the default. In general, we recommend 2-3 tasks per CPU core in your cluster.

除非您将每项操作的并行度设置得足够高，否则集群将无法充分利用。Spark会自动的根据文件的大小为其设置一定数量的“map” task。(您可以通过`SparkContext.textFile等`参数控制map task 数量)*（这里应该指的是Partitions数 ）*，对于例如groupByKey和reduceByKey这种“reduce ”task的数量由它父RDD最大的partitions决定。你可以将并性级别作为第二参数（具体参照`spark.PairRDDFunctions`的文档）传给诸如`sc.textFile`、`sc.parallelize`的方法。或者通过配置 Spark的`spark.default.parallelism`参数调整并行度。通常我们推荐每个CPU核上应并性运行2-3个task。
### reduce task内存的使用
> Sometimes, you will get an OutOfMemoryError not because your RDDs don’t fit in memory, but because the working set of one of your tasks, such as one of the reduce tasks in groupByKey, was too large. Spark’s shuffle operations (sortByKey, groupByKey, reduceByKey, join, etc) build a hash table within each task to perform the grouping, which can often be large. The simplest fix here is to increase the level of parallelism, so that each task’s input set is smaller. Spark can efficiently support tasks as short as 200 ms, because it reuses one executor JVM across many tasks and it has a low task launching cost, so you can safely increase the level of parallelism to more than the number of cores in your clusters.

有时候，出现OutOfMemoryError不是因为没有足够内存分配给你的RDD，而是由于一个task引起的，例如groupByKey中的一个reduce任务它大了。Spark的shuffle操作（sortByKey，groupByKey，reduceByKey，join等）每个task会创建一个hash表执行分组，该表一般很大。一个简单的解决办法是增加并行度，这样每个task的输入集会更小。Spark可以有效支持200ms的短task，因为可以在多个task中重复使用一个executor JVM并且task启动成本何低，因此你可以放心的将并行度设置的高于集群的cpu核数。
### 广播大变量
> Using the broadcast functionality available in SparkContext can greatly reduce the size of each serialized task, and the cost of launching a job over a cluster. If your tasks use any large object from the driver program inside of them (e.g. a static lookup table), consider turning it into a broadcast variable. Spark prints the serialized size of each task on the master, so you can look at that to decide whether your tasks are too large; in general tasks larger than about 20 KB are probably worth optimizing.

使用SparkContext中广播功能可以大大减少每个序列化任务的大小以及通过群集启动job的成本。如果你的task用到了driver程序中的一些大对象（例如用于查询的静态表），请考虑将它转化为广播变量。Spark会在master上打印每个task序列化的大小，因此您可以以此确定task是否太大; 一般来说大于20KB的task可能值得优化。

### 数据本地性级别
> Data locality can have a major impact on the performance of Spark jobs. If data and the code that operates on it are together then computation tends to be fast. But if code and data are separated, one must move to the other. Typically it is faster to ship serialized code from place to place than a chunk of data because code size is much smaller than data. Spark builds its scheduling around this general principle of data locality.

> Data locality is how close data is to the code processing it. There are several levels of locality based on the data’s current location. In order from closest to farthest:

> - **PROCESS_LOCAL** data is in the same JVM as the running code. This is the best locality possible
> - **NODE_LOCAL** data is on the same node. Examples might be in HDFS on the same node, or in another executor on the same node. This is a little slower than PROCESS_LOCAL because the data has to travel between processes
> - **NO_PREF** data is accessed equally quickly from anywhere and has no locality preference
> - **RACK_LOCAL** data is on the same rack of servers. Data is on a different server on the same rack so needs to be sent over the network, typically through a single switch
> - **ANY** data is elsewhere on the network and not in the same rack

> Spark prefers to schedule all tasks at the best locality level, but this is not always possible. In situations where there is no unprocessed data on any idle executor, Spark switches to lower locality levels. There are two options: a) wait until a busy CPU frees up to start a task on data on the same server, or b) immediately start a new task in a farther away place that requires moving data there.

> What Spark typically does is wait a bit in the hopes that a busy CPU frees up. Once that timeout expires, it starts moving the data from far away to the free CPU. The wait timeout for fallback between each level can be configured individually or all together in one parameter; see the [`spark.locality`](http://spark.apache.org/docs/1.6.0/configuration.html#scheduling) parameters on the configuration page for details. You should increase these settings if your tasks are long and see poor locality, but the default usually works well.
 
 数据本地级别会对Spark任务产生重大影响。如果数据和在其上运行的代码在一起那么计算速度会很快。如果代码和数据是分离的，那么有一方必须移动。一般来说移动序列化的代码比移动数据要快的多，因为代码比数据小的多。Spark根据这一原则构建其task分配算法（taskScheduler负责）。
数据本地性级别表示的是代码与数据的距离。以是从近到远的五个本地性级别：
- **PROCESS_LOCAL** 数据和代码在同一个JVM上。这是最理想的情况
- **NODE_LOCAL** 数据和代码在同一个节点上。例如HDFS数据与executor在同一节点上，或是数据在同节点的另一个executor上。此级别比PROCESS_LOCAL慢一点，因为必须进行进程间的传输。
- **NO_PREF** 数据从任何地方访问速度都一样，无任何本地性偏好.
- **RACK_LOCAL** 数据来源于同一机架。数在同一机架的不同节点上，需要通过网络经过交换机传输过来。
- **ANY** 数据在网络的任何地方并且不在同一机架上。

Spark会使用最优的本地性级别安排所有task，但也不是总是可行。如果有空闲的executor和未处理的数据，Spark会降低数据本地级别。这里有2个选项：a. 等待CPU释放资源启动一个新的task，这个task可以达到更高的本地级别。b. 立刻移动数据在有空闲资源的机器上开始新的task。
一般来说spark会等待一会，希望CPU能释放资源。如果超时，会开始将数据移动到其它有空闲资源的机器上。各级别间的超时时间可以通过1个变量设置也可以分开设置。使用`spark.locality.wait`参数同一设置，默认是3s，也可使用以下3个参数分别设置`spark.locality.wait.node`、`spark.locality.wait.process`、`spark.locality.wait.rack`
如果你的task的本地性很差并且执行时间很长，你可能需要调大超时时间。一般来说默认配置适用大多数场景。

## 总结
这是一个简短的指南，指出Spark调优应注意的主要问题——最重要的是数据序列化和内存调优。对于大多数程序，切换到Kryo序列化并以序列化格式保存数据将解决常见的性能问题。随时欢迎来信（user@spark.apache.org ，dev@spark.apache.org ）询问Spark调优的问题。