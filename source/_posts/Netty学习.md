---
title: Netty学习
author: XIA
categories:
  - null
tags:
  - null
date: 2020-07-15 21:12:04
---

# IO读写基本原理

当应用程序进行IO读写时，实际上底层都是用到了read或write这两个系统调用。但是这两个系统调用并不是直接将数据从物理设备读取到内存，也不是直接将数据写回物理设备。这里会涉及两个缓存，内核缓存与进程缓存。实际上调用read时是将数据从内核缓冲区复制到进程缓冲区，而write是将数据从进程缓冲区复制到内核缓冲区。所以程序的IO操作实际上都是和缓存在打交道。

操作系统设置内核缓存与进程缓存，其实是为了减少频繁的设备之间进行物理交换。因为与物理设备的直接读写会涉及到系统的中断。当有了缓冲区就可以不必每次读写都实际地进行物理设备的读写操作，例如write时，只需要将进程缓存的数据复制到内核缓存，待内核缓存的数据量达到一定规模再统一进行实际的物理写操作。而从内核缓存实际写入物理设备的过程是操作系统完成的，并不需要程序员关心。

在操作系统中只有一个内核缓冲区，而每个进程都有自己的独立进程缓冲区。

一个读写操作可以分为两个阶段：数据准备阶段和数据复制阶段。以一个java操作socket的例子来说明这两个阶段。

![image-20200715214646944](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20200715214646944.png)

用户进程空间中运行的是java程序，当一个请求到来时，操作系统会将网卡中的数据复制到内核缓冲区，这个阶段就是数据的准备阶段；当数据准备阶段完成，就会将数据从内核缓冲区复制到进程缓冲区，这个阶段就是数据复制阶段。这样程序的读操作才算是完成。之后程序就可以根据进程缓冲区中的数据进行计算，当需要进行响应时，需要将响应数据从进程缓存复制到内核缓存(write系统调用)，然后再写回网卡。

在linux的io模型中，所谓的阻塞指的是内核缓冲区到物理设备的阻塞等待。

# 四种IO模型

> 阻塞IO，指的是需要内核IO操作彻底完成后，才返回到用户空间执行用户的操作。阻塞指的是用户空间程序的执行状态。传统的IO模型都是同步阻塞IO。在Java中，默认创建的socket都是阻塞的。
>
> 同步IO，是一种用户空间与内核空间的IO发起方式。同步IO是指用户空间的线程是主动发起IO请求的一方，内核空间是被动接受方。异步IO则反过来，是指系统内核是主动发起IO请求的一方，用户空间的线程是被动接受方。

所以阻塞是内核到物理设备的阻塞等待。同步是内核缓存与进程缓存间的复制行为调用关系，同步时是程序调用内核缓存进行复制，而异步则是操作系统自动将内核缓存数据复制到进程缓存，然后操作系统通知应用进程或执行程序指定的回调。

## 同步阻塞IO

在Java中默认情况下所有的socket都是同步阻塞的，也就是在一次io调用中不管是数据准备阶段还是复制数据阶段都需要等待。等到这两个流程都执行结束才算是这次操作完成，程序才可以继续往下运行。

![image-20200715222920866](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20200715222920866.png)

优点：开发简单，在阻塞等待数据时用户线程被挂起，基本不会占用cpu。

缺点：一般情况下，每个连接都会被分配一个线程，当并发量增大时，线程在等待io时所消耗的性能资源是很大的。而且大量线程也会带来大量的内存开销、线程切换开销。

## 同步非阻塞NIO

在linux中可以将socket变成非阻塞模式，使用非阻塞模式读写，也就是NIO。

在NIO模式下，一旦程序开始IO调用，就会出现两种情况：

1. 在内核缓存中没有数据情况下，系统调用立即返回，返回一个调用失败的信息。
2. 在内核缓存中有数据的情况下，是阻塞的，直到数据从内核缓冲区复制到进程缓冲区。

![image-20200715224802595](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20200715224802595.png)

