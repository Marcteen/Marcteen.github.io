---
title: Quick Configuring of TensorFlow
date: 2017-04-14 10:23:10
tags: [deep_learing，tensor_flow]
categories:
---
# 使用Anaconda(python3.6)配置TensorFlow
<!--more-->

## 安装配置anaconda
官网下载安装脚本，运行即可

	wget https://repo.continuum.io/archive/Anaconda3-4.3.1-Linux-x86_64.sh
	bash Anaconda3-4.3.1-Linux-x86_64.sh
然后创建一个conda环境，并可以对其指定Python运行版本

	conda create -n tensorflow
激活这个运行环境

	source activate tensorflow
停用这个运行环境

	source deactivate tensorflow

## 安装TensorFlow
激活后，命令提示符前显示当前使用的环境配置名，那么进行TensorFlow的安装
	
	pip install --ignore-installed --upgrade https://storage.googleapis.com/tensorflow/linux/cpu/tensorflow-1.0.1-cp36-cp36m-linux_x86_64.whl
注意最后一个url参数需要在[这里](https://www.tensorflow.org/install/install_linux)按照版本匹配选择。完成后，就可以开始安装验证了。

## 验证刚才安装的Tensorflow
保持刚才的conda环境，进入python，运行如下脚本

	import tensorflow as tf
	hello = tf.constant("Hello, TensorFlow!")
	sess = tf.Session()
	print(sess.run(hello))
import报错如下

	ImportError: /lib64/libc.so.6: version `GLIBC_2.16' not found (required by /home/tseg/anaconda3/lib/python3.6/site-packages/tensorflow/python/_pywrap_tensorflow.so)
静态链接库的版本问题，可以查看glibc支持的版本

	strings /lib64/libc.so.6 |grep GLIBC
输出最高只有2.12，需要更高版本来支持，一些攻略会建议给用户自己安装一个libc，但是在添加到动态链接库缓存后会出现[严重问题](http://blog.csdn.net/levy_cui/article/details/51251095)，所以还是直接升级为妙

	mkdir ~/lib_local
	cd lib_local
	wget http://ftp.gnu.org/gnu/glibc/glibc-2.16.0.tar.gz
	tar -xzvf glibc-2.16.0.tar.gz
	cd glibc-2.16.0
建立构建目录并进入
	
	mkdir build
	cd build
进行配置（指定编译后库文件的安装位置，这里指定到用户目录内）

	../configure --prefix=/usr --disable-profile --enable-add-ons --with-headers=/usr/include --with-binutils=/usr/bin
然后启动编译，需要等待一定时间

	make -j
然后安装

	sudo make install
此时再次检查libc.so.6内容，可以看到增加了新版本2.16，但是，这时候python载入报错信息变成glibc_2.17了。。。检查了一下另一个已经成功运行tensorflow-1.0的机器，发现glibc版本一直到了2.18，重新安装编译2.18之后，问题解决，再次执行刚才的测试程序成功执行，但是有如下提示

	W tensorflow/core/platform/cpu_feature_guard.cc:45] The TensorFlow library wasn't compiled to use SSE3 instructions, but these are available on your machine and could speed up CPU computations.
	W tensorflow/core/platform/cpu_feature_guard.cc:45] The TensorFlow library wasn't compiled to use SSE4.1 instructions, but these are available on your machine and could speed up CPU computations.
	W tensorflow/core/platform/cpu_feature_guard.cc:45] The TensorFlow library wasn't compiled to use SSE4.2 instructions, but these are available on your machine and could speed up CPU computations.
	W tensorflow/core/platform/cpu_feature_guard.cc:45] The TensorFlow library wasn't compiled to use AVX instructions, but these are available on your machine and could speed up CPU computations.
这只是表示，如果TensorFlow用进行编译安装，可以获得更好性能（刚才直接使用anaconda安装的），一般没有大碍，可以参考[这里](http://stackoverflow.com/questions/42334777/tensorflow-installation-using-sse-instructions-with-pip)


