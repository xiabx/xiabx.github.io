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

# Reactor线程模型

先说结论：Reactor线程模型的思想就是基于IO复用和线程池的结合。Reactor是为了更好搭配NIO多路复用机制而产生的一种线程模型。Reactor线程模型使用少量的线程处理大量连接，这主要归功于NIO中选择器的功劳。

在反应器模式中有两个重要的组件：Reactor反应器和Handler处理器。

+ Reactor反应器：利用Selector查询IO事件，当检测到IO事件，将其发送给响应的Handler处理器。
+ Handler处理器：真正处理IO事件，包括连接建立、业务逻辑处理、数据读写等。

Reactor线程模型分为单线程和多线程版本。单线程版本中，Reactor反应器和Handler处理器在同一个线程中执行。多线程版本中会将Handler处理放到线程池中，不再和Reactor放在同一条线程，而且可以使用多个Reactor反应器，提高事件分发效率。

## 单线程版本

在Channel与Handler处理器进行绑定时，需要使用Channel注册到Selector时的绑定键对象SelectionKey。该对象拥有两个重要的成员方法：

+ void attach(Object o)：该方法可以将任何对象作为附件添加到SelectionKey对象，相当于附件属性的setter方法。可以使用该方法将Handler处理器绑定到Selection对象。
+ Object attachment()：此方法的作用是取出之前通过attach(Object o)添加到SelectionKey选择键实例的附件，相当于附件属性的getter方法，与attach(Object o)配套使用。

还是根据示例看吧，一个Echo服务器。

以下是单线程版本Reactor线程模型。在代码中，Reactor反应器和Handler处理器的处理都在同一个线程中。这样虽然使用到的线程减少了，但是当Handler发生阻塞，那么Reactor反应器也会阻塞，这时将会造成服务不可用的情况。所以还需要Reactor线程模型的多线程版本。

```java
//反应器，对应Reactor线程模型的Reactor反应器。
class EchoServerReactor implements Runnable {
    Selector selector;
    ServerSocketChannel serverSocket;

    EchoServerReactor() throws IOException {
        //Reactor初始化
        selector = Selector.open();
        serverSocket = ServerSocketChannel.open();

        InetSocketAddress address =
                new InetSocketAddress(NioDemoConfig.SOCKET_SERVER_IP,
                        NioDemoConfig.SOCKET_SERVER_PORT);
        serverSocket.socket().bind(address);
        //非阻塞
        serverSocket.configureBlocking(false);

        //分步处理,第一步,接收accept事件
        SelectionKey sk =
                serverSocket.register(selector, SelectionKey.OP_ACCEPT);
        //绑定Handler，用于slelector选择到了serverSocket这个channel时，可以从绑定键中取出Handler进行执行业务逻辑。
        sk.attach(new AcceptorHandler());
    }

    public void run() {
        try {
            while (!Thread.interrupted()) {
                selector.select();
                Set<SelectionKey> selected = selector.selectedKeys();
                Iterator<SelectionKey> it = selected.iterator();
                while (it.hasNext()) {
                    //Reactor负责dispatch收到的事件
                    SelectionKey sk = it.next();
                    //将事件分发到Handler
                    dispatch(sk);
                }
                selected.clear();
            }
        } catch (IOException ex) {
            ex.printStackTrace();
        }
    }

    void dispatch(SelectionKey sk) {
        Runnable handler = (Runnable) sk.attachment();
        //调用之前attach绑定到选择键的handler处理器对象
        if (handler != null) {
            handler.run();
        }
    }

    // Handler:新连接处理器
    class AcceptorHandler implements Runnable {
        public void run() {
            try {
                SocketChannel channel = serverSocket.accept();
                if (channel != null)
                    //将新连接SocketChannel进行Selector注册，并绑定Handler
                    new EchoHandler(selector, channel);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }


    public static void main(String[] args) throws IOException {
        new Thread(new EchoServerReactor()).start();
    }
}
```

```java
class EchoHandler implements Runnable {
    final SocketChannel channel;
    final SelectionKey sk;
    final ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
    static final int RECIEVING = 0, SENDING = 1;
    int state = RECIEVING;

    EchoHandler(Selector selector, SocketChannel c) throws IOException {
        channel = c;
        c.configureBlocking(false);
        //仅仅取得选择键，后设置感兴趣的IO事件
        sk = channel.register(selector, 0);

        //将Handler作为选择键的附件
        sk.attach(this);

        //第二步,注册Read就绪事件
        sk.interestOps(SelectionKey.OP_READ);
        selector.wakeup();
    }

    public void run() {

        try {

            if (state == SENDING) {
                //写入通道
                channel.write(byteBuffer);
                //写完后,准备开始从通道读,byteBuffer切换成写模式
                byteBuffer.clear();
                //写完后,注册read就绪事件
                sk.interestOps(SelectionKey.OP_READ);
                //写完后,进入接收的状态
                state = RECIEVING;
            } else if (state == RECIEVING) {
                //从通道读
                int length = 0;
                while ((length = channel.read(byteBuffer)) > 0) {
                    Logger.info(new String(byteBuffer.array(), 0, length));
                }
                //读完后，准备开始写入通道,byteBuffer切换成读模式
                byteBuffer.flip();
                //读完后，注册write就绪事件
                sk.interestOps(SelectionKey.OP_WRITE);
                //读完后,进入发送的状态
                state = SENDING;
            }
            //处理结束了, 这里不能关闭select key，需要重复使用
            //sk.cancel();
        } catch (IOException ex) {
            ex.printStackTrace();
        }
    }


}
```

