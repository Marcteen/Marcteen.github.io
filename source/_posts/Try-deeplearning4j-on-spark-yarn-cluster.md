---
title: Try deeplearning4j on spark yarn cluster
date: 2016-11-17 16:52:47
tags: [Deep Learning, Spark]
categories: [Trials]
---
# 尝试在Spark on Yarn集群上运行DeepLearning4j样例程序
Deeplearning4j看起来是一个较为规整的java深度学习工具包，基于jvm实现，考虑了底层线性代数科学运算的问题，使用ND4J作为底层支持，而且有一个较为友好的主页，虽然有一个[测评](https://spark-summit.org/2016/events/which-is-deeper-comparison-of-deep-learning-frameworks-on-spark/)指出其性能和拟合效果都比较不够好。。（怎么会这样呢），还是先拿它开刀好了
<!--more-->

## 准备样例工程目录

	git clone https://github.com/deeplearning4j/dl4j-examples.git
 	cd dl4j-examples/
 	mvn clean install

我使用的是idea打开，也会自动解析pom.xml，maven依赖的下载速度即时是挂了代理也要相当一段时间。看了看依赖十几个，到时候集群上还得提供这些jar包才行。本地试着在idea上编译运行了一下，可以跑，但是速度非常慢，呵呵。

## 如何本机使用样例

代码全是java，看来可以学着怎么使用java开发spark应用程序了。使用的样例是MnistMLPExample，其成员变量使用了@Parameter字段进行注解，像这样

	@Parameter(names = "-useSparkLocal", description = "Use spark local (helper for testing/running without spark submit)", arity = 1)
    private boolean useSparkLocal = true;
然后注释帮助里指出，当使用spark-submit的时候，注意在应用程序参数里加上如下内容，从而避免使用local模式，发挥集群的效力（但现在的情况看起来，我们的环境不能提供任何性能保证，悲伤。）

	-useSparkLocal false
感觉很棒的一个方法，不知道是什么原理，看来还是得好好学一下java基础。

## 尝试进行集群运行
1.尝试使用依赖项jar包

Deeplearning主页指明，如果需要在spark应用中使用dl4j，那么在工程中添加如下依赖

	<dependency>
        <groupId>org.deeplearning4j</groupId>
        <artifactId>dl4j-spark_${scala.binary.version}</artifactId>
        <version>${dl4j.version}</version>
    </dependency>

于是打包样例程序的时候，只输出了dl4j-spark-example，然后取巧得将上面提到的依赖放到了集群SPARK_HOME/lib/目录下（注意选择对应的scala版本号）。然后提交执行，妥妥地找不到类。。

2.尝试将所有依赖打包进入spark应用程序

这里使用idea创建artifact的时候使用from module with dependency项创建，这样打包输出的依赖jar包就会被加上extract前缀，这样的方法在之前有同学成功过。但是提交执行后出现了如下错误

	java.lang.SecurityException: Invalid signature file digest for Manifest main attributes

日志信息并没有让我明白为什么，网上有人指出是log4j依赖版本的问题，但是检查依赖项后发现没有出现版本问题。暂时无解。

3.使用依赖项dl4j-spark_xx.jar包，在提交命令中追加依赖，像下面这样

	spark-submit \
	\--class org.deeplearning4j.mlp.MnistMLPExample \
	\--master yarn \
	\--deploy-mode cluster \
	\--driver-memory 8G \
	\--executor-memory 20G \
	\--num-executors 4 \
	\--jars /home/tseg/users/lc/dl14j-spark_2.10-0.6.0.jar \
	/home/tseg/users/lc/dl4j-examples.jar \
	\-userSparkLocal false
似乎还是不行，依然会有类找不到的问题。
随后发现为了运行这个样例程序，需要的不仅仅是dl14j-spark_2.10-0.6.0.jar，仔细在idea中的代码import以及artifact的模块依赖中检查其依赖项，还包括了dl4j的nn，core（包含谜一样的输入数据的生成过程方法，这个后续开发中应该不会再用到了吧）这些子项目包，还有nd4j，slf4j这些，同时入口函数还使用了jcommander，这些在集群上都是没有的，于是将这些包extracted到应用程序jar中，提交执行就不会再报错了，但是此时的应用程序jar包会很臃肿，有34M，而且提交执行之后集群ApplicationMaster不会有任何输出信息，暂时不知道问题出哪里。

记录一个小问题，使用yarn application -kill时出现了报错，指出jvm达到资源上线，无法再分配资源，这是因为linux默认进程数的限制导致，解决方法是在如下文件中修改即可，可指定hadoop，spark进程限制等等

	/etc/security/limits.d/90.nproc.conf

4.尝试将所有依赖jar添加到集群

首先需要导出所有的依赖包，maven工具可以用一个命令实现，在工程目录下输入

	mvn dependency:copy-dependencies -DoutputDirectory=lib
然后就可以在工程下的lib目录找到所有依赖的jar包了。这里先尝试了是将他们放大集群各节点的$SPARK_HOME/lib/下就能使用。

4.1 首先是通过idea中artifact打成fat-jar，然后传至各节点的对应lib目录，提交执行，失败
，那么为什么实验室集群这个目录下还有其他jar呢？例如jblas.jar。

4.2 检验一下上一条是否由fat-jar引起，刚才报错找不到jcommander相关的类，那么将这个依赖的jar包单独拷贝到各个节点的lib文件夹，提交执行，报错信息没有任何变化。结论是似乎单纯将其放置到lib文件夹并不起作用。

4.3尝试一下在hdfs中上传所需jar包，分别使用SparkConf.setJars()与SparkContext.addJar()进行添加，都不起作用。。不管是fat-jar还是原始的jar（测试jcommander得出无效结论);同时也排除了hdfs路径存在问题的可能性，因为写不写ip端口都不行。

4.4在$SPARK_HOME/conf/spark-defaults.conf中配置如下属性

	spark.driver.extraJars  xxx.jar:xxx.jars...
	spark.executor.extraJars xxx.jar:xxx.jars...

指定一个所有节点都一样的额外jar包存放目录就可以了，但是需要每个节点都进行配置且保存一份jar包，有一点麻烦，这样不会造成jar包的上传。配置文件是可以使用$SPARK_HOME这样的字段进行配置的。

虽然程序成功进入了RUNNING状态，但是ApplicationMaster一直没有更新输出，可能数据加载还是有问题的，改天自己准备数据跑一跑试试。

另外使用jcommander处理入口类参数的时候，注意要在创建相关spark配置及上下文类之前进行，否则输入的参数可能不会生效分（注意代码逻辑哦）。
