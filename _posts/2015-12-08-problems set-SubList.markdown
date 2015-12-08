---
layout: post
title: 小心使用ArrayList.subList方法
date: 2015-12-08
comments: true
archive: false
label: problem
---

### 先看一段代码

```
ArrayList<String> original = new ArrayList<String>(
		        Arrays.asList("first", "second", "third"));
List<String> subList = original.subList(0, original.size());

original.remove(0);

System.out.println("subList:" + subList);
```

执行这段代码会抛出如下异常<br/>

```
Exception in thread "main" java.util.ConcurrentModificationException
	at java.util.ArrayList$SubList.checkForComodification(ArrayList.java:1231)
	at java.util.ArrayList$SubList.listIterator(ArrayList.java:1091)
	at java.util.AbstractList.listIterator(AbstractList.java:299)
	at java.util.ArrayList$SubList.iterator(ArrayList.java:1087)
	at java.util.AbstractCollection.toString(AbstractCollection.java:454)
	at java.lang.String.valueOf(String.java:2981)
	at java.lang.StringBuilder.append(StringBuilder.java:131)
	........
```

### 问题来源
这个异常是在list的迭代器上迭代的时候抛出来的。调用```System.out.println```的时候，首先会计算参数，也就是```"subList:" + subList```，这个时候会调用list方法的```toString```方法，而这个方法正好就使用了迭代器。

```
public String toString()
{
    Iterator<E> it = iterator();
    if (! it.hasNext())
        return "[]";

    StringBuilder sb = new StringBuilder();
    sb.append('[');
    for (;;)
     {
        E e = it.next();
        sb.append(e == this ? "(this Collection)" : e);
        if (! it.hasNext())
            return sb.append(']').toString();
        sb.append(',').append(' ');
    }
}
```

<br/>
就算使用了迭代器，那么又为什么会抛出```ConcurrentModificationException```异常呢？一般而言，这个异常的抛出是在操作迭代器的时候，这个迭代器所属的List发生了结构性变化。结构性变化包括对这个List增加元素或者减少元素以及扩容等操作。但是从上面的程序看，在打印subList的时候，并没有对这个List做结构性的变化操作。那又是为什么呢？
<br/>

从错误栈中可以看到，在调用```listIterator```的时候，紧接着调用了```checkForComodification```方法，在这个方法里面报错了。

```
private void checkForComodification()
{
    if (ArrayList.this.modCount != this.modCount)
        throw new ConcurrentModificationException();
}
```

```

SubList(AbstractList<E> parent,int offset, int fromIndex, int toIndex)
{
    this.parent = parent;
    this.parentOffset = fromIndex;
    this.offset = offset + fromIndex;
    this.size = toIndex - fromIndex;
    this.modCount = ArrayList.this.modCount;
}
```

```this.modCount```是在构造方法里面设置的，也就是在调用```subList```方法的时候获取的，而到了打印的时候，由于前面执行了语句```original.remove(0)```，从而original中的modCount发生了变化，所以这个时候便会有```ArrayList.this.modCount != this.modCount```，于是便抛出异常了。

<br/>

<div align="left">
<i>
<font color="purple">
所以小心使用list.subList方法，尤其是把方法的返回值作为参数传给其他众多方法的时候，很有可能会出现这个错误。
</font>
</i>
</div>