## 多线程版本

对单线程版本主要有两方面升级：

1. 首先是升级Handler处理器。既要使用多线程，又要尽可能的高效率，则可以考虑使用线程池。
2. 其次是升级Reactor反应器。可以考虑引入多个Selector选择器，提升选择大量通道的能力。

将Handler处理都放在单独的线程池中，与Reactor反应器隔离，避免服务器阻塞。

```java
class MultiThreadEchoServerReactor {
    ServerSocketChannel serverSocket;
    AtomicInteger next = new AtomicInteger(0);
    //selectors集合,引入多个selector选择器
    Selector[] selectors = new Selector[2];
    //引入多个子反应器
    SubReactor[] subReactors = null;


    MultiThreadEchoServerReactor() throws IOException {
        //初始化多个selector选择器
        selectors[0] = Selector.open();
        selectors[1] = Selector.open();
        serverSocket = ServerSocketChannel.open();

        InetSocketAddress address =
                new InetSocketAddress(NioDemoConfig.SOCKET_SERVER_IP,
                        NioDemoConfig.SOCKET_SERVER_PORT);
        serverSocket.socket().bind(address);
        //非阻塞
        serverSocket.configureBlocking(false);

        //第一个selector,负责监控新连接事件
        SelectionKey sk =
                serverSocket.register(selectors[0], SelectionKey.OP_ACCEPT);
        //附加新连接处理handler处理器到SelectionKey（选择键）
        sk.attach(new AcceptorHandler());

        //第一个子反应器，一子反应器负责一个选择器
        SubReactor subReactor1 = new SubReactor(selectors[0]);
        //第二个子反应器，一子反应器负责一个选择器
        SubReactor subReactor2 = new SubReactor(selectors[1]);
        subReactors = new SubReactor[]{subReactor1, subReactor2};
    }

    private void startService() {
        // 一子反应器对应一条线程
        new Thread(subReactors[0]).start();
        new Thread(subReactors[1]).start();
    }

    //反应器
    class SubReactor implements Runnable {
        //每条线程负责一个选择器的查询
        final Selector selector;

        public SubReactor(Selector selector) {
            this.selector = selector;
        }

        public void run() {
            try {
                while (!Thread.interrupted()) {
                    selector.select();
                    Set<SelectionKey> keySet = selector.selectedKeys();
                    Iterator<SelectionKey> it = keySet.iterator();
                    while (it.hasNext()) {
                        //Reactor负责dispatch收到的事件
                        SelectionKey sk = it.next();
                        dispatch(sk);
                    }
                    keySet.clear();
                }
            } catch (IOException ex) {
                ex.printStackTrace();
            }
        }


        void dispatch(SelectionKey sk) {
            Runnable handler = (Runnable) sk.attachment();
            //调用之前attach绑定到选择键的handler处理器对象
            if (handler != null) {
                handler.run();
            }
        }
    }


    // Handler:新连接处理器
    class AcceptorHandler implements Runnable {
        public void run() {
            try {
                SocketChannel channel = serverSocket.accept();
                if (channel != null)
                    new MultiThreadEchoHandler(selectors[next.get()], channel);
            } catch (IOException e) {
                e.printStackTrace();
            }
            if (next.incrementAndGet() == selectors.length) {
                next.set(0);
            }
        }
    }


    public static void main(String[] args) throws IOException {
        MultiThreadEchoServerReactor server =
                new MultiThreadEchoServerReactor();
        server.startService();
    }

}
```

