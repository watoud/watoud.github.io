---
layout: post
title: 使用bat脚本运行java程序
date: 2015-11-14
comments: true
archive: true
tag: script
---
有时偶尔会用java语言写一些控制台程序，然后又不想在eclipse里面跑，比如写一个client-server的网络应用程序，运行的时候，要启动一个server，再启动一个client，这样在一个clipse里面就不方便了，所以在windows环境下，就使用bat脚本来执行吧。
    
cur-dir    
     : lib\  		*运行程序所需的jar包就放在这里*    
     : script.bat 	*运行程序的脚本*

```
@echo off

set MAINCLASS=net.watoud.demo.nio.echo.Server
set RUN_JAVA=java
set LIB=./lib
set MYCLASSPATH=
set CLASSPATH=%CLASSPATH%

setlocal enabledelayedexpansion

for %%i in (%LIB%/*.jar) do (set MYCLASSPATH=!MYCLASSPATH!;%~dp0lib\%%i)

set CLASSPATH=%CLASSPATH%!MYCLASSPATH!

setlocal disabledelayedexpansion

%RUN_JAVA% %MAINCLASS% -classpath "%CLASSPATH%"
```
