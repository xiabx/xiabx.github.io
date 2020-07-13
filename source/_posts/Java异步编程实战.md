---
title: Java异步编程实战
author: XIA
categories:
  - null
tags:
  - null
date: 2020-06-30 20:33:47
---

# 认识异步编程

## 异步编程场景

有时候我们会有这么一种场景，需要异步的处理一些事情，但是不需要知道结果。这时候可以调用线程异步打印日志，将主线程中需要打印的日志放到一个内存队列，然后负责日志打印的线程从队列中获取打印信息然后打印日志。

有时候需要异步执行任务时，可以直接开启一个线程来执行，也可以使用spring框架的@Async注解把一个任务异步化。

有时候我们需要获得异步执行任务的结果，这时候就可以使用Future对象，Future实质是通过两个线程间的共享内存来实现结果传递。

Future确实可以获取异步任务的执行结果，但是获取的过程是阻塞的。在JDK8中CompletableFuture弥补了这个缺陷。它可以设置回调函数，解放了主线程在获取异步任务执行结果时的同步等待，让主线程彻底解放出来。

通过JDK8提供的Stream和CompletableFuture可以比较完美的实现异步编程，但是还存在：流只可以用一次、缺少与时间相关操作、无法指定线程池的问题。这时候Reactor、RxJava这些异步编程框架就解决了这些问题。

使用CompletableFuture结合Netty可以实现网络请求的异步调用，Netty实现异步非阻塞功能，CompletableFuture实现编排功能。

在servlet3.0之前，容器对请求的处理都是一个线程对应一个请求的模式。这样处理很耗时而且容易把容器内线程池中线程耗尽。在servlet3.0提供了异步能力，让容器线程可以及时释放，让业务自己的线程池处理后续任务。但是io还是阻塞的。

在servlet3.1中实现了非阻塞IO。但是servlet受限于规范导致异步并不彻底，比如Filter执行等。所以诞生了webflux基于Netty实现天然异步、非阻塞处理的框架。

# 显式使用线程和线程池实现异步编程

## 直接使用线程

直接使用线程较直接且简单，但是存在三个问题：1.每次运行异步任务都会创建与销毁一个线程，开销较大。2.返回值问题。3.每次都需要显式创建线程属于命令式编程。

解决以上三个问题的方法就是：1.使用线程池2.使用Future来接受返回值3.使用Java提供的类库

## 使用线程池

使用线程池解决了上面直接使用线程带来的第一个问题，使用时只需要将任务提交到线程池即可，然后线程池会自动分配线程执行任务。

**线程池ThreadPoolExecutor原理**

ThreadPoolExecutor中的属性：

```java
    //用来标记线程池状态（高3位）与线程个数（低29位），默认RUNNING状态，线程个数为0
	private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
	//线程个数掩码位数，为Integer的二进制位数-3的剩余位数
    private static final int COUNT_BITS = Integer.SIZE - 3;
	//线程的最大个数，低29位
    private static final int COUNT_MASK = (1 << COUNT_BITS) - 1;

    // 线程池的主要状态
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;

```



# 基于反应式编程实现异步编程

反应式编程是一种涉及数据流和变化传播的异步编程范式。例如在命令式编程中a=b+c，在b和c相加赋值给a后，再改变b或者c的值对a没有影响。而在反应式编程中a的值则会随b和c的值变化而变化。

**为什么需要异步反应式编程库**