```java
class MultiThreadEchoHandler implements Runnable {
    final SocketChannel channel;
    final SelectionKey sk;
    final ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
    static final int RECIEVING = 0, SENDING = 1;
    int state = RECIEVING;
    //引入线程池
    static ExecutorService pool = Executors.newFixedThreadPool(4);

    MultiThreadEchoHandler(Selector selector, SocketChannel c) throws IOException {
        channel = c;
        c.configureBlocking(false);
        //仅仅取得选择键，后设置感兴趣的IO事件
        sk = channel.register(selector, 0);
        //将本Handler作为sk选择键的附件，方便事件dispatch
        sk.attach(this);
        //向sk选择键注册Read就绪事件
        sk.interestOps(SelectionKey.OP_READ);
        selector.wakeup();
    }

    public void run() {
        //异步任务，在独立的线程池中执行
        pool.execute(new AsyncTask());
    }

    //异步任务，不在Reactor线程中执行
    public synchronized void asyncRun() {
        try {
            if (state == SENDING) {
                //写入通道
                channel.write(byteBuffer);
                //写完后,准备开始从通道读,byteBuffer切换成写模式
                byteBuffer.clear();
                //写完后,注册read就绪事件
                sk.interestOps(SelectionKey.OP_READ);
                //写完后,进入接收的状态
                state = RECIEVING;
            } else if (state == RECIEVING) {
                //从通道读
                int length = 0;
                while ((length = channel.read(byteBuffer)) > 0) {
                    Logger.info(new String(byteBuffer.array(), 0, length));
                }
                //读完后，准备开始写入通道,byteBuffer切换成读模式
                byteBuffer.flip();
                //读完后，注册write就绪事件
                sk.interestOps(SelectionKey.OP_WRITE);
                //读完后,进入发送的状态
                state = SENDING;
            }
            //处理结束了, 这里不能关闭select key，需要重复使用
            //sk.cancel();
        } catch (IOException ex) {
            ex.printStackTrace();
        }
    }

    //异步任务的内部类
    class AsyncTask implements Runnable {
        public void run() {
            MultiThreadEchoHandler.this.asyncRun();
        }
    }

}
```

# Future异步回调模式

## join异步阻塞

假设有三件任务A、B、C。C任务需要等到A、B都完成后才可以执行。

此时可以将A、B、C这三件任务放到三个线程中，在C任务线程中调用A、B任务线程的join方法进行阻塞。

join操作的原理是：阻塞当前的线程，直到准备合并的目标线程的执行完成。

但是join有个缺陷就是没有返回值，我们在C任务线程中无法知道A、B任务线程的结果。

## FutureTask异步回调之重武器

Runnable在Java多线程中表示线程业务代码的抽象接口。但是Runnable有一个很严重的问题就是，他没有返回值，所以Runnable不可以应用到有返回值的场景。

为了解决Runnable接口的问题，Java定义了一个新的和Runnable类似的接口—— Callable接口。并且将其中的代表业务处理的方法命名为call, call方法有返回值。

```java
@FunctionalInterface
public interface Callable<V> {
    
    V call() throws Exception;
}
```

Callable接口可以对应到Runnable接口；Callable接口的call方法可以对应到Runnable接口的run方法。相比较而言，Callable接口的功能更强大一些。

Callable接口与Runnable接口相比，还有一个很大的不同：Callable接口的实例不能作为Thread线程实例的target来使用；而Runnable接口实例可以作为Thread线程实例的target构造参数，开启一个Thread线程。

Java中的线程类型，只有一个Thread类，没有其他的类型。如果Callable实例需要异步执行，就要想办法赋值给Thread的target成员，一个Runnable类型的成员。为此，Java提供了在Callable实例和Thread的target成员之间一个搭桥的类——FutureTask类。

FutureTask类的构造函数的参数为Callable类型，实际上是对Callable类型的二次封装，可以执行Callable的call方法。FutureTask类间接地继承了Runnable接口，从而可以作为Thread实例的target执行目标。

FutureTask类就像一座搭在Callable实例与Thread线程实例之间的桥。FutureTask类的内部封装一个Callable实例，然后自身又作为Thread线程的target。

## Netty的异步回调模式

Netty继承和扩展了JDK Future系列异步回调的API，定义了自身的Future系列接口和类，实现了异步任务的监控、异步执行结果的获取。引入了GenericFutureListener接口作为Future执行完毕的回调接口。

```java
public interface GenericFutureListener<F extends Future<?>> extends EventListener {

    void operationComplete(F future) throws Exception;
}
```

GenericFutureListener只有一个方法，该方法在异步任务完成时回调。EventListener是一个空接口，仅作标识作用。

通过Netty中的Future接口中addListener与removeListener可以对Future注册的GenericFutureListener进行管理。

Netty的Future接口一般不会直接使用，而是会使用子接口。Netty有一系列的子接口，代表不同类型的异步任务，如ChannelFuture。ChannelFuture子接口表示通道IO操作的异步任务；如果在通道的异步IO操作完成后，需要执行回调操作，就需要使用到ChannelFuture接口。

在Netty中所有的操作都是异步的，在处理网络连接通道的输入和输出处理时都会返回ChannelFuture实例。

```java
Bootstrap b = new Bootstrap();
//连接操作是异步的，返回Future
ChannelFuture f = b.connect();
//设置Future回调
f.addListener((ChannelFuture futureListener) ->
{
    if (futureListener.isSuccess()) {
        Logger.info("EchoClient客户端连接成功!");

    } else {
        Logger.info("EchoClient客户端连接失败!");
    }

});
```

# Netty中的Reactor模式

回忆一下Reactor模式，其实就是选择器Selector所在的线程成为反应器线程，然后轮询注册在之上的Channel，轮询到事件时就响应事件，事件执行逻辑在Handler中。在NIO中通过SelectionKey的附件保存Handler。单线程Reactor模式为Reactor线程的事件轮询与Handler的事件处理都位于一个线程。多线程Reactor是将Handler的处理放到线程池中，不会阻塞Reactor线程。在多线程Reactor模式中Reactor线程也会可以有多个。

