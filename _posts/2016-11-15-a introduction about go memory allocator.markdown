---
layout: post
title: GO内存分配浅析
comments: true
archive: true
tag: [queue, memory]
---
## tcmalloc简介
在GO内存管理实现的源代码中，有注释表示GO内存分配机制是基于tcmalloc实现的，首先不妨看一下tcmalloc究竟如何处理的，有什么优势？具体内容见http://goog-perftools.sourceforge.net/doc/tcmalloc.html

**tcmalloc具有以下三大优势：**

- 速度更快，有测试表明，tcmalloc比glibc 2.3 malloc以及其他内存方式要快。
- 降低多线程环境下的锁竞争。
- 分配小对象时空间使用率更高。

**tcmalloc概览**tcmalloc给每个线程都分配一个线程本地缓存，小对象从线程本地缓存中分配，当线程本地缓存不够时，则从中央堆内存中获取一部分到线程本地缓存，而周期性的垃圾回收则会把内存从线程本地缓存移动到中央堆内存中。

![](/images/watoud/go-memory/tcmalloc-overview.gif)

对于小于32k的小对象与与超过32k的大对象，tcmalloc有不同的处理方式。大对象直接从中央堆内存中按照页级粒度进行分配，每一页是一个4k对齐的内存区域。所以大的对象总是按照页对齐的，并总是占用整数个页。

一连串的页可以分割成一系列小对象，每一个都具有相同的大小，比如一页可以分割成32个大小为128字节的对象。

**小对象的分配**每一个小对象的大小都可以映射到大约170个大小级别中的一个。比如对961到1024字节大小内存的分配会向上舍入到1024比特。大小级别是按照一定的间隔排列的，他们相隔8字节、16字节、32字节，诸如此类，最大相隔256字节，这个时候大小级别中的大小至少大于等于2k。

每一个线程各自都拥有一个独立的自由对象链表，每个大小等级的对象都有，如下如所示。

![](/images/watoud/go-memory/threadCache.gif)


