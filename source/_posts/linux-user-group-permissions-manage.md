---
title: linux user/group permissions manage
date: 2017-03-16 09:25:49
tags: [linux, user, permission]
categories: [Tricks]
---

# 一些管理linux用户的笔记
<!--more-->

## 添加用户，以及一些基本配置
	
	useradd -r -m -s /bin/bash username
	passwd username
	"username's password"
注意useradd的参数，此时我们可以加入账户自动过期的功能，比如

	useradd ... -e YYYY/MM/DD
## 查看用户添加的默认配置

	useradd -D
可以修改这个默认配置，例如修改过期时间

	useradd -D -e "xx/xx/xx"
查看用户时效信息

	chage -l xxx
修改用户属性

	usermod -e xx/xx/xx username
调整账户过期
	
	change -E xx/xx/xx username

## 如果要赋予用户sudo的功能，使用root进行如下操作

	visudo
找到
	
	root ALL=(ALL)ALL
在下方添加下列语句即可

	xxx ALL=(ALL) ALL
如果希望使用更短的密码，修改文件
	
	sudo vim /etc/login.defs
修改最短密码长度为

	PASS_MIN_LEN 4

## 通过密码禁用账户

	passwd -l username # 禁用
	passwd -u username # 启用
## 通过usermod禁用账户

	usermod -L username #禁用
	usermod -U username #启用
## 删除用户
	
	userdel -r username
## 更改用户名及用户目录

	usermod -l oldname newname
然后修改用户目录名称，并将当前用户目录指定到新名称目录

	vim /etc/passwd

## 修改欢迎信息

首先新建文件，保存要打印的文本
	
	vim /etc/update-motd.d/gpu_tips
然后修改文件

	vim /etc/update-motd.d/00-header
在其中加入一行

	cat /etc/update-motd.d/gpu_tips
保存，可以用如下命令测试一下
	
	run-parts /etc/update-motd.d
欢迎信息

	########################### Manggis GPU Server Tips ############################
	1.Please ensure that all the environment variables are available by calling:    #
	"source /etc/profile" in case that it is not loaded automatically, or add it    #
	to your ~/.bashrc.                                                              #
	                                                                                #
	2.The cuda and caffe(and other tools for deep learning as well) are sensitive   #
	to the change of system configure, so we do not provide "sudo" for you account  #
	by default to avoid the computing enviroment from ruined.                       #
	                                                                                #
	3.If you want to install something, install it for yourself only by compiling   #
	the corresponding source code and install it in your own directory. If you      #
	think it is a global need, ask the manager to use apt-get, for now, the manager #
	is liuchang.                                                                    #
	                                                                                #
	4.Some paths for the libs:                                                      #
	        anaconda_4.3.14: /usr/local/anacoda                                     #
	        cuda_8.0.28: /usr/loca/cuda                                             #
	        cuda_8.0_samples: /root/NVIDIA_CUDA-8.0_Samples                         #
	        ffmpeg_2.8.11: /usr/local/ffmpeg                                        #
	        opencv_2.4.13: /usr/local/opencv                                        #
	        matlab_r2014a: /usr/local/matlab                                        #
	        caffe: /usr/loca/caffe                                                  #
	################################################################################


