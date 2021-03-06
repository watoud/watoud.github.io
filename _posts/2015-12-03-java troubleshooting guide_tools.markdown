---
layout: post
title: Java故障排查指南之工具篇
date: 2015-12-03
comments: true
archive: true
tag: [tools]
---
### Java启动参数配置
1. 使用core文件。linux系统下使用```ulimit -c unlimited```
2. 添加-XX:+HeapDumpOnOutOfMemoryError jvm配置项<br/>
```-XX:+HeapDumpOnOutOfMemoryError  -XX:HeapDumpPath=${path\to\file}```
3. 在Java命令行添加参数```-verbose:gc```，日志会打印到控制台上 <br/>
```-Xloggc:${path\to\file}```，日志打印到目标文件

### 诊断工具
1. jps:查看系统java进程信息<br/>
参数设置：<br/>
-m 输出main method的参数<br/>
-v 输出jvm参数<br/>
-l 输出完全的包名，应用主类名，jar的完全路径名 <br/>
  ![jcmd帮助命令](/images/watoud/jdkTools/jps/jps-lvm.png)  

2. jcmd:用于向jvm发送诊断信息<br/>
-l 列出所有java虚拟机信息	<br/>
-help 列出该虚拟机支持的所有命令. eg: <b> jcmd *processId* help </b>  <br/>
	  ![jcmd帮助命令](/images/watoud/jdkTools/jcmd/jcmd-help.png)   <br/>
查看当前jvm支持的命令之后，就可以知道能够使用那些命令了. 比如：<br/>
jcmd *processId* VM.uptime  		查看系统运行时间<br/>
jcmd *processId* Thread.print 		查看线程堆栈信息<br/>
详细使用说明见文档:[http://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr006.html#BABEHABG](http://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr006.html#BABEHABG)

3. jmap：统计内存相关信息<br/>
获取堆转储文件：<b> jmap -dump:format=b,file=es.dump *pid*  </b> <br/>
	  ![获取转储文件](/images/watoud/jdkTools/jmap/jmap-dump.png)   <br/>
查看堆信息：<b> jmap -heap *pid* </b>  <br/>
	 ![获取堆信息](/images/watoud/jdkTools/jmap/jmap-heap.png)   <br/>

 
| 堆配置项     | 说明              |参数设置 |
| :------------- | :---------------- | :---------- |
| MaxHeapSize   | JVM堆的最大大小| -XX:MaxHeapSize= |
| NewSize       | 新生代大小      | -XX:NewSize= |
| MaxNewSize   | 新生代最大大小  | -XX:MaxNewSize= |
| OldSize  | 老生代大小| -XX:OldSize= |
| NewRatio  | 新生代和老生代的大小比率 | -XX:NewRatio= |
| SurvivorRatio | Eden区与Survivor区的比值 | -XX:SurvivorRatio= |
| MetaspaceSize(jdk8) | Metaspace使用的是本地<br/>内存而不是堆内存| -XX:MetaspaceSize= |

除了查看堆栈信息之外，jmap还没用来查看堆中类的统计信息<br/>
查看类信息：<b> jmap -histo *pid* <br/>   </b>
	  ![查看histo信息](/images/watoud/jdkTools/jmap/jmap-histo.png)  <br/>
查看类加载器统计信息: <b> jmap -clstats *pid* </b>
	  ![类加载器信息](/images/watoud/jdkTools/jmap/jmap-clstats.png)   <br/>

4. jinfo: 获取Java进程的配置信息 <br/>
获取Java进程的配置信息： <b> jinfo *pid* </b> <br/>
	  ![Java进程配置信息](/images/watoud/jdkTools/jinfo/jinfo.png)   <br/>

5. jhat：分析Java堆,以html的形式显示出来 <br/>
执行命令： <b> jhat */dump/file/path*  </b> <br/>
	 ![Jhat读取dump文件](/images/watoud/jdkTools/jhat/jhat-server.png)  <br/>
之后就可以使用浏览器在本地查看Java分析信息了 <br/>
	  ![浏览dump信息](/images/watoud/jdkTools/jhat/jhat-client.png)  <br/>
我们可以从页面上查询所有的类、根对象、对象实例数等等 <br/>
	  ![浏览实例信息](/images/watoud/jdkTools/jhat/jhat-instance.png)   <br/>
堆dump文件的分析,除了使用jdk自带的jhat外，还可以用Eclipse MAT工具 <br/>

6. jstat：获取JVM性能以及资源消耗信息 <br/>
查看GC信息：<b>  jstat -gcutil *pid* *interval* *count* </b> <br/>
	  ![浏览实例信息](/images/watoud/jdkTools/jstat/jstat-gcutil.png)  <br/>
查看元空间信息：<b>  jstat -gcmetacapacity *pid* *interval* *count* </b> <br/>
 	  ![浏览实例信息](/images/watoud/jdkTools/jstat/jstat-metaspace.png)   <br/>

<div align="right" >
<font color="purple">
<i><b>详细说明请见官方文档</b></i>
</font>
</div>