## Netty中的Channel

Channel是Netty中比较重要的角色，原因是：反应器模式和通道紧密相关，反应器的查询和分发的IO事件都来自于Channel通道组件。

Netty中不直接使用NIO中的Channel通道，而是对Channel通道组件进行了自己的封装。在Netty中有一系列通道，对于每一种协议，Netty都实现了自己的通道。Netty除了提供NIO通道还提供了OIO类型通道。常用Channel如下：

+ NioSocketChannel：异步非阻塞TCP Socket传输通道。
+ NioServerSocketChannel：异步非阻塞TCP Socket服务器端监听通道。
+  NioDatagramChannel：异步非阻塞的UDP传输通道。
+  NioSctpChannel：异步非阻塞Sctp传输通道。
+ NioSctpServerChannel：异步非阻塞Sctp服务器端监听通道。
+  OioSocketChannel：同步阻塞式TCP Socket传输通道。
+  OioServerSocketChannel：同步阻塞式TCP Socket服务器端监听通道。
+  OioDatagramChannel：同步阻塞式UDP传输通道。
+ OioSctpChannel：同步阻塞式Sctp传输通道。
+ OioSctpServerChannel：同步阻塞式Sctp服务器端监听通道。

在Netty的NioSocketChannel内部封装了一个java NIO的SelectableChannel成员。通过内部的Java NIO通道，Netty的NioSocketChannel通道上的IO操作，最终会落地到Java NIO的SelectableChannel底层通道。

![image-20200720211459193](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20200720211459193.png)

## Netty中的Reactor反应器

在反应器模式中，一个反应器会负责一个事件处理线程，不断地轮询，通过Selector选择器不断查询注册过的IO事件（选择键）。如果查询到IO事件，则分发给Handler业务处理器。

Netty中的反应器有多个实现类，与Channel通道类有关系。对应于NioSocketChannel通道，Netty的反应器类为：NioEventLoop。

NioEventLoop有两个重要的成员属性，一个是Thread类成员，另一个是Java NIO的选择器Selector。

![image-20200720211803485](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20200720211803485.png)

在Netty中，EventLoop反应器可以注册多个NettyChannel。

![image-20200720211908336](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20200720211908336.png)

## Netty中的Handler

在Netty中，EventLoop反应器内部有一个Java NIO选择器成员 执行以上事件的查询，然后进行对应的事件分发。事件分发（Dispatch）的目标就是Netty自己的Handler处理器。

Netty的Handler处理器分为两大类：第一类是ChannelInboundHandler通道入站处理器；第二类是ChannelOutboundHandler通道出站处理器。二者都继承了ChannelHandler处理器接口。继承关系如下：

![image-20200720213337461](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20200720213337461.png)

ChannelOutboundHandler与ChannelInboundHandler处理器，都有自己的默认adapter实现，在实现自己的Handler时只需要继承Adapter即可。

## Netty的流水线(pipeline)

Netty中Channel与Handler之间是多对多的关系，也就是一个Channel可以绑定多个Handler，一个Handler可以被多个Channel绑定。在Netty中通过ChannelPipeline来组织Channel与Handler之间的关系。

ChannelPipeline的实现方式是一个双向链表，所有的Handler处理器被包装为双向链表的结点。

一个Netty通道拥有一条Handler处理器流水线，成员的名称叫作pipeline。这里就是Channel与事件Handler的绑定原理，通过Netty的Channel中Pipeline绑定。

Handler处理器分为入站处理器和出站处理器。这两种处理器都在同一条流水线上。

![image-20200720214513416](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20200720214513416.png)

入站的IO操作只会且只能从Inbound入站处理器类型的Handler流过；出站的IO操作只会且只能从Outbound出站处理器类型的Handler流过。

# Bootstrap启动类

Bootstrap类是Netty提供的一个便利工厂类，可以通过它完成客户端与服务端的Netty组件组转，以及Netty的初始化。

在Netty中，有两个启动器类，分别用在服务端和客户端。

![image-20200720215513812](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20200720215513812.png)

## 父子通道

在Netty中，每一个NioSocketChannel通道所封装的是Java NIO通道，再往下就对应到了操作系统底层的socket描述符。理论上操作系统的socket描述符分为两类，连接监听类型和数据传输类型。

Netty中的NioServerSocketChannel通道用来监听连接，NioSocketChannel用来传输数据的通道。

在Netty中，将有接收关系的NioServerSocketChannel和NioSocketChannel，叫作父子通道。其中，NioServerSocketChannel负责服务器连接监听和接收，也叫父通道（ParentChannel）。对应于每一个接收到的NioSocketChannel传输类通道，也叫子通道（ChildChannel）。

## EventLoopGroup线程组

在Netty中一个EventLoop相当于一个子反应器。多个Eventloop线程组成一个EventLoopGroup线程组。

也就是说，Netty的EventLoopGroup线程组就是一个多线程版本的反应器。而其中的单个EventLoop线程对应于一个子反应器（SubReactor）。

