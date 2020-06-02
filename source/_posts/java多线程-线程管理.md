---
title: java多线程-线程管理
author: XIA
categories:
  - null
tags:
  - null
date: 2020-03-09 18:36:52
---

# 线程组

线程组可以用来表示一组相似的线程。一个线程组可以包含多个线程以及其他线程组。一个线程组包含其他线程组的时候，我们称这个线程组为其他线程组父线程组。

当创建一个线程时没有指定线程组，那么默认属于父线程所属线程组。在jvm创建main线程时会为其指定一个线程组，因此java中任何一个线程都有与之关联的线程组。

由于线程组当初是处于安全的考虑，设计用来隔离不同Applet的，但是并没有达到目的。现在线程组的使用很少了，基本上可以忽略它的存在。

# 线程的未捕获异常与监控

如果线程在运行期间抛出了未捕获异常（Uncaught Exception），那么随着run方法的退出，相应的线程也会终止。但是多数情况下，我们希望可以捕获到该异常，在适当情况下做一些补救，例如将异常信息记录到日志中，或启动一条替代线程继续执行。

JDK1.5为了解决这个问题引入了UncaughtExceptionHandler接口，该接口在Thread类内部，它只定义了一个方法：`void uncaughtException(Thread t,Throwable e)`。该方法的两个参数为：异常终止的线程本身和导致线程提前终止的异常。我们在调用start方法启动线程前，设eh是一个UncaughtExceptionHandler实例，调用thread.setUncaughtExceptionHandler(eh)来为thread关联一个UncaughtExceptionHandler实例，用来处理线程抛出未被捕获异常。

线程组本身也实现了UncaughtExceptionHandler接口。如果一个线程没有关联的UncaughtException实例，那么该线程异常终止前其所属线程组的uncaughtException方法会被调用。线程组的uncaughtException方法会调用父线程组的uncaughtException方法，如果一个线程组没有父线程组时，则会调用`Thread.getDefaultUncaughtExceptionHandler();`来获取默认UncaughtExceptionHandler来处理，该默认处理器可以使用setter方法设置。

**ThreadGroup#uncaughtException**

```java
public void uncaughtException(Thread t, Throwable e) {
    if (parent != null) {
        parent.uncaughtException(t, e);
    } else {
        Thread.UncaughtExceptionHandler ueh =
            Thread.getDefaultUncaughtExceptionHandler();
        if (ueh != null) {
            ueh.uncaughtException(t, e);
        } else if (!(e instanceof ThreadDeath)) {
            System.err.print("Exception in thread \""
                             + t.getName() + "\" ");
            e.printStackTrace(System.err);
        }
    }
}
```

**UncaughtExceptionHandler选择优先级示例图：**

![image-20200309222031958](https://xbxblog2.bj.bcebos.com/java%E5%A4%9A%E7%BA%BF%E7%A8%8B-%E7%BA%BF%E7%A8%8B%E7%AE%A1%E7%90%86%2Fimage-20200309222031958.png)

# 线程工厂

在JDK1.5后，引入了创建线程实例的工厂类接口ThreadFactory：

```java
public interface ThreadFactory {

    Thread newThread(Runnable r);
}
```

newThread方法可以用来创建线程，该方法的r代表所创建的线程需要执行的任务。

在实际使用中我们可以通过实现该接口，在newThread方法中封装线程创建的逻辑，并进行一些公共的配置。

# 线程池

线程池内部可以预先创建一定数量的工作者线程，客户端代码并不需要向线程池借用线程而是将其需要执行的任务作为一个对象交给线程池，线程持可能将这些任务缓存在队列中，而线程池内部的各个工作线程则不断地从队列中取出任务并执行。因此线程池可以看作基于生产者-消费者的一种服务，该线程池内部维护的工作者线程相当于消费者，线程池的客户端相当于生产者，线程池内部用于缓存任务的队列相当于传输通道。

![image-20200309223515685](https://xbxblog2.bj.bcebos.com/java%E5%A4%9A%E7%BA%BF%E7%A8%8B-%E7%BA%BF%E7%A8%8B%E7%AE%A1%E7%90%86%2Fimage-20200309223515685.png)

`java.util.concurrent.ThreadPoolExecutor`类就是一个线程池，客户端代码可以调用submit(Runnable task)方法向其提交任务。

线程池内部维护的工作者线程数量被称为线程池大小，其有三种形态：

+ 当前线程池大小：表示线程池中实际工作者线程的数量
+ 最大线程池大小：表示线程池中允许的工作者线程数量上限
+ 核心线程池大小：表示一个不大于最大线程池大小的工作者线程数量上限

当前线程池大小是线程池中的工作者线程进行计数的结果，最大线程池大小和核心线程池大小都可以进行配置。

当前线程池大小是随着线程池接收到的任务数量而逐渐向核心线程池大小靠拢的，即核心线程是逐渐被创建与启动的。ThreadPoolExecutor.prestartAllCoreThreads可以在线程池未接收到任何任务的情况下预先建立并启动所有核心线程。

------

**一个ThreadPoolExecutor的构造方法：**

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) 
```

+ corePoolSize:线程池核心大小
+ maximumPoolSize：最大线程池大小
+ keepAliveTime，unit：在当前线程池大小超过核心线程池大小的时候，超过核心线程池大小的部分工作者线程空闲时间超过keepAliveTime所指定的事件后会被清理掉
+ workQueue：保存工作任务的阻塞队列，称为工作队列
+ threadFactory：用于创建工作者线程的工厂
+ handler：当工作队列满并且当前线程池大小达到最大线程池大小的情况下，客户端试图提交的任务会被拒绝。为了提高线程池的可靠性，引入RejectedExecutionHandler接口，来封装被拒绝的任务的处理策略。

-----

关闭线程池的两个方法：ThreadPoolExecutor.shutdown()和ThreadPoolExecutor.shutdownNow()

+ ThreadPoolExecutor.shutdown():使用该方法关闭线程池时，已提交的任务会被继续执行，而新提交的任务则会被拒绝
+ ThreadPoolExecutor.shutdownNow()：使用该方法停止线程池会将正在执行的任务停止，已经提交的任务不会被执行。该方法的返回值是已提交而为被执行的任务列表。停止正在执行的线程的方式是向其发送中断通知。

----------

**任务处理结果、异常与任务的取消**

如果我们需要获取任务执行后的返回结果，可以使用另一个提交任务的submit方法：

`public <T> Future<T> submit(Callable<T> task) `

Callable代表的任务可以有返回值，也可以抛出异常。在任务执行结束后调用Future.get()可以获取任务执行后的结果以及执行过程中发生的未捕获异常进行捕获。

Future.get()是阻塞方法，当调用该方法时，相关线程还没有执行结束，那么执行该方法的线程将被暂停。









