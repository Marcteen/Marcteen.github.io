---
title: Openstack managing notes
date: 2017-01-09 16:28:57
tags: [openstack, fuel]
categories: [Tricks]
---
# openstack集群管理的一些常用命令
<!--more-->

## 使用fuel远程到任意节点，提供身份认证信息

首先要登录到fuel上去，这个fuel具体在哪不是很清楚，是一个内网ip，那么登录上去之后，可以查看一下集群节点列表
	
	fuel nodes
或者

	fuel node list

注意登录的时候，所用主机名和上述输出列表中显示内容不一致，需要将编号意外的文字换成node，像这样

	ssh node-6
登录之后，可以使用各种openstack相关命令进行操作，但是首先要提供身份信息，使用如下命令即可，

	source openrc # opensrc中包含了所有的身份证书信息

## glance查看镜像信息
* 列出所有可用镜像
	
		glance image-list
* 镜像详细信息

		glance image-show

* 修改镜像信息

		glance image-update --{Property_name} {new_value} {image_id}

## 批量配置免密登录
需要注意的是，authorized_keys文件权限必须是600，否则验证会失败，首先在本地生成密钥

	cd ~/.ssh
	rm ./id_rsa*
	ssh-keygen -t rsa
在需要登录的节点上将生成的公钥授权

	cat ./id_rsa.pub >>　．/authorized_keys
如果在本地添加，可以简单在本地验证免密登录是否配置成功

	ssh localhost
因为其他节点都没有任何已授权公钥，可以直接将这个文件scp过去，可以使用一个shell脚本实现

	#! /bin/bash

	for line in "cat $1"
	do
		scp $2 ${line}:~/.ssh/
	done
但是这样放过去的文件权限还是不对的，需要修改文件权限，但是一个一个手动修改显然效率不高，而目前还没有实现免密码登录，我们可以使用expect来帮助执行，首先安装

	sudo yum install expect
然后是expect脚本

	#!/bin/bash
	expect <<EOF
	set timeout 10
	set hosts [lindex $1] # shell变量使用$前缀
	set file [open \$hosts r] # expect变量使用\$前缀
	while {[gets \$file line] != -1} { # 括号处有严格空格格式
        spawn ssh \$line
        expect "*password:"
        send "tseg\n"
        expect "Last*"
        send "chmod 600 ~/.ssh/authorized_keys\n"
        expect "$" // shell 命令提示符，好像没用，捕获不到
        send "exit\n"
        puts "finish action on \$line"
	}
	close \$file
	EOF
尝试了很多次都不行，主要是expect没有采用嵌套循环的形式处理所有可能情况，比如有的节点不用输密码，有的则要，并且登录后打印的第一条信息是“Last login...”，最后全部权限改好之后发现还是没有免密码效果，疑惑良久。最后发现是上午新建了公钥，忘了给所有节点重新传一份了（这也可以用expect自动输入密码）。嗯有时候事情没有按照预想的推进，基本上就是因为它所依赖的前提条件没有满足（太基础，太久，到这一步已经不会去回想了，都假设一定正确了），而不是说这依然还需要其他后续处理。关键找一个人帮忙，对方没有被困组，很有可能打破僵局，或者自己学会重启，成为打破僵局的人。
到了这一步，我所认为的条件是
* authorized_key权限正确

我忘记的条件是
* authorized_key内容正确

## 一个更为完善的expect命令脚本

	#! /bin/bash
	expect <<EOF
	set timeout 7
	set hosts [lindex $1]
	set file [open \$hosts r]
	while {[gets \$file line] != -1} {
        spawn ssh \$line # 发送命令，会产生新的shell进程
        expect { # 并行匹配，同时捕获Are或password进行对应动作
            "Are" {
                send "yes\n"
				exp_continue # 继续后面的捕获，这里表现为捕获Are后，依然会继续捕获后面的Are以及password并行捕获执行
            }
			“Are” { # 有可能出现主机名验证不一致，再次确认，这是hosts设定导致的问题
				send "yes\n"
				exp_continue
			}
            "password" {
                send "r00tme\n"
            } # 注意并行的 "捕获串""空格"{xxx xxx}严格格式
        }
        expect "#" # 命令提示符，发现$和用户名都不起作用，一定会等到超时才下一步？？但这个#确实可以
        send "shutdown -h now\n"
        expect "for" #shutdown 命令占用终端时输出，这里可以立即捕获并执行exit
        send "exit\n"
        puts "finish action on \$line"
	}
	close \$file
	EOF



	
