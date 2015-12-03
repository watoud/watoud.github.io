---
layout: post
title: JavaSE官方文档阅读纪要之Concurrency
date: 2015-12-01
comments: true
archive: true
tag: [concurrency]
---
### 线程中断
有两种情况。一种是线程频繁调用会抛出InterruptedException异常的方法，这个时候若需要响应中断，只需要在catch到InterruptedException异常的时候直接退出就行了。

````
for (int i = 0; i < importantInfo.length; i++)
{
    // Pause for 4 seconds
    try
    {
        Thread.sleep(4000);
    }
    catch (InterruptedException e)
    {
        // We've been interrupted: no more messages.
        return;
    }
    // Print a message
    System.out.println(importantInfo[i]);
}
````

有不少方法会抛出InterruptedException异常，比如```sleep```方法。
<br/><br/>

另一种情况下，一个线程可能执行很长一段时间，并且期间并不会出现调用可能抛出InterruptedException异常的方法。这个时候就需要我们周期性的调用方法```Thread.interrupted```。

````
for (int i = 0; i < inputs.length; i++)
{
    heavyCrunch(inputs[i]);
    if (Thread.interrupted())
    {
        // We've been interrupted: no more crunching.
        return;
    }
}
````
上述的简单例子中，接收到中断请求后就直接退出了，但是在比较复杂的场景下，一般会抛出InterruptedException异常。

````
if (Thread.interrupted())
{
    throw new InterruptedException();
}
````
### 线程同步
线程启动和终止的happens-before关系。当一条语句执行```Thread.start```方法，任何与```Thread.start```有happens-before关系的语句，都与所有新线程执行的语句具有hanppens-before关系。此外，当一个线程终止并导致另一个调用```Thread.join```方法而阻塞的线程重新执行时，已经结束的那个线程所执行的所有语句，与成功执行join方法后所执行的所有语句具有happens-before关系。

<br/>
同步方法(synchronized)的影响。1.不可能出现同一个对象的两个synchronized方法被交叉调用执行的情况。当一个线程执行某个对象的synchronized方法时，其它所有调用那个对象的synchronized方法的线程都将被挂起，直到第一个调用synchronized方法的线程执行完成。2.当synchronized执行结束的时候，它将与稍后调用同一个对象的synchronized方法的操作自动建立起happens-before关系。此外，对构造方法加synchronized关键字是没有意义的，因为在一个对象处于创建阶段的时候，只有创建这个对象的线程可以访问这个对象。

<br/>
监视器锁与同步方法。每一个对象都关联着一个内部锁，同步方法的实现是建立在内部锁或者说监视器锁的基础上的，内部锁提供了同步操作所需要的两种特性：对对象状态的互斥访问以及happens-before关系的建立。需要注意的是，静态的同步方法与非静态的同步方法，它们所获取的内部锁是不一样的，一个类的非静态的同步方法所获取的内部锁是该类的实例所关联的内部类，而一个类的静态同步方法所关联的内部类是这个类所对应的class实例。

<br/>
原子操作。如下两类操作可以看成是原子操作：1.对引用变量以及原始变量(long以及double除外)的读写操作是原子的，2.对所有声明为volatile的变量(包括long以及double)的读写操作是原子的。

### 设计不可变对象
1. 不提供setter方法
2. 所有字段加上final以及private修饰符
3. 避免子类对方法进行重载。可以对类加上final标记，或者把构造方法设置成私有的而提供工厂方法
4. 如果字段的值是对可变对象的引用，那么需要避免对该可变对象进行修改。一方面不提供修改可变对象的方法，另一方面避免把可变对象的引用逸出。

### 高级并发对象
1.Lock对象比隐式锁(内部锁或者监视器锁)优越的地方：可以在获取不到锁的时候放弃。Lock对象有tryLock方法，可以在获取不到锁的时候立即返回，或者超时后返回；另外一方面，lockInterruptibly也允许在锁尚未获取的时候被其它线程中断。

### 参考资料
http://docs.oracle.com/javase/tutorial/essential/concurrency/index.html
