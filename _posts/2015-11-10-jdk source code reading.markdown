---
layout: post
title: JDK源码阅读之List
date: 2015-11-10
comments: true
archive: true
tag: list
---

### 实现方式
List有两种典型的实现方式，一种采用**数组(ArrayList)**，一种采用**双向链表(LinkedList)**，在使用时性能有所不同。

| cost        	| ArrayList     |   LinkedList  |
| :-----------  | :----------:  | :-----------: |
| size	        | o(1)			| o(1)   	    |
| isEmpty       | o(1)          | o(1) 	    	|
| get			| o(1)          | o(n)       	|
| set	        | o(1)			| o(n)   	    |
| add           | o(1)          | o(1) 	    	|
| contains      | o(n)          | o(n) 	    	|

### 迭代器的使用    
创建了迭代器之后, 如果list不是在当前迭代器的操作下而发生了结构性变化, 那么该迭代器的下一次操作将立马抛出`ConcurrentModificationException`异常. 
典型的结构性变化有添加元素、删除元素以及**扩容**等.

### ArrayList的扩容    
ArrayList默认大小为10(`DEFAULT_CAPACITY = 10`), 使用默认构造函数创建一个ArrayList对象时，容量大小为0. 

```
public ArrayList()
{
	// 这是一个长度为0的数组
	this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

当往ArrayList中添加元素时, 先判断当前的容量是否足够, 如果不够则先进行扩容.新的容量的大小为原来大小的1.5倍, ```newCapacity = oldCapacity + (oldCapacity >> 1)```.
    
有一点需要注意, 当初始容量大小为0时, 使用默认构造函数创建的ArrayList对象在扩容时，容量大小最少为10.

```
private void ensureCapacityInternal(int minCapacity)
{
	if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
 	{
		minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
	}

	ensureExplicitCapacity(minCapacity);
}
```

而使用带容量大小的参数的构造函数创建的容量大小为0的对象，在扩容时则没有这种特殊性。
    
ArrayList在扩容时, 是对ArrayList当前的数组内容进行复制, `elementData = Arrays.copyOf(elementData, newCapacity)`.
 