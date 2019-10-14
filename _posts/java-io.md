---
title: JAVA IO 操作
categories:
- JAVA
keywords:
- bio,nio,aio
---

# 概要
在[上一节](http://blog.gaiaproject.club/io/)我们介绍了操作系统的IO操作，详细描述了网络IO的5种模式。每种模式主都有2个角色，分别是系统内核和用户程序，而上节主要是以系统内核的角度来定义描述这5种模式。本章将从用户程序的角度再来说说IO交互模式。这里我们选择的用户程序是我们熟知的JVM虚拟机，着重介绍JAVA是如何运用操作系统提供的IO模型实现自己的BIO，NIO，AIO操作的。  

由于涉及知识太多且作者能力有限，所以本文主要写作方式是以点盖面，列出作者认为主要的知识点，在整体逻辑上可能会比较碎。

<!-- more-->

# JVM
本小节将聊聊对JVM几点认识，方便后续讲解的开展。  

首先我认为某种意义上JVM的功能与操作系统类似。它向下屏蔽了操作系统的差异，向上为字节码提供了统一的运行环境。比如针对文件外的各种IO操作，关于CPU的各种线程管理以及对用户透明的内存管理。也正是由于它对操作系统差异的屏蔽，使得它具有一次编译到处运行的特点。即Java字节码可以运行在任何操作系统上，只要这个操作系统上有JVM。  
![jvm抽象](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/jvm.bmp)  


其次JVM作为用户程序，是要依托于操作系统运行的。因此JVM可以实现的IO操作是基于操作系统提供了相应的支持。所以在学习JVM的IO模式前了解操作系统支持的IO模型很有必要。

# 操作系统多路复用I/O
在上一节我们已经介绍了操作系统多路复用I/O，但是没有细说这种模式细分的三种机制：select、poll、epoll。由于本文会用到，下面再来详细说下这3种机制的区别。不过在此之前我们先回顾下多路复用I/O的内容。多路复用I/O的本质是，通过一种机制，让单个进程可以监控多个文件描述符（文件、socket、管道等在linux看来都是文件描述符），一旦某个数据就绪（数据保存在内核空间），能够通知程序进行相应读写操作。

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
本章将逐个介绍BIO，NIO，AIO。介绍时首先会以点带面的介绍相关概念，然后会列出该模式下一个网络I/O服务端的例子。首先我们需要明确JAVA I/O就是数据输入（Input）到JVM中，或从JVM中输出（Output）。Input Output指的是进入JVM或出从JVM出。JAVA在诞生初期仅有BIO然后在1.4版本引入了NIO，1.7对NIO进行了扩展有了NIO 2。

## BIO
java BIO（blocking IO）或称JAVA传统IO，是java 1.0提出的IO模型，也是最简单最直接的IO模型。与它相关的概念比较多，这里提几点比较重要的；
1. java传统IO是基于流模式实现的同步交互，在读取/写入数据时，当先线程会被阻塞；
2. 传统IO的输入输出源一般为文件、网络链接、pipe（管道）、buffer（内存缓存）、System.out(标准输出)、System.in(标准输入)、System.error(错误输出)；
3. java在对BIO的实现时使用了大量装饰者模式

```java
public class BioServer {
    public static void main(String[] args) {
        try (ServerSocket server = new ServerSocket(1986)) {
                    while (true) {
                        Socket socket = socket = server.accept(); //阻塞
                        Socket finalSocket = socket;
                        new Thread(() -> {
                            try (BufferedReader in = new BufferedReader(new InputStreamReader(finalSocket.getInputStream()));
                                 PrintWriter out = new PrintWriter(finalSocket.getOutputStream(), true)) {
                                String body;
                                // read 时阻塞
                                while ((body = in.readLine()) != null && body.length() != 0) {
                                    System.out.println("server received : " + body);
                                    out.print("copy right \n");
                                    out.flush();
                                }
                            } catch (IOException e) {}
                        }).start();
                    }
                } catch (Exception e) {}
    }
}
```

## NIO
1. nio相较于bio除了使用了操作系统的多路复用IO，所以NIO解释为new IO更准确，而不是non-blocking IO。
2. nio属于同步非阻塞IO，因为从整体流程上看，在数据进入JVM或从JVM输出完成前，这个链接的处理流程无法继续进行，是被阻塞的所以是同步。但是由于多路复用IO的机制，线程不用专注等待某个链接数据准备完成，而是直接去处理已经准备完成的链接。所以从线程的角度讲，线程会去处理数据准备好的链接，线程没有被阻塞。在后面的例子中我们可以看见一个线程同时可以处理多个链接。
3. nio新引入了buffer、channel和selector三个概念。三者的关系大致可以理解为selector上管理了多个channel，当channel中有用户关注的读\写\注册事件发生时，用户可以从selector获取到相应channel并向其输入或读出包含数据的buffer。
4. NIO是面向channel的，对应的有关于socket的SocketChannel(客户端channel)、ServerSocketChannel(服务端channel)、DatagramChannel(UDP channel)以及针对文件的FileChannel
5. 从channel读写的数据是以buffer为基准的
6. Buffer，缓冲区就是一个可读写的数组
7. 描述buffer主要有4个变量：容量(Capacity)、上界(Limit)、位置(Position)、标记(Mark)。
8. 容量(Capacity)： 缓冲区能够容纳的数据元素的最大数量，buffer构造完成后不能改变，因此buffer的构建一般使用静态方法
9. 上界(Limit)：读写上限
10. 位置(Position): 下一个要被读或写的元素的位置
11. 标记(Mark)一个标记位置，调用reset方法可以将position置为标记的位置
![buffer](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/buffer.bmp)
12. 与BIO面向stream相对NIO是面向Channel
13. channel的源主要有文件和网络socket
14. 与BIO中stream对byte数组或char数组进行读写不同，NIO中是对channel中的Buffer进行读写
15. selector是一个channel管理器，是实现同时管理多个channel的NIO核心组件
16. selector可以理解为是操作系统多路复用IO中select/epoll/poll机制的外包类
17. JVM默认使用poll机制，如果监测系统是高版本（2.4+ 存疑）linux则会自动使用epoll机制

```java
public class EpollServer {
        public static void main(String[] args) {
            try {
                ServerSocketChannel ssc = ServerSocketChannel.open();
                ssc.socket().bind(new InetSocketAddress("127.0.0.1", 8000));
                ssc.configureBlocking(false);
                //在有多个链接时，该select上会有1个关注注册的channel，有多条已链接的channel每个channel可能关注读也可能在关注写
                Selector selector = Selector.open();
                // 注册 channel，并且指定感兴趣的事件是 Accept
                ssc.register(selector, SelectionKey.OP_ACCEPT);
                ByteBuffer readBuff = ByteBuffer.allocate(1024);
                ByteBuffer writeBuff = ByteBuffer.allocate(128);
                writeBuff.put("received".getBytes());
                writeBuff.flip();
                while (true) {
                    int nReady = selector.select(); //阻塞等待感兴趣的事件
                    Set<SelectionKey> keys = selector.selectedKeys();
                    Iterator<SelectionKey> it = keys.iterator();
                    while (it.hasNext()) {
                        SelectionKey key = it.next();
                        it.remove();
                        if (key.isAcceptable()) {
                            // 创建新的连接，并且把连接注册到selector上，而且，
                            // 声明这个channel只对读操作感兴趣。
                            SocketChannel socketChannel = ssc.accept();
                            socketChannel.configureBlocking(false);
                            socketChannel.register(selector, SelectionKey.OP_READ);
                        } else if (key.isReadable()) {
                            // 处理读事件，然后表明这个链接的channel只对写事件感兴趣
                            SocketChannel socketChannel = (SocketChannel) key.channel();
                            readBuff.clear();
                            socketChannel.read(readBuff);
                            readBuff.flip();
                            System.out.println("received : " + new String(readBuff.array()));
                            key.interestOps(SelectionKey.OP_WRITE);
                        } else if (key.isWritable()) {
                            // channel可写时写入数据，并将channel的关注事件再切到读上
                            writeBuff.rewind();
                            SocketChannel socketChannel = (SocketChannel) key.channel();
                            socketChannel.write(writeBuff);
                            key.interestOps(SelectionKey.OP_READ);
                        }
                    }
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```
## AIO
1. Java AIO 又称异步非阻塞IO
2. AIO向内核发送I/O命令后不会等待数据而是直接去做别的事了。等内核完成I/O操作会自动调用注册的成功或失败回调事件
3. 在window中使用LOCP实现AIO，在Linux中使用多路复用I/O的epoll机制模拟实现AIO
4. AIO编码比NIO友好，但依然推荐使用对NIO进行封装的Netty进行网络编程
```java
public class AsyncTest {
    public static void main(String[] args) {
        try (AsynchronousServerSocketChannel server =
                     // 设置线程池来运行回调(CompletionHandler)
                     AsynchronousServerSocketChannel.open(AsynchronousChannelGroup.withFixedThreadPool(10, x -> new Thread(x, "myThread")))
                             .bind(new InetSocketAddress("localhost", 9832))) {
            server.accept(null, new CompletionHandler<AsynchronousSocketChannel, Object>() {
                final ByteBuffer buffer = ByteBuffer.allocate(1024); // AIO读写数据也是针对buffer的
                @Override
                public void completed(AsynchronousSocketChannel result, Object attachment) {
                    Future<Integer> writeResult = null;
                    try {
                        buffer.clear();
                        // read结果是使用Future封装的对象，需要使用get真正阻塞读取
                        result.read(buffer).get(100, TimeUnit.SECONDS);
                        System.out.println("In server: " + new String(buffer.array()));
                        //将数据写回客户端
                        buffer.flip();
                        writeResult = result.write(buffer);
                    } catch (InterruptedException | ExecutionException | TimeoutException e) {
                        e.printStackTrace();
                    } finally {
                        server.accept(null, this); // 继续接收
                        try {
                            writeResult.get(); // 写完
                            result.close();
                        } catch (InterruptedException | IOException | ExecutionException e) {
                            e.printStackTrace();
                        }
                    }
                }
                @Override
                public void failed(Throwable exc, Object attachment) {
                    System.out.println("failed:" + exc);
                }
            });
            Thread.sleep(1000);
        } catch (Exception e) {
        }
    }
}

```