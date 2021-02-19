---
title: Java多线程-基础
author: XIA
categories:
  - 多线程
tags:
  - null
date: 2019-02-28 17:56:01
---

# 认识线程

## 进程、线程、任务

三者的关系：进程是程序运行的实例，一个进程可以包含多个线程，每个线程索要完成的工作就是任务。

程序与进程的关系就好比播放器中播放的视频与相应的视频文件的关系，程序从静态角度刻画事物，进程从动态角度刻画事物。

进程和线程的关系可以比作饭店和员工，一个饭店对外提供餐饮服务，而服务最终由员工完成。而员工们则共享饭店的资源。

任务则代表线程要完成的工作，可以是一次网络下载、一次文件解压等。

## 线程的创建、启动与运行

Java中创建一个线程就是创建一个Thread类的实例，线程的任务逻辑位于Thread类的run方法中。启动一个线程则是调用start方法。创建的每个线程都有自己的名字。但是创建Thread对象需要额外分配调用栈，所以创建线程对象的成本会比创建普通对象的成本要高一些。Thread类有两个常用构造器：`Thread()和Thread(Runnable target)`。

Thread的run方法，target表示Runnable对象。

```java
public void run() {
    if (target != null) {
        target.run();
    }
}
```

不管是哪种方式创建的线程，一旦run方法执行结束，线程所占用的资源就会被jvm回收。且每个线程实例的start方法只能调用一次。

## 线程属性

