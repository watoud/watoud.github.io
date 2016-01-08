---
layout: post
title: graphviz使用简介
date: 2016-01-07
comments: true
archive: true
tag: [graphviz]
---

### 有向图及其属性设置
```
digraph simple
{
	a -> b -> c;

	b -> d;
}
```

其中```digraph```指明图的类型为有向图，```simple```为这个有向图的名称， ```a```、```b```等代表节点，而```->```则代表节点之间的有向边，上述语句编译后生成的图片如下图所示。
<center>
![生成的图片](/assets/images/graphviz/simple-none.png)
</center>
<br/>

如果需要对节点```a```的形状和颜色进行控制，比方说形状改为方形，颜色改为红色。

```
digraph simple
{
	a [shape = "box", style = "filled", fillcolor= "red"];

	a -> b -> c;

	b -> d;
}
```

在上述语句中，```[]```是对节点或者边的属性进行设置，```shape```设置节点的形状，```style```设置节点的风格，```fillcolor```设置节点的填充颜色，上述语句编译后生成的图片如下图所示。
<center>
![生成的图片](/assets/images/graphviz/simple-2.png) <br/>
</center>
<br/>

给节点或者边添加文本，比如给节点```a```添加文本*start*，```b```添加文件*this is a test*，```a```到```b```的边添加文本*ready*，同时节点```b```希望以纯文本的形式展现等等。

```
digraph simple
{
	a [shape = "box", style = "filled", fillcolor= "red", label = "start"];

	b [shape = plaintext, fontcolor = "#40e0d0", label = "this is a test"];

	c [shape = point, label = "over", width = 0.5, margin = 0, color = "#a0522d"];

	d [shape = "diamond", style = "rounded", width = 0.3, label = "^_^"];

	a -> b [label = "ready", fontcolor = "blue"];

	b -> c;

	b -> d;
}
```

在上述语句中，```fontcolor```设置字体的颜色，```width```设置节点的大小，```margin```设置节点与边的距离，上述语句执行后生成如下图片。
<center>
![生成的图片](/assets/images/graphviz/simple-3.png)
</center>
<br/>

### 组合图
下面展示如果绘制一颗二叉树

```
digraph binaryTree
{
	bgcolor = "antiquewhite";

	node [ shape = record, height=.1 style = "filled" fillcolor = "pink3"];

	node0 [label = "<f0> |<f1> G|<f2> "];
	node1 [label = "<f0> |<f1> E|<f2> "];
	node2 [label = "<f0> |<f1> B|<f2> "];
	node3 [label = "<f0> |<f1> F|<f2> "];
	node4 [label = "<f0> |<f1> R|<f2> "];
	node5 [label = "<f0> |<f1> H|<f2> "];
	node6 [label = "<f0> |<f1> Y|<f2> "];
	node7 [label = "<f0> |<f1> A|<f2> "];
	node8 [label = "<f0> |<f1> C|<f2> "];

	"node0":f2 -> "node4":f1;
	"node0":f0 -> "node1":f1;
	"node1":f0 -> "node2":f1;
	"node1":f2 -> "node3":f1;
	"node2":f2 -> "node8":f1;
	"node2":f0 -> "node7":f1;
	"node4":f2 -> "node6":f1;
	"node4":f0 -> "node5":f1;
}
```

上述语句执行后生成如下图所示，语句中的颜色名称可以从链接 (http://www.graphviz.org/content/color-names) 中查看。
<center>
![生成的图片](/assets/images/graphviz/simple-4.png)
</center>
<br/>

```
digraph structs
{
	node [shape=record];
	struct1 [shape=record,label="<f0> left|<f1> middle|<f2> right"];
	struct2 [shape=record,label="<f0> one|<f1> two"];
	struct3 [shape=record,label="hello\nworld |{ b |{c|<here> d|e}| f}| g | h"];
	struct1:f1 -> struct2:f0;
	struct1:f2 -> struct3:here;
}
```

上述语句生成的图形如右图所示 ![生成的图片](/assets/images/graphviz/simple-5.png) <br/>

在有向图```structs```中加上语句```rankdir = "LR";```后生成如下所示图片。
<center>
![生成的图片](/assets/images/graphviz/simple-6.png)
</center>
<br/>

有关上述标签语法的使用详情请见官方文档说明 (http://www.graphviz.org/content/node-shapes#record) <br/>

除了上述的方式外，label还可以使用```HTML```标签的方式，这种方式处理更为细致。

```
digraph structs
{
	node [shape=record];

	struct1 [shape=plaintext,label=<
		<TABLE  BORDER="0" CELLBORDER="1" CELLSPACING="0">
			<TR>
				<TD BGCOLOR="red" port="f0">left</TD>
				<TD BGCOLOR="brown2" port="f1">middle</TD>
				<TD BGCOLOR="cyan3"  port="f2">right</TD>
			</TR>
		</TABLE>
	 >];

	struct2 [shape=plaintext,label=<
		<TABLE  BORDER="0" CELLBORDER="1" CELLSPACING="0">
			<TR>
				<TD BGCOLOR="oldlace" port="f0"><i>one</i></TD>
				<TD BGCOLOR="peachpuff1" port="f1"><u>two</u></TD>
			</TR>
		</TABLE>
	 >];

	struct3 [shape=plaintext,label=<
		<TABLE  BORDER="0" CELLBORDER="1" CELLSPACING="0">
			<TR>
				<TD BGCOLOR="oldlace" ROWSPAN="3"><SUB>hello</SUB><SUP>world</SUP></TD>
				<TD BGCOLOR="peachpuff1" COLSPAN="3" ALIGN="LEFT">b</TD>
				<TD BGCOLOR="khaki" ROWSPAN="3">g</TD>
				<TD BGCOLOR="ivory" ROWSPAN="3">h</TD>
			</TR>
			<TR>
				<TD BGCOLOR="lightcyan">c</TD>
				<TD BGCOLOR="tan" PORT="here">d</TD>
				<TD BGCOLOR="orange">e</TD>
			</TR>
			<TR>
				<TD COLSPAN="3" BGCOLOR="palegreen" ALIGN="right">f</TD>
			</TR>
		</TABLE>
	 >];

	struct1:f1 -> struct2:f0;
	struct1:f2 -> struct3:here;
}
```

上述语句生成如右所示图片 ![生成的图片](/assets/images/graphviz/simple-8.png) <br/>


```
digraph G {

	subgraph cluster_0
	{
		style=filled;
		color=moccasin;
		node [style=filled,color=palegreen];
		a0 -> a1 -> a2 -> a3;
		label = "process #1";
	}

	subgraph cluster_1
	{
		node [style=filled, fillcolor = darksalmon];
		b0 -> b1 -> b2 -> b3;
		label = "process #2";
		color=deeppink;
	}
	start -> a0;
	start -> b0;
	a1 -> b3;
	b2 -> a3;
	a3 -> a0;
	a3 -> end;
	b3 -> end;

	start [shape=Mdiamond];
	end [shape=Msquare];
}
```
上述语句生成的图形如右图所示 ![生成的图片](/assets/images/graphviz/simple-7.png) <br/>


### 阅读资料
- http://www.graphviz.org/Documentation.php

