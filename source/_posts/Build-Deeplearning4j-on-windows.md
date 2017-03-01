---
title: Build Deeplearning4j on windows
date: 2017-01-16 09:47:36
tags:[deepLearning, maven, java]
categories:[Trials]
---
# 记录一下使用maven自己编译deeplearning4j的过程
<!--more-->
由于深度学习模型参数众多（和一般几个到十几个的相比），所以就决定按照其各种的继承关系，利用JCommander进行参数处理，却发现源码中的继承关系定义有误，到聊天室反映之后倒是马上就处理了，见[这里](https://github.com/deeplearning4j/deeplearning4j/pull/2673/files)，但是显然不会很快发布到maven仓库里，想利用还是得自己编译。使用Idea打开查看源码，总是出现一些奇怪的报错，比如成员方法未定义（甚至是构造方法未定义，这要怎么加啊。。），pom里也是有一些错，比如依赖太新而仓库里并没有发布。。那么还是使用maven的命令行进行编译吧。

## deeplearning4j-0.7.3-snapshot编译尝试
首先下载deeplearning4j仓库

	git clone -b master git@github.com:deeplearning4j/deeplearning4j.git

进入源码目录，进行打包

	mvn clean package -DskipTests=true
报错表示javadoc生成失败，那么跳过javadoc生成

	mvn clean package -DskipTests=true -Dmaven.javadoc.skip=true
依然报错，原因是datavec指定的版本太新，maven仓库里也没有，后来编译datavec就会出现nd4j的问题，依然是版本太新找不到依赖。然而nd4j依然不是最开始的地方，首先应该编译libnd4j，这个也是可以在github上找到的

	mvn clone -b master git@github.com:deeplearning4j/libnd4j.git

按照windows编译的说明，首先安装[msys2](https://msys2.github.io/)，完成后运行。可以查看版本号，注意一直使用mingw64.exe

	pacman --version
进行更新（命令命名好奇怪）

	pacman -Sy pacman
完成后关闭msys2，然后再次启动，更新其他部分

	pacman -Syu
重启没msys2再运行
	pacman -Su
再次重启，输入如下命令进行开发环境配置

	pacman -S mingw-w64-x86_64-gcc mingw-w64-x86_64-cmake mingw-w64-x86_64-extra-cmake-modules make pkg-config grep sed gzip tar mingw64/mingw-w64-x86_64-openblas mingw-w64-x86_64-lz4
嗯这个装的好慢，那么在此可以将msys加入到系统的PATH环境变量中，上述脚本完成后，进入libnd4j目录，进行cpu后端编译，步骤参考[这里](https://github.com/deeplearning4j/libnd4j/blob/master/windows.md)

	bash ./buildnativeoperations.sh
同时还可以选择cuda后端编译，但是目前。。不太想编译，直接跳过，那么，将libnd4j的目录加入到环境变量中（感觉其实就是msys2.mingw64的环境变量）
	
	export LIBND4J_HOME=`pwd`
然后克隆nd4j

	git clone -b master git@github.com:deeplearning4j/nd4j.git
并进入到其目录中，注意刚才跳过了cuda后端，故在编译的时候要让maven跳过这一部分

	mvn clean install -DskipTests -Dmaven.javadoc.skip=true -pl '!org.nd4j:nd4j-cuda-7.5,!org.nd4j:nd4j-cuda-7.5-platform,!org.nd4j:nd4j-tests'
呃，提示找不到7.5，懒得仔细查pom了，看了看刚才的指示页面，发现还有一个8.0的字样，于是将上述命令改成了

	mvn clean install -DskipTests -Dmaven.javadoc.skip=true -pl '!org.nd4j:nd4j-cuda-8.0,!org.nd4j:nd4j-cuda-8.0-platform,!org.nd4j:nd4j-tests'
报错消失，但是换了一个报错，执行到nd4j-aeron的时候出错了，不知道为什么，windows下的shell总会出现一些奇怪的乱码，而且中文不会乱码，比如这个时候，最关键的报错信息显示为:

	 ▒▒Ч▒▒Դ▒▒▒а▒: 1.8

## 跳回0.6.0
这要如何是好？是不是因为系统没有1.8的jdk？看了看报错的插件是[maven-compiler-plugin](http://maven.apache.org/plugins/maven-compiler-plugin/)，可以在编译的时候[Setting the -source and -target of the Java Compiler](http://maven.apache.org/plugins/maven-compiler-plugin/examples/set-compiler-source-and-target.html)，看起来像是可以令jdk1.8编译出来的jar包可以在1.7环境下运行，但是，这需要代码中没有使用1.8新特性，而集群上现在还是jdk1.8，所以这是不行的。。仔细想想0.7.3的deeplearning4j还是太新了，反正只是一个继承关系的修改，使用更先前的版本（比如0.7.1，），当然也要避免使用利用了jdk1.8特性的版本。当然这还需要自己检查，一检查就发现这几版deeplearning4j其实还是改动挺大的，结果还是退回到了0.6.0

	git clone -b deeplearning4j-0.6.0 --depth=1  git@github.com:deeplearning4j/deeplearning4j.git
高于这个release都使用了jdk1.8特性，这就比较不好了，需要我们其他运行环境全部升级支持到1.8才可以。另外由于本地没有准备cuda本地库，所以就需要跳过cuda相关的编译，命令如下

	mvn clean package -DskipTests=true -Dmaven.javadoc.skip=true -pl '!org.deeplearning4j:deeplearning4j-cuda-7.5'
其实就是直接忽略了一些模块的效果，如果有多个需要跳过，命令最后面的字符串内用逗号隔开。这个可以后面需要了再重新编译一下。

随后编译成功，因为是本地的jar包，需要发布到本地仓库，才能通过开发工程的pom.xml将其使用上。其实手动改动的只有一个地方，所以需要使用的也只有一个jar包，也就是./deeplearning4j/deeplearning4j-nn-0.6.0.jar，进入这个目录，使用如下命令

	mvn install:install-file \
	-Dfile=deeplearning4j-nn-0.6.0.jar \
	-DgroupId=com.dl \
	-DartifactId=deeplearning4j-nn \
	-Dversion=0.6.0-var \
	-Dpackaging=jar
为了与远程仓库中的依赖相区分，这里指定的是一个自定义groupId。这时在~/.m2/下就可以找到刚才添加的jar包了。

## 修改应用开发工程的依赖
因为是开发的spark应用，所需要引入的依赖为
	
	<groupId>org.deeplearning4j</groupId>
      <artifactId>dl4j-spark_${scala.binary.version}</artifactId>
      <version>${dl4j.version}</version>
然后通过链式依赖关系，deeplearning4j的内容会被加载，这里我们可以让其排除其原本指定的deeplearning4j-nn，而转向使用刚才添加到本地maven仓库的jar包。

	<dependency>
      <groupId>org.deeplearning4j</groupId>
      <artifactId>dl4j-spark_${scala.binary.version}</artifactId>
      <version>${dl4j.version}</version>
      <exclusions>
        <exclusion>
          <artifactId>spark-core_2.10</artifactId>
          <groupId>org.apache.spark</groupId>
        </exclusion>
        <exclusion>
          <artifactId>spark-core_2.11</artifactId>
          <groupId>org.apache.spark</groupId>
        </exclusion>
		<exclusion>
			<artifactId>deeplearning4j-nn</artifactId>
			<groupId>org.deeplearning4j</groupId>
		</exclusion>
      </exclusions>
    </dependency>
然后在依赖中加入刚才添加的自编译deeplearning4j-nn

	<dependency>
		<groupId>com.dl</groupId>
		<artifactId>deeplearning4j-nn</artifactId>
		<version>0.6.0-var</version>
	</dependency>
如果找不到，仔细安装的时候各项字段的内容，容易出小错误。
	
