---
layout: post
title: gradle环境快速搭建
date: 2017-09-23
comments: true
archive: true
tag: gradle
---
> 最近在gitub上搜了一下kafka和elasticsearch的源码，发现这两个热门的开源软件都在使用gradle做项目依赖管理，elasticsearch更是从maven切换到了gradle，顿时感觉再不学习一下gradle就要落伍了，于是根据官网指导，学着使用gradle编译了一把java代码。

### gradle安装
> 环境设置相对比较方法，首先设置java运行环境，使用最新版本的话（4.2）至少需要jdk 1.7，接下来到官网下载最新的安装包，解压并把bin目录加入到PATH路径下。
> 设置好后执行命令**gradle -v**检查一下。
> ![gradle -v](/images/gradle/gradle-v.png)

### 编译hello world
> 创建目录gradle-demo，并在该目录下执行命令**gradle init --type java-application**，执行命令后，如下所示
> ![gradle init](/images/gradle/gradle-init.png)
> ![gradle init](/images/gradle/gradle-init-2.png)
> 命令执行成功后生成的目录结构如下所示
> ![gradle success](/images/gradle/after-init.png)
> 生成的目录结构可以分成两部分，一部分是java源代码目录结构，**src**及其下的目录结构，剩下的为gradle编译相关的文件及目录。
> **hello world**的代码已经生成，接下里只需编译就好，执行命令**gradlew build**
> ![gradle build](/images/gradle/gradle-build.png)
> 然后再执行命令**gradlew run**
> ![gradle run](/images/gradle/gradle-run.png)

### 给eclipse安装gradle插件
> 在eclipse marketplace中搜索Buildship并安装
> ![gradle plugin install](/images/gradle/gradle-plugin.png)
> 安装好插件后，把上面的**hello world**工程导入eclipse
> ![import to eclipse](/images/gradle/import-to-eclipse.png)
> ![run in eclipse](/images/gradle/run-in-eclipse.png)

### 阅读资料
> - https://gradle.org/install/
> - http://www.vogella.com/tutorials/EclipseGradle/article.html
> - http://www.vogella.com/tutorials/EclipseGradle/article.html