Netty程序不会直接使用Eventloop线程，而是使用EventLoopGroup线程组。EventLoopGroup的构造函数有一个参数，用于指定内部的线程数。

为了及时接受（Accept）到新连接，在服务器端，一般有两个独立的反应器，一个反应器负责新连接的监听和接受，另一个反应器负责IO事件处理。对应到Netty服务器程序中，则是设置两个EventLoopGroup线程组，一个EventLoopGroup负责新连接的监听和接受，一个EventLoopGroup负责IO事件处理。

通常负责新连接监听的EventLoopGroup线程组，查询父类通道的IO事件。另一个EventLoopGroup线程组负责查询所有子通道的IO事件，执行Handler处理器。

## Bootstrap启动流程

Bootstrap的启动流程，也就是Netty组件的组装、配置，以及Netty服务器或者客户端的启动流程。总共可以分为八步。

```java
//创建一个服务器端的启动器
ServerBootstrap b = new ServerBootstrap();
```

1. 创建reactor线程组，并赋值给ServerBootstrap。当然也可以只设置一个，只不过这样就会使监听和工作的线程组

   ```java
   //创建reactor 线程组
   EventLoopGroup bossLoopGroup = new NioEventLoopGroup(1);
   EventLoopGroup workerLoopGroup = new NioEventLoopGroup();
   
   //1 设置reactor 线程组
   b.group(bossLoopGroup, workerLoopGroup);
   ```

2. 设置通道IO类型

   ```java
   //2 设置nio类型的channel
   b.channel(NioServerSocketChannel.class);
   ```

3. 设置监听端口

   ```java
   //3 设置监听端口
   b.localAddress(8080);
   ```

4. 设置传输通道的配置选项

   ```java
   //4 设置通道的参数
   b.option(ChannelOption.SO_KEEPALIVE, true);
   b.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
   b.childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
   ```

   Bootstrap的option() 选项设置方法。对于服务器的Bootstrap而言，这个方法的作用是：给父通道（Parent Channel）接收连接通道设置一些选项。

   如果要给子通道（Child Channel）设置一些通道选项，则需要用另外一个childOption()设置方法。

   可以设置哪些通道选项（ChannelOption）呢？在上面的代码中，设置了一个底层TCP相关的选项ChannelOption.SO_KEEPALIVE。该选项表示：是否开启TCP底层心跳机制，true为开启，false为关闭。

5. 装配子通道Pipeline

   ```java
   //5 装配子通道流水线
   b.childHandler(new ChannelInitializer<SocketChannel>() {
       //有连接到达时会创建一个channel
       protected void initChannel(SocketChannel ch) throws Exception {
           // pipeline管理子通道channel中的Handler
           // 向子channel流水线添加一个handler处理器
           ch.pipeline().addLast(NettyEchoServerHandler.INSTANCE);
       }
   });
   ```

   如果为父通道设置流水线，可以使用ServerBootstrap的handler方法，为父类设置ChannelInitializer初始化器。

6. 开始绑定服务器的端口

   ```java
   // 6 开始绑定server
   // 通过调用sync同步方法阻塞直到绑定成功
   ChannelFuture channelFuture = b.bind().sync();
   Logger.info(" 服务器启动成功，监听端口: " +
           channelFuture.channel().localAddress());
   ```

   b.bind()方法的功能：返回一个端口绑定Netty的异步任务channelFuture。在这里，并没有给channelFuture异步任务增加回调监听器，而是阻塞channelFuture异步任务，直到端口绑定任务执行完成。

7. 阻塞直到通道关闭

   ```java
   // 7 等待通道关闭的异步任务结束
   // 服务监听通道会一直等待通道关闭的异步任务结束
   ChannelFuture closeFuture = channelFuture.channel().closeFuture();
   closeFuture.sync();
   ```

8. 关闭EventLoopGroup

   关闭Reactor反应器线程组，同时关闭内部的子反应器线程，这也会关闭内部的Selector选择器，内部轮询线程以及负责查询的所有子通道。

   ```java
   // 8 优雅关闭EventLoopGroup，
   // 释放掉所有资源包括创建的线程
   workerLoopGroup.shutdownGracefully();
   bossLoopGroup.shutdownGracefully();
   ```

# Channel

在Netty中，Channel代表着网络连接。通道的抽象类AbstractChannel的构造函数可以知道，每个Channel都包含一个Pipeline与一个父通道。

```java
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    unsafe = newUnsafe();
    pipeline = new DefaultChannelPipeline(this);
}
```

如果是监听通道，则父通道为null。如果是传输通道，则父通道为监听通道。

Channel中的常用方法：

+ ChannelFuture connect(SocketAddress address)：连接远程服务器。
+ ChannelFuture bind（SocketAddress address): 绑定监听地址，开始监听新的客户端连接。
+ ChannelFuture close()：关闭连接通道。
+ Channel read()：读取通道数据，并且启动入站处理。
+ ChannelFuture write（Object o）：启程出站流水处理，把处理后的最终数据写到底层Java NIO通道。
+ Channel flush()：将缓冲区中的数据立即写出到对端。并不是每一次write操作都是将数据直接写出到对端，write操作的作用在大部分情况下仅仅是写入到操作系统的缓冲区，操作系统会将根据缓冲区的情况，决定什么时候把数据写到对端。而执行flush()方法立即将缓冲区的数据写到对端。