在发起一个非阻塞的socket调用时，如果此时内核缓存数据还没有准备好，就会返回一个失败信息，并立即返回。此时程序可以继续执行其他工作，之后再发起调用，若此时内核缓存数据已经准备好，则阻塞开始复制数据到进程缓存了。

优点：每次发起IO调用，再内核准备数据的阶段可以直接返回，用户线程不会阻塞，实时性较好。

缺点：会不断轮询内核，这将大量占用cpu时间。

**Java中的NIO并不是这种模型**

## 多路复用模型

多路复用模型避免了NIO模型中轮询等待的问题。它通过linux的select、epoll系统调用，将需要原来在应用中轮询的文件描述符注册进去，之后select、epoll会监视这些文件描述符，一旦就绪就会通知应用程序。

![image-20200715230235800](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20200715230235800.png)

优点：与一个线程维护一个连接的阻塞IO模式相比，使用select/epoll的最大优势在于，一个选择器查询线程可以同时处理成千上万个连接（Connection）。系统不必创建大量的线程，也不必维护这些线程，从而大大减小了系统的开销。

缺点：本质上，select/epoll系统调用是阻塞式的，属于同步IO。都需要在读写事件就绪后，由系统调用本身负责进行读写，也就是说这个读写过程是阻塞的。

**Java的NIO就是这种模型**

## 异步IO模型

也就是AIO。当用户线程进行系统调用，向内核注册某个IO操作，在整个操作期间(包括数据准备、数据复制)完成后，通知应用程序或执行指定的回调方法。

在AIO中，整个内核的数据处理过程，包括从网卡复制数据到内核缓存和从内核缓存复制数据到进程缓存都是不需要阻塞的。

![image-20200715231223412](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20200715231223412.png)

但是AIO在linux中支持并不好，所以现在的程序普遍使用的多路复用模型。

# NIO编程入门

## 传统BIO编程

传统bio的线程模型是线程个数和并发访问数呈1:1的关系，当一个请求到来后会为之创建对应的处理线程，用来处理到来的请求。当请求处理完成后线程就会被销毁。

这种线程模型最大的问题 就是缺乏伸缩性，当客户端访问量增加后，服务端的线程个数会迅速膨胀，将导致系统性能急剧下降，导致堆栈溢出，无法创建新线程等问题。

这种IO模型的编程方式通常是一个线程对server线程进行阻塞，当有请求连接进入时，就会新建线程去处理这个链接，然后server线程会继续阻塞。

![image-20200713225941736](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20200713231158473.png)

## 伪异步IO编程

为了解决传统同步阻塞IO面临的一个链路需要一个线程处理的问题，而引入了一个专门用于处理客户端请求的线程池。通过线程池可以灵活的调配资源，设置线程的最大值，防止由于海量链接并发导致线程耗尽。

通过线程池和任务队列可以实现这种伪异步的io框架，当有新的请求到来时，将客户端的请求封装为Task，投放到线程池的任务队列中，让线程池使用内部维护的线程处理任务。

![image-20200713231158473](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20200713231158473.png)

使用伪异步IO存在的弊端主要是，在进行读取和写入操作时，会被一直阻塞。由于InpuStream与OutputStream的阻塞特性，只有在数据被读取或写入完毕或者发生异常时才会返回。

假设我的系统需要使用一个第三方的系统，我们无法保证第三方系统的性能与稳定性，比如当我们对第三方系统进行一次请求响应的平均时间由于第三方的原因变成60秒，这样就会导致我们的系统中线程池中线程都会被阻塞60秒的时间，此时如果请求量较大就会将线程池中线程耗尽，导致客户以为我们系统崩溃的假象。

## NIO编程

自JDK1.4开始Java开始引入了NIO包。

NIO与传统的BIO主要区别在，BIO是面向流的，NIO是面向缓冲区的。所谓的面向流就是以流的方式读取或写入一个或多个字节，不能随意更改指针顺序。面向缓冲区的NIO，只需要将缓冲区的数据写入到通道或从通道中读取数据到缓冲区。

