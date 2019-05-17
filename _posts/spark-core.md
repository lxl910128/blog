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

根据上文所述，Spark在进行集群计算时主要涉及的概念有：driver进程，ClusterManager服务，worker节点，Executor进程，Task任务。除了已上概念外，还有Application、Job、Stage。下面将逐个简要介绍。
## Application
Application是用户编写的Spark应用程序。一般Spark应用的基本思路是，首先用户需要初始化SparkContext，然后使用SparkContext读取数据,得到表示数据的RDD。接着对RDD进行计算，最后将计算结果保存。以下是统计文件各个单次出现次数的spark应用。
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
该进程是运行我们编写的spark应用的
# 主要流程
## 任务初始化
## 任务流程
# 总结