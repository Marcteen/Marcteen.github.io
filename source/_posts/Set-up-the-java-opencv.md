---
title: Set up the java opencv
date: 2017-07-29 19:22:21
tags:[opencv, java]
categories:[Trails]
---
# 编译并使用opencv官方提供的java接口
<!--more-->

## 使用cmake来编译

按照[这里](http://docs.opencv.org/2.4.11/doc/tutorials/introduction/desktop_java/java_dev_intro.html)的文档，进行源码的编译，首先下载源码，解压后进入目录，新建build文件夹

	mkdir build
然后进行配置，页面上指出关闭so的生成后，java关联的动态库就会包含一切运行所需实现，不再需要其他外部运行环境中的库。

	cmake -DBUILD_SHARED_LIBS=OFF -DWITH_FFMPEG=1 -DWITH_CUDA=0 ..
因为最终运行的集群没有GPU，所以排除cuda依赖，如果提示没有cmake，用如下命令安装

	apt-get install cmake
然后启动编译，并使用多核进行加速

	make -j
	
发现报了个错，提示jni有问题，遂将服务器的jdk配上，但是java依然是available，检查后是缺少了ant，那么安装一下吧

	apt-get install ant
再次执行，编译成功，终端打印出了输出文件的路径,但其实我们需要的文件是

	build/lib/libopenxxxx.so
	build/bin/openxxxx.jar
	
拷贝到虚拟机中的id工程，尝试运行，发现如下提示

	/usr/lib64/libstdc++.so.6: version `GLIBCXX_3.4.15' not found
有可能是libstdc++版本太低，尝试更新

	sudo yum install libstdc++.so.6
安装之后，发现GLIBC版本信息没有任何变化，好吧，就手动安装一个，下载下列文件

	libstdc++6_4.7.2-5_i386.deb
转化一下

	ar -x libstdc++6_4.7.2-5_i386.deb && tar xvf data.tar.gz
将其下的usr/lib/...的文件放到/usr/lib64中，重建硬链接
	
	ln libstdc++.so.6 libfile
再次执行scala程序，报错信息出现

	libstdc++.so.6: wrong ELF class: ELFCLASS32
难道是32位和64位的问题？用如下命令检查

	file libfile
输出为

	libstdc++.so.6.0.17: ELF 32-bit LSB shared object, Intel 80386, version 1 (SYSV), dynamically linked, stripped
果然，32位编译的共享库不行，于是在[这里](http://www.cnblogs.com/cppskill/p/5748032.html)重新下载了64位版本，再次尝试进行配置，报错信息变成了

	version `GLIBC_2.15' not found
我觉得适止而可，联想到上次配置tensorflow遇到的问题，我决定直接升级到2.18，参考[这里](https://marcteen.github.io/2017/04/14/Quick-Configuring-of-TensorFlow/#more)，至此，spark程序可以单机顺利运行了

## 读取视频
虽然可以顺利进行，从输入日志里看好像也是正确访问了视频文件，但是检查RDD后发现，其实它一帧图像也没有。。

首先想到的是很有可能是确实其他依赖，甚至连ffmpeg也没有，查了查，确实如果缺少其他读取视频所需的库，opencv这个功能只会静悄悄地失败，那就先配上它再说吧，参考(这里)[https://marcteen.github.io/2017/03/11/Configuring-cuda-toolkit-on-Ubuntu14-04/#more]

继续编译ffmpeg，果然，还是有很多依赖没解决，先使用apt-get安装依赖，如果需要GCC 4.6+，参考(这里)[http://www.linuxidc.com/Linux/2012-12/75982.htm]进行升级
## gcc升级
	wget ftp://gcc.gnu.org/pub/gcc/infrastructure/{gmp-4.3.2.tar.bz2,mpc-0.8.1.tar.gz,mpfr-2.4.2.tar.bz2}
	wget http://ftp.gnu.org/gnu/gcc/gcc-4.6.1/gcc-4.6.1.tar.bz2
先装gmp

	tar jxf gmp-4.3.2.tar.bz2
	cd gmp-4.3.2
	./configure -prefix=/usr/local/gmp/
	make -j
	make install
mpfr
 	
	tar jxf mpfr-2.4.2.tar.bz2
	cd mpfr-2.4.2
	./configure -prefix=/usr/local/mpfr -with-gmp=/usr/local/gmp
	make -j
	make install
mpc	
	
	tar xzf mpc-0.8.1.tar.gz
	cd mpc-0.8.1
	./configure -prefix=/usr/local/mpc -with-mpfr=/usr/local/mpfr -with-gmp=/usr/local/gmp
	make -j
	make install
	
终于轮到gcc了，注意不要使用多核编译

	tar jxf gcc-4.6.1.tar.bz2
	cd gcc-4.6.1
	./configure -prefix=/usr/local/gcc -enable-threads=posix -disable-checking -disable-multilib -enable-languages=c,c++ -with-gmp=/usr/local/gmp -with-mpfr=/usr/local/mpfr -with-mpc=/usr/local/mpc
	make
	make install
出现如下报错
	
	cannot compute suffix of object files: cannot compile
(这里)[https://my.oschina.net/zchking/blog/97704]指出是因为前面依赖没能加载的问题，那么将他们都进行动态链接库加载就好了

	vim /etc/ld.so.conf.d/gcc_dependency.conf
添加如下内容

	/usr/local/mpfr/lib
	/usr/local/mpc/lib
	/usr/local/gmp/lib
然后

	sudo ldconfig
依然编译不成功，按照提示查看config.log，发现如下报错

	error: ppl_c.h: No such file or directory
	
### 半路调到cloog-ppl
查了查，是cloog-ppl依赖的问题，那么下载cloog和ppl

	http://www.cloog.org/
	http://gcc.parentingamerica.com/infrastructure/
	tar -zxvf cloog-parma-0.16.1.tar.gz
	cd cloog-parma-0.16.1
	./configure -prefix=/usr
此时出错说找不到gmp的头文件，大概是因为前面只有运行环境，但还是缺少了开发环境，用yum安装

	yum install gmp-devel
再次执行，报错变成了缺少ppl的头文件，遂如法炮制

	yum install gmp-devel
终于成功，接着编译安装

	make -j
	make install
此时先回投尝试gcc编译，configure.log中依然会存在错误，choke错误？？，(这里)[http://blog.erdemagaoglu.com/post/3444247672/compiling-gcc-45-on-debian-unstable]有一些说法，这些报错信息坑很多，这里的信息实际上没什么用，真正应该关注的是ppl的版本检查提示，完全不相关的信息。那么还是先编译ppl吧，yum当中的ppl是0.10,肯定不能达到gcc configure要求的0.11	

	tar -xzvf ppl-0.11.tar.gz
	cd tar ppl-0.11
	./configure -with-gmp=/usr/local/gmp --enable-cxx
	make -j
	make install
完成后，又报错。。
	
	error while loading shared libraries: libpwl.so.5
使用whereis命令，发现其在

	/usr/local/lib/libpwl.so.5
于是将这个路径也加到ld.so.conf.d当中，ldconfig之后重试，gcc编译成功！然后将原来的gcc链接备份一下，引入新的gcc

	mkdir -p  /data/backup/`date +%Y%m%d`  
	mv /usr/bin/{gcc,g++}      /data/backup/`date +%Y%m%d`
	ln -s /usr/local/gcc/bin/gcc          /usr/bin/gcc
	ln -s /usr/local/gcc/bin/g++          /usr/bin/g++ 
	
## 回到ffmpeg
首先是安装ffmpeg的依赖，参考(这里)[http://blog.csdn.net/xuezhisdc/article/details/48691797]进行，条目是有点多的,但是最好是在安装命令后追加，建议apt就用其默认的ubuntu cn源，之前替换阿里云源之后总会出现依赖unmet的现象，不过倒是不太确认是否真的是这个原因所致。同时，先按照官网教程配置caffe/caffeonspark，再进行这个编译会比较顺利

	--force-yes
否则有些依赖会因为校验的问题没能安装成功，也不太容易发现问题，完成后下载ffmpeg源码并进行编译

	wget http://ffmpeg.org/releases/ffmpeg-2.8.tar.bz2
	tar -xf ffmpeg-2.8.tar.bz2
	cd ffmpeg-2.8
	 ./configure --enable-gpl --enable-version3 --enable-nonfree --enable-postproc --enable-pthreads --enable-libmp3lame --enable-libtheora --enable-libx264 --enable-libxvid --enable-libvorbis --enable-pic --enable-shared
	 make -j
	 make install
	 
	 当然，仅仅使用opencv生成的单个so还是不能覆盖所有需求，它依然会依赖系统所拥有的动态链接库，然而linux系统混乱的.so文件管理经常让人找不出缺失的so文件来自于哪一个软件工具。。然后，再次按照上面opencv的说明重新编译，所生成文件就可以在系统里通过java调用进行视频帧的处理了。
	 
## Dlib安装

那么，还是尝试一下这个可以检测68人脸关键点的工具吧，再尝试看看能不能封装到java里面，首先是下载源码，编译安装，先用最新版，突然发现dlib自带一整套图像功能，似乎没有其他依赖，先试试看看吧

	wget http://dlib.net/files/dlib-19.4.tar.bz2
	tar -jxvf dlib-19.4.tar.bz2
	cd dlib-14.9
先编译运行一下单元测试

	cd dlib/test
	mkdir build
	cd build
	cmake ..
	cmake --build . --config Release
	./dtest --runall
跑了许久，确实跑完了，不过好像也没什么用，那么尝试一下人脸检测的调用，发现dlib提供的example里就有人脸检测以及landmark标注程序，按照官网编译之后，其build目录下就会有对应的可执行程序，另外，我么可以通过ldd命令查看可执行文件或者so文件所依赖使用的动态库情况

	ldd executable_file/.so
可以仿照example的方式进行自己应用的cmake编译配置，完成后，发现工程里提供的jpg全部都可以识别出人脸，移到gpu服务器上也是一样，但是视频帧就会出问题，只有开发机能够识别人脸，其他的机器一直是0人脸。检查cmake的不同，只发现gpu服务器上没有lapack，那么先安装试试

	sudo apt-get install liblapack-dev
安装完之后，依然显示找不到lapack模块，但是却找到了lapack。。。不得其解，重新检查处理的图片，发现图片全是灰色的，原来还是文件的问题。
	 




