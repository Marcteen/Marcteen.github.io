---
title: Configuring cuda(toolkit) on Ubuntu14.04
date: 2017-03-11 11:40:20
tags: [cuda, unbuntu]
categories: [Trials]
---
# 包含一个删除重装的cuda8.0配置过程
<!--more-->

[参考内容](http://blog.csdn.net/xuezhisdc/article/details/48651003)

## 使用.run文件安装cuda，apt-get安装驱动，pascal架构的titan x必须使用cuda8.0

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
上述命令输出为nvidia，此时切换到cuda8.0所在目录，执行
	
	./cuda_8.0.28_linux.run
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
可以查询一下GPU的状态

	nvidia-smi
然后可以到执行显卡信息查询，如果exmaple下面没有bin，说明是未编译的源码，先执行

	make
然后就可以执行deviceQuery的例子了

	NVIDIA_CUDA-8.0_Samples/bin/x86_64/linux/release/deviceQuery
稍等片刻，各个GPU的信息都能够显示出来，此时可以恢复图形界面服务
	
	sudo service lightdm start
然后配置cudnn，比较简单，主要是文件的复制，进入cudnn目录，拷贝相应文件到目标位置

	cp include/cudnn.h /usr/local/cuda/include/
	cp lib64/* /usr/local/cuda/lib64
此时可以通过运行cudnn的例子进行验证。

然后matlab之前已经成功安装完成，此时再验证一下，首先不要忘记开启vncserver
	
	vncserver :1
然后打开终端，输入matlab，能够看到matlab启动即可，然后是opencv，猜想之前的ffmpeg编译安装有问题，因此，先卸载掉ffmpeg

## 再次尝试编译ffmpeg

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
英吹思婷，其实也可以不用它，只是一个加速编译的工具，在./configure后追加一个flag就可以实现

	--disable-yasm
不过还是先试着重新安装一下

	apt-get --purge remove yasm
	apt-get install yasm
果然yasm正常了，再次configure之后，提示信息变成了
	
	ERROR: libx264 not found
回头发现刚才脚本里安装的是x264，[其他教程](http://blog.csdn.net/u012891472/article/details/51482460)里安装的是libx264-dev，那么就删掉重新安装一次试试

	apt-get --purge remove x264
	apt-get install libx264-dev
再次configure，看起来很顺利，没有奇怪的提示了，接下来尝试编译

	make -j # -j可加速编译
	make install
	
最后看到了这些，估计没什么问题了,ffmpeg编译成功

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
## 编译opencv的过程

首先建议cuda8.0以及最新版本caffe搭配3.2使用，因为后面在测试caffe的时候发现找不到3.x的opencv库。继续编译opencv前，检查了一下教程给出的CMAKE的参数，发现CUDA_GENERATION参数前没有-D字段，这显然是不对的，加上后发现这个参数并不支持Pascal选项，遂改为Auto，命令如下
	
	mkdir build
	cd build
	cmake -D CMAKE_BUILD_TYPE=RELEASE -D CMAKE_INSTALL_PREFIX=/usr/local -D WITH_TBB=ON -D BUILD_NEW_PYTHON_SUPPORT=ON -D WITH_V4L=ON -D INSTALL_C_EXAMPLES=ON -D INSTALL_PYTHON_EXAMPLES=ON -D BUILD_EXAMPLES=ON -D WITH_QT=ON -D WITH_OPENGL=ON -D CUDA_GENERATION=Auto ..
	make -j
执行出现如下问题

	Linking CXX shared library ../../lib/libopencv_highgui.so
	/usr/bin/ld: /usr/local/lib/libavcodec.a(avpacket.o): relocation R_X86_64_32 against `.rodata.str1.1' can not be used when making a shared object; recompile with -fPIC
	/usr/local/lib/libavcodec.a: error adding symbols: Bad value
	collect2: error: ld returned 1 exit status
	make[2]: *** [lib/libopencv_highgui.so.2.4.11] Error 1
	make[1]: *** [modules/highgui/CMakeFiles/opencv_highgui.dir/all] Error 2
	make: *** [all] Error 2
注意看报错信息，当前是将opencv编译成动态链接库（遇到问题时，正要链接出libopencv_highgui.so），需要使用其他库生成的静态链接库（libavcodec.a），但是指出这个静态链接库在编译时没有开启-fPIC选项，看上面个的命令，显然编译ffmpeg的时候，-fPIC是加上了的，查了一下，编译后生成的/usr/local/ffmpeg/lib目录下有一个同名文件，但这并不是报错中指出的/usr/local/lib/libavcodec.a，说明编译时并没有引用我们自己编译生成的ffmpeg库文件，那么在/etc/ld.so.conf.d/下新建文件ffmpeg.conf，并加载库文件至cache
	
	echo /usr/local/ffmpeg/lib >>　/etc/ld.so.conf.d/ffmpeg.conf
	ldconfig
然后还是在build目录中，执行
	
	make clean
然后就可以继续make了，然后执行到了65%，报错如下

	/home/lthpc/opencv/opencv-2.4.11/modules/gpu/src/graphcuts.cpp:120:54: error: ‘NppiGraphcutState’ has not been declared
	/home/lthpc/opencv/opencv-2.4.11/modules/gpu/src/graphcuts.cpp:135:18: error: expected type-specifier before ‘NppiGraphcutState’
	/home/lthpc/opencv/opencv-2.4.11/modules/gpu/src/graphcuts.cpp:141:9: error: ‘NppiGraphcutState’ does not name a type
	/home/lthpc/opencv/opencv-2.4.11/modules/gpu/src/graphcuts.cpp: In constructor ‘{anonymous}::NppiGraphcutStateHandler::NppiGraphcutStateHandler(NppiSize, Npp8u*, {anonymous}::init_func_t)’:
	/home/lthpc/opencv/opencv-2.4.11/modules/gpu/src/graphcuts.cpp:127:13: error: ‘pState’ was not declared in this scope
	/home/lthpc/opencv/opencv-2.4.11/modules/gpu/src/graphcuts.cpp: In destructor ‘{anonymous}::NppiGraphcutStateHandler::~NppiGraphcutStateHandler()’:
	/home/lthpc/opencv/opencv-2.4.11/modules/gpu/src/graphcuts.cpp:132:13: error: ‘pState’ was not declared in this scope
	/home/lthpc/opencv/opencv-2.4.11/modules/gpu/src/graphcuts.cpp:132:13: error: ‘nppiGraphcutFree’ was not declared in this scope
	/home/lthpc/opencv/opencv-2.4.11/modules/gpu/src/graphcuts.cpp: In function ‘void cv::gpu::graphcut(cv::gpu::GpuMat&, cv::gpu::GpuMat&, cv::gpu::GpuMat&, cv::gpu::GpuMat&, cv::gpu::GpuMat&, cv::gpu::GpuMat&, cv::gpu::GpuMat&, cv::gpu::Stream&)’:
	/home/lthpc/opencv/opencv-2.4.11/modules/gpu/src/graphcuts.cpp:174:5: error: ‘nppiGraphcutGetSize’ was not declared in this scope
	/home/lthpc/opencv/opencv-2.4.11/modules/gpu/src/graphcuts.cpp:182:61: error: ‘nppiGraphcutInitAlloc’ was not declared in this scope
	/home/lthpc/opencv/opencv-2.4.11/modules/gpu/src/graphcuts.cpp:190:9: error: ‘nppiGraphcut_32s8u’ was not declared in this scope
	/home/lthpc/opencv/opencv-2.4.11/modules/gpu/src/graphcuts.cpp:195:9: error: ‘nppiGraphcut_32f8u’ was not declared in this scope
	/home/lthpc/opencv/opencv-2.4.11/modules/gpu/src/graphcuts.cpp: In function ‘void cv::gpu::graphcut(cv::gpu::GpuMat&, cv::gpu::GpuMat&, cv::gpu::GpuMat&, cv::gpu::GpuMat&, cv::gpu::GpuMat&, cv::gpu::GpuMat&, cv::gpu::GpuMat&, cv::gpu::GpuMat&, cv::gpu::GpuMat&, cv::gpu::GpuMat&, cv::gpu::GpuMat&, cv::gpu::Stream&)’:
	/home/lthpc/opencv/opencv-2.4.11/modules/gpu/src/graphcuts.cpp:246:5: error: ‘nppiGraphcut8GetSize’ was not declared in this scope
	/home/lthpc/opencv/opencv-2.4.11/modules/gpu/src/graphcuts.cpp:254:61: error: ‘nppiGraphcut8InitAlloc’ was not declared in this scope
	/home/lthpc/opencv/opencv-2.4.11/modules/gpu/src/graphcuts.cpp:264:9: error: ‘nppiGraphcut8_32s8u’ was not declared in this scope
	/home/lthpc/opencv/opencv-2.4.11/modules/gpu/src/graphcuts.cpp:271:9: error: ‘nppiGraphcut8_32f8u’ was not declared in this scope
	make[2]: *** [modules/gpu/CMakeFiles/opencv_gpu.dir/src/graphcuts.cpp.o] Error 1
	make[2]: *** Waiting for unfinished jobs....
	make[1]: *** [modules/gpu/CMakeFiles/opencv_gpu.dir/all] Error 2
	make: *** [all] Error 2
[这里](http://blog.csdn.net/caozhantao/article/details/51479172)表示是opencv与cuda8.0的版本兼容问题，更换为16年发布的v2.4.13（或者最新v3.2.0）都可以解决。然后安装编译的库文件到系统，并将opencv库加入到系统的库路径中去

	make install
	echo /usr/local/lib >> /etc/ld.so.conf.d/opencv.conf
	ldconfig
### 可能遇到的其他错误及解决办法

#### [LIBTIFF问题](http://answers.opencv.org/question/35642/libtiff_40-link-errors/)
报错信息如

	/usr/lib/libopencv_highgui.so.2.4: undefined reference to TIFFIsTiled@LIBTIFF_4.0' 1> /usr/lib/libopencv_highgui.so.2.4: undefined reference toTIFFOpen@LIBTIFF_4.0' 1> /usr/lib/libopencv_highgui.so.2.4: undefined reference to TIFFReadEncodedStrip@LIBTIFF_4.0' 1> /usr/lib/libopencv_highgui.so.2.4: undefined reference toTIFFSetField@LIBTIFF_4.0' 1> /usr/lib/libopencv_highgui.so.2.4: undefined reference to TIFFWriteScanline@LIBTIFF_4.0' 1> /usr/lib/libopencv_highgui.so.2.4: undefined reference toTIFFGetField@LIBTIFF_4.0' 1> /usr/lib/libopencv_highgui.so.2.4: undefined reference to TIFFScanlineSize@LIBTIFF_4.0' 1> /usr/lib/libopencv_highgui.so.2.4: undefined reference toTIFFSetWarningHandler@LIBTIFF_4.0' 1> /usr/lib/libopencv_highgui.so.2.4: undefined reference to TIFFSetErrorHandler@LIBTIFF_4.0' 1> /usr/lib/libopencv_highgui.so.2.4: undefined reference toTIFFReadEncodedTile@LIBTIFF_4.0' 1> /usr/lib/libopencv_highgui.so.2.4: undefined reference to TIFFReadRGBATile@LIBTIFF_4.0' 1> /usr/lib/libopencv_highgui.so.2.4: undefined reference toTIFFClose@LIBTIFF_4.0' 1> /usr/lib/libopencv_highgui.so.2.4: undefined reference to TIFFRGBAImageOK@LIBTIFF_4.0' 1> /usr/lib/libopencv_highgui.so.2.4: undefined reference toTIFFReadRGBAStrip@LIBTIFF_4.0'
解决方法：CMAKE命令追加，同时还可以尝试通过apt-get安装/卸载重装libtiff-dev
	
	-DBUILD_TIFF=ON
#### [Eigen/Eigenvalues问题](https://stackoverflow.com/questions/31425454/fatal-error-eigen-eigenvalues-in-ubuntu)
报错信息如

	opencv-xxx/modules/calib3d/src/dls.cpp:11:31: fatal error: Eigen/Eigenvalues: No such file or directory
	xxx
	compilation terminated.
	make[2]: *** [modules/calib3d/CMakeFiles/opencv_calib3d.dir /src/dls.cpp.o] Error 1
	make[1]: *** [modules/calib3d/CMakeFiles/opencv_calib3d.dir/all] Error 2
	make: *** [all] Error 2
这是因为系统对特征值库的引用不对，首先确认已经安装了eigen3，可使用apt-get重装，然后在cmake命令后追加如下内容保证正确引用

	 -D EIGEN_INCLUDE_PATH=/usr/include/eigen3
eigen3的路径可以通过如下方法获得
	
	whereis eigen3
#### ippicv下载龟速
运气好的话，大概5，6分钟能够下载完成，同时也可以自己手动下载
	
	wget https://raw.githubusercontent.com/Itseez/opencv_3rdparty/81a676001ca8075ada498583e4166079e5744668/ippicv/ippicv_linux_20151201.tgz
然后将这个文件放入到opencv源码工程的如下路径中

	3rdparty/ippicv/downloads/linux-808b791a6eac9ed78d32a7666804320e


#### undefined reference

	../../lib/libopencv_videoio.so.3.2.0: undefined reference to `av_packet_unref'
	../../lib/libopencv_videoio.so.3.2.0: undefined reference to `avformat_get_mov_video_tags'
	collect2: error: ld returned 1 exit status
	make[2]: *** [bin/opencv_visualisation] Error 1
	make[1]: *** [apps/visualisation/CMakeFiles/opencv_visualisation.dir/all] Error 2

这个不知道是怎么回事，查了查是属于ffmpeg的功能，尝试重新
	
	ldconfig -v
还有就是
	
	apt-get install libavcodec-dev
	apt-get --purge remove libavcodec-dev
他就莫名其妙地好了。。。或许是库cache有延迟吧。。

#### 卸载opencv的命令
因为opencv的编译过程没有指定prefix，所以卸载好像没有那么方便，可以使用如下命令进行比较彻底的清楚，然后就可以重来了

	rm -r /usr/local/include/opencv2 /usr/local/include/opencv /usr/include/opencv /usr/include/opencv2 /usr/local/share/opencv /usr/local/share/OpenCV /usr/share/opencv /usr/share/OpenCV /usr/local/bin/opencv* /usr/local/lib/libopencv*
## 按照简洁的官网教程继续
转到caffe[官网教程](http://caffe.berkeleyvision.org/install_apt.html)继续，虽然官网教程没有指出要编译安装opencv，并且直接使用apt-get安装了libopencv，但这并不能说明自己编译安装opencv是不必要的。

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
	
	export ANACONDA_HOME="/usr/local/anaconda"
	export PATH="$ANACONDA_HOME/bin:$PATH"
然后验证一下当前系统使用的python

	which python
安装boost和pyCuda，后者需要前者

	conda search boost
	conda install boost
然后克隆一个caffe仓库到本地

	git clone https://github.com/BVLC/caffe.git
修改配置文件Makefile.config，去掉如下语句的注释
	
	USE_CUDNN := 1
	OPENCV_VERSION := 3
注意，因为启用了opencv3，我们需要在Makefile文件中进行相应的修改

	LiBRARIES += opencv_core opencv_highgui opencv_imgproc
在这一行后面追加

	opencv_imgcodecs
如果使用了其他的python（例如anaconda python），修改其中的python指向，文件中都有相关语句，matlab同理，这些通过改变语句注释即可，很方便，如下


	MATLAB_DIR := /usr/local
	MATLAB_DIR := /usr/local/MATLAB/R2014a
	ANACONDA_HOME := /usr/local/anaconda
	PYTHON_INCLUDE := $(ANACONDA_HOME)/include \
                 $(ANACONDA_HOME)/include/python2.7 \
                 $(ANACONDA_HOME)/lib/python2.7/site-packages/numpy/core/include
	PYTHON_LIB := $(ANACONDA_HOME)/lib
	WITH_PYTHON_LAYER := 1
然后就可以启动make了，建议使用多核加速 -j

	make all -j
	make test -j
	make runtest -j
执行runtest报错为
	.build_release/tools/caffe: error while loading shared libraries: libboost_system.so.1.61.0: cannot open shared object file: No such file or directory
执行如下命令，会发现anacoda包含了该库，但是库文件不在系统的库路径当中
	locate libboost_system.so.1.61.0
	/usr/local/anaconda/lib/libboost_system.so.1.61.0
这会很自然地让人想到将这个路径加入到/etc/ld.so.conf.d当中，但这却会有使得系统无法正常启动的风险，原因在如上路径中加入动态链接库有可能引入冲突，导致系统在启动时加载一些库失败，正确的做法是在用户的~/.bashrc或者/etc/profile当中追加

	export LD_LIBRARY_PATH=$ANACONDA_HOME/lib:$LD_LIBRARY_PATH
如果没有这一步，caffe实际上找不到的不只这一个库，这里就显示出了anaconda的便利，当然，配置使用还是要稍稍谨慎一些。测试应该全部通过，此时这个caffe目录就是安装好的caffe包了，我们可以继续编译python及matlab要使用的caffe库文件

	make pycaffe -j
	make matcaffe -j
编译matcaffe出错了，输出为
	
	MEX matlab/+caffe/private/caffe_.cpp
	Building with 'g++'.
	/usr/local/caffe/matlab/+caffe/private/caffe_.cpp: In function ‘void delete_solver(int, mxArray**, int, const mxArray**)’:
	/usr/local/caffe/matlab/+caffe/private/caffe_.cpp:208:3: warning: lambda expressions only available with -std=c++11 or -std=gnu++11 [enabled by default]
	/usr/local/caffe/matlab/+caffe/private/caffe_.cpp:208:4: error: no matching function for call to ‘remove_if(std::vector<boost::shared_ptr<caffe::Solver<float> > >::iterator, std::vector<boost::shared_ptr<caffe::Solver<float> > >::iterator, delete_solver(int, mxArray**, int, const mxArray**)::<lambda(const boost::shared_ptr<caffe::Solver<float> >&)>)’
	/usr/local/caffe/matlab/+caffe/private/caffe_.cpp:208:4: note: candidate is:
	In file included from /usr/include/c++/4.7/algorithm:63:0,
	                 from ./include/caffe/blob.hpp:4,
	                 from ./include/caffe/caffe.hpp:7,
	                 from /usr/local/caffe/matlab/+caffe/private/caffe_.cpp:18:
	/usr/include/c++/4.7/bits/stl_algo.h:1166:5: note: template<class _FIter, class _Predicate> _FIter std::remove_if(_FIter, _FIter, _Predicate)
	/usr/local/caffe/matlab/+caffe/private/caffe_.cpp:208:4: error: template argument for ‘template<class _FIter, class _Predicate> _FIter std::remove_if(_FIter, _FIter, _Predicate)’ uses local type ‘delete_solver(int, mxArray**, int, const mxArray**)::<lambda(const boost::shared_ptr<caffe::Solver<float> >&)>’
	/usr/local/caffe/matlab/+caffe/private/caffe_.cpp:208:4: error:   trying to instantiate ‘template<class _FIter, class _Predicate> _FIter std::remove_if(_FIter, _FIter, _Predicate)’
	/usr/local/caffe/matlab/+caffe/private/caffe_.cpp: In function ‘void delete_net(int, mxArray**, int, const mxArray**)’:
	/usr/local/caffe/matlab/+caffe/private/caffe_.cpp:293:3: warning: lambda expressions only available with -std=c++11 or -std=gnu++11 [enabled by default]
	/usr/local/caffe/matlab/+caffe/private/caffe_.cpp:293:4: error: no matching function for call to ‘remove_if(std::vector<boost::shared_ptr<caffe::Net<float> > >::iterator, std::vector<boost::shared_ptr<caffe::Net<float> > >::iterator, delete_net(int, mxArray**, int, const mxArray**)::<lambda(const boost::shared_ptr<caffe::Net<float> >&)>)’
	/usr/local/caffe/matlab/+caffe/private/caffe_.cpp:293:4: note: candidate is:
	In file included from /usr/include/c++/4.7/algorithm:63:0,
	                 from ./include/caffe/blob.hpp:4,
	                 from ./include/caffe/caffe.hpp:7,
	                 from /usr/local/caffe/matlab/+caffe/private/caffe_.cpp:18:
	/usr/include/c++/4.7/bits/stl_algo.h:1166:5: note: template<class _FIter, class _Predicate> _FIter std::remove_if(_FIter, _FIter, _Predicate)
	/usr/local/caffe/matlab/+caffe/private/caffe_.cpp:293:4: error: template argument for ‘template<class _FIter, class _Predicate> _FIter std::remove_if(_FIter, _FIter, _Predicate)’ uses local type ‘delete_net(int, mxArray**, int, const mxArray**)::<lambda(const boost::shared_ptr<caffe::Net<float> >&)>’
	/usr/local/caffe/matlab/+caffe/private/caffe_.cpp:293:4: error:   trying to instantiate ‘template<class _FIter, class _Predicate> _FIter std::remove_if(_FIter, _FIter, _Predicate)’
	In file included from /usr/local/anaconda/include/boost/filesystem/path_traits.hpp:23:0,
	                 from /usr/local/anaconda/include/boost/filesystem/path.hpp:25,
	                 from /usr/local/anaconda/include/boost/filesystem.hpp:16,
	                 from ./include/caffe/util/io.hpp:4,
	                 from ./include/caffe/caffe.hpp:18,
	                 from /usr/local/caffe/matlab/+caffe/private/caffe_.cpp:18:
	/usr/local/anaconda/include/boost/system/error_code.hpp: At global scope:
	/usr/local/anaconda/include/boost/system/error_code.hpp:221:36: warning: ‘boost::system::posix_category’ defined but not used [-Wunused-variable]
	/usr/local/anaconda/include/boost/system/error_code.hpp:222:36: warning: ‘boost::system::errno_ecat’ defined but not used [-Wunused-variable]
	/usr/local/anaconda/include/boost/system/error_code.hpp:223:36: warning: ‘boost::system::native_ecat’ defined but not used [-Wunused-variable]
	
	make: *** [matlab/+caffe/private/caffe_.mexa64] Error 255
在[这里](http://caffecn.cn/?/question/1113)找到了解决方法，编辑caffe目录下的文件Makefile，添加一行代码
	
	CXXFLAGS += -std=c++11
保存后再次执行，就可以了。为了让python能够使用caffe，需要配置python环境变量，在/etc/profile当中添加如下内容
	
	export CAFFE_HOME=/usr/local/caffe
	export PYTHONPATH=$CAFFE_HOME/python:$PYTHONPATH
此时可以进入python测试一下
	
	python
	import caffe
出现一些找不到依赖的情况，使用conda安装一下应该就可以了

### 其他可能的问题

#### kEmptyString’ is not a member of ‘google::protobuf::internel

可能是系统内存在多个版本的libprotobuf和protobuf-compiler，如果安装了anaconda，一般都包含了这个两个工具，不要再使用apt-get单独安装，卸载后可排除错误

#### fatal error: caffe/proto/caffe.pb.h: No such file or directory  #include "caffe/proto/caffe.pb.h"

caffe目录内缺失了东西，使用protoc生成一下就行，在caffe目录中

	protoc src/caffe/proto/caffe.proto --cpp_out=.
	mkdir include/caffe/proto
	mv src/caffe/proto/caffe.pb.h includ/caffe/proto
然后再重新编译即可

#### 使用draw_net.py报错attributeError:int object has no attribute '_values'

这个有人指出是protobuf的问题，但是实际上很可能是caffe源码自己的bug，对应文件为

	caffe_dir/python/caffe.draw.py
github上从rc4开始，这一文件被更新，但是这个更新全是差评，大家都指出这直接导致了上面的错误，所以一个解决方法是，使用rc3的对应文件替换，这样一般可以解决。

	