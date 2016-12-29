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

	git clone --depth=1 https://github.com/deeplearning4j/dl4j-examples.git
 	cd dl4j-examples/
 	mvn clean install

我使用的是idea打开，也会自动解析pom.xml，maven依赖的下载速度即时是挂了代理也要相当一段时间。看了看依赖十几个，到时候集群上还得提供这些jar包才行。本地试着在idea上编译运行了一下，可以跑，但是速度非常慢，呵呵。这里的--depth参数是为了避免git将仓库整个历史都克隆下来。

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

于是打包样例程序的时候，只输出了dl4j-spark-example，然后取巧得将上面提到的依赖放到了集群SPARK_HOME/lib/目录下（注意选择对应的scala版本号）。尝试后发现好不到了。当然这里只一共这一个jar也是不够的。这是因为忽略了链式依赖，每一个依赖不可能将其所有的依赖都打包到自身，所以正确的做法是将这个依赖加入到maven当中，然后idea或者命令行maven都能将所有链式依赖一并下载，然后将集群运行时不能提供的依赖挑出来，一并打包成jar或者放置到集群上。下面为了简化测试过程，仅使用jcommander依赖进行。

2.尝试将所有依赖打包进入spark应用程序

犯了好几次错，总结来说应该像这样：
使用Idea建立maven工程，然后讲dl4j-spark样例中的pom抄过来（依赖项，properties这些），然后令其自动解析，这样所有依赖及其链式依赖都会被下载到本地，打包的时候挑出集群没有的一并加入（Extract方式），这样正常是可以执行的。

3.使用依赖项jcommander.jar包，有下面两种方法

	\--jars /home/tseg/spark-1.5.1-bin-hadoop2.6/extraJars/dl4j_extra/jcommander.jar 
	\--driver-class-path /home/tseg/spark-1.5.1-bin-hadoop2.6/extraJars/dl4j_extra/jcommander.jar
第一个表示了在运行时所有节点都需要的依赖，第二个则表示driver节点需要的依赖。但值得注意的是，这个两个参数并不支持通配符*，所以需要多个jar的时候，需要用逗号隔开多个完整路径，不是很方便。

后来发现另外
dl4j官方提供的spark例子会从网络上下载数据集，所以离线的离线的集群还是不要直接尝试了，否则很久都不会执行的，需要自己编写程序从HDFS读取数据。

4.尝试将依赖jar包直接部署至集群

首先需要导出所有的依赖包，maven工具可以用一个命令实现，在工程目录下输入

	mvn dependency:copy-dependencies -DoutputDirectory=lib
然后就可以在工程下的lib目录找到所有依赖的jar包了。

4.1首先是测试集群上的$SPARK_HOME/lib目录是否可以起作用，首先只是在Driver节点的对应目录添加jcommander.jar，报错找不到类，然后将其余worker节点也放上相应jar包，这会集群有一个节点挂掉了。。将其他的worker也添加jar包之后，依然找不到类jcommander。为了避免这个坏节点的影响，我又在另一个集群上进行了相同测试，结论为，这个方法没有用。


4.3（有问题）尝试一下在hdfs中上传所需jar包，分别使用SparkConf.setJars()与SparkContext.addJar()进行添加，都不起作用。。不管是fat-jar还是原始的jar（测试jcommander得出无效结论);同时也排除了hdfs路径存在问题的可能性，因为写不写ip端口都不行，但是有其他同学实验室这么做的，有待进一步尝试。

4.4（有问题）在$SPARK_HOME/conf/spark-defaults.conf中配置如下属性

	spark.driver.extraJars  xxx.jar:xxx.jars...
	spark.executor.extraJars xxx.jar:xxx.jars...

指定一个所有节点都一样的额外jar包存放目录就可以了，但是需要每个节点都进行配置且保存一份jar包，有一点麻烦，这样不会造成jar包的上传（嗯网上别人是这么说的）。配置文件是可以使用$SPARK_HOME这样的字段进行配置的。结论是，这样配置也是不行的，不管是不是写出每一个jar的具体路径，或是使用通配符，日志中都会报找不到类的异常，而且所有节点都已经保持配置文件一致。

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

接下来有了大反转，使用包括了所有集群缺失依赖的应用jar包，将其在另一个没有作上述修改的集群进行提交，之前未跑通时也会出现类似错误信息，但是现在却不会输出任何错误了（除了一些诡异的container failed）。无解，看来其实一开始解决依赖的问题就可以了。然后试着将另一个集群上已经布置的graphbuilder删掉，虽然还是会出现报错，但是完全不影响正常执行，看来是日志输等级设置导致的差异。

3.应用入口Object中定义了

	var numPossibleFeatures

并使用jcommander进行参数读取，在driver上，这个参数已经按照传入的参数字符串被正确赋值，但是通过map传到executor上就变成0（和设定的默认值相同）了，后来发现只要是类似的静态变量，直接在Rdd的算子中进行引用，都没有办法在worker上获得正确的值，解决办法有下面几种：
* 建立一个复制值的局部变量，或者按照dl4j例子里的那样，将为入口类创建class，然后在将所有过程代买写到其成成员方法里去，再在main中进行调用。
* 将所有Rdd算子过程写成子函数，通过传参的形式为其创建等值局部变量，这样还可以使得入口方法流程更为清晰。

后来看到了这样一种说法

- 序列化保存的时对象的状态，静态变量属于类的状态，所以序列化并不保存静态变量。
- 若基类没有实现序列化接口，子类实现了序列化接口。
序列化时基类对象不会被序列化，反序列化时通过无参构造函数构建基类对象。

这应该算是比较容易遇到的问题了，昨晚学弟就遇到了第二个问题。

4.另外使用jcommander处理入口类参数的时候，注意要在创建相关spark配置及上下文类之前进行，否则输入的参数可能不会生效分（注意代码逻辑哦）。