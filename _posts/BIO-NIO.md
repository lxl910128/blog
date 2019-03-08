---
title: JAVA BIO 与 NIO 简介
categories:
- JAVA
tags:
- IO
---
# 概述
I/O操作是系统中十分基础且重要的操作，同样也很容易产生性能瓶颈。本文以网络IO为使用场景，首先介绍IO是什么以及Linux常见的5中IO模型，然后介绍java BIO，分析了它的不足，最后介绍NIO是如何解决BIO的问题及NIO的Reactor模式。

<!-- more -->

# 什么是IO
IO操作的基础是缓冲(buffer)和缓冲的处理方式。所有的I/O无外乎是指数据进入或移除缓存。java进程请求操作系统进行IO操作就可以理解为以下2种。JVM将数据写入缓存，操作系统从缓存中取出数据写入文件或socket，这种是数据从JVM中送出去就是out操作。JVM从缓冲中读取操作系统放入的数据，这种是数据进入JVM就是in操作。  
下图显示数据如何从外部源，例如一个磁盘，移动到JVM的存储区域（堆）中。首先，JVM使用系统调用（read()）请求内核将数据写入缓冲。接到请求后内核向磁盘控制硬件发出命令获取磁盘中的数据。磁盘控制器通过DMA直接将数据写入内核的内存缓冲区。当写入完成后，内核将数据从临时缓冲拷贝到JVM指定的缓冲中。  
![data-buffering-at-os-level](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/data-buffering-at-os-level.png)

# IO模型
可以JAVA的IO操作是与内核操作紧密相关的。为了更好的理解JAVA的IO操作，我们不得不说说内核（Linux）为我们提供的IO模型，它们分别是：完全阻塞模型、非阻塞模型、IO多路复用模型、SIGIO信号模型以及异步IO模型。  
在介绍前我们首先需要明确`同步异步`、`阻塞非阻塞`这两组形容词的意思。  

>
* 同步和异步关注的是消息通信机制 (synchronous communication/ asynchronous communication)
   * 同步就是在发出一个*调用*时，在没有得到结果之前*调用者*需要等待或者轮询*被调用者*，直到被调用者返回数据。换句话说，就是是由*调用者*主动等待这个*调用*的结果。
   * 异步则是相反，当一个异步过程调用发出后，调用者不会立刻得到结果。而是在*调用*发出后，*被调用者*通过状态、通知来通知调用者，或被掉者处理完后运行调用者设置的回调函数。
* 阻塞和非阻塞关注的是程序在等待调用结果（消息，返回值）时的状态
   * 阻塞调用是指调用结果返回之前，当前线程会被挂起。调用线程只有在得到结果之后才会继续运行。
   * 非阻塞调用指在不能立刻得到结果之前，该调用不会阻塞当前线程。
>

## 完全阻塞模型
![blockingIO](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/blockingIO.png)  
比如在网络IO的模型中，在客户端和服务端已建立连接后，服务端发起read请求，在客户端发送完前服务端当前线程会一直休眠。

## 非阻塞式IO
把非阻塞的文件描述符称为非阻塞I/O。可以通过设置SOCK_NONBLOCK标记创建非阻塞的socket fd，或者使用fcntl将fd设置为非阻塞。  
对非阻塞fd调用系统接口时，不需要等待事件发生而立即返回，事件没有发生，接口返回-1，此时需要通过errno的值来区分是否出错，有过网络编程的经验的应该都了解这点。不同的接口，立即返回时的errno值不尽相同，如，recv、send、accept errno通常被设置为EAGIN 或者EWOULDBLOCK，connect 则为EINPRO- GRESS 。  
![notBlockingIO](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/notBlockingIO.png)

## IO多路复用
最常用的I/O事件通知机制就是I/O复用(I/O multiplexing)。Linux 环境中使用select/poll/epoll_wait 实现I/O复用，I/O复用接口本身是阻塞的，在应用程序中通过I/O复用接口向内核注册fd所关注的事件，当关注事件触发时，通过I/O复用接口的返回值通知到应用程序。I/O复用接口可以同时监听多个I/O事件以提高事件处理效率。  
![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/IO-multiplexing.png)

