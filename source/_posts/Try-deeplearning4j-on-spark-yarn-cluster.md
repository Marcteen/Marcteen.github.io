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

于是打包样例程序的时候，只输出了dl4j-spark-example，然后取巧得将上面提到的依赖放到了集群SPARK_HOME/lib/目录下（注意选择对应的scala版本号）。然后提交执行，妥妥地找不到类，后面解释为什么，还是有很多要学习的地方。

2.尝试将所有依赖打包进入spark应用程序

犯了好几次错，总结来说应该像这样：
使用Idea建立maven工程，然后讲dl4j-spark样例中的pom抄过来（依赖项，properties这些），然后令其自动解析，这样所有依赖及其链式依赖都会被下载到本地，打包的时候挑出集群没有的一并加入（Extract方式），这样正常是可以执行的。

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
后来发现是因为忽略了链式依赖，每一个依赖不可能将其所有的依赖都打包到自身，所以正确的做法是将这个依赖加入到maven当中，然后idea或者命令行maven都能将所有链式依赖一并下载，然后将集群运行时不能提供的依赖挑出来，一并打包成jar或者放置到集群上。另外
dl4j官方提供的spark例子会从网络上下载数据集，所以离线的离线的集群还是不要直接尝试了，否则很久都不会执行的，需要自己编写程序从HDFS读取数据。

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

虽然程序成功进入了RUNNING状态，似乎jcommander能够找到，由于数据下载的问题，实际上spark计算一直没有运行。

另外使用jcommander处理入口类参数的时候，注意要在创建相关spark配置及上下文类之前进行，否则输入的参数可能不会生效分（注意代码逻辑哦）。

##记录一个小问题，关于进程数量限制
使用yarn application -kill时出现了报错，指出jvm达到资源上线，无法再分配资源，这是因为linux默认进程数的限制导致，解决方法是在如下文件中修改即可，可指定hadoop，spark进程限制等等

	/etc/security/limits.d/90.nproc.conf

## 集群执行问题
本质上deeplearning4j-spark本身并不需要修改集群的配置，只需提供依赖即可执行，但是在尝试过程中还是遇到了一些问题。
重写dl4j-spark应用之后，本地使用idea在local模式下可以直接执行，注意运行时依赖需要在idea的module setting-model的dependency选项卡中将需要的依赖设置为compile。
在集群上提交执行后遇到了这些问题

1.graphbuilder目录找不到

这是是reflection抛出的异常，不太明白这个依赖是什么来头。但是查看了服务器目录，driver节点上居然真的有这个目录，但是worker上没有，将目录拷贝到所有节点上之后，异常消失。

2.hadoop contrib目录找不到

依然是reflection抛出的异常，百度后发现这是一个hadoop第三方的调度工具，hadoopo2.x默认不会提供这个模块，但是解决方法不是将这个工具人工引入，而是修改hadoop-env.sh，将下列代码注释掉（这里我修改了所有集群节点上的文件），并重启hadoop

	#Extra Java CLASSPATH elements.  Automatically insert capacity-scheduler.
	for f in $HADOOP_HOME/contrib/capacity-scheduler/*.jar; do
		if [ "$HADOOP_CLASSPATH" ]; then
			export HADOOP_CLASSPATH=$HADOOP_CLASSPATH:$f
		else
			export HADOOP_CLASSPATH=$f
		fi
	done

其实仔细看，这个明明是不存在的，却依然会构成影响。

3.应用入口Object中定义了

	var numPossibleFeatures

并使用jcommander进行参数读取，在driver上，这个参数已经按照传入的参数字符串被正确赋值，但是通过map传到excutor上就变成0（和设定的默认值相同）了，等待发现原因。