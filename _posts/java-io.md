---
title: JAVA IO 操作
categories:
- JAVA
keywords:
- bio,nio,aio
---

# 概要
在[上一节](http://blog.gaiaproject.club/io/)我们介绍了操作系统的IO操作，详细描述了网络IO的5种模式。每种模式主都有2个角色，分别是系统内核和用户程序，而上节主要是以系统内核的角度来定义描述这5种模式。本章将从用户程序的角度再来说说IO交互模式。这里我们选择的用户程序是我们熟知的JVM虚拟机，着重介绍JAVA是如何运用操作系统提供的IO模型实现自己的BIO，NIO，AIO操作的。

# JVM
本小节将聊聊对JVM几个点的认识，方便后续讲解的开展。  


首先我认为某种意义上JVM的功能与操作系统类似。它向下屏蔽了操作系统的差异，向上为使用者提供了统一的资源使用方式。比如关于IO的各种IO操作，关于CPU的各种线程管理以及对用户透明的内存管理。也正是由于它对操作系统差异的屏蔽，使得它具有一次编译到处运行的特点。即Java字节码可以运行在任何操作系统上，只要这个操作系统上有JVM。  


其次JVM作为用户程序，是要依托于操作系统运行的。因此JVM可以实现的IO操作是基于操作系统提供相应的实现。所以在学习JVM的IO模式前了解操作系统支持的IO模型很有必要。

# 操作系统多路复用I/O
在上一节我们已经介绍了操作系统多路复用I/O，但是没有细说这种模式细分的三种机制：select、poll、epoll。由于本文会用到，下面再来详细说下这3中机制的区别。不过再次之前我们先回顾下多路复用I/O的内容。多路复用I/O的本质是，通过一种机制，让单个进程可以监控多个文件描述符（文件、socket、管道等在linux看来都是文件描述符），一旦某个数据就绪（数据保存在内和空间），能够通知程序进行相应读写操作。

## select机制
**基本原理**：select函数可以监控文件描述符3种状态，分别为writefds(可写)、readfds（可读）、exceptfds（出错）。调用select函数后会阻塞，并轮询所有文件描述符直到有描述符变为上述3种状态中的1种或者超时，函数将返回。当select函数返回后，可以通过遍历文件描述符集合fdset，来找到就绪的描述符。具体函数如下：
```cgo 
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```
**缺点**：传入的文件描述符的集合fd_set不宜过大，过大影响性能，主要体现在需要将集合从用户态传入内核态以及在内核中需要遍历fd_set。同时内核对fd_set的大小有限制，最大不能超过1024。

## poll机制
**基本原理**：poll的机制与select类似，与select在本质上没多大区别，管理多个描述符也是用轮询的方式，但是poll机制引入了新的结构用于存储被监控的文件描述符，同时取消了监控文件描述符数量的限制。poll函数原型如下，:
```cgo
// fds是pollfd类型的数组 , nfds 是数组的长度, timeout 超时
int poll(struct pollfd *fds, nfds_t nfds, int timeout);

typedef struct pollfd {
        int fd;                         // 需要被检测或选择的文件描述符
        short events;                   // 对文件描述符fd上感兴趣的事件
        short revents;                  // 文件描述符fd上当前实际发生的事件
} pollfd_t;
```

## epoll机制
**基本原理**：epoll是基于事件驱动的I/O方式，相对于select来说，epoll没有描述符个数限制，将用户关心的文件描述符的事件存放在内核的一个事件表中，这样在用户空间和内核空间的拷贝只需要一次。linux中提供的epoll相关函数如下：
```cgo
int epoll_create(int size); 
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);  
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

1. **epoll_create**函数主要用来初始化eventpoll结构体，返回epoll句柄，该结构体主要包括一个保存所有监控的文件描述符的红黑树的根结点和一个双向链表，这个链表内存储着准备就绪的文件描述符。
2. **epoll_ctl**函数是注册要监听的事件类型，`epfd`是epoll句柄，`op`表示fd的操作类型，包括：EPOLL_CTL_ADD(注册新的fd到epfd中)，EPOLL_CTL_MOD(修改已注册的fd的监听事件)，EPOLL_CTL_DEL(从epfd中删除一个fd)。`fd`是监听的文件描述符，`event`表示监听的事件。调用该函数监听的fd会被添加到红黑树中，并且监听对象的事件会和驱动(网卡)建立回调关系，如果相应事件发生会将对象放到双向链表中。
3. **epoll_wait**函数主要就是返回双向链表中准备就绪的对象。

从上述函数的基本功能可以发现，内核在其cache中维护了需要监控的文件描述符，用户程序只增量的告诉内核对监控对象的操作就好了，不用向select和poll那样每次都告诉内核全部监控的文件描述符。同时内核和设备驱动之间建立了回调的模式，当关注的事件发生，设备驱动会告诉内核将对应文件描述符放入就绪的双向链表中，而不是让内核轮询所有fd检查关注事件是否发生。  

为了提高效率epoll除了上述处理还支持了2中触发模式：`LT(水平出发,默认)`及`ET(边缘出发)`。在LT模式下，只要这个文件描述符还有数据可读，每次 epoll_wait都会返回它的事件，提醒用户程序去操作。在ET模式下，如果检测到有 I/O 事件时，通过 epoll_wait调用会得到有事件通知的文件描述符，对于每一个被通知的文件描述符，如可读，则必须将该文件描述符一直读到空。因为边缘触发只在文件描述符从未就绪变为就绪时通知用户程序1次，即使用户程序没读完数据，在下次文件描述符变为就绪前内核也不会再通知用户程序了。使用ET可以避免epoll_wait每次都返回大量你不需要读写的就绪文件描述符。


epoll机制在处理高并发且逻辑不复杂的业务中表现优异，著名的nginx就是使用的epoll机制实现的。epoll机制其实内容很多，[这篇文章](https://blog.csdn.net/daaikuaichuan/article/details/83862311)我认为介绍的比较详细。

# JAVA IO操作
关于JAVA IO很多基本概念，这里以点的方式提一些我认为比较重要的。
1. JAVA I/O就是数据输入（Input）到JVM中，或输出（Output）到JVM中。Input Output指的是进入JVM或出从JVM出。
2. 在JAVA中主要的输入输出源是文件或网络。
3. 从输入输出数据的类型来分，JAVA有面向字节流的Input/Output也有面向字符流的Read/Write。
4. JAVA传统的I/O方式是bio（blocking IO）。它是基于流模式实现的。它的交互模式是同步的，在读取/写入流数据时，读写完成前线程会被阻塞。
5. JAVA在1.4版本提出了nio（new IO）。注意NIO不是non-blocking IO。它是在操作系统多路复用IO的基础上实现的。
6. JAVA在1.7版本提出了nio 2.0 也称为aio。
6. nio属于同步非阻塞IO，因为从整体流程上看，在数据进入JVM或从JVM输出完成前，这个链接的处理流成无法继续进行，是被阻塞的。但是由于多路复用IO的机制，线程不用专注等待某个链接数据准备完成，而是直接去处理已经准备完成的链接。所以从线程的角度讲，线程会去处理数据准备好的链接，线程没有被阻塞。
7. nio相较于bio除了使用了操作系统的多路复用IO
7. nio新引入了buffer、channel和selector三个概念。
8. 