## SIGIO
除了I/O复用方式通知I/O事件，还可以通过SIGIO信号来通知I/O事件，如图所示。两者不同的是，在等待数据达到期间，I/O复用是会阻塞应用程序，而SIGIO方式是不会阻塞应用程序的。  
![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/SIGIO.png)

## 异步调用IO（AIO）
POSIX规范定义了一组异步操作I/O的接口，不用关心fd 是阻塞还是非阻塞，异步I/O是由内核接管应用层对fd的I/O操作。异步I/O向应用层通知I/O操作完成的事件，这与前面介绍的I/O 复用模型、SIGIO模型通知事件就绪的方式明显不同。以aio_read 实现异步读取IO数据为例，如图所示，在等待I/O操作完成期间，不会阻塞应用程序。  
![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/asyncIO.png)

# BIO
JAVA中BIO一般对应使用的是操作系统的完全阻塞模型。  
以网络IO为使用场景，JAVA传统的同步阻塞（BIO）模型一般使用如下设计，服务端创建ServerSocket对象绑定IP，启动监听端口，等待客户端连接。客户端构建Socket对象发起连接请求。服务端在接收到连接后同样回得到一个socket对象。然后双方就可以从socket中获取inputStream或outputStream从而进行信息交互。需要特别注意的是服务端在等待连接接入时会阻塞线程，C/S（客户端和服务端）在等待接收数据时也会阻塞进程。如果一方不断将数据写入socket流中而另一端没有接收的话，那么数据首先会堆积在对方协议栈的接收缓冲区里，如果堆满会导致写操作被阻塞。（另，注意区分传输层TCP的ack和应用层的recv）  
在这种情况下通常使用的模型是：在采用BIO通信模型的服务端，一般使用一个独立的Acceptor线程负责监听客户端的连接，它接收到客户端连接请求后为每个客户端创建一个新的线程进行链路处理，处理主要包括接收信息，处理业务逻辑以及回写信息，处理完成后销毁线程。如下图所示：
![bio mode](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/BIO-mode.png)  
在这个模型中，我们为每个连接创建了线程，并且有个专门的线程负责接收连接。这样做的主要原因是socket.accept(),socket.write()，三个主要函数是同步阻塞的，如果是单线程的话同一时间只能做这3件事中的一件，即写数据时不能再接收新的连接。开启多线程，就可以让CPU去处理更多的事情。其实这也是所有使用多线程的本质： 1. 利用多核。 2. 当I/O阻塞系统，但CPU空闲的时候，可以利用多线程使用CPU资源。   
![bio pool](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/BIO-POOL.png)
现在的多线程一般都使用线程池，可以让线程的创建和回收成本相对较低(如上图伪异步IO模型图)。在活动连接数不是特别高（小于单机1000）的情况下，这种模型是比较不错的，可以让每一个连接专注于自己的I/O并且编程模型简单，也不用过多考虑系统的过载、限流等问题。线程池本身就是一个天然的漏斗，可以缓冲一些系统处理不了的连接或请求。  
不过，这个模型最本质的问题在于，严重依赖于线程。但线程是很”贵”的资源，主要表现在：
 1. 线程的创建和销毁成本很高，在Linux这样的操作系统中，线程本质上就是一个进程。创建和销毁都是重量级的系统函数。 
 2. 线程本身占用较大内存，像Java的线程栈，一般至少分配512K～1M的空间，如果系统中的线程数过千，恐怕整个JVM的内存都会被吃掉一半。 
 3. 线程的切换成本是很高的。操作系统发生线程切换的时候，需要保留线程的上下文，然后执行系统调用。如果线程数过高，可能执行线程切换的时间甚至会大于线程执行的时间，这时候带来的表现往往是系统load偏高、CPU sy使用率特别高（超过20%以上)，导致系统几乎陷入不可用的状态。 
 4. 容易造成锯齿状的系统负载。因为系统负载是用活动线程数或CPU核心数，一旦线程数量高但外部网络环境不是很稳定，就很容易造成大量请求的结果同时返回，激活大量阻塞线程从而使系统负载压力过大。
 
