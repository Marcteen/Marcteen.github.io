---
title: Using Maven to build an DeepLearning4j on Spark
date: 2016-11-28 10:30:15
tags: [Maven, Scala, DeepLearning4j, idea]
categories: [Tricks]

---
# 在学习使用DeepLearning4j on Spark的同时，练习使用Maven构建工具
<!--more-->
都知道Maven可以直接初始化一个项目工程目录，模板有很多，但是Idea里似乎默认没用内置一个合适的，看起来都很奇怪，因为是Spark Scala工程，所以我采用了下列Archetype

	Archetype Group Id : net.alchim31.maven
	Archetype Artifact Id : scala-archetype-simple
	Archetype Version : 1.6
	//如果下载失败在添加Repository URL试试
	Repository URL : http://github.com/davidB/scala-archetype-simple

然后所需要的依赖在pom.xml中添加即可，这个有一个[网站](http://mvnrepository.com/)可以很方便地查询所需依赖的写法。暂时记录到这里。
当然，我发现这个模板有一个bug，test中有一个JUnitRunner没有导入。

## 然后还遇到了一些奇怪的问题。
1.make工程出现EOF异常

这个没有找到原因，使用invalidate cache不起作用，但是重启电脑后就好了。

2.scala指出RDD对hadoop有不恰当引用。
后来发现是因为spark依赖的scope问题，改为compile就不会报错了，但是这个应该设置为provided，实测命令行mvn编译不会受到这个影响。

## 依赖项的scope
参考[官方文档](http://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html)，这是一个可以控制依赖作用域的依赖项属性字段，也会影响依赖是否会在idea的module-settings的artifact以及library等选项中可见，具体的定义不再赘述，不知是不是由于理解不到位，这个属性并没有想象的那样可以解决从构建到配置集群运行时环境的问题。只简单记录一下：

配置dependency管理插件

	<plugin>
        <artifactId>maven-dependency-plugin</artifactId>
        <configuration>
          <outputDirectory>${project.build.directory}/lib</outputDirectory>
          <excludeTransitive>false</excludeTransitive>
          <stripVersion>true</stripVersion>
          <includeScope>provided</includeScope>
          <excludeGroupIds>
            org.apache.spark,
			xxx,xxx
          </excludeGroupIds>
			<excludeArtifactIds>
            org.apache.spark,xxx,xxx
          </excludeArtifactIds>
        </configuration>
     </plugin>
这里可以直接设置依赖拷贝命令的一些行为，比如是否加入链式依赖（能够保证将所有真正所需依赖拷贝出来），是否有不需要拷贝的项（比如十分确定运行时环境能够提供的依赖。
这样，拷贝工程依赖只需输入
	
	mvn dependency:copy-dependencies
当然，也可以在后面追加参数，像这样
	
	mvn dependency:copy-dependencies -DoutputDirectory=lib -DincludeScope=compile



