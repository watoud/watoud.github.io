---
layout: post
title: Java故障排查指南之工具篇
date: 2015-12-03
comments: true
archive: true
tag: [troubleshooting ]
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
<center> ![jcmd帮助命令](/assets/images/jdkTools/jps/jps-lvm.png) </center>

2. jcmd:用于向jvm发送诊断信息<br/>
-l 列出所有java虚拟机信息	<br/>
-help 列出该虚拟机支持的所有命令. eg: <b> jcmd *processId* help </b>  <br/>
	<center> ![jcmd帮助命令](/assets/images/jdkTools/jcmd/jcmd-help.png) </center> <br/>
查看当前jvm支持的命令之后，就可以知道能够使用那些命令了. 比如：<br/>
jcmd *processId* VM.uptime  		查看系统运行时间<br/>
jcmd *processId* Thread.print 		查看线程堆栈信息<br/>
详细使用说明见文档:[http://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr006.html#BABEHABG](http://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr006.html#BABEHABG)

3. jmap：统计内存相关信息<br/>
获取堆转储文件：<b> jmap -dump:format=b,file=es.dump *pid*  </b> <br/>
	<center> ![获取转储文件](/assets/images/jdkTools/jmap/jmap-dump.png) </center> <br/>
查看堆信息：<b> jmap -heap *pid* </b>  <br/>
	<center> ![获取堆信息](/assets/images/jdkTools/jmap/jmap-heap.png)  </center> <br/>
<center>
-----------------------------------------------
| 堆配置项     | 说明              |参数设置 |
| :------------- | :---------------- | :---------- |
| MaxHeapSize   | JVM堆的最大大小| -XX:MaxHeapSize= |
| NewSize       | 新生代大小      | -XX:NewSize= |
| MaxNewSize   | 新生代最大大小  | -XX:MaxNewSize= |
| OldSize  | 老生代大小| -XX:OldSize= |
| NewRatio  | 新生代和老生代的大小比率 | -XX:NewRatio= |
| SurvivorRatio | Eden区与Survivor区的比值 | -XX:SurvivorRatio= |
| MetaspaceSize(jdk8) | Metaspace使用的是本地<br/>内存而不是堆内存| -XX:MetaspaceSize= |
</center>
除了查看堆栈信息之外，jmap还没用来查看堆中类的统计信息<br/>
查看类信息：<b> jmap -histo *pid* <br/>   </b>
	<center> ![查看histo信息](/assets/images/jdkTools/jmap/jmap-histo.png) </center> <br/>
查看类加载器统计信息: <b> jmap -clstats *pid* </b>
	<center> ![类加载器信息](/assets/images/jdkTools/jmap/jmap-clstats.png) </center> <br/>

4. jinfo: 获取Java进程的配置信息 <br/>
获取Java进程的配置信息： <b> jinfo *pid* </b> <br/>
	<center> ![Java进程配置信息](/assets/images/jdkTools/jinfo/jinfo.png) </center> <br/>





