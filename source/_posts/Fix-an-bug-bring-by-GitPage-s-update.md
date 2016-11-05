---
title: Fix an bug bring by GitPage's update
date: 2016-11-05 11:23:29
tags: [debug, hexo, 前端]
categories: [trials]
---
# 据说是GitPage默默上线新版导致的bug
昨晚发现托管于Git的博客打开之后只显示框架，内容链接全部都显示为空白，可是特定位置的链接还是可以点击，在本地预览一切正常。刚开始还以为是网速不行，但随后就发现[知乎](https://www.zhihu.com/question/52251404)上有人遇到了差不多的问题，使用的也是next主题，很奇怪昨晚就是打不开这个页面，手机客户端倒是很正常。有一个回答参考[这里](https://www.zhihu.com/question/52272725/answer/129830961)，指出是Jekyll3.3的新特性造成的问题，并给出解决方法：在blog目录下

	vim .nojekyll
添加
	!vendor/*
	!node_modules/*
	