# NIO
试想在BIO中主要的问题是由于IO操作同步阻塞的问题，我们需要开很多线程处理每个连接的读写操作。那么为了提高效率，我们争取使用1个线程完成所由连接读写操作。    
为了实现上述目标，就需要使用JAVA的NIO了,它主要用到了操作系统的。在NIO中各个连接（channel）和处理读写的操作之间有selector层，selector可帮助我们选出当前有哪些连接（channel）处在就绪状态，即读就绪，写就序，有新连接到来等，如果没有就绪的连接selector会阻塞。有了selector以后我们就需要注册当这几个事件到来的时候所对应的处理器。然后在合适的时机告诉事件选择器：我对这个事件感兴趣。对于写操作，就是写不出去的时候对写事件感兴趣；对于读操作，就是完成连接和系统没有办法承载新读入的数据的时；对于accept，一般是服务器刚启动的时候；而对于connect，一般是connect失败需要重连或者直接异步调用connect的时候。然后，用一个死循环选择就绪的事件，会执行系统调用（Linux 2.6之前是select、poll，2.6之后是epoll，Windows是IOCP），还会阻塞的等待新事件的到来。新事件到来的时候，会在selector上注册标记位，标示可读、可写或者有连接到来。注意，select是阻塞的，无论是通过操作系统的通知（epoll）还是不停的轮询(select，poll)，这个函数是阻塞的。所以你可以放心大胆地在一个while(true)里面调用这个函数而不用担心CPU空转。  
伪代码表示如下：
```java
class nio{
    public static void main(){
        ServerChannel server =  new ServerChannel(IP,PORT); //启动服务端监听
        Selector selector = new Selector();//创建选择器
        server.register(selector, ACCEPT);//选择器关注连接事件
        Channel channel;
        while(channel = selector.select()){//选择就绪的连接
            if(channel.event==accept){
               channel.accept();//接收连接
               channel.register(selector,OP_READ);//连接接下来关注读事件
            }else if(channel.event==read){
                channel.read();//读取数据
                channel.register(selector,OP_WRITE);//连接接下来关注读事件
            }else if(channel.even==write){
                channel.write();//写数据
            }
        }
    }
}
```
上述程序是最简单的NIO Reactor模式：注册所有感兴趣的事件处理器，单线程轮询选择就绪事件，执行事件处理器。  
由上面的示例我们可以大概体会到NIO是如何解决线程过多的问题。NIO由原来的阻塞读写（占用线程）变成了单线程轮询事件，找到可以进行读写的网络描述符进行读写。除了事件的轮询是阻塞的（没有可干的事情必须要阻塞），剩余的I/O操作都是纯CPU操作，没有必要开启多线程。并且由于线程的节约，连接数大的时候因为线程切换带来的问题也随之解决，进而为处理海量连接提供了可能。
单线程处理I/O的效率确实非常高，没有线程切换，只是拼命的读、写、选择事件。但现在的服务器，一般都是多核处理器，如果能够利用多核心进行I/O，无疑对效率会有更大的提高。  
仔细分析一下我们需要的线程，其实主要包括以下几种： 1. 事件分发器，单线程选择就绪的事件。 2. I/O处理器，包括connect、read、write等，这种纯CPU操作，一般开启CPU核心个线程就可以。 3. 业务线程，在处理完I/O后，业务一般还会有自己的业务逻辑，有的还会有其他的阻塞I/O，如DB操作，RPC等。只要有阻塞，就需要单独的线程。Java的Selector对于Linux系统来说，有一个致命限制：同一个channel的select不能被并发的调用。因此，如果有多个I/O线程，必须保证：一个socket只能属于一个IoThread，而一个IoThread可以管理多个socket。另外连接的处理和读写的处理通常可以选择分开，这样对于海量连接的注册和读写就可以分发。虽然read()和write()是比较高效无阻塞的函数，但毕竟会占用CPU，如果面对更高的并发则无能为力。
使用线程的NIO Reactor模式如下图：
![](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/NIOReactor.png)

# 总结
NIO还有很多东西，比如Proactor模式等，由机会再继续学习学习。netty主要使用的是NIO，后续可以根据netty再深入了解NIO。本文主要是对多篇blog的理解摘抄，后续还需继续学习。





# 参考及引用
https://blog.csdn.net/anxpp/article/details/51512200  
https://tech.meituan.com/2016/11/04/nio.html
https://howtodoinjava.com/java/io/how-java-io-works-internally-at-lower-level/
https://www.zhihu.com/question/19732473
https://www.ibm.com/developerworks/cn/linux/l-async/
https://zhuanlan.zhihu.com/p/27382996