# Handler

在Reactor反应器经典模型中，反应器查询到IO事件后，分发到Handler业务处理器，由Handler完成IO操作和业务处理。

整个的IO处理操作环节包括：从通道读数据包、数据包解码、业务处理、目标数据编码、把数据包写到通道，然后由通道发送到对端。

![image-20200721220220654](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20200721220220654.png)

Netty的Handler分为入站处理器和出站处理器，数据包解码和业务处理属于入站处理器，目标数据编码和把数据包写到通道属于出站处理器的工作。

## ChannelInboundHandler出站处理器

当数据或者信息入站到Netty通道时，Netty将触发入站处理器ChannelInboundHandler所对应的入站API，进行入站操作处理。

![image-20200721221432068](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20200721221432068.png)

+ channelRegistered：当通道注册完成后，Netty会调用fireChannelRegistered，触发通道注册事件。通道会启动该入站操作的流水线处理，在通道注册过的入站处理器Handler的channelRegistered方法，会被调用到。
+ channelActive：当通道激活完成后，Netty会调用fireChannelActive，在通道注册过的入站处理器Handler的channelActive方法，会被调用到。
+ channelRead：当通道缓冲区可读，Netty会调用fireChannelRead，在通道注册过的入站处理器Handler的channelRead方法，会被调用到。
+ channelReadComplete：当通道缓冲区读完，Netty会调用fireChannelReadComplete，在通道注册过的入站处理器Handler的channelReadComplete方法，会被调用到。
+ channelInactive：当连接被断开或者不可用，Netty会调用fireChannelInactive，在通道注册过的入站处理器Handler的channelInactive方法，会被调用到。
+ exceptionCaught：当通道处理过程发生异常时，Netty会调用fireExceptionCaught，在通道注册过的处理器Handler的exceptionCaught方法，会被调用到。

Netty中，ChannelInboundHandler的默认实现为ChannelInboundHandlerAdapter，在实际开发中，只需要继承ChannelInboundHandlerAdapter，重写自己需要的方法即可。

## ChannelOutboundHandler出站处理器

在业务处理完成后，需要操作Java NIO底层通道时，可以通过一系列ChannelOutboundHandler出站处理器，完成Netty通道到底层通道的操作。如建立底层连接、断开底层连接、写入底层Java NIO通道等。

![image-20200721222715208](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20200721222715208.png)

+ bind：监听地址（IP+端口）绑定：完成底层Java IO通道的IP地址绑定。如果使用TCP传输协议，这个方法用于服务器端。
+ connect：连接服务端：完成底层Java IO通道的服务器端的连接操作。如果使用TCP传输协议，这个方法用于客户端。
+ write：写数据到底层：完成Netty通道向底层Java IO通道的数据写入操作。此方法仅仅是触发一下操作而已，并不是完成实际的数据写入操作。
+ flush：腾空缓冲区中的数据，把这些数据写到对端：将底层缓存区的数据腾空，立即写出到对端。
+ read：从底层读数据：完成Netty通道从Java IO通道的数据读取。
+ disConnect：断开服务器连接：断开底层Java IO通道的服务器端连接。如果使用TCP传输协议，此方法主要用于客户端。
+ close：主动关闭通道：关闭底层的通道，例如服务器端的新连接监听通道。

# Pipeline

Pipeline是Netty用来组织Handler的组件，它内部是一个双向链表，支持动态添加和删除Handler。

## Pipeline入站处理流程

