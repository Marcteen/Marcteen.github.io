---
title: Open up an port on Linux
date: 2016-11-08 09:43:12
tags:
categories:
---
# 为Linux系统开放特定端口
linux系统默认会开启防火墙（嗯应该是个系统就会开吧），那么如何开放特定端口满足实际需求呢？
<!--more-->
输入如下命令

	iptables -A INPUT -p tcp --dport port_number -j ACCEPT
	iptables -A OUTPUT -p tcp --dport port_number -j ACCEPT
然后保存并重启防火墙

	/etc/rc.d/init.d/iptables save
	/etc/init.d/iptables restart
查看端口是否开放

	/sbin/iptables -L -n

完成。