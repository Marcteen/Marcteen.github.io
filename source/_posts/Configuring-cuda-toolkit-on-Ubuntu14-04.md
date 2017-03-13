---
title: Configuring cuda(toolkit) on Ubuntu14.04
date: 2017-03-11 11:40:20
tags: [cuda, unbuntu]
categories: [Trials]
---
# 包含一个删除重装的cuda7.0配置过程
<!--more-->

[参考内容](http://blog.csdn.net/xuezhisdc/article/details/48651003)

## 使用.run文件安装cuda，apt-get安装驱动

首先为了保证不再出现其他问题，先将之前配置的所有nvidia驱动以及cuda删除，首先关闭图形环境

	sudo service lightdm stop

如果使用的是.run方式安装cuda，则进入/usr/local/cuda/bin后使用如下命令卸载

	sudo .../uninstall_cuda_toolkit_7.0.pl

如果是其他安装方式，例如.deb，则可以使用
	
	sudo apt-get --purge remove nvidia*
此时直接运行.run文件，提示Nouveau禁用有问题，重启之后依然不能直接通过run安装驱动及cuda，报错信息不变，卒。[这里](http://blog.csdn.net/xuezhisdc/article/details/47075401)指出要先安装显卡驱动，再安装cuda，否则会出错。

那么吸就先通过apt-get单独安装驱动
	
	sudo apt-get install nvidia-346
运行后显示安装的是367.57，显示安装完成后，检查驱动版本号
	
	cat /proc/driver/nvidia/version
发现没有这个nvidia目录，这说明驱动没有加载。重启后再试，就可以了
	
	NVRM version: NVIDIA UNIX x86_64 Kernel Module  367.57  Mon Oct  3 20:37:01 PDT 2016
	GCC version:  gcc version 4.7.3 (Ubuntu/Linaro 4.7.3-12ubuntu1)
查看一下显卡信息

	lspci | grep VGA
还可以查看一下此时加载的驱动

	sudo prime-slelct query
上述命令输出为nvidia，此时切换到cuda7.0所在目录，执行
	
	./cuda_7.0.28_linux.run
长按d移至最下方，按照提示输入，除了不安装驱动，不安装openGL之外，其他的都为接受和默认值即可，安装完成后，因为选择了不安装驱动，会出现提示信息
	
	***WARNING: Incomplete installation! This installation did not install the CUDA Driver. A driver of version at least 346.00 is required for CUDA 7.0 functionality to work.
	To install the driver using this installer, run the following command, replacing <CudaInstaller> with the name of this run file:
    sudo <CudaInstaller>.run -silent -driver
都是唬人的，按照其他提示在/etc/profile中添加如下内容

	export PATH=$PATH:/usr/local/cuda-7.0/bin
	export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda/lib64
然后source使其立即生效

	source /etc/profile
然后在/etc/ld.so.conf.d下新建cuda.conf,添加如下内容

	/usr/local/cuda/lib64
	/lib
	/usr/lib
	/usr/lib32

此时可验证一下cuda是否正常配置完成

	nvcc --version # 或者 -V
应该可以看到输出

	nvcc: NVIDIA (R) Cuda compiler driver
	Copyright (c) 2005-2015 NVIDIA Corporation
	Built on Mon_Feb_16_22:59:02_CST_2015
	Cuda compilation tools, release 7.0, V7.0.27
然后可以到执行显卡信息查询

	NVIDIA_CUDA-7.0_Samples/bin/x86_64/linux/release/deviceQuery
稍等片刻，各个GPU的信息都能够显示出来，此时可以恢复图形界面服务
	
	sudo service lightdm start

然后配置cudnn，比较简单，主要是文件的复制和替换软链接

	cd cudnn-6.5-linux-x64-v2
	sudo cp lib* /usr/local/cuda/lib64/
	sudo cp cudnn.h /usr/local/cuda/include/
然后处理软连接

	cd /usr/local/cuda/lib64/ 
	sudo rm -rf libcudnn.so libcudnn.so.6.5 
	sudo ln -s libcudnn.so.6.5.48 libcudnn.so.6.5 
	sudo ln -s libcudnn.so.6.5 libcudnn.so 
此时可以通过运行cudnn的例子进行验证。

然后matlab之前已经成功安装完成，此时再验证一下，首先不要忘记开启vncserver
	
	vncserver :1
然后打开终端，输入matlab，能够看到matlab启动即可，然后是opencv，猜想之前的ffmpeg编译安装有问题，因此，先卸载掉ffmpeg

## 再次尝试编译ffmpeg（不建议参照）

	rm -rf /usr/local/bin/ffmpeg /usr/local/bin/ffprob /usr/local/bin/ffserver
尝试重新编译安装，因为要使用GNU来进行编译，稍微了解一下这些命令的基本情况还是必须的，[参考这里](http://blog.csdn.net/nemo2011/article/details/7384501)，首先要通过
	
	./configure
为make做好准备，否则make会无法进行，报一些找不到文件和目标之类的错误。其中提到了一个关键的文件
	
	/etc/ld.so.conf
这个文件记录了编译过程中要使用的动态链接库的路径，默认情况下，只会包含/lib和/usr/lib这两个目录下的库文件，其他要使用的库的目录的绝对路劲需要写到这个当中，然后，执行

	ldconfig
才能使得上述文件的路径被缓存到/etc/ld.so.cache当中，从而使得这些库能够在编译过程中被使用。


首先重新进行一次依赖的安装，反正如果已经安装成功的话，再安装一次也不会怎么样（通过apt-get），但是直接拷贝教程里的命令执行，会出现一些如下的信息

	"The following packages cannot be authenticated”
	and "E: There are problems and -y was used without --force-yes"
查到解决方法是：
	
	sudo apt-get update
或者修改配置文件/etc/apt/apt.conf，在其中增加如下内容

	APT::Get::AllowUnauthenticated 1
确保网络可用，执行第一个命令后，成功消除以上问题，所有依赖得以安装。然后开始ffmpeg的编译安装。
获得ffmpeg2.8源码后，进入目录，执行

	./configure --prefix=/usr/local/ffmpeg --enable-gpl --enable-version3 --enable-nonfree --enable-postproc --enable-pthreads --enable-libfaac --enable-libmp3lame --enable-libtheora --enable-libx264 --enable-libxvid --enable-x11grab --enable-libvorbis --enable-pic --enable-shared
注意./configure最后俩额外添加的enable字段，之前通过其他方法成功安装ffmpeg后，编译opencv会出错，参看[这里](http://stackoverflow.com/questions/25539034/opencv-make-fails-recompile-with-fpic)，执行后出现如下提示

	yasm/nasm not found or too old. Use --disable-yasm for a crippled build.

	If you think configure made a mistake, make sure you are using the latest
	version from Git.  If the latest version fails, report the problem to the
	ffmpeg-user@ffmpeg.org mailing list or IRC #ffmpeg on irc.freenode.net.
	Include the log file "config.log" produced by configure as this will help
	solve the problem.
此时尝试重新安装yasm
	
	sudo apt-get install yasm
提示yasm已是最新版本，奇怪的是直接输入yasm又会有如下提示

	The program 'yasm' is currently not installed. You can install it by typing:
	apt-get install yasm
英吹思婷，其实就不要用它好了，在./configure后追加一个flag就可以实现
	--disable-yasm
不过还是先试着重新安装一下
	apt-get --purge remove yasm
	apt-get install yasm
果然yasm正常了，再次configure之后，提示信息变成了
	
	ERROR: libx264 not found
回头发现刚才脚本里安装的是x264，[其他教程](http://blog.csdn.net/u012891472/article/details/51482460)里安装的是libx264-dev，那么就删掉重新安装一次试试

	apt-get --purge remove yasm
	apt-get install libx264-dev
再次configure，看起来很顺利，没有奇怪的提示了，接下来尝试编译

	make -j # -j可加速编译
	
最后看到了这些，估计没什么问题了

	LD	ffmpeg_g
	LD	ffplay_g
	LD	ffprobe_g
	LD	ffserver_g
	CP	ffprobe
	STRIP	ffprobe
	CP	ffplay
	CP	ffserver
	STRIP	ffplay
	STRIP	ffserver
	CP	ffmpeg
	STRIP	ffmpeg
然后继续编译opencv，依然是这个报错信息
	
		Linking CXX shared library ../../lib/libopencv_highgui.so
	/usr/bin/ld: /usr/local/lib/libavcodec.a(avpacket.o): relocation R_X86_64_32 against `.rodata.str1.1' can not be used when making a shared object; recompile with -fPIC
	/usr/local/lib/libavcodec.a: error adding symbols: Bad value
	collect2: error: ld returned 1 exit status
	make[2]: *** [lib/libopencv_highgui.so.2.4.11] Error 1
	make[1]: *** [modules/highgui/CMakeFiles/opencv_highgui.dir/all] Error 2
	make: *** [all] Error 2
检查了一下CMAKE的参数，发现CUDA_GENERATION参数前没有-D字段，这显然是不对的，加上后发现并没有这个参数并没有Pascal选项，遂改为Auto，命令如下
	
	cmake -D CMAKE_BUILD_TYPE=RELEASE -D CMAKE_INSTALL_PREFIX=/usr/local -D WITH_TBB=ON -D BUILD_NEW_PYTHON_SUPPORT=ON -D WITH_V4L=ON -D INSTALL_C_EXAMPLES=ON -D INSTALL_PYTHON_EXAMPLES=ON -D BUILD_EXAMPLES=ON -D WITH_QT=ON -D WITH_OPENGL=ON -D CUDA_GENERATION=Auto ..
执行出现如下问题

	nvcc fatal   : Unsupported gpu architecture 'compute_61'
	CMake Error at cuda_compile_generated_matrix_operations.cu.o.cmake:208 (message):
	  Error generating
	  /home/lthpc/opencv/opencv-2.4.11/build/modules/core/CMakeFiles/cuda_compile.dir/__/dynamicuda/src/cuda/./cuda_compile_generated_matrix_operations.cu.o
查到[这里](http://blog.csdn.net/jacke121/article/details/55007527)，但是找不到这个所谓的makefile.config，可能cmake没有这个东西？也不知道为什么，这些源码包的编译一会用cmake，一会用gcc，试了是opencv3.0，码，使用3.0，然后执行
	make -j
依然会出现一样的nvcc报错信息。然后发现，似乎pascal架构的显卡必须使用cuda8.0，比如[这里](http://www.cnblogs.com/yaoyaoliu/p/5850993.html)，网上找到的关于这个问题的解决方法，发布时间都早于pascal面世，而打patch的方法，检查了一下下载的opencv源码里面已经是修改后的内容


那么还是先清理一下opencv的碎片吧

	rm -r /usr/local/include/opencv2 /usr/local/include/opencv /usr/include/opencv /usr/include/opencv2 /usr/local/share/opencv /usr/local/share/OpenCV /usr/share/opencv /usr/share/OpenCV /usr/local/bin/opencv* /usr/local/lib/libopencv*
	
## 按照简洁的官网教程继续
转到caffe[官网教程](http://caffe.berkeleyvision.org/install_apt.html)继续，嗯这个opencv直接使用apt-get解决了，那好吧

	sudo apt-get install libprotobuf-dev libleveldb-dev libsnappy-dev libopencv-dev libhdf5-serial-dev protobuf-compiler
	sudo apt-get install --no-install-recommends libboost-all-dev
最搞笑的是服务器的网关验证老是自动断开，让人误以为是仓库不响应。。使用apt-get会让本地磁盘的缓存越来越大，可以定期清理一下
	
	apt-get clean
	
然后是安装blas，虽然GPU自己可以使用cuBlas，但是有些layer只有cpu实现，所以为cpu安装blas还是有必要的，服务器是因特尔处理器，但是mkl还要用邮箱申请序列号。。那就先用atlas好了
	
	sudo apt-get install libatlas-base-dev
	
然后是安装python的dev包，配合系统自带的python提供对pycaffe接口构建的支持。

剩下的一些依赖

	sudo apt-get install libgflags-dev libgoogle-glog-dev liblmdb-dev
官网建议使用anaconda，好处就是很多caffe,python等等依赖都包含了，而且也可以方便地安装其他依赖，就连blas库都可以，太方便了，那么安装一下

	wget https://repo.continuum.io/archive/Anaconda2-4.3.1-Linux-x86_64.sh
	bash Anaconda2-4.3.1-Linux-x86_64.sh
因为想给所有用户使用，安装路径设置为

	/usr/local/anaconda # 其实是少打了一个2
安装完成后，在/etc/profile中加入

	export PATH="/usr/local/anaconda/bin:$PATH"
然后验证一下当前系统使用的python

	which python
安装boost和pyCuda，后者需要前者

	conda search boost
	conda install boost
制定一个符合caffe要求群的版本即可，然后安装通过anaconda的pip安装theano（为何要装这个？）
	
	pip install theano -i http://pypi.douban.com/simple



