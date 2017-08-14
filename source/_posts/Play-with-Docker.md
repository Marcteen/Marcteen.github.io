---
title: Play with Docker
date: 2017-08-06 09:57:10
tags: [docker, caffeonspark]
categories: [Trials]
---
# 尝试基于官方提供的caffeonsparkdocker，定制图像处理集群
<!--more-->
## docker环境在centos下的部署
先查看内核版本号

	uname -a
(这里)指出[http://www.cnblogs.com/saneri/p/6178536.html]首先要升级内核，否则会卡，使用如下命令，6.x系统都适用

	rpm -Uvh http://www.elrepo.org/elrepo-release-6-8.el6.elrepo.noarch.rpm
	sudo yum install -y epel-release

然后是修改Grub引导顺序

	vim /etc/grub.conf
一般新安装的内核排在最前，修改如下
	
	default=0
保存后重启，通过再次查看内核版本进行验证，完成后继续配置。首先安装yum源

	yum -y install http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
进行docker安装

	yum install docker-io
启动docker

	service docker start

