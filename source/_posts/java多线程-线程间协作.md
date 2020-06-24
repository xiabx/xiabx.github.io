---
title: java多线程-线程间协作
author: XIA
categories:
  - 多线程
tags:
  - null
date: 2019-03-03 20:26:06
---

# 等待和通知

在单线程程序中，当要执行的操作需要满足一定条件才可以执行时，我们可以将该操作放在一个if语句中。而在多线程程序中我们可以将操作方法体放在while循环体中，然后将该线程暂停，当其他线程所作操作满足该线程执行条件后，再使用其他线程唤醒该线程，然后检测执行条件后继续执行。

这涉及两个操作：等待和通知。他们使用Object.wait()和Object.notify()方法实现。由于任何类都继承自Object类，所以所有对象都会包含这两个方法。

调用wait和notify方法的对象需要是对这两个方法进行同步的锁对象。调用wait方法的模板代码：

![image-20200303205951119](https://xbxblog2.bj.bcebos.com/java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%BA%BF%E7%A8%8B%E9%97%B4%E5%8D%8F%E4%BD%9C%2Fimage-20200303205951119.png)

这里当wait方法被调用后不会立即被返回，而是在wait方法内将锁释放，然后再将该线程进行暂停。等该线程收到其他线程通知后，再继续被暂停的wait方法，尝试获取锁，当获取锁后继续执行该线程。

在其他线程调用通知方法notify后也不会立即释放锁，而是在包含调用notify方法的临界区执行结束后才会释放锁。所以 notify的调用应尽量放在临界区代码的结束处。notify方法的模板代码：

![image-20200303210530345](https://xbxblog2.bj.bcebos.com/java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%BA%BF%E7%A8%8B%E9%97%B4%E5%8D%8F%E4%BD%9C%2Fimage-20200303210530345.png)

等待线程：执行wait方法的线程。通知线程：执行notify方法的线程。

notify方法会唤醒相应锁对象上的任意一个线程，notifyAll方法则会唤醒相应锁对象上的全部线程。

**wait与notify实现原理**

jvm会为每个对象维护一个入口集用来存储申请该对象内部锁的线程，此外jvm还会维护一个等待集用来存储该对象上的等待线程。当Object.wait方法被调用后，相应线程会被暂停并释放锁，同时将该线程引用存储在等待集中。当Object.notify被调用后，等待集中任意一条线程被唤醒，被唤醒的线程会继续待在等待集中，直到申请到锁以后才会从等待集中移除，然后继续执行。其中wait方法的伪代码类似：

![image-20200303211648421](https://xbxblog2.bj.bcebos.com/java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%BA%BF%E7%A8%8B%E9%97%B4%E5%8D%8F%E4%BD%9C%2Fimage-20200303211648421.png)

# wait/notify的问题

+ 过早唤醒：假设有两个等待线程T1、T2同步在一个锁对象someObject上，当一个通知线程更新了共享变量s后并调用someObject调用notifyAll方法。此时T1、T2两个线程都被唤醒，但是更新的共享变量s只满足了T1唤醒后继续执行的条件，所以这时T2被唤醒就属于过早唤醒。解决这个问题可以使用Condition接口解决。
+ 信号丢失：如果等待线程执行Object.wait前没有判断保护条件是否已经成立，那么就可能会出现在Object.wait方法调用前notify方法已经被执行，这样执行被暂停的线程就会永远得不到通知。解决这个问题的方法就是在执行wait方法前先对共享变量状态进行判断。
+ 上下文切换问题：因为等待和通知的过程需要涉及到多次锁的释放、获取、竞争。所以上下文会存在多次上下文切换问题。减少上下文切换问题可以在正确的前提下使用notify代替notifyAll。

# Condition

JDK1.5以后出现了`java.util.concurent.locks.Condition`接口，该接口在显式锁中使用，用来代替内部锁中的wait/notify方法。此接口解决了wait/notify方法导致的过早唤醒问题，还解决Object.wait(long)方法不能区分是由等待超时而导致的问题。

该接口与显示锁配合使用，使用一个显式锁对象调用newCondition方法可以创建一个相应的Condition接口实例。然后调用该Condition实例的await/signal/signalAll这三个方法的作用同Object.wait/notify/notifyAll。同样调用Condition实例的相应方法时可需要调用线程持有相应显式锁。

当一个显式锁重复调用newCondition方法时可以创建多个Condition实例，可以使用这些Condition实例实现来控制唤醒粒度，解决过早唤醒问题。

Condition.awaitUntil(Date deadline)可以实现带超时时间限制的等待，如果该方法返回值为true则表示进行的等待未达到deadline。

使用示例：

![image-20200303225659850](https://xbxblog2.bj.bcebos.com/java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%BA%BF%E7%A8%8B%E9%97%B4%E5%8D%8F%E4%BD%9C%2Fimage-20200303225659850.png)

# CountDownLatch

CountDownLatch字面意思为倒计时协调器。假设有这么个场景，有一个线程可能需要等待其他线程执行完特定操作后才可以继续运行。那么对于这种情景可以使用wait/notify或者Confition来实现。不过可以使用另一个工具类CountDownLatch实现，他的原理就是构造时指定一个数值作为需要执行的目标步骤数目，调用该对象的await方法所在的线程会被暂停，当每执行一个步骤结束后调用countDown方式使数值减一，当减到零时就会通知暂停线程继续运行。

# CyclicBarrier

CyclicBarrier的字面意思是：可循环（Cyclic）使用的屏障（Barrier）。它要做的事情是，让一组线程到达一个屏障（也可以叫屏障点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被拦屏障拦截的线程才会继续干活，线程进入屏障通过CylicBarrier的await()方法。 

# 生产者-消费者模式

生产者负责创建数据或任务，消费者负责处理数据或任务。生产者和消费者分别可以是一个或多个线程。生产者与消费者之间的数据或任务传输途径是通道。也可以从通道的角度理解生产者消费者，往通道里存数据的线程为生产者，从通道里取数据的线程为消费者。

![image-20200307105947811](https://xbxblog2.bj.bcebos.com/java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%BA%BF%E7%A8%8B%E9%97%B4%E5%8D%8F%E4%BD%9C%2Fimage-20200307105947811.png)

**阻塞队列**

当通道为空时候，消费者无法从通道中取出产品，此时消费者线程可以处于等待状态；当通道存储空间满的时候，生产者无法往通道中存储产品，此时生产者线程可以处于等待状态。拥有这样特性的通道被称为阻塞队列。

阻塞队列可以按照容量是否有界来划分，分为有界队列和无界队列，有界队列的容量限制由程序指定，无界队列的容量限制最大为2的31次方减一。JDK1.5引入的接口`java.util.concurrent.BlockingQueue`定义了一种线程安全的队列阻塞队列。定义put相当于生产者将产品安全发布到消费者线程，take相当于从队列中取出产品。该接口的主要实现类包括：ArrayBlockingQueue、LinkedBlockingQueue、SynchronousQueue。

+ ArrayBlockingQueue：内部使用数组作为其存储空间，数组的存储空间是预先分配的，所以这个队列是有界队列。由于该队列中put和take操作使用的是同一个锁，所以会导致锁的高争用，进而导致较多的上下文切换。适用于并发程度较低情况下。
+ LinkedBlockingQueue：既可以实现无界队列也可以实现有界队列。该队列内部维护着一个链表，所以每次put与take都会导致节点的创建和删除，会给垃圾回收带来负担。其优点是内部使用两个显式锁分别用于take和put操作，降低了锁的争用。适用于并发程度较高情况下。
+ SynchronousQueue：该队列是一个特殊的有界队列，内部并没有维护存储元素的空间。当生产者线程执行了put操作时如果没有消费者执行take，那么生产者将被暂停直到消费者执行take。类似的如果消费者执行take此时没有生产者执行put，那么此时消费者也会被暂停，直到生产者执行put。适用于消费者处理能力与生产者处理能力相差不大情况下。

**流量控制与信号量：semaphore**

无界队列的一个好处就是put操作并不会导致生产者线程被阻塞，因此无界队列不会影响生产者的步伐。但是在队列数据积压的情况下会对内存资源占用过多。因此我们需要在使用无界队列时进行流量控制来限制生产者的生产速率，来达到无界队列中不会积压过多产品。

JDK1.5引入的标准库`java.util.concurrent.Semaphore`可以来实现流量控制。我们将代码访问特定资源或执行特定操作看作是一种虚拟资源，Semaphore相当于虚拟资源的配额管理器，它控制同一时间内对虚拟资源的访问次数。他的原理相当于一个“锁”，当执行Semaphore.acquire()方法用于申请一个配额，如果可以成功获取配额会立即返回，当前配额会减少1。否则执行线程会被暂停，被保存在Semaphore内部维护的一个等待队列中。Semaphore.release()会使当前配额增加1，并唤醒相应的Semaphore中的任意一个等待线程。

```java
  public void put(P product) throws InterruptedException {
    semaphore.acquire();// 申请一个配额
    try {
      queue.put(product);// 访问虚拟资源
    } finally {
      semaphore.release();// 返还一个配额
    }
  }
```

注意：acquire()和release()总是配对使用的，release()应总是放在finally块中。

**双缓冲与Exchanger**

双缓冲是指使用两个缓冲区，生产者和消费者分别使用一个，生产者往其中一个缓冲区中存储数据，消费者往另一个缓冲区中取出并处理数据。当生产者的缓冲区存满，消费者的缓冲区处理完后，此时这两个缓冲区进行交换，生产者就可以往刚刚被消费者处理完的空的缓冲区存储数据，消费者就可以在刚才生产者存满的缓冲区中取出数据。

JDK1.5引入的`java.util.concurrent.Exchanger`就可以实现双缓冲。其原理类似于CyclicBarrier，当Exchanger.exchange(V)执行时，相当于CyclicBarrier.await()执行。只有两个缓冲区的线程都执行了Exchanger.exchange(V)才会将缓冲区交换。

# 线程中断机制

当一个线程请求另一个线程停止其正在执行的操作，比如一个线程在执行一个大文件的下载操作，如果另一个线程要中途取消这个下载操作，这时就需要使用线程中断机制。

线程中断机制也像是一种协议，java平台为每个线程都维护了一个被称为中断标记的布尔型变量，用来表示线程是否收到了中断，中断标记为true表示线程收到了中断，目标线程可以通过Thread.currentThread().isInterrupted()来获取中断标记，也可以通过Thread.interrupted()来获取并重置中断标记。当一个线程的interrupted()方法被调用，就相当于该线程的中断标记设置为true。

目标线程在执行一些操作前都会对中断标记进行检测，当检测到中断标记为true时，则会抛出InterruptedException等异常。然后我们捕获该类型异常进行处理，比如将线程中断，向上抛出，忽视等等。

能够对中断做出相应的一些方法：

![image-20200307132039100](https://xbxblog2.bj.bcebos.com/java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%BA%BF%E7%A8%8B%E9%97%B4%E5%8D%8F%E4%BD%9C%2Fimage-20200307132039100.png)


