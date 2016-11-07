---
title: submit a new working directory to github
date: 2016-11-07 23:59:56
tags:
categories:
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

