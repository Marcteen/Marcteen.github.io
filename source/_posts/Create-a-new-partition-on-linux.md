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
## 进行分区创建
 
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
输入n进行分区创建。

## 创建新分区，同时扩展根分区VolGroup-lv_root
这里注意到了一个问题，根目录挂载的根分区容量/dev/mapper/VolGroup-lv_root太小了，仅剩余2.1G，最好还是先扩展一下。

	fdisk /dev/vda
	n
	p #使用主分区类型
	20806
	117656 #增加约50G空间
因为主分区已经存在，剩余空间使用extended进行分区创建就可以了

	n
	e
	117656
	2080507 #全部用掉
注意扩展分区是不能直接使用的，要先在其上创建逻辑分区，然后才能格式化，并将其挂载到目录上去。依然是在fdisk内，输入
	
	N
	117657
	2080507
	
最后使用w保存并退出，重启后就生效了。查看分区信息如下，

	      Device Boot      Start         End      Blocks   Id  System
	/dev/vda1   *           3        1018      512000   83  Linux
	Partition 1 does not end on cylinder boundary.
	/dev/vda2            1018       20806     9972736   8e  Linux LVM
	Partition 2 does not end on cylinder boundary.
	/dev/vda3           20806      117656    48812864   83  Linux
	/dev/vda4          117657     2080507   989276904    5  Extended
	/dev/vda5          117657     2080507   989276872+  83  Linux
然后进行格式化
	
	mkfs.ext4 /dev/vda3
	mkfs.ext4 /dev/vda5

写inode表，需要一定时间，耐心等待。然后开始进行根分区的扩展。将vda3添加为物理卷

	pvcreate /dev/vda3
这时可以查看当前系统的物理键（PV）信息
	
	pvdisplay
	
	--- Physical volume ---
	PV Name               /dev/vda2
	VG Name               VolGroup
	PV Size               9.51 GiB / not usable 3.00 MiB
	Allocatable           yes (but full)
	PE Size               4.00 MiB
	Total PE              2434
	Free PE               0
	Allocated PE          2434
	PV UUID               oVndmw-Yi8V-K6Sd-duzJ-1c1p-1Bbk-K1UWfp
	   
	"/dev/vda3" is a new physical volume of "46.55 GiB"
	--- NEW Physical volume ---
	PV Name               /dev/vda3
	VG Name               
	PV Size               46.55 GiB
	Allocatable           NO
	PE Size               0   
	Total PE              0
	Free PE               0
	Allocated PE          0
	PV UUID               u2NNGd-EZlp-aLhy-Jknv-R0TP-tvA9-1YOGJi
这里给出了我们新添加的物理卷信息，另外是当前卷组情况

	vgdisplay
	
	--- Volume group ---
	VG Name               VolGroup
	System ID             
	Format                lvm2
	Metadata Areas        1
	Metadata Sequence No  3
	VG Access             read/write
	VG Status             resizable
	MAX LV                0
	Cur LV                2
	Open LV               2
	Max PV                0
	Cur PV                1
	Act PV                1
	VG Size               9.51 GiB
	PE Size               4.00 MiB
	Total PE              2434
	Alloc PE / Size       2434 / 9.51 GiB
	Free  PE / Size       0 / 0   
	VG UUID               Uf48rr-ft4p-Nh4V-2dln-0cpJ-gyI2-Q7Hi9Jwqw
将分区vda3转换为扩展分区

	vgextend VolGroup /dev/vda3
提示成功扩展后，查看当前逻辑卷

	lvdisplay
好像没有什么特别的，再查看一下扩展后的卷组情况

	vgdisplay
可以发现VolGroup的size已经成功扩展，这时我们就可以将新增的逻辑卷全部扩展到"/"分区中

	lvextend -L +46.54G /dev/VolGroup/lv_root
刷新根分区大小

	resize2fs /dev/VolGroup/lv_root 
这里再使用df查看系统挂载情况，可以看到"/"目录扩容成功。

##挂载新的逻辑分区，并使用跳板将原有文件迁移
计划将vda5挂载到/home目录，但是这个目录是非空的，需要先进行一下预处理，这个方法叫做跳板，效果是将原先的数据复制到新的分区下，首先新建一个目录，作为跳板目录，并将待vda5挂载到其上

	mkdir /jump_board
	mount /dev/vda5 /jump_board
然后将/home中的数据全部复制到跳板文件夹中，同时可以将原来/home中的文件删除（那我刚才直接使用mv命令那也是一样的效果），这样可以节省原先分区的空间，并将vda5挂载到/home，解除跳板目录的挂载并将其删除
	
	cp -R /home/* /jump_board
	mount /dev/vda5 /home
	umount /jump_board
	rm -rf /jump_board	
另外可以设置开机自动挂载

	echo  "/dev/vda5  /home    ext4    defaults    0 0" >> /etc/fstab

但是这样会有一个问题，/home目录下的文件被root用户捋了一遍，这会导致文件所有者易主，原用户将不再具有原本的读写权限，解决办法是使用如下命令纠正文件夹及其所含文件的所有者

	sudo chown -R tseg /home/tseg

