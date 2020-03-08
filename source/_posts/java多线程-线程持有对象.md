---
title: java多线程-线程持有对象
author: XIA
categories:
  - null
tags:
  - null
date: 2020-03-07 19:05:34
---

# 线程持有对象

如果多个线程需要共享同一个非线程安全对象，那么我们往往需要借助锁来保障线程安全。事实上，我们也可以选择不共享非线程安全的对象，对于非线程安全的对象，每个线程都创建一个该对象的实例，各个线程仅访问各自的实例，且一个线程不能访问另一个线程创建的实例。这种各个线程创建各自的实例，一个实例只能被一个线程访问的对象就被称为线程持有对象。

`ThreadLocal<T>`相当于线程访问其线程特有对象的代理，即各个线程通过这个对象可以创建并访问各自线程持有对象。一个线程可以使用不同的ThreadLocal实例来创建并访问其不同的线程持有对象，多个线程使用同一个ThreadLocal实例所访问到的对象是类型T的不同实例。

![image-20200307224817168](https://xbxblog2.bj.bcebos.com/java%E5%A4%9A%E7%BA%BF%E7%A8%8B-threadLocal%2Fimage-20200307224817168.png)

ThreadLocal常用方法：

![image-20200307224911192](https://xbxblog2.bj.bcebos.com/java%E5%A4%9A%E7%BA%BF%E7%A8%8B-threadLocal%2Fimage-20200307224911192.png)

ThreadLocal实例通常会被作为某个类的静态字段使用，这时因为如果把ThreadLocal实例作为实例变量使用会导致每次创建类的实例都会导致ThreadLocal被创建。

# 实现原理

每个线程实例内部会维护一个类似HashMap的对象，称之为ThreadLocalMap。每个ThreadLocalMap内部会包含若干Entry(key-value形式)。Entry的key是一个ThreadLocal实例，value使一个线程特有对象。因此Entry的作用就相当于为其所属线程建立一个ThreadLocal与一个线程特有对象之间的对应关系。

![image-20200308121423797](https://xbxblog2.bj.bcebos.com/java%E5%A4%9A%E7%BA%BF%E7%A8%8B-threadLocal%2Fimage-20200308121423797.png)

# 典型应用场景

+ 需要使用非线程安全对象，但又不希望因此引入锁
+ 使用线程安全对象，但希望避免其使用锁的开销和相关问题
+ 隐式参数传递
+ 特定于线程的单例模式

# ThreadLocal可能导致的问题

+ **退化与数据错乱**

  由于线程和任务之间是一对多的关系，一个线程可以先后执行多个任务，因此线程持有对象就相当于一个线程所执行的多个任务之间的共享对象。如果线程持有对象随任务的执行而改变，那么这个线程可能就会看到前一个任务执行所留下的“痕迹”。这种情况可能会导致后续任务执行发生混乱，使线程持有对象退化为任务特有对象。

  解决这个问题可以在使用该线程持有对象的任务执行前将线程持有对象进行重置或清理，使之恢复到初始状态。

+ **内存泄漏问题**

  由ThreadLocalMap中的Entry不能被及时清理导致。

