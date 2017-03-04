---
title: Set an extra JDK for new IDEA
date: 2016-12-10 11:16:13
tags: [idea, linux, jdk]
categories: [Tricks]
---
# 今天给linux虚拟机安装idea，版本很新，发现jdk的一些问题
<!--more-->
虚拟机内配置好了各种环境，jdk1.7也是妥妥的，但是安装好idea之后，新建scala工程最后一步总是提醒没有合适的jdk1.8，查询之后发现新版idea现在需要1.8来运行了，可以单独为其配置一个，而保持系统默认的为1.7，找到[官网教程](https://intellij-support.jetbrains.com/hc/en-us/articles/206544879-Selecting-the-JDK-version-the-IDE-will-run-under)看了一下，呵呵screw you，还说别人oracle的网站confusing，你们也一样，说了半天唯修改配置文件的地方也是很confusing，因为按照说明到配置文件目录里根本就找不到那个配置文件。同时也按照唯一清晰的指示进行了如下配置

	export $IDEA_JDK_64=/PATHTOJDK1.8
	
然后重启idea，并没有任何变化。

后来看到了[这里](http://jingyan.baidu.com/article/6b97984dc9f1a31ca2b0bf24.html)，按照他的说法修改了工程默认的jdk，指向1.8路径，报错消失。