```java
public class InPipeline {
    static class SimpleInHandlerA extends ChannelInboundHandlerAdapter {
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            Logger.info("入站处理器 A: 被回调 ");
            super.channelRead(ctx, msg);
        }
    }
    static class SimpleInHandlerB extends ChannelInboundHandlerAdapter {
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            Logger.info("入站处理器 B: 被回调 ");
            super.channelRead(ctx, msg);
        }
    }
    static class SimpleInHandlerC extends ChannelInboundHandlerAdapter {
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            Logger.info("入站处理器 C: 被回调 ");
            super.channelRead(ctx, msg);
        }
    }


    @Test
    public void testPipelineInBound() {
        ChannelInitializer i = new ChannelInitializer<EmbeddedChannel>() {
            protected void initChannel(EmbeddedChannel ch) {
                ch.pipeline().addLast(new SimpleInHandlerA());
                ch.pipeline().addLast(new SimpleInHandlerB());
                ch.pipeline().addLast(new SimpleInHandlerC());

            }
        };
        EmbeddedChannel channel = new EmbeddedChannel(i);
        ByteBuf buf = Unpooled.buffer();
        buf.writeInt(1);
        //向通道写一个入站报文
        channel.writeInbound(buf);
        try {
            Thread.sleep(Integer.MAX_VALUE);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

在channelRead()方法中，调用父类的channelRead()方法，会自动调用下一个inBoundHandler的channelRead()方法。这样就形成了调用链。

![image-20200721225138658](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20200721225138658.png)

## Pipeline出站处理流程

类似于入站处理，调用链也是通过调用父类方法实现。

## ChannelHandlerContext上下文

Pipeline是用来组织Handler处理器的组件，如：

```java
ChannelInitializer i = new ChannelInitializer<EmbeddedChannel>() {
    protected void initChannel(EmbeddedChannel ch) {
        ch.pipeline().addLast(new SimpleInHandlerA());
        ch.pipeline().addLast(new SimpleInHandlerB());
        ch.pipeline().addLast(new SimpleInHandlerC());

    }
};
```

其实，在addLast时会将Handler包装为ChannelHandlerContext对象。在ChannelHandlerContext对象中会包含一系列相关组件，如：

```java
private final boolean inbound;
private final boolean outbound;
private final AbstractChannel channel;
private final DefaultChannelPipeline pipeline;
private final String name;
private boolean removed;
```

Channel、Handler、ChannelHandlerContext三者的关系为：Channel通道拥有一条ChannelPipeline通道流水线，每一个流水线节点为一个ChannelHandlerContext通道处理器上下文对象，每一个上下文中包裹了一个ChannelHandler通道处理器。在ChannelHandler通道处理器的入站/出站处理方法中，Netty都会传递一个Context上下文实例作为实际参数。通过Context实例的实参，在业务处理中，可以获取ChannelPipeline通道流水线的实例或者Channel通道的实例。

## 截断流水线处理

在pipeline中出发handler链的继续执行有两种方式，一个是在channel中调用父类的相应方法，另一个是ChannelHandlerContext的fireChannelXxx()方法。

如果需要截断流程只需要在链中不调用这两个方法即可。

## Handler业务处理器的热拔插

pipeline是一个双向链表，在程序执行过程中可以动态的进行增加和删除业务处理器Handler。主要涉及的方法如下：

![image-20200721233318034](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20200721233318034.png)

# ByteBuf

Netty提供了ByteBuf来替代Java NIO的ByteBuffer缓冲区，以操纵内存缓冲区。

ByteBuf内部是一个字节数组，从逻辑上来分，可以分为四个部分。

![image-20200721233757849](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20200721233757849.png)

第一个部分是已用字节，表示已经使用完的废弃的无效字节；第二部分是可读字节，这部分数据是ByteBuf保存的有效数据，从ByteBuf中读取的数据都来自这一部分；第三部分是可写字节，写入到ByteBuf的数据都会写到这一部分中；第四部分是可扩容字节，表示的是该ByteBuf最多还能扩容的大小。

ByteBuf中有三个重要属性：

![image-20200721233913553](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20200721233913553.png)

+ readerIndex：指示读取的起始位置，没读取一个字节，readerIndex自动增加1
+ writerIndex：指示写入的起始位置。每写一个字节，writerIndex自动增加1。一旦增加到writerIndex与capacity()容量相等，则表示ByteBuf已经不可写了。
+ maxCapacity：表示ByteBuf可以扩容的最大容量。当向ByteBuf写数据的时候，如果容量不足，可以进行扩容。扩容的最大限度由maxCapacity的值来设定，超过maxCapacity就会报错。

## 主要方法

ByteBuf的方法大致可以分为三组。

第一组：容量系列· 

+ capacity()：表示ByteBuf的容量，它的值是以下三部分之和：废弃的字节数、可读字节数和可写字节数。· 
+ maxCapacity()：表示ByteBuf最大能够容纳的最大字节数。当向ByteBuf中写数据的时候，如果发现容量不足，则进行扩容，直到扩容到maxCapacity设定的上限。

第二组：写入系列· 

+ isWritable() ：表示ByteBuf是否可写。如果capacity()容量大于writerIndex指针的位置，则表示可写，否则为不可写。注意：如果isWritable()返回false，并不代表不能再往ByteBuf中写数据了。如果Netty发现往ByteBuf中写数据写不进去的话，会自动扩容ByteBuf。
+  writableBytes() ：取得可写入的字节数，它的值等于容量capacity()减去writerIndex。
+ maxWritableBytes() ：取得最大的可写字节数，它的值等于最大容量maxCapacity减去writerIndex。
+ writeBytes(byte[] src) ：把src字节数组中的数据全部写到ByteBuf。这是最为常用的一个方法。
+ writeTYPE(TYPE value）：写入基础数据类型的数据。TYPE表示基础数据类型，包含了8大基础数据类型。具体如下：writeByte()、 writeBoolean()、writeChar()、writeShort()、writeInt()、writeLong()、writeFloat()、writeDouble()。
+ setTYPE(TYPE value）：基础数据类型的设置，不改变writerIndex指针值，包含了8大基础数据类型的设置。具体如下：setByte()、 setBoolean()、setChar()、setShort()、setInt()、setLong()、setFloat()、setDouble()。setType系列与writeTYPE系列的不同：setType系列不改变写指针writerIndex的值；writeTYPE系列会改变写指针writerIndex的值。
+ markWriterIndex()与resetWriterIndex()：这两个方法一起介绍。前一个方法表示把当前的写指针writerIndex属性的值保存在markedWriterIndex属性中；后一个方法表示把之前保存的markedWriterIndex的值恢复到写指针writerIndex属性中。markedWriterIndex属性相当于一个暂存属性，也定义在AbstractByteBuf抽象基类中。

第三组：读取系列

+ isReadable( ) ：返回ByteBuf是否可读。如果writerIndex指针的值大于readerIndex指针的值，则表示可读，否则为不可读。
+ readableBytes( ) ：返回表示ByteBuf当前可读取的字节数，它的值等于writerIndex减去readerIndex。
+ readBytes(byte[] dst)：读取ByteBuf中的数据。将数据从ByteBuf读取到dst字节数组中，这里dst字节数组的大小，通常等于readableBytes()。这个方法也是最为常用的一个方法之一。
+ readType()：读取基础数据类型，可以读取8大基础数据类型。具体如下：readByte()、readBoolean()、readChar()、readShort()、readInt()、readLong()、readFloat()、readDouble()。
+  getTYPE(TYPE value）：读取基础数据类型，并且不改变指针值。具体如下：getByte()、 getBoolean()、getChar()、getShort()、getInt()、getLong()、getFloat()、getDouble()。getType系列与readTYPE系列的不同：getType系列不会改变读指针readerIndex的值；readTYPE系列会改变读指针readerIndex的值。
+  markReaderIndex( )与resetReaderIndex( ) ：这两个方法一起介绍。前一个方法表示把当前的读指针ReaderIndex保存在markedReaderIndex属性中。后一个方法表示把保存在markedReaderIndex属性的值恢复到读指针ReaderIndex中。markedReaderIndex属性定义在AbstractByteBuf抽象基类中。

# 编解码器

# 解码器Decoder

它是一个InBound入站处理器，解码器负责处理“入站数据”。它能将上一站Inbound入站处理器传过来的输入（Input）数据，进行数据的解码或者格式转换，然后输出（Output）到下一站Inbound入站处理器。

一个标准的解码器将输入类型为ByteBuf缓冲区的数据进行解码，输出一个一个的Java POJO对象。Netty内置了这个解码器，叫作ByteToMessageDecoder，位在Netty的io.netty.handler.codec包中。

## ByteToMessageDecoder

ByteToMessageDecoder继承自ChannelInboundHandlerAdapter适配器，是一个入站处理器，实现了从ByteBuf到Java POJO对象的解码功能。

其工作流程为：首先，它将上一站传过来的输入到Bytebuf中的数据进行解码，解码出一个`List<Object>`对象列表；然后，迭代`List<Object>`列表，逐个将Java POJO对象传入下一站Inbound入站处理器。

![image-20200722000627789](https://blog-1253099784.cos.ap-nanjing.myqcloud.com/image-20200722000627789.png)

自己实现一个解码器，首先继承ByteToMessageDecoder抽象类。然后，实现其基类的decode抽象方法。将解码的逻辑，写入此方法。

总体来说，如果要实现一个自己的ByteBuf解码器，流程大致如下：

1. 首先继承ByteToMessageDecoder抽象类。
2. 然后实现其基类的decode抽象方法。将ByteBuf到POJO解码的逻辑写入此方法。将Bytebuf二进制数据，解码成一个一个的Java POJO对象。
3. 在子类的decode方法中，需要将解码后的Java POJO对象，放入decode的`List<Object>`实参中。这个实参是ByteToMessageDecoder父类传入的，也就是父类的结果收集列表。

## MessageToMessageDecoder

MessageToMessageDecoder用来将一种POJO对象解码成另外一种POJO对象。

使用时需要继承MessageToMessageDecoder然后重写decode方法。

## 开箱即用的Decoder

固定长度数据包解码器——FixedLengthFrameDecoder

行分割数据包解码器——LineBasedFrameDecoder

自定义分隔符数据包解码器——DelimiterBasedFrameDecoder

自定义长度数据包解码器——LengthFieldBasedFrameDecoder

# 编码器Encoder

接口MessageToByteEncoder实现POJO编码为ButeBuf

MessageToMessageEncoder将POJO编码为POJO

## 编解码器 Codec

编解码器同时拥有编码器和解码器的功能。

ByteToMessageCodec编解码器，完成POJO到ByteBuf数据包的配套的编码器和解码器的基类，叫作ByteToMessageCodec，它是一个抽象类。从功能上说，继承它，就等同于继承了ByteToMessageDecoder解码器和MessageToByteEncoder编码器这两个基类。

CombinedChannelDuplexHandler组合器。前面的编码器和解码器相结合是通过继承完成的。继承的方式有其不足，在于：将编码器和解码器的逻辑强制性地放在同一个类中，在只需要编码或者解码单边操作的流水线上，逻辑上不大合适。CombinedChannelDuplexHandler使用组合的 方式将编码器和解码器组合起来













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



