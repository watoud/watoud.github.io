---
layout: post
title: JDK同步工具类
date: 2016-01-12
comments: true
archive: true
tag: [Java, Synchronizer]
---

在Java开发中，多线程间的同步操作常使用```Synchronized```关键字以及JDK提供的几个常用同步工具类:```Lock```、```CountDownLatch```、```CyclicBarrier```以及```Semaphore```。Synchronized的实现是基于JVM提供的监视器锁，而JDK提供的几个同步工具类的实现都是基于```volatile```变量和```CAS```操作。

### Lock与Synchronized的差别
Java中的每个对象实例都拥有一个监视器锁，而Synchronized的实现就是基于对象的监视器锁。既然有了Synchronized提供的同步功能，那么为什么JDK又提供了Lock同步工具类呢？下面来看下Lock与Synchronized的差别。

1. Lock可以让多个线程同时访问某个共享资源（共享锁），而Synchronized不具备这种能力。
2. Lock提供非阻塞的方式获取锁，在获取锁的过程中可中断，提供锁超时，而Synchronized不具备这种功能。
3. Lock可以是非可重入的，而Synchronized是可重入的。
4. Synchronized是非公平的，而Lock可以是公平的也可以是非公平的。
5. 此外，与Lock相关联的Condition也比使用Object的监视器方法wait等更加灵活。

Lock虽然在使用使用上比Synchronized灵活许多，但是Synchronized相对更加安全，
使用Synchronized不必当心监视器锁的释放，而使用Lock则必须由调用者在获取锁后再进行释放，否则会导致后面需要获取锁的操作一直失败或者阻塞。此外，Lock的实现遵循监视器锁的内存同步语义。

### 同步工具类的实现
JDK提供的常用同步工具类在实现上都用到了抽象类```AbstractQueuedSynchronizer```，下面重点分析下该类的实现细节。<br/>

![aa](/images/watoud/AQS/AQS-1.png)      ![bb](/images/watoud/AQS/AQS-2.png)

![cc](/images/watoud/AQS/AQS-3.png)

抽象类```AbstractQueuedSynchronizer```使用```volatile```、```CAS```以及队列提供了一个同步操作框架，JDK中的锁以及其它几个同步工具都是基于```AbstractQueuedSynchronizer```实现的。 <br/>

首先看下```AbstractQueuedSynchronizer```的字段。

```
 	/**
     *	头节点
     */
    private transient volatile Node head;

    /**
	 *	尾节点
     */
    private transient volatile Node tail;

    /**
     * 同步状态值
     */
    private volatile int state;
```

这个三个字段都使用volatile进行了修饰，因为三个字段值会被不同的线程进行修改，使用volatile能保证字段值在不同线程之间都是可见的。其中state使用volatile进行修饰，还用到了volatile的happens-before关系的副作用。在使用同步工具的时候，都会有类似lock以及unlock的成对操作，而lock以及unlock的happens-before关系就是依赖对volatile修饰的state的操作来实现的。lock的时候，