JDK的NIO包含三个主要组件：Buffer、Channel、Selector。

### Buffer

缓存区Buffer实际上是一个内存块，类似于数组，其实内部也是使用数组实现。Buffer作为一个抽象类，Buffer的实现类包括了Java的基本数据类型：ByteBuffer、CharBuffer、DoubleBuffer、FloatBuffer等。

Buffer主要包括三个属性：capacity(容量)、position(读写位置)、limit(读写的限制)。

![image-20200715232718691](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20200715232718691.png)

常用方法：

+ allocate()：创建缓冲区，如`IntBuffer.allocate(20)`创建一个可以包含20个int类型的缓冲区。
+ put()：写入到缓冲区。
+ flip()：反转，用于转换缓冲区的读写模式。详细原理可以看源码。
+ get()：从缓冲区读取数据。
+ mark()与reset()：将position的值保存起来，放到mark属性中。reset()方法将mark的值恢复到position中。
+ clear()：清空缓冲区。

使用Java NIO Buffer类的基本步骤如下：

1. 使用创建子类实例对象的allocate()方法，创建一个Buffer类的实例对象。
2. 调用put方法，将数据写入到缓冲区中。
3. 写入完成后，在开始读取数据前，调用Buffer.flip()方法，将缓冲区转换为读模式。
4. 调用get方法，从缓冲区中读取数据。
5. 读取完成后，调用Buffer.clear() 或Buffer.compact()方法，将缓冲区转换为写入模式。

----



### Channel

NIO中一个连接就是用一个Channel（通道）来表示。从更广泛的层面来说，一个通道可以表示一个底层的文件描述符，例如硬件设备、文件、网络连接等。除了可以对应到底层文件描述符，Java NIO的通道还可以更加细化。例如，对应不同的网络传输协议类型，在Java中都有不同的NIO Channel（通道）实现。

四种常见Channel实现：FileChannel、SocketChannel、ServerSocketChannel、DatagramChannel。

1. FileChannel文件通道，用于文件的数据读写。
2. SocketChannel套接字通道，用于Socket套接字TCP连接的数据读写。
3. ServerSocketChannel服务器嵌套字通道（或服务器监听通道），允许我们监听TCP连接请求，为每个监听到的请求，创建一个SocketChannel套接字通道。
4. DatagramChannel数据报通道，用于UDP协议的数据读写。



***FileChannel***

可以通过文件的输入流或输出流获取FileChannel，也可以通过RandomAccessFile来获取FileChannel。

```java
new FileInputStream(file).getChannel();   //通过输入流获取channel

new RandomAccessFile("filename","rw").getChannel();  //通过RandomAccessFile获取Channel
```

可以通过FileChannel的read(Buffer)方法将数据读取到Buffer中。也可以通过FileChannel的write(Buffer)将数据从Buffer写回Channel。由于性能原因可能操作系统不会将数据实时写回磁盘，还可以使用FileChannel的force()方法将数据强制写回磁盘。



***SocketChannel与*ServerSocketChannel**

在NIO中，涉及网络连接的通道有两个，一个是SocketChannel负责连接传输，另一个是ServerSocketChannel负责连接的监听。

无论是ServerSocketChannel，还是SocketChannel，都支持阻塞和非阻塞两种模式。

1. socketChannel.configureBlocking（false）设置为非阻塞模式。
2. socketChannel.configureBlocking（true）设置为阻塞模式。

可以通过SocketChannel的静态方法open()获得一个SocketChannel实例，然后通过connect()方法对服务器进行连接。如果是ServerSocketChannel则可以使用open方法获取实例后，使用accept方法监听套接字，获得新的SocketChannel通道。

可以通过SocketChannel的read(Buffer)和write(Buffer)对数据进行读取和写入。



### Selector

选择器的使命是完成IO的多路复用。一个通道代表一条连接通路，通过选择器可以同时监控多个通道的IO（输入输出）状况。选择器和通道的关系，是监控和被监控的关系。

