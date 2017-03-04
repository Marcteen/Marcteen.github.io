---
title: Set up Jupyter with anaconda
date: 2016-12-08 23:34:18
tags: [python, anaconda]
categories: [Trials]
---
# 发现有一个叫做Jupyter的工具很不错，在浏览器里使用ide，试试看看
<!--more-->
这里是[Jupiter主页](http://jupyter.org),具体来说叫做Jupyter notebook，定义为一个开源交互式数据科学以及科学计算平台，支持超过四十多种语言，嘿嘿，scala也可以用。

进入安装的说明页面，果然建议使用anaconda进行安装，正好已经安装过了，应该是符合要求的。不过为了保险还是查看一下提供的anaconda安装页面。不太确定当前系统是否默认使用ananconda内置的python。输入如下命令可以查看anaconda已经安装的库

	conda list
## 确定系统当前默认使用的是哪一个python
之前折腾了一下caffe，虽然没有成功，但是python这块应该是没有问题的，为了保险起见，还是检查一下。上面命令输出信息表示包含的python版本为2.7.12。但是系统默认使用的是哪一个呢，首先输入命令查看python版本

	python --version
显示了如下信息

	Python 2.7.12 :: Anaconda custom (x86_64)
看来现在系统默认使用的就是anaconda中提供的python了，这个是怎么实现的呢，参考[这里](http://stackoverflow.com/questions/22773432/mac-using-default-python-despite-anaconda-install)，检查~/.bash_profile内容，发现应该是anaconda在安装的时候就自动修改了，将其lib路径追加到了PATH路径前面，使得其被优先采用。

## 使用anaconda安装jupiter
嗯，安装安装说明，anaconda似乎已经包含了我所需要的Jupyter notebook，那么在终端输入下面命令

	Jupyter notebook
启动成功，浏览器自动打开了
	
	http://localhost:8888/tree
看了一下，不太明白是如何使用的，而且默认工作目录路径是怎么样的呢？实际上页面的Conda标签页是可以进行相关配置的，这个按照需要进行吧。那么如何终止运行呢，在启动Jupyter的命令行按下ctrl+c即可。

## pip安装其他软件包注意事项
考虑到anaconda也自带一个pip，通过这个来安装其他软件工具包的时候，也应该使用ananconda自带的pip安装，从而避免麻烦，那么系统是否默认采用这个pip呢，输入如下命令可以进行检查

	which pip
输出信息为/Users/Marcteen/anaconda/bin/pip，确认完成。

##  修改Jupyter默认目录

首先生成一个配置文件，输入如下命令

	jupyter notebook --generate-config
	
修改如下字段的值，注意去掉注释

	c.NotebookApp.notebook_dir
不过到底有没有修改的必要呢？并不知道，我就单独为它创建一个文件夹吧，就使用~/NotebookDir好了。另外这些参数也可以在启动notebook的时候指定，像下面这样

	jupyter notebook --NotebookApp.notebook_dir=xxx
只要配置的属性名正确（错误会被忽视，无报错输出），就可以看到路径已经被成功修改啦。


