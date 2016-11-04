---
title: Use Kvm to customize an mirror for openstack
date: 2016-11-03 10:15:31
tags: [Linux, KVM, openstack]
categories: [trials]
---

# 在linux上使用kvm，qemu，vnc等工具进行虚拟机定制以及qcow2镜像的生成
由于，这里有一个基于openstack的教学项目，所以，真的要把这个镜像做出来了，不然月底要完蛋。
服务器上应该是已经安装好了kvm相关环境，但是学弟遇到了一些问题一直没有解决，今天我自己看一看吧。用服务器来做总比用虚拟机强。

## 基础环境检查与简单配置
[参考内容1](http://www.111cn.net/sys/CentOS/87211.htm)
[参考内容2](http://www.vpsee.com/2012/02/create-centos-kvm-image-for-openstack-nova/)
Intel cpu

	cat /proc/cpuinfo | grep "vmx"
AMD cpu

	cat /proc/cpuinfo | grep "svm"

有输出内容就表示满足条件，否则进入bios更改虚拟化设置。然后安装kvm以及虚拟化所需相关软件包

	yum install -y kvm virt-* libvirts bridge-utils qemu-img

* kvm：软件包中含有KVM内核模块，它在默认linux内核中提供kvm管理程序
* libvirts：安装虚拟机管理工具，使用virsh等命令来管理和控制虚拟机
* bridge-utils：设置网络网卡桥接
* virt-*：创建、克隆虚拟机命令，使用qemu命令来创建磁盘等。
* qemu-img：安装qemu组件，使用qemu命令来创建磁盘等。

加载kvm模块

	modprobe kvm-intel

检查kvm模块是否被加载

	lsmod | grep kvm

最好是重启再确认一次

	reboot

## 安装vncserver 
不是很确定服务器上是否有vnc server，检查一下，[参考链接](http://blog.163.com/likaifeng@126/blog/static/3209731020147614916768/)
	
	ls /etc/sysconfig | grep "vncservers"
返回为空，那么就先安装一下吧

	sudo yum install tigervnc tigervnc-server -y
配置一下

	sudo vim /etc/sysconfig/vncservers
去掉注释并修改

	VNCSERVERS="2:node4"
	VNCSERVERARGS[2]="-geometry 1024x768  -nolisten tcp -localhost"
然后配置一下当前用户的vnc登录密码

	vncpasswd

验证两次，然后启动vnc服务

	vncserver &
也可以设置其随系统启动

	chkconfig --level 5 vncserver on
	chkconfig --list | grep vnc

这时候就可以使用vncviewer之类的客户端进行登录了，输入如下主机名:1即可

	node4:1
## 准备centOS-6.7镜像，创建KVM虚拟机并启动
我们可以查看虚拟机列表，当然现在暂时没有任何虚拟机实例在运行
	
	virsh list
创建虚拟机硬盘文件，据说10GB可以了，不然后面使用费时费力

	kvm-img create -f raw PDME.img 10G
很不幸，出现 kvm-img: command not found了。搜索后发现好像qemu-img也是一样的

	qemu-img create -f raw PDME.img 10G
成功创建镜像，那么就可以使用centOS镜像将系统安装在这个创建的硬盘文件中了
	
	sudo kvm -m 512 -cdrom CentOS-6.7-x86_64-bin-DVD1.iso\
	 -drive file=PDME.img -boot d -net nic -net tap -nographic -vnc :0
-vnc参数用于打开vnc访问，这样就可以通过其他机器登录到这个界面安装系统了，同时注意须加上-net nic -net tap才能建立镜像到kvm网桥virbr0的映射，网络才通。

不过这时候依然出现了找不到kvm命令的提示，我觉得适可而止惹。查阅后发现是新版本系统会把命令藏起来，推荐使用新命令virtual-install/virsh进行操作，而把qemu-kvm转移到了不起眼的地方/usr/libexex，那么就自己链接过来使用吧

	sudo ln -sf /usr/libexec/qemu-kvm /usr/bin/kvm

这好像还是和kvm-img没有关系。运行kvm创建虚拟机，报错如下

	/etc/qemu-ifup: could not launch network script
	kvm: -net tap: Device 'tap' could not be initialized
### 尝试网桥配置
似乎是网络方面的问题？搜了一下遇到这个问题的人还是不少，不过想起自己似乎略过了针对kvm进行网络方面的配置。
虚拟机采用桥接方式，使其可以获得与物理机同样级别的IP，参考[这里](http://www.centoscn.com/image-text/config/2016/0218/6765.html)
，另外还有[更详细的参考](http://blog.csdn.net/hzhsan/article/details/44098537/)，包括了NAT模式的介绍。
下面我们为其配置网桥模式

	cd /etc/sysconfig/network-scripts
	sudo cp ifcfg-em1 ifcfg-br1
编辑ifcfg-em1，添加内容
	
	BRIDGE=br1
编辑ifcfg-br1,参考内容为

	DEVICE=br1
	TYPE=Bridge
	ONBOOT=yes
	NM_CONTROLLED=yes
	BOOTPROTO=dhcp

因为我们这里没用固定的网段分配，指定静态ip是不明智的，所以直接将IPADDR等相关项省略，对于NM_CONTROLLED字段，参考内容中说明RedHat系统需要设置为NO，而CentOS似乎没有特别需要注意的说法，就保持默认的yes好了。

保存好后，重启network service

	sudo service NetworkManager stop
	sudo service network restart
	sudo service NetworkManager start
可是呢，无论如何尝试，在ifcfg-em1中添加BRIDGE=br0之后，服务器就断网了。。于是尝试virt-install,可供参考的[内容](http://www.361way.com/virt-install/2721.html)，后来发现只要不恢复NetworkManager就好像没有问题，虽然ifconfig里em*一堆的error，[这里](https://www.chenyudong.com/archives/libvirt-kvm-bridge-network.html#i)也是这样，看来应该没有大碍吧。还有[这里](http://www.cnblogs.com/jankie/archive/2012/10/19/2730826.html)提到，确实需要关闭NetworkManager才可以使网桥正常运行。

    virt-install -n PDMECENTOS \
    -r 512 -vcpus=1 \
    -c /home/node4/techbase/CentOS-6.7-x86_64-bin-DVD1.iso \
    --hvm --os-type=linux \
    --disk /home/node4/techbase/PDME.img \
    --graphics vnc,listen=0.0.0.0,port=7789 \
    --force --autostart
好像是可以成功的，自动打开了一个VNC连接至创建的虚拟机实例，但是一直没动静，过了一会连接至物理服务器的VNC黑掉了，重新连接还是黑的并提示未加密链接。

## 笑着活下去。记录一下常用virsh命令
- virsh list #显示本地活动虚拟机
- virsh list --all #显示本地所有的虚拟机（活动的+不活动的）
- virsh define ubuntu.xml #通过配置文件定义一个虚拟机（这个虚拟机还不是活动的）
- virsh start ubuntu #启动名字为ubuntu的非活动虚拟机
- virsh create ubuntu.xml #创建虚拟机（创建后，虚拟机立即执行，成为活动主机）
- virsh suspend ubuntu #暂停虚拟机
- virsh resume ubuntu #启动暂停的虚拟机
- virsh shutdown ubuntu #正常关闭虚拟机
- virsh destroy ubuntu #强制关闭虚拟机
- virsh undefine ubuntu #移除ubuntu虚拟机
- virsh dumpxml ubuntu #显示虚拟机的当前配置文件（可能和定义虚拟机时的配置不同，因为当虚拟机启动时，需要给虚拟机分配id号、uuid、vnc端口号等等）
- virsh setmem ubuntu 512000 #给不活动虚拟机设置内存大小
- virsh setvcpus ubuntu 4 # 给不活动虚拟机设置cpu个数
- 针对虚拟机操作
- virsh domid win03_3 #查看虚拟机的标识符
- virsh domname win03_2 #查看虚拟机的名称
- virsh domuuid win03_2 #查看虚拟机的 UUID
- virsh domstate win03_2 #查看虚拟机目前的状态
- virsh dominfo win03_2 #查看虚拟机的信息
- virsh console name #控制台进入指定虚拟机实例

于是又经历了一次漫长的服务器重启，上面列出的virt-install命令经过了一些修改，直接使用创建的img作为磁盘文件。当vncviewer启动并弹出之后，最好不要去动窗口，这回造成安装过程中断。使用键盘在操作界面进行安装过程就可以啦。

不过疑问还是有的，virt-install创建的虚拟机实例是如何配置所在ip及端口的呢？上面只是直接给vnc指定了一组参数。嗯在下面好像就发现了。

进入安装好的系统之后，只有一个root用户，此时可以先创建一个普通用户并设置密码

	useradd -d /home/student student
	passwd student
为了能够使该账户能够使用sudo，我们需要将其添加到sudo配置文件中去

	su
	visudo
	/ALL
找到root ALL=(ALL) ALL，在下方添加

	student ALL=(ALL) ALL
退出并保存，完成。

## 配置virt vnc连接
上面提到了一个vnc连接kvm虚拟机的参数配置问题，这下也不得不面对一下了，因为很开心地设置好用户之后将虚拟机示例shutdown，然后现在重新启动之后不知道怎么再用vnc连接了，

	vncviewer 0.0.0.0:1

对了，记录一个Mac上好用的VNC，[chicken of vnc](https://sourceforge.net/projects/cotvnc/),恩主要是因为免费，图标简直太可爱。当然功能也是简单够用的，但是注意最好不要在建立连对就是下面的命令不起作用，输密码验证错误，我输的默认密码。。接的时候选全屏模式，会抽风。
那么接着尝试解决vnc连不上虚拟机的问题，尝试如下方法

	virsh console PDMECENTOS
结果是卡住不动了，搜索之后找到了[类似问题](http://www.linuxidc.com/Linux/2014-10/107891.htm)，可是人家是vnc能连上虚拟机的情况啊。。。

然后事情出现了转机，查看[这里](http://blog.csdn.net/taiyang1987912/article/details/50474219)，前面配置虚拟主机vnc没啥好说的，默认就是那样，注意后面

	virsh edit PDMECENTOS
将端口值改为-1，之前被我照着另一个很像的教程改成了5910，最原本的默认值不记得了，嗯这里应该就可以配置和vnc相关的参数了，改好之后启动虚拟机，查看qemu-kvm运行的端口，不要使用：1，感觉这个需要虚拟机配置vncserver后进行验证才可用，通过进程端口直接连接显然更佳。

	netstat -tunlp
记得把终端全屏才能看见进程信息列，然后前面的ip:port就是vncviewer登陆所使用的参数啦。这个时候虚拟机并没有图形界面,我们可以安装一下
	
	yum -y groupinstall Desktop
	yum -y groupinstall "X Window System"
安装完成后就可以启动图形界面了

	startx
添加中文支持

	yum -y groupinstall chinese-support

设置开机自动进入图形界面

	vim /etc/inittab
将id:x:initdefautl中的x改成5即可
	
## 配置virt虚拟客户机桥接上网
[参考内容](https://www.chenyudong.com/archives/libvirt-kvm-bridge-network.html#i)

	virsh edit PDMECENTOS
添加内容

	<interface type="bridge"> <!--虚拟机网络连接方式-->
	<source bridge="br0" /> <!-- 当前主机网桥的名称-->
	<mac address="00:16:e4:9a:b3:6a" /> <!--为虚拟机分配mac地址，务必唯一，否则dhcp获得同样ip,引起冲突，而我是从已经创建好的虚拟机里面抄的，后来发现这样做十分正确-->
	</interface>
然后启动网卡，我们可以设置其随开机启动

	ifconfig eth0 up
	vim /etc/sysconfig/network-scripts/ifcfg-eth0 #置ONBOOT=true
	
	这个时候运行ifconfig就可以看到eth0的信息啦，然而没有卵用，最后发现安装虚拟机应该还是需要root，否则重启虚拟机的时候tap vnet会有权限错误
很奇怪重启服务器后执行命令会遇到locale unsupport错误，可以这么解决

	export LC_ALL=C
另外是修改系统语言的方法，感觉更有效，虽然好像对虚拟机（vnc字符界面下）没有效果，但是对虚拟主机有效果。
	
	vim /etc/profile
	/export #找到export xx xx xxx语句，在前增加LANG="zh_CN.UTF-8",在后追加LANG
然后重启系统即可生效，后来发现在工作机上直接ssh虚拟机无乱码，但是在服务器通过vnc到字符界面却会中文显示小方块，真是累，而在图形界面下又是正常的。
			
然后virt-install出现了无法启动vnc显示的问题，又照上面修改了虚拟机的监听端口为-1，然后再用vncviewer重连就好了（改之前重连也会无法启动显示），再然后直接安装又能启动vnc显示了，我晕，完全不懂怎么好的。对了，root用户就不要在普通用户目录里安装了，会有蜜汁文件权限问题。。
	
最后只得注意的是对虚拟机配置的变更很可能要重启后才生效，不管什么情况，试一试最好，我这里的网桥配置就是删除libvirt自带的nat虚拟网桥再重启虚拟机就生效了。嗯不要使用普通用户运行virsh，很坑。另一个例子，我们进入图形界面后，可以给右键菜单添加打开终端的选项，也需要重启系统

	sudo apt-get install nautilus-open-terminal #ubuntu
	sudo yum install nautilus-open-terminal #centos
另外virsh shutdown命令需要对虚拟机进行[一定配置](http://www.bubuko.com/infodetail-771662.html)才可用。
	


	
	

