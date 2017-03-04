---
title: Set background picture for hexo neXt
date: 2016-11-06 15:33:27
tags: [hexo]
categories: [Trials]
---
# 为hexo博客设置自己喜欢的背景吧
next主题在hexo中排名第一，看起来确实简洁大方，不过其实我不太喜欢白色背景，因为还是感觉电子屏幕上深色背景加浅色字体才不那么刺眼。哈哈但是我还不知道要怎么自己修改。今天就重新为博客设置一下背景图片先吧，之前看到有人用next主题把内容的白色背景改成了半透明，这样这个背景的内容都能看见，羡煞我也，但是也还是不知道怎么弄。那就先弄一个简单的背景吧
<!--more-->
首先，需要找到喜欢的图片，如果图太小的话，渲染的时候会自动进行重复平铺。背景图片涉及主题方面的修改，首先是将图片放到如下目录，以next主题为例

	themes/next/source/images/
然后进入当前主题相应的scheme下

	themes/next/source/css/_schemes/Pisces
	
编辑index.styl文件，在最上方添加或修改代码

	body { background:url(/images/background.jpg)}
保存并退出，重新生成部署就可了。
恩以后一定要弄一个透明背景色，让背景图片全部都能看见。