选择器提供了独特的API方法，能够选出（select）所监控的通道拥有哪些已经准备好的、就绪的IO操作事件。

一般来说，一个单线程处理一个选择器，一个选择器可以监控很多通道。通过选择器，一个单线程可以处理数百、数千、数万、甚至更多的通道。

通道和选择器之间的关系，通过register（注册）的方式完成。调用通道的Channel.register（Selector sel, int ops）方法，可以将通道实例注册到一个选择器中。register方法有两个参数：第一个参数，指定通道注册到的选择器实例；第二个参数，指定选择器要监控的IO事件类型。可供选择器监控的通道IO事件类型，包括以下四种：

1. 可读：SelectionKey.OP_READ
2. 可写：SelectionKey.OP_WRITE
3. 连接：SelectionKey.OP_CONNECT
4. 接收：SelectionKey.OP_ACCEPT

> **什么样的Channel可以被注册到Selector中？**
>
> 并不是所有的通道，都是可以被选择器监控或选择的。比方说，FileChannel文件通道就不能被选择器复用。判断一个通道能否被选择器监控或选择，有一个前提：判断它是否继承了抽象类SelectableChannel（可选择通道）。如果继承了SelectableChannel，则可以被选择，否则不能。简单地说，一条通道若能被选择，必须继承SelectableChannel类。SelectableChannel类，是何方神圣呢？它提供了实现通道的可选择性所需要的公共方法。JavaNIO中所有网络链接Socket套接字通道，都继承了SelectableChannel类，都是可选择的。而FileChannel文件通道，并没有继承SelectableChannel，因此不是可选择通道。

在通道注册到selector上之后，就可以使用selector的select()方法来选择通道的就绪状态，一旦在通道中发生了某些IO事件（就绪状态达成），并且是在选择器中注册过的IO事件，就会被选择器选中，并放入SelectionKey选择键的集合中。

在编程时，选择键的功能是很强大的。通过SelectionKey选择键，不仅仅可以获得通道的IO事件类型，比方说SelectionKey.OP_READ；还可以获得发生IO事件所在的通道；另外，也可以获得选出选择键的选择器实例。

使用选择器，主要有以下三步：

1. 获取选择器实例；
2. 将通道注册到选择器中；
3. 轮询感兴趣的IO就绪事件（选择键集合）。

## AIO编程

JDK1.7以后，java推出了NIO2.0也就是AIO库。AIO是基于Linux的AIO模型的。在使用时就是通过注册回调事件来对IO结果响应。















# 使用Netty进行简单的开发

使用Netty的步骤：包括建立NIO线程组，初始化并设置启动类，设置Handler等。

# TCP粘包/拆包问题

由于Netty之间建立的是tcp的请求通道，并不是针对某一个应用层协议，所以它本身不能控制应用层数据组合。所以就会产生粘包、拆包问题。

解决的方式就是在Netty应用中加入适当的分割协议，如根据指定长度、根据特定的字符进行分割。Netty已经为我们提供了很多用于处理粘包、拆包的Handler。使用时只需要将用于处理粘包、拆包的Handler加入pipeline即可。如：

![image-20200714215537434](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20200714215537434.png)

Netty提供了很多处理粘包与拆包策略的Handler可供使用：

+ LineBasedFrameDecoder:根据换行符处理半包问题
+ DelimitedBasedFrameDecoder：根据分隔符处理半包问题
+ FixedLengthFrameDecoder：根据固定长度处理半包问题
+ StringDecoder：将数据编码为字符串

# 编解码技术

Java自带的编解码技术有着以下缺点：

+ 无法跨语言
+ 序列化后码流太大
+ 序列化性能低

所以需要使用第三方的编解码框架来应对编解码需求，主流的有Protobuf、Thrift、Marshalling等。

Netty本身提供了对这些编解码器的支持，开箱即用。

# HTTP支持&WebSocket支持



