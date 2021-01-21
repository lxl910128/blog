---
title: spark-stream 知识点总结
date: 2021/1/14 22:00:10
categories:
- spark
tags:
- spark
- spark-stream 
keywords:
- spark-stream , spark
---
# 知识点

1. spark stream 支持使用相同的代码进行批处理、在流数据上做即席查询、历史数据加入流。

2. Spark Streaming接收实时输入数据流，并将数据分成批处理，然后由Spark引擎进行处理，以生成批处理的最终结果流。
<!-- more -->
3. 开发spark stream基本部署：

   1. 定义Dstream输入源
   2. 通过指定`transformation`和`output`操作对Dstream的操作
   3. 开始接受数据`streamContext.start()`
   4. 等待程序终止（因出错），`streamingContext.awaitTermination()`或手动终止`streamingContext.stop()`

4. 官方注意点：

   1. context一但启动，不能有新的stream计算再被设置或添加
   2. context被停止，则它不能再被重启
   3. 1个JVM只能有一个`StreamingContext`
   4. `SparkConext`可以用来重复创建多个`StreamingContexts`，只要在创建新的`StreamingContext`前不停止`SparkContext`且前一个`StreamingContext`已经停止。

5. `Dstream`是spark steaming的基本抽象。表示连续的数据流。

6. `Dstream`内部由连续一系列`RDD`表示，每个RDD都包含来自特定间隔的数据

7. 在`DStream`上执行的任何操作都转换为对基础RDD的操作

8. 输入`Dstream`都与一个`Receiver`关联，该对象接受数据并将其存储到spark内存中以供后续计算使用

9. 可以同时创建多个Dstream接收不同数据源的数据

10. 本地调试`local[n]`,n>要运行的接收者的数量

11. 在集群上运行，分配给Spark Streaming应用程序的内核数必须大于接收器数。否则，系统将接收数据，但无法处理它。

12. 文件作为`Receiver`

    1. 可以监视hdfs目录
    2. 设置监控可以使用`glob`通配符，即linux使用的通配符
    3. 所有文件必须同一种格式
    4. spark关注文件修改时间而非创建时间
    5. 处理后，在当前窗口中对文件的更改将不会导致重新读取该文件。即：*忽略更新*
    6. 调用[`FileSystem.setTimes()`](https://hadoop.apache.org/docs/current/api/org/apache/hadoop/fs/FileSystem.html#setTimes-org.apache.hadoop.fs.Path-long-long-)修复时间戳是一种在以后的窗口中拾取文件的方法，即使其内容没有更改。

13. 可以自定义Receivers，详见：https://spark.apache.org/docs/2.4.7/streaming-custom-receivers.html

14. 接收器分为`Reliable Receiver`可靠接收器和`Unreliable Receiver`不可靠接收器

    1. 可靠接收器允许发送数据以确定接收，使用时认真考虑确认语义
    2. 不可靠接收器没有确认接收的复杂机制

15. `Transformations`允许修改DStream的数据，常规RDD操作Dstream都支持

    1. `count()`返回单值RDD表示RDD接收的条数
    2. `reduce()`和`reduceByKey()`是`Transformations`
    3. `join()`、`cogroup()`和`union()`用于合并2个RDD
    4. `join()`，同RDD不先聚合，直接与RDD2按key聚合，所以是笛卡尔积的形式
    5. `cogroup`，同RDD先聚合，再与另一RDD按key聚合，所以结果中通1key仅有1个值
    6. `transform()`， 对Dstream的每个RDD上应用RDD-to-RDD的操作，使用此方法可以在Dstream使用任意RDD的操作，如何与已加载的RDD（来源于存量数据）做join操作等
    7. `updateStateByKey()`,返回一个新的状态Dstream，用于维护每个键的状态数据。
       1. 第一步，定义一个状态
       2. 第二步，定义更新状态的函数
    8. 窗口操作，可以在滑动窗口上应用`Transformations`。重要的有2个参数，一个是`窗口时间长度`和`滑动间隔时长`。比如Dstream设置的间隔是10S，窗口时长为30S，滑动间隔为20S，那么每个窗口就有3个DStream，且新窗口的第一个RDD是上一个窗口的最后一个RDD
       1. 窗口操作可以直接改变Dstream的间隔
       2. 一个Dstream可以用窗口产生多个DStream，多个Dstream之间可以做`join`等操作

16. OutPut操作允许将DStream的数据推出到外部系统。类似action操作，此操作触发所有Transformations的实际执行

    1. `println()`在driver节点打印Dstream前10条

    2. `saveAsTextFiles()`、`saveAsObjectFiles()`、`saveAsHadoopFiles()`结果保存到文件系统

    3. `foreachRDD(func)`，最通用输出运算符，特别注意func是运行在**driver**上的。十分重要

       1. 注意经常出现在`driver`中创建连接，在worker中使用连接，这样非常不对！

          ```scala
          dstream.foreachRDD { rdd =>
            val connection = createNewConnection()  //在 driver 执行
            // 要求将连接序列化并传给work，但连接很少能序列化传输
            rdd.foreach { record =>
              connection.send(record) // 在 work执行
            }
          }
          ```

       2. 在避免1的同时，可能会犯为每个记录创建连接的错误！

          ```scala
          dstream.foreachRDD { rdd =>
            rdd.foreach { record => // 在work执行，但每个记录都创建了连接
              val connection = createNewConnection() 
              connection.send(record)
              connection.close()
            }
          }
          ```

       3. 最佳解决方案是在`foreachRDD()`中使用`foreachPartition`,该方案为每个partition创建1个连接且在work上执行

          ```scala
          dstream.foreachRDD { rdd =>
            rdd.foreachPartition { partitionOfRecords => // 每个批次建一个
              val connection = createNewConnection() // work内创建
              partitionOfRecords.foreach(record => connection.send(record))
              connection.close()
            }
          }
          ```

       4. 进一步优化，可以在多个RDD /批次之间重用连接对象

          ```scala
          dstream.foreachRDD { rdd =>
            rdd.foreachPartition { partitionOfRecords =>
              // ConnectionPool is a static, lazily initialized pool of connections
              val connection = ConnectionPool.getConnection()
              partitionOfRecords.foreach(record => connection.send(record))
              ConnectionPool.returnConnection(connection)  // return to the pool for future reuse
            }
          }
          ```

    4. 注意，Dstream的output操作类似RDD的aciton操作，如果没有则不会触发前面Transformations的执行，foreachRDD()内没有RDD的action也不会触发任何执行！系统将仅仅是接收数据，然后丢弃

    5. output执行逻辑`one-at-a-time`，并且按application定义的顺序执行

17. spark stream可以很方便的和spark sql结合使用，即在`foreachRDD`中使用`createOrReplaceTempView`将流式数据注册成表即可

18. 使用`streamingContext.remember(Minutes(5))`可以让spark stream缓存5分钟的数据，方便配合spark sql 使用

19. `DStreams`也可以使用`persist()`，将数据持久存储在内存中。使用场景是同一个Dstream需要经行多次操作。

20. 窗口操作默认进行`persist()`，无需人工进行缓存

21. 对于接入的是网络流式数据的默认的持久性级别设置为将数据复制到两个节点以实现容错

22. 为了能使spark stream 7*24小时运行，需要设置`Checkpointing`，注意区别其余`persist()`缓存的区别。

