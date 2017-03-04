---
title: submit a new working directory to github
date: 2016-11-07 23:59:56
tags: [git]
categories: [Trials]
---
# 如何将本地工程与github仓库关联
今天开始刷LeetCode了，决定使用eclipse，并将工程使用git进行管理。
<!--more-->
1.首先创建好本地java工程，提交了第一个题目的解twoSum。

2.然后在github上新建一个空白仓库

3.终端进入工程目录，进行相关初始化设置

	git init
	git add .
	git commit -m "first commit"
	git remote add origin git@github.com:Marcteen/leeTs.git
	git push origin master
完成。

对了，将仓库同步下来的时候，有一点注意一下，源码大小不大但是整个工程目录巨大无比。后来发现是.git目录很大，主要原因应该是该仓库commit很多，导致本地仓库记录了很多分支信息，解决的一个思路是控制clone的深度，因为目前往往不会参与到开源项目的贡献中,历史信息与分支暂时都没什么用。

	git clone xxx --depth 1
这样就可以只把仓库当前最新的内容同步下来了。

