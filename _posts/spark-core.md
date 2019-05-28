---
title: spark核心流程分析
categories:
- spark
tags:
- 流程分析
---
# 概述
# 主要概念

// 先以官方文档介绍各个角色
// 分述各个角色
// 介绍在 standalone、yarn-client、yarn-cluster中各个角色的差异

Spark官方文档对集群的概述如下。Spark应用在集群上作为相互独立的进程组来运行，并通过main程序中的SparkContext来协调，称之为`driver`程序。
具体来说，为了运行在集群上，SparkContext首先需要到`Cluster Manager`上获取计算资源（CPU/内存）。目前Cluster Manager主要有Spark自带的Standalone Cluster Manager、
Mesos以及Yarn。SparkContext通过Cluster Manager可以知道哪些`Worker节点`有满足需求的计算资源，并在这些节点上启动用于计算、存储数据的`Executor`进程。
接下来，driver将会把应用代码（即通过JAR报传递给SparkContext）发送至Executor。Executor启动多个线程并行的运行接收到的`task`。具体如下图所示:
![spark集群示意图](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/spark%E9%9B%86%E7%BE%A4%E9%97%B4%E5%9B%BE.jpg)  

根据上文所述，Spark在进行集群计算时主要涉及的概念有：driver进程，ClusterManager服务，worker，Executor进程，Task任务。除了已上概念外，还有Application、Job、Stage。下面将逐个简要介绍。
## Application
Application是用户编写的Spark应用程序，最终通常最终以jar包的形式体现。一般Spark应用的基本思路是，首先用户需要初始化SparkContext，然后使用SparkContext读取数据,得到表示数据的RDD。接着对RDD进行计算，最后将计算结果保存。以下是统计文件各个单次出现次数的spark应用。在打成JAR包后，通常使用`spark-submit`命令将应用提交到Spark集群运行。
```scala
object SimpleApp {
  def main(args: Array[String]) {
    val logFile = "YOUR_SPARK_HOME/README.md" 
    val conf = new SparkConf().setAppName("Simple Application")
    val sc = new SparkContext(conf)
    val logData = sc.textFile(logFile, 2).cache()
    val words = logData.flatMap { line => line.split(" ") } //words同样是RDD类型    
    val pairs = words.map { word => (word, 1) }
    val wordCounts = pairs.reduceByKey(_ + _)   
    wordCounts.collect().foreach(println)
    
    sc.stop() 

   }
}
```
## driver进程
官方对driver进程的说明是:执行Application的main()方法的进程，并且负责创建SparkContext。  

从上这就话可以看出2点，第一点，driver进程是负责真正执行我们spark程序的；第二点，driver需要创建sparkContext，换句话说driver需要负责spark计算的初始化工作。  

关于第一点，需要补充的是，执行spark Application的driver并不一定在用户提交spark应用(即运行`spark-submit`)的机器上。通常Spark的部署模式有两种，不同的部署模式driver运行的地方也不一样。在`client`模式中，client提交Application的进程负责启动driver。在`cluster`模式下，driver被集群中的一个worker进程启动，client在完成提交Application的责任后就可以退出，无需等待spark Application结束。  

从第二点我们可以得知，driver主要的职责是创建sparkContext，即初始化Spark Application运行的环境。在初始化时主要创建了`DAGScheduler`，`TaskScheduler`，`TaskSchedulerBackerend`。  

其中`DAGScheduler`主要作用原文如下：

> The high-level scheduling layer that implements stage-oriented scheduling. It computes a DAG of stages for each job, keeps track of which RDDs and stage outputs are materialized, and finds a minimal schedule to run the job. It then submits stages as TaskSets to an underlying TaskScheduler implementation that runs them on the cluster.

DAGScheduler在更高层面实现了面向`stage`的调度机制。它会为每个`job`计算一个`stage`的DAG(有向无环图)，追踪RDD和stage的输出存在何处（内存或磁盘），并且找到一个消耗最小的计划来执行`job`(消耗最小即数据尽可能不被传输)。然后，DAGScheduler将`stages`作为`TaskSets`交给底层的`TaskScheduler`，后者负责在集群中运行这些`TaskSets`。

## clusterManager服务
## worker
## exector进程
## task任务
## Job对象
## stage对象
`stage`的创建是通过在shuffle边界破坏RDD图来创建的。
## TaskSet
`TaskSet`包含完整且独立的任务。这些任务可基于集群中已有的数据即刻运行。（例如根据上一个`stage`输出文件）如果任务失败可能是因为数据不可用了。

# 主要流程
## 任务初始化
## 任务流程
# 总结