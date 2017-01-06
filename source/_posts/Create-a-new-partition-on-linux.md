---
title: Create a new partition and mount it on linux
date: 2017-01-06 11:31:24
tags: [linux, disk]
categories: [Trials]
---
# 如何在Linux系统中为硬盘创建分区并挂载到特定目录下呢？
<!--more-->
参考内容：
[资料](http://www.linuxidc.com/Linux/2013-03/80860.htm)
[还是资料](http://blog.chinaunix.net/uid-25829053-id-3067619.html)

## 查看磁盘分区情况
输入命令

	sudo fdisk -l

每个磁盘都会显示为

	Disk dev/xxx

使用df命令则可以看到分区挂载到系统目录的情况

虚拟机里显示主硬盘为dev/vda，1TB并没有全部用上，那么给它继续分区。进入fdisk命令

	fdisk /dev/vda
输入m查看帮助

	a   toggle a bootable flag
	b   edit bsd disklabel
	c   toggle the dos compatibility flag
	d   delete a partition
	l   list known partition types
	m   print this menu
	n   add a new partition
	o   create a new empty DOS partition table
	p   print the partition table
	q   quit without saving changes
	s   create a new empty Sun disklabel
	t   change a partition's system id
	u   change display/entry units
	v   verify the partition table
	w   write table to disk and exit
	x   extra functionality (experts only)
输入n进入分区添加，注意如果这块硬盘之前就存在其他分区，要注意后面分区的起止柱树（建议接着已存在分区的末尾），因为前面的分区可能不是从1开始的，这时用默认值就会出现无法使用后方剩余空间的问题
输入n进行分区创建，因为主分区已经存在，这里使用extended就可以了，然后输入标号（自增不就可以了），然后输入起止柱数，最后使用w保存并退出，重启后就生效了。查看分区信息如下，新分区为vda3

	Device Boot      Start         End      Blocks   Id  System
	/dev/vda1   *           3        1018      512000   83  Linux
	Partition 1 does not end on cylinder boundary.
	/dev/vda2            1018       20806     9972736   8e  Linux LVM
	Partition 2 does not end on cylinder boundary.
	/dev/vda3           20806     2080507  1038089768    5  Extended
这时还不能使用，因为创建的是扩展分区，还需要在其上创建逻辑分区才可以格式化

	sudo fdisk /dev/vda
然后输入N，直接按两下回车选取默认的其实值就可以了，然后退出并保存。这时候查看分区多了一项
	/dev/vda5           20806     2080507  1038089736+  83  Linux
对其进行格式化

	mkfs -t ext3 -c /dev/vda5
如果硬盘比较大的话，检查坏块的时间会比较久，耐心等待吧，哦，等完后要写inode表，还等再多等一会

然后挂载
	mount /dev/vda5 /home/xxx


