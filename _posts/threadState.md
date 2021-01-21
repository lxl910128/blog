---
title: JAVA线程状态及状态间转换介绍
date: 2019/3/7 9:29:10
categories:
- JAVA
tags:
- Thread
---
# 概述
JAVA线程是生产及面试中重要的内容，弄清楚线程的6个状态及状态间如何转换是学好线程的基础。本文首先翻译JDK源码及文档，概述6种状态如何定义，然后根据网上资料总结各状态间如何转化，最终得（zhuan）出（zai）线程转换示意图如下：
![线程状态转化图](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/threadState.jpg)
<!--more-->

# 线程6种状态
根据`java.lang.Thread.State`中的定义，JAVA线程有New，Runnable，Blocked，Waiting，Timed Waiting，Terminated6种状态。  


## New
```
Thread state for a thread which has not yet started.
```
线程还没有开始，即线程对象刚创建但还未调用start()方法。
## Runnable
>
Thread state for a runnable thread. A thread in the runnable state is executing in the Java virtual machine but it may be waiting for other resources from the operating system such as processor.
.
表示线程可运行。该状态表示线程正在JVM中运行，但是该线程也可能在等待操作系统资源例如CPU。
## Blocked
>
Thread state for a thread blocked waiting for a monitor lock. A thread in the blocked state is waiting for a monitor lock to enter a synchronized block/method or reenter a synchronized block/method after calling ‘Object.wait’ .
>
Blocked状态表示线程被阻塞等待一个监视锁（monitor lock）。线程在blocked状态就是等待一个监视锁(monitor lock)用以进入一个同步的代码块或方法或者在调用'Object.wait'后重新进入同步代码块或方法。
## Waiting
>
Thread state for a waiting thread.A thread is in the waiting state due to calling one of the following methods:
* 'Object.wait' with no timeout
* 'Thread.join' with no timeout
* 'LockSupport.park'
A thread in the waiting state is waiting for another thread to perform a particular action.
For example, a thread that has called Object.wait() on an object is waiting for another thread to call Object.notify() or Object.notifyAll() on that object. A thread that has called Thread.join() is waiting for a specified thread to terminate.
>
该状态表示线程在等待其它线程。由于调用'Object.wait'(未设置超时)，'Thread.join'(未设置超时)，'LockSupport.park'线程处于Waiting状态。 
线程处于Waiting状态就是等待其它线程执行特定方法。比如，一个线程调用了一个对象的Object.wait()，那么它就是在等待其它线程调用这个对象的Object.notify()或Object.notifyAll()。一个线程调用了另一个线程Thread.join()就是等待那个线程终止。
## Timed Waiting
>
Thread state for a waiting thread with a specified waiting time. A thread is in timed waiting state due to calling one of the following methods with a specified positive waiting time:
* 'Thread.sleep' with  timeout
* 'Object.wait' with  timeout
* 'LockSupport.parkNanos'
* 'LockSupport.parkUntil'
>
该状态表示等待其它线程但有超时时间。在调用了'Thread.sleep'，'Object.wait','LockSupport.parkNanos'，'LockSupport.parkUntil'并设置了超时间间，那么线程就进入了timedWaiting状态。
## Terminated
>
Thread state for a terminated thread.The thread has completed execution.
>
该状态表示线程终止，表示线程运行完成。

# 状态间转换
如文章开头的图，已总结很清楚，不再赘述。面关于图的部分细节进行补充。
* 图中Runnable中包含两个部分Ready和Running，Ready表示等待系统分配资源，Running表示获得系统资源并运行，但是对于JVM层面它都是在运行的，在JAVA2之前这2部分时分开的。
* 程序占有系统资源的时间(CPU时间)是有限的，当CPU时间用完程序就会从Running转到Ready状态。JVM提供了Thread.yield()方法使线程可以主动让出CPU使用时间。[此文](https://blog.csdn.net/dabing69221/article/details/17426953)有关于yield方法的介绍。
* Waiting和TimedWaiting状态由于其它线线程调用notify()或notifyAll()方法结束时，状态转化为Blocked状态。（存疑）
* 进入到Runnable状态时，都是先进入Ready状态等待CPU时间。
* Blocked和waiting是有去别的，间单可以理解为waiting是主动放弃对CPU的占有等待一个通知（吊丝男等待白富美回国的消息，才能继续自己赢娶白富美走上人生巅峰的计划）。而Blocked是被动放弃CPU时间，因为需要用的资源被其它线程占用（由于白富美正被高富帅占用吊丝男只能暂停自己人生巅峰计划）。
* wait()方法会释放CPU执行权和占有的锁。
* sleep(long)方法仅释放CPU使用权，锁仍然占用；线程进入TIMED_WAITING状态，与yield相比，它会使线程较长时间得不到运行。
* yield()方法仅释放CPU执行权，锁仍然占用，线程会进入Ready状态，会在短时间内再次执行。
* wait和notify/notifyAll必须配套使用，即必须使用同一把锁调用；
* wait和notify/notifyAll必须放在一个同步块中
* 调用wait和notify/notifyAll的对象必须是他们所处同步块的锁对象。

当线程执行wait()方法时候，会释放当前对象的锁，然后让出CPU，线程进入等待状态。  
只有当线程A中notify/notifyAll() 被执行时候，才会唤醒其它一个或多个正处于等待状态的线程，然后线程A继续往下执行，直到执行完synchronized代码块的代码或是中途遇到wait()，再次释放锁。  
也就是说，notify/notifyAll() 的执行只是唤醒沉睡的线程，而不会立即释放锁，锁的释放要看代码块的具体执行情况。所以在编程中，尽量在使用了notify/notifyAll() 后立即退出临界区，以唤醒其他线程。


# 参考
https://www.jianshu.com/p/a8abe097d4ed  
https://www.cnblogs.com/nwnu-daizh/p/8036156.html  
https://www.uml-diagrams.org/java-thread-uml-state-machine-diagram-example.html  
https://blog.csdn.net/dabing69221/article/details/17426953