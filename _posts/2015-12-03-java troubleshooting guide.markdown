---
layout: post
title: Java故障排查指南
date: 2015-12-03
comments: true
archive: true
tag: [troubleshooting ]
---
### Java启动参数配置
1. 使用core文件。linux系统下使用```ulimit -c unlimited```
2. 添加-XX:+HeapDumpOnOutOfMemoryError虚拟机标志.<br/>
```-XX:+HeapDumpOnOutOfMemoryError  -XX:HeapDumpPath=${path\to\file}```
3. 在Java命令行添加参数```-verbose:gc```，日志会打印到控制台上，与```-Xloggc:${path\to\file}```共存时，以后者为准

### 诊断工具
1. jps:查看系统java进程信息<br/>
参数设置：<br/>
-m 输出main method的参数<br/>
-v 输出jvm参数<br/>
-l 输出完全的包名，应用主类名，jar的完全路径名 <br/>
<br/>
2. jcmd:用于向jvm发送诊断信息<br/>
-l 列出所有java虚拟机信息	<br/>
-help 列出该虚拟机支持的所有命令. eg: jcmd *processId* help. 查看当前jvm支持的命令之后，就可以知道能够使用那些命令了. 比如：<br/>
jcmd *processId* VM.uptime  		查看系统运行时间<br/>
jcmd *processId* Thread.print 		查看线程堆栈信息<br/>
详细使用说明见链接    
http://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr006.html#BABEHABG