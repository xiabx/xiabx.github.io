---
title: Java异步编程实战
author: XIA
categories:
  - null
tags:
  - null
date: 2020-06-30 20:33:47
---

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































