---
title: How to launch apps with shell command
date: 2016-11-02 08:08:48
tags: [linux, Unix, OSX, link]
categories: [tricks]
---

# 就像vim那样在shell终端启动应用程序吧！
自从习惯使用终端进行操作之后，就越来越习惯这么做了，当然在没有完全发挥vim潜能的情况下，使用图形化编辑器还是一个比较好的选择。那么，如果能够像vim那样输入一个命令就启动编辑器，也是一个很棒的功能。

<!--more-->

下面给出了两种实现方法，按需要来吧。感觉第一种比较厉害，但是不是一直有效，第二种方法的适用程度应该更高

1.在usr/local/bin/ 目录下创建软件可执行程序的软连接。
以sublime text为例
	
	sudo ln -s /Application/Sublime\ Text.app/Contents/SharedSupport/bin/subl /usr/local/bin/subl
	
然后就可以像这样在终端进行使用啦

	subl [filename]
	
2.使用alias自定义shell命令，那么这次使用MWeb lite为例，因为使用第一种方法为它创建命令，执行后会报错
切换到~

	vim .bash_profile
	
添加如下内容

	alias mwebl='open -a /Applications/MWeb\ Lite.app/Contents/MacOS/MWeb\ Lite'
	
然后保存并退出，执行如下命令

	source .bash_profile
	
那么就可以像这样使用MWeb lite编辑md文件了

	mwebl ./source/_post/somepost.md
	
这样使用hexo新建md之后就可以接着敲命令将其打开进行编辑了，哈哈不要用鼠标点来点去。

3.其实也可以将第二种方法的命令放到一个sh中，但是可以把sh脚本作为命令吗？其实把这个放在更具体的应用下好像也不错。


