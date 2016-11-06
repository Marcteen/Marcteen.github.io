---
title: first post
date: 2016-10-31 23:45:36
tags: [hexo, post, git]
categories: [trials]
comments: true
---
# 如何使用git管理hexo源码目录并在多台电脑上同步
费了些力气，终于是把hexo-gitPage搭起来了，因为觉得自己无论如何要开始做好记录，以便以后能够回顾。git是熟悉了又忘记，这篇文章就先记录一下如何使用git将hexo工程的源码一并管理，方便在不同的机器上撰写并发布post。

<!--more-->

[hexo main page](https://hexo.io)
[Markdown gramma](http://www.appinn.com/markdown/#link)
[Markdown入门指南](http://www.jianshu.com/p/1e402922ee32/)
[hexo目录结构及作用](http://www.tuicool.com/articles/fiYVbaY)
[neXt主题配置文档](https://github.com/iissnan/hexo-theme-next/wiki)

## 首次配置hexo与node.js环境
1.安装node.js，这里直接去[node.js官网](https://nodejs.org/en/)下载pkg进行安装就可以了。
2.安装hexo

	mkdir blog//临时初始化目录
	cd blog
	npm install hexo --no-optional #避免执行中报异常的方法
	npm init
	npm install
	npm install hexo-deployer-git
	
3.在github上创见博客仓库Marcteen.github.io， [配置ssh登录密钥](http://www.jianshu.com/p/a655bbc178e3)

	git clone git@github.com:Marcteen/Marcteen.github.io.git
	git branch hexo #管理工程源码的分支
	git checkout hexo
	git push origin hexo #将分支发布到github上
此时我们可以在github上将仓库的默认分支设置为hexo，进一步的，我们可以为当前工程目录设置默认的push分支。

	git config --global push.default "current"
	
4.将刚才初始化的blog内容完全复制到博客仓库的本地目录中。

	cp ../blog/* ./
	rm -rf ../blog #可以考虑将其删除，或者刚才直接使用mv命令
	
5.配置.gitignore
可以适当地忽略一些不需要同步的文件。
	
6.将hexo内容同步到github仓库的hexo分支。
	git add .
	git commit -m “提交信息”
	git push origin hexo
	
7.配置部署参数

	vim _config.yml
	
为deploy字段追加内容

	type: git
  	repository: git@github.com:Marcteen/Marcteen.github.io.git
  	branch: master
	
到此应该就告一段落了
## 在新机器上同步hexo仓库并进行撰文
1.首先将博客仓库clone下来

git clone git@github.com:Marcteen/Marcteen.github.io

但是这时候只有一个hexo分支，这是为什么呢？[参考内容](http://ilewen.com/questions/1940)

2.重新配置hexo工程，这个可以参考上文首次配置中的第二步就可以了，唯一的不同就是不要执行hexo init，不然git工程配置文件就挂了。后来有次在实验室的win7机器上，突然出现了local hexo未安装的异常，无论如何安装都不行，即使重新clone下来后按上述步骤配置也不行，而且git-bash也已经由管理员身份运行了。后来发现，重启的电脑后，重新clone并配置，就一切正常。

## 发布博文注意事项
1.新建并编辑博文，[参考](http://blog.csdn.net/wizardforcel/article/details/40684575)

	hexo new “title”
	
常用的文章属性有

* layout #post或page
* title	#文章的标题	 
* date	#文件的创建日期
* updated	#文件的修改日期
* comments	#是否开启评论	
* tags	#标签	 
* categories	#分类	 
* permalink	#url中的名字
	
2.依次执行,进行工程源码的远程同步，这样就没有只能在本地写博文的问题啦！

	git add .
	git commit -m “提交信息”
	git push origin hexo
	
3.然后才进行发布，感觉这样的顺序是最佳的。
	hexo clean
	hexo g
	hexo -d
	
4.本地预览。有时候github的速度真的挺慢，本地查看也不错，同时也可以让它不要占用一个终端

	nohup hexo s &
	然后按下ctrl+c，就能够不中断本地预览并退出了。不过这样要结束本地预览呢，毕竟这不是正式环境，进程号又不好记，那么就通过下面的命令发现进程号并杀掉就好了
	
	ps aux | grep hexo
	kill -9 pidOfHexo
	
## 一点用git进行同步与合并分支的内容
由于可能使用多台设备进行文章的编辑，所以需要进行同步动作，保持当前工作目录处于最新进展。
首先是从远程主机取回更新，

	git fetch origin hexo #<主机名> <分支名>
	
然后将更新合并到本地分支，

	git merge origin/hexo
	
或者使用

	git rebase

实际中可能遇到冲突的情况，对于文件内容冲突，可以在文件中进行相关内容的编辑；而删除冲突则可直接将工作目录中的对应文件删除，然后执行

	git add ...
	git commit -m "message"

然后就可以进行merge/rebase了。	由于hexo博客涉及分支内容较少，此处暂时给出少量相关内容，以后再按需学习git吧~
	

	
	


