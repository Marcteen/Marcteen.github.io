---
title: Using Maven to build an DeepLearning4j on Spark
date: 2016-11-28 10:30:15
tags: [Maven, Scala, DeepLearning4j, idea]
categories: [Tricks]

---
#在学习使用DeepLearning4j on Spark的同时，练习使用Maven构建工具
<!--more-->
都知道Maven可以直接初始化一个项目工程目录，模板有很多，但是Idea里似乎默认没用内置一个合适的，看起来都很奇怪，因为是Spark Scala工程，所以我采用了下列Archetype

	Archetype Group Id : net.alchim31.maven
	Archetype Artifact Id : scala-archetype-simple
	Archetype Version : 1.6
	//如果下载失败在添加Repository URL试试
	Repository URL : http://github.com/davidB/scala-archetype-simple

然后所需要的依赖在pom.xml中添加即可，这个有一个[网站](http://mvnrepository.com/)可以很方便地查询所需依赖的写法。暂时记录到这里。
当然，我发现这个模板有一个bug，test中有一个JUnitRunner没有导入。

然后还遇到了一些奇怪的问题。
1.make工程出现EOF异常

这个没有找到原因，使用invalidate cache不起作用，但是重启电脑后就好了。

2.scala指出RDD对hadoop有不恰当引用。
后来发现是因为spark依赖的scope问题，改为compile就不会报错了，但是这个应该设置为provided，实测命令行mvn编译不会受到这个影响。

