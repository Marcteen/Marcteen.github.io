---
title: Set up the CaffeonSpark
date: 2017-08-05 20:36:35
tags: [caffe, spark, ubuntu]
categories: [Trials]
---
# 尝试在Ubuntu系统上搭建Spark on Yarn with Caffe集群（无GPU）
<!--more-->
这里采用一个刚刚配置好spark on yarn的ubuntu14系统进行操作。参考了一下caffeonspark仓库里提供的Docker for CPU。首先可以添加一下国内快速的镜像，例如阿里云，然后执行

	apt-get update
接着再dockfile当中列出来的依赖，有一堆出现了安装不成功的问题，也不知道是不是之前的一些其他配置导致的，还是回头转向(这里)[http://blog.csdn.net/sadonmyown/article/details/72781393]，先安装maven
	
	wget http://mirror.bit.edu.cn/apache/maven/maven-3/3.5.0/binaries/apache-maven-3.5.0-bin.tar.gz
	tar -zxvf apache-maven-3.5.0-bin.tar.gz
添加如下环境变量

	export M2_HOME=/home/tseg/apache-maven-3.5.0
	export PATH=$PATH:$M2_HOME/bin
使配置生效
	
	source ~/.bashrc
使用apt-get安装其他依赖

	sudo apt-get install libprotobuf-dev protobuf-compiler
	sudo apt-get install libleveldb-dev
	sudo apt-get install libsnappy-dev
	sudo apt-get install libopencv-dev
	sudo apt-get install libhdf5-serial-dev
	sudo apt-get install --no-install-recommends libboost-all-dev
	sudo apt-get install libopenblas-dev
	sudo apt-get install Python-dev
	sudo apt-get install libgflags-dev
	sudo apt-get install libgoogle-glog-dev
	sudo apt-get install liblmdb-dev
然后修改caffe编译配置

	cd /CaffeOnSpark/caffe-public
	cp Makefile.config.example  Makefile.config
	vim  Makefile.config
反注释cpu only标记，修改blas字段，并添加java include，

	CPU_ONLY := 1
	BLAS :=open
	INCLUDE_DIRS += ${JAVA_HOME}/include
返回caffeonspark根目录，执行编译

	sudo make build
出现报错信息

	python/caffe/_caffe.cpp:10:31: fatal error: numpy/arrayobject.h: No such file or directory
还是依赖的缺失，安装如下项目

	sudo apt-get install python-numpy
然后报错为

	/bin/sh: 1: mvn: not found
似乎是因为sudo的时候会把环境变量覆盖，以前完全都没有意识到，可以零时新建一个alias，令sudo保留当前环境变量信息

	alias sudo="sudo env PATH=$PATH"
重试后报错为

	./include/common.hpp:7:17: fatal error: jni.h: No such file or directory
于是将上面的include dir字段修改为绝对路径

	INCLUDE_DIRS += /usr/lib/jvm/jdk1.7.0_80/include
然后报错又变成了。。
	
	fatal error: jni_md.h: No such file or directory
find了一下发现这个文件就在jdk include的linux文件夹里，真是活久见，如果是直接使用root进行，jdk的include就像最开始那样写一句就够了。。。所以还是建议为root配好环境变量之后，用root来编译，最后把文件夹的权限给原来的用户就好。那么把这个路径也加到MakeFile.config里去。另外，重新尝试编译之前，先执行clean命令比较好

	sudo make clean
否则可能因为文件读写权限的问题导致一些maven goal失败，从而使得build过程不成功，再次重试，编译成功，所有maven的步骤都成功了。但是最后的python测试代码，因为服务器spark版本的问题，访问一个sqlAPI的时候报错，不过感觉应该问题不大。

## 集群运行测试
将完成编译部署的镜像直接复制，配成了spark on yarn集群，参考(官方教程)[https://github.com/yahoo/CaffeOnSpark/wiki/GetStarted_yarn]测试执行情况，其中设置链接库的命令较为关键

	export LD_LIBRARY_PATH=${CAFFE_ON_SPARK}/caffe-public/distribute/lib:${CAFFE_ON_SPARK}/caffe-distri/distribute/lib
这需要在每一个slave节点上执行，否则会出现任务一直不能分配到计算资源的情况（提交后始终处于accepted），估计是spark和yarn做不到把系统链接库配置一块统一分发的地步，毕竟是“本地库”嘛，不然怎么宁愿一直等也不会报出任何明显的错误，不过日志文件里可能有。然后在提交，果然很快就能进入计算状态，可是又出现了新的报错

	17/08/08 20:10:26 ERROR yarn.ApplicationMaster: User class threw exception: java.lang.NullPointerException
	java.lang.NullPointerException
		at scala.collection.mutable.ArrayOps$ofRef$.length$extension(ArrayOps.scala:114)
		at scala.collection.mutable.ArrayOps$ofRef.length(ArrayOps.scala:114)
		at scala.collection.IndexedSeqOptimized$class.foreach(IndexedSeqOptimized.scala:32)
		at scala.collection.mutable.ArrayOps$ofRef.foreach(ArrayOps.scala:108)
		at com.yahoo.ml.caffe.LmdbRDD.localLMDBFile(LmdbRDD.scala:184)
		at com.yahoo.ml.caffe.LmdbRDD.com$yahoo$ml$caffe$LmdbRDD$$openDB(LmdbRDD.scala:201)
		at com.yahoo.ml.caffe.LmdbRDD.getPartitions(LmdbRDD.scala:45)
		at org.apache.spark.rdd.RDD$$anonfun$partitions$2.apply(RDD.scala:239)
		at org.apache.spark.rdd.RDD$$anonfun$partitions$2.apply(RDD.scala:237)
		at scala.Option.getOrElse(Option.scala:120)
		at org.apache.spark.rdd.RDD.partitions(RDD.scala:237)
		at org.apache.spark.rdd.MapPartitionsRDD.getPartitions(MapPartitionsRDD.scala:35)
		at org.apache.spark.rdd.RDD$$anonfun$partitions$2.apply(RDD.scala:239)
		at org.apache.spark.rdd.RDD$$anonfun$partitions$2.apply(RDD.scala:237)
		at scala.Option.getOrElse(Option.scala:120)
		at org.apache.spark.rdd.RDD.partitions(RDD.scala:237)
		at org.apache.spark.SparkContext.runJob(SparkContext.scala:1919)
		at org.apache.spark.rdd.RDD.count(RDD.scala:1121)
		at com.yahoo.ml.caffe.CaffeOnSpark.trainWithValidation(CaffeOnSpark.scala:257)
		at com.yahoo.ml.caffe.CaffeOnSpark$.main(CaffeOnSpark.scala:42)
		at com.yahoo.ml.caffe.CaffeOnSpark.main(CaffeOnSpark.scala)
		at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
		at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
		at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
		at java.lang.reflect.Method.invoke(Method.java:606)
		at org.apache.spark.deploy.yarn.ApplicationMaster$$anon$2.run(ApplicationMaster.scala:525)
也不知道是不是因为spark版本的问题呢，但仔细看了一下提交的命令，完全看不出输入文件路径写在神马地方。。怕是根本没有读到数据。于是照着另一个standalone的执行说明，修改了配置json文件中的路径。然后还是发现了奇怪的问题，spark提交任务后居然一直都无法进入running状态，container似乎就是不能正确反馈回来，最终发现，是防火墙的锅，嗯关上就好了，自己开放端口终归是容易出问题。至此，caffeonspark已经可以启动计算了。

## 集群视频人脸检测

将之前的应用打包之后，提交运行，注意要把libopencv的so文件放到所有物理节点上，并确保其路径处在LD_LIBRARY_PATH当中，参考提交命令如下

	spark-submit --master yarn --deploy-mode cluster \ 	--driver-memory 4G \ 	--executor-memory 4G \ 	--num-executors 2 \ 	--executor-cores 2 \ 	--conf spark.driver.extraLibraryPath="${LD_LIBRARY_PATH}" \ 	--conf spark.executorEnv.LD_LIBRARY_PATH="${LD_LIBRARY_PATH}" \ 	--class com.gyllen.sparkface.VideoFaceRetrieve \ 	/home/tseg/user/lc/face.jar \ 	hdfs:///user/tseg/lc/input
另外opencv需要加载检测器的配置xml文件，建议将这个加载过程放到transformation函数体内，直接填写物理机的本地磁盘绝对路径，只要所有节点上都在相应地方放好文件，是可以顺利读取配置，生成有效的检测器对象的（就算是一个nativeObj）

	
	
       

	

