---
layout: post
title: tail queue串串烧
comments: true
archive: true
tag: [queue, tail queue]
---
## 邂逅tail queue
最近阅读GO内存分配相关源代码，看到下面一段代码。

~~~~~go
// Linked list structure is based on BSD's "tail queue" data structure.
type mSpanList struct {
	first *mspan  // first span in list, or nil if none
	last  **mspan // last span's next field, or first if none
}

type mspan struct {
	next *mspan     // next span in list, or nil if none
	prev **mspan    // previous span's next field, or list head's first field if none

	// 其他字段信息
}
~~~~~

`prev **mspan`、`last  **mspan`这两个字段定义为二级指针，一开始看到这样的定义觉得很怪异，跟正常的双向链表的定义不一样。从代码注释中可以看到，这样的链表是基于一种被称为尾队列（tail queue）的数据结构来构建的。看到这样的一种链表的定义，很容易就会想到为什么要这样设计，这样设计有什么好处？


本人以前一直使用Java，这也是首次见到这样的双向链表的设计。从网上搜索到的相关介绍比较少（感觉是自己搜索技能不够），大部分的说法是这种设计对链表的各种操作更方便快捷。是否方便快捷暂且不说，其实这里比较令人奇怪的是`prev`定义为指向上一个节点的`next`字段，那反向遍历的时候，拿到`prev`字段进行解指针操作，最终得到的不还是当前的节点吗，那究竟是如何进行反向遍历的呢？

## tail queue的常见操作
一般双向链表都可以在两端进行插入删除，进行正向遍历以及反向遍历，这里面其他操作都好说，但是删除最后一个元素以及反向遍历，tail queue在操作的时候却需要用到一些指针操作技巧。下面首先看一下tail queue的初始化以及插入元素之后的状态。

![初始化](/images/watoud/tailqueue/init.png)
![初始化](/images/watoud/tailqueue/common.png)

下面是自己尝试实现的tail queue插入、删除以及遍历操作。

~~~~~go
type tqlistEntry struct {
	next *tqlistEntry  // next entry in list, or nil if none
	prev **tqlistEntry // previous entry's next field, or list head's first field if none

	val interface{}
}

type tqlist struct {
	first *tqlistEntry  // first entry in list, or nil if none
	last  **tqlistEntry // last entry's next field, or first if none
}

// 初始化
func (list *tqlist) listInit() {
	list.first = nil
	list.last = &list.first
}

// 是否为空
func (list *tqlist) isEmpty() bool {
	return list.first == nil
}

func (list *tqlist) insertFront(entry *tqlistEntry) {
	entry.next = list.first
	if list.first != nil {
		list.first.prev = &entry.next
	} else {
		list.last = &entry.next
	}
	list.first = entry
	entry.prev = &list.first
}

// 在后面插入元素相当简洁
func (list *tqlist) insertBack(entry *tqlistEntry) {
	entry.next = nil
	entry.prev = list.last
	*list.last = entry
	list.last = &entry.next
}

// 调用时需要确保要删除的元素在当前的链表中
func (list *tqlist) remove(entry *tqlistEntry) {
	if entry.next != nil {
		entry.next.prev = entry.prev
	} else {
		list.last = entry.prev
	}
	*entry.prev = entry.next
	entry.next = nil
	entry.prev = nil
}

func (list *tqlist) removeFirst() {
	if list.isEmpty() || list.first.next == *list.last {
		// 空或者只有一个元素的时候直接初始化
		list.listInit()
		return
	}
	newFirst := list.first.next
	list.first.prev = nil
	list.first.next = nil
	list.first = newFirst
	newFirst.prev = &list.first
}

func (list *tqlist) removeLast() {
	if list.isEmpty() || list.first.next == *list.last {
		// 空或者只有一个元素的时候直接初始化
		list.listInit()
		return
	}
	// 指针的应用 如果仅仅解指针的话是达不到目的的
	listEntry := (*tqlistEntry)(unsafe.Pointer(&(*list.last)))
	list.last = listEntry.prev
	*listEntry.prev = listEntry.next
	listEntry.prev = nil
	listEntry.next = nil
}

type accessFunc func(*tqlistEntry)

func (list *tqlist) forerwardTraversal(accessor accessFunc) {
	cur := list.first
	for cur != nil {
		accessor(cur)
		cur = cur.next
	}
}

func (list *tqlist) backrwardTraversal(accessor accessFunc) {
	cur := (*tqlistEntry)(unsafe.Pointer(&(*list.last)))
	for cur != nil && cur.next != list.first {
		accessor(cur)
		cur = (*tqlistEntry)(unsafe.Pointer(&(*cur.prev)))
	}
}
~~~~~

tail queue在反向遍历的时候需要对指针进行类型转换以获取所需要的对象，跟一般的双向链表相比，它在插入以及删除操作上确实相对更加简洁，但是却并不那么直观了。







