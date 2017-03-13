---
title: Configuring cuda(toolkit) on Ubuntu14.04
date: 2017-03-11 11:40:20
tags: [cuda, unbuntu]
categories: [Trials]
---
# 包含一个删除重装的cuda7.0配置过程
<!--more-->

[参考内容](http://blog.csdn.net/xuezhisdc/article/details/48651003)

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
发现没有这个nvidia目录，这说明驱动没有加载。首先重启后再试，发现驱动成功加载
	
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

	rm -rf /usr/local/bin/ffmpeg /usr/local/bin/ffprob /usr/local/bin/ffserver
尝试重新编译安装