![image-20200228201910397](https://xbxblog2.bj.bcebos.com/java%E5%A4%9A%E7%BA%BF%E7%A8%8B-%E5%9F%BA%E7%A1%80%2Fimage-20200228201910397.png)

## Thread类常用方法

![image-20200228202105891](https://xbxblog2.bj.bcebos.com/java%E5%A4%9A%E7%BA%BF%E7%A8%8B-%E5%9F%BA%E7%A1%80%2Fimage-20200228202105891.png)

## 线程的层次关系

线程之间存在着层级关系，例如我们在main线程中创建了线程thread-01，那么main就称为线程thread-01的父线程，反之亦然。

默认情况下父线程的优先级、是否为守护线程的属性决定子线程的这些属性。但是父子线程的生命周期却没有必然的联系。

## 线程的生命周期

线程的状态可以使用Thread.getState()方法获取。线程的状态包括以下几种：

+ NEW:线程已经被创建但是未启动就处于该状态。
+ RUNNABLE:处于该状态的线程被称为活跃线程。该状态包括两个子状态：READY和RUNNING。前者表示可以线程可以被调度器调度使之处于RUNNING状态，而后者表示该状态的线程正在运行。执行Thread.yield()的线程可能其状态会由RUNNING转为READY。
+ BLOCKED：一个线程发起一个阻塞式IO操作或者申请一个其他线程独占的资源，相应的线程就会处于该状态。处于该状态的线程不会占用cpu资源，当等待的资源获得后可以转为RUNNABLE状态。
+ WAITING：一个线程执行了`Object.wait(),Thread.join(),LockSupport.patk(object) `方法后会处于该等待状态。当`Object.notify()/notifyAll()和LockSupport.unpark(object)`方法被调用后等待状态收到通知就会变为RUNNABLE状态。
+ TIMED_WAITING：与WAITING类似，不同之处在于该状态不会无限期的等下去，当在指定时间内没有收到指定的通知后会自动转变为RUNNABLE状态。
+ TERMINATED：已经执行结束的线程处于该状态。

![image-20200228204718636](https://xbxblog2.bj.bcebos.com/java%E5%A4%9A%E7%BA%BF%E7%A8%8B-%E5%9F%BA%E7%A1%80%2Fimage-20200228204718636.png)



# 多线程编程目标与挑战

## 串行、并发与并行

假设由三件事情：A、B、C。串行的处理方式为先处理A，待A完成再做B然后再C。并发的方式为处理A时若处理过程中A需要等待则会在等待时去处理B。而并行则是A、B、C一起做。

并发的极致就是并行。

## 竞态

在多个线程同时操作一个属性时，由于多线程的运行顺序是受操作系统控制的，在运行期由于竞态条件的存在会产生属性的操作结果与预期不符的情况。

一个竞态例子：

```java
public class TicketSell implements Runnable {

    private int ticketNum = 10;

    public void sell(){
        if (ticketNum > 0){
            System.out.println(Thread.currentThread().getName()+":  " +ticketNum);
            ticketNum--;
        }
    }

    @Override
    public void run() {
        for (int i = 0; i < 10; i++) {
            sell();
        }
    }
}


class TicketMain{
    public static void main(String[] args) {
        TicketSell ticketSell = new TicketSell();
        Thread thread = new Thread(ticketSell);
        Thread thread1 = new Thread(ticketSell);
        thread.start();
        thread1.start();
    }
}

```

![image-20200228225915379](https://xbxblog2.bj.bcebos.com/java%E5%A4%9A%E7%BA%BF%E7%A8%8B-%E5%9F%BA%E7%A1%80%2Fimage-20200228225915379.png)

这里两个线程都对ticketNum属性进行操作，当两个线程操作过程中没有实行同步机制则会出现竞态。其原因是当某个线程在判断if条件后暂停或者执行ticketNum--操作过程中被暂停。

这个例子包含两种竞态条件：read-modify-write和check-then-act

read-modify-write使竞态发生的原因在于执行`ticketNum--`过程中，实际上`ticketNum--`分解为三条指令，`分别为读取ticketNum值到寄存器--将寄存器中的ticketNum减1--将计算后的ticketNum赋值到内存中的ticketNum`。这三步若是中间被暂停使另一个线程运行更新了ticketNum值，则会导致丢失更新的问题。

check-then-act发生在if判断后，若判断ticketNum > 0为true后，该线程被暂停，另一个线程随后将ticketNum 更新，那么上一个线程就产生了读到过期数据的脏读。

若要避免竞态可以在sell方法使用synchronized关键字。

## 线程安全性

如果一个类在多线程环境下在其使用发不必为其做任何改变的情况下也能正常运行，那么这个类就是线程安全的。

一个类如果不是线程安全的，那么在多线程环境下使用就会存在线程安全问题。线程安全问题表现在三个方面：原子性、可见性、有序性。

### 原子性

原子性的含义为不可分割，那么什么是不可分割？假设有两个线程，当一个线程对一组共享变量操作时，从另一个线程看来这个操作要么还没进行要么已经结束。另一个线程时不可以获取到正在操作共享变量所产生的中间结果的。

以一个设置ip与端口的例子。当一个线程在执行updateHostInfo方法，一个线程执行connnect方法。那么如果没有保证更新ip与port属性的原子性，那么就可能存在一个线程更新了ip属性但是还未更新port属性，另一个线程则读取了新设置的ip与原始port。

```java
public class AtomicityExample {
    
    private HostInfo hostInfo;
    
    public void updateHostInfo(String ip,int port){
        hostInfo.setIp(ip);
        hostInfo.setPort(port);
    }
    
    public void connnect(){
        String ip = hostInfo.getIp();
        int port = hostInfo.getPort();
        //do connect
    }
    
    
    public  static class HostInfo{
        private String ip;
        private int port;

        // getter  setter
    }
    
}
```

保障原子性有两种方式：使用锁和CAS

还可以使用java语言特性来对原子性进行保障，如下，使用新建一个对象，然后对hostInfo引用进行更新，在java中对引用型变量的读写操作都是原子性的。

```java
public void updateHostInfo(String ip,int port){
    HostInfo newHostInfo = new HostInfo(ip, port);
    hostInfo = newHostInfo;//原子操作
}
```

### 可见性

如果一个线程对共享变量更新以后，其他线程在访问这个变量时无法读取到这个变量更新后的结果，那么这种情况就发生了可见性问题。

发生可见性问题的原因有两种：JIT编译器对代码优化产生的副作用导致和计算机存储结构的原因。前者是编译器在运行时为了提高代码效率将变量编译为常量导致，后者则是由于存储结构设计导致，位于寄存器中的写缓冲器刷新不及时以及无效化队列未及时更新。

这两种原因都可以使用volatile关键字修饰变量进行解决。

单处理器也会发生可见性问题，这是因为单处理器系统是通过时间片分配来实现多线程的，时间片分配时会将寄存器中的内容保存在线程上下文中，一个线程无法访问其他线程的上下文，这就导致了可见性问题。

根据java语言规范，父线程在启动子线程之前对共享变量的更新对子线程是可见的。同时也保证一个线程终止后该线程对共享变量的更新对于调用该线程join方法的线程而言是可见的。

但是在当前处理器的设计中，可见性问题出现的概率非常低，但是仍不能存在侥幸心理。

> **相对新值和最新值**
>
> 对于同一个共享变里而言，一个线程更新了该变量的值之后，其他线程能够读取到这个更新后的值，那么这个值就被称为该变里的相对新值 。如果读取这个共享变量的线程在读取并使用该变量的时候其他线程无法更新该变量的值，那么该线程读取到的相对新值就被称为该变量的最新值。

### 有序性

有序性是指一个线程看另一个线程的指令运行顺序，可能由于编译器或者存储子系统的原因导致实际顺序并不是按照源代码指定的顺序运行的。

使程序运行顺序发生改变的原因是由于重排序产生的，当然这在单线程程序中并不会有什么影响，因为重排序都是编译器或处理器为了提高性能按照一定规则进行的，但是在多线程程序中就会产生问题。

一个程序的运行顺序有以下几个：

+ 源代码顺序：源代码中指定的指令操作顺序
+ 程序顺序：源代码经过javac编译器、JIT编译器编译后生成的指令的顺序
+ 执行顺序：处理器上实际执行的顺序
+ 感知顺序：其他处理器所看到的该处理器的指令执行顺序，表现在内存访问操作的顺序

在此基础上，将重排序分为指令重排序和存储子系统重排序：

![image-20200229174625154](https://xbxblog2.bj.bcebos.com/java%E5%A4%9A%E7%BA%BF%E7%A8%8B-%E5%9F%BA%E7%A1%80%2Fimage-20200229174625154.png)

**指令重排序**

根据上面表格可以知道，指令重排序表现为两种。

第一种主要发生在JIT编译器中，通常javac编译器不会对指令进行重排序。代表性的例子是new对象过程的重排序。如下图，创建一个新对象会被分解为三个指令，JIT有时会为了执行效率将指令3重排序到指令2之前，此时如果在存在多个线程操作该新建对象，那么就可能导致其操作的对象是一个并未初始化的对象。

![image-20200229175148675](https://xbxblog2.bj.bcebos.com/java%E5%A4%9A%E7%BA%BF%E7%A8%8B-%E5%9F%BA%E7%A1%80%2Fimage-20200229175148675.png)

第二中情况发生在处理器重排序中，有时处理器并不会按照指令的顺序执行，他会根据指令的准备状态进行"猜测执行"。常见例子是if语句，有时处理器并不会先判断if语句的结果，而是先处理if语句块中的指令，然后将结果保存到重排序缓冲器中，然后再去计算if语句的结果，若为true则将重排序缓冲器中的内容保存到内存中，若为false则丢弃结果。这么做在单线程中没有问题，多线程情况下就会产生问题了。

**存储子系统重排序**

存储子系统是指写缓冲器和高速缓存。存储子系统重排序并不是像只能怪重排序那样，实实在在的将执行顺序改变了。存储子系统重排序是另一个线程看一个线程对内存的操作顺序发生了改变，这种改变是由于存储子系统的原因导致没有将数据按照指令顺序进行存取。该类型的重排序包括以下几种：

![image-20200229181037160](https://xbxblog2.bj.bcebos.com/java%E5%A4%9A%E7%BA%BF%E7%A8%8B-%E5%9F%BA%E7%A1%80%2Fimage-20200229181037160.png)

每种cpu对这些重排序的支持程度是不同的。

StoreStore重排序举例：一个线程先后更新了a、b两个变量的值，这两个变量的值被保存在写缓冲器中。但是写缓冲器并没有遵顼FIFO先入先出原则，导致b变量的更新值早于a变量的更新值被写入高速缓存或内存中。此时就发生了重排序，在多线程环境下这可能就要出问题。

## 上下文切换

在单核处理器上实现多线程是通过时间片轮转实现的，当一个线程因为时间片用尽或者由于自身原因(sleep、wait等)导致导致线程需要切换，此时会将该线程的上下文打包进内存中以待下次继续运行，上下文主要包括寄存器和程序计数器中的内容。

从java角度来看，一个线程在RUNNABLE与非RUNNABLE之间的转换就发生了上下文切换。

并不是只有单核处理器才会产生上下文切换，由于线程/进程远远比处理器核数多，所以多核处理器同样也存在上下文切换问题。
