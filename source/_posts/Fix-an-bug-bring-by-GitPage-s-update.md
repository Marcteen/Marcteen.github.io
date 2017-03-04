---
title: Fix an bug bring by GitPage's update
date: 2016-11-05 11:23:29
tags: [debug, hexo, 前端]
categories: [Trials]
---
# 据说是GitPage默默上线新版导致的bug
昨晚发现托管于Git的博客打开之后只显示框架，内容链接全部都显示为空白，可是特定位置的链接还是可以点击，在本地预览一切正常。

<!--more-->

刚开始还以为是网速不行，但随后就发现[知乎](https://www.zhihu.com/question/52251404)上有人遇到了差不多的问题，使用的也是next主题，很奇怪昨晚就是打不开这个页面，手机客户端倒是很正常。有一个回答参考[这里](https://www.zhihu.com/question/52272725/answer/129830961)，指出是Jekyll3.3的新特性造成的问题，并给出解决方法：在工程目录下

	vim .nojekyll
添加

	!vendor/*
随后提交push并部署，发现没有变化，估计应该是将这个文件添加到master分支，但是这个要怎么做呢？直接把.nojekyll放到public里，deploy是不会将其添加到master上去的，手动切换分支，add，fetch，merge总是好多冲突，而且这样下次一deploy还会有用吗？

后来发现github在仓库分支页面可以直接在线编辑并添加文件，于是在master分支里加上了上述文件，果然排除的问题，但是这样只是暂时的，hexo -d使用的force push，下一次提交就会把它冲掉了。

后来发现知乎有前辈更新了回答，解决方案是将theme目录下的相应文件夹改名，同时更新配置文件中的相关信息。随后就发现github上next的仓库已经更新了，果断clone下来，问题排除。


