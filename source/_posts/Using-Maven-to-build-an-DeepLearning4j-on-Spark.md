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

## 使用maven build算法工程

idea与maven混用的感觉并不是很好，因为依赖管理千差万别（或许吧），而且spark官网对于应用的第三方依赖解决方案并没有一个特别友好的方式（必须一个一个添加？），大概是我们项目没有构建也没有一个标准的模块化的原因吧，于是还是决定试一试使用maven，直接通过pom的配置来进行jar的生成，idea里面逐个手动添加的方式有点累。

1. 那么，按照dl4j官方例子的pom抄过来

		<dependencies>
		    <dependency>
		      <groupId>org.scala-lang</groupId>
		      <artifactId>scala-library</artifactId>
		      <version>${scala.version}</version>
		    </dependency>
	    
		    <dependency>
		      <groupId>org.nd4j</groupId>
		      <artifactId>nd4j-native-platform</artifactId>
		      <version>${nd4j.version}</version>
		    </dependency>
		
		    <dependency>
		      <groupId>org.deeplearning4j</groupId>
		      <artifactId>dl4j-spark_${scala.binary.version}</artifactId>
		      <version>${dl4j.version}</version>
		      <exclusions>
		        <exclusion>
		          <artifactId>spark-core_2.10</artifactId>
		          <groupId>org.apache.spark</groupId>
		        </exclusion>
		      </exclusions>
		      <scope>provided</scope>
		    </dependency>
		
		    <dependency>
		      <groupId>org.apache.spark</groupId>
		      <artifactId>spark-core_${scala.binary.version}</artifactId>
		      <version>${spark.version}</version>
		      <exclusions>
		        <exclusion>
		          <groupId>javax.servlet</groupId>
		          <artifactId>servlet-api</artifactId>
		        </exclusion>
		      </exclusions>
		      <scope>${spark.scope}</scope>
		    </dependency>
		
		
		    <dependency>
		      <groupId>com.beust</groupId>
		      <artifactId>jcommander</artifactId>
		      <version>${jcommander.version}</version>
		      <scope>provided</scope>
		    </dependency>
		</dependencies>
可以看到，将spark依赖设置为provided了，表明由运行时环境提供，也就是spark集群。

2. 使用maven打jar包

随便看了一下网上相关的内容，打包也有好几种方式，assembly，或者引入maven-shade-plugin??不是很明白，先找了个最简单的来，好像没有很特别的配置，在工程目录输入如下命令

	mvn clean package

下载了一些东西，然后报错了。。看起来是因为把测试的依赖删掉了，但是测试的代码并没有删掉，那好吧先把测试的依赖加回来。在此执行，注意到一些scala版本依赖的警告，似乎是要求好几种2.10.x，不知道会有致命问题吗？

然后又报错了，好像自动执行了TEST,然后测试没有通过，拜托我代码都没动，那就跳过测试好了。输入如下命令

	mvn clean package -DskipTests
果然成功了，打好的jar包存放于 

	ProjectDir/target/xxx.jar
看了一下大小，只有17K。。查看一下包含的内容，并没有把其他依赖包括进行。。这应该是需要配置插件才可以进行的。

3. 将外部依赖打入jar包

参看了：[这里](http://www.cnblogs.com/xinsheng/p/4109573.html),有点不太明白
又看了[这里](http://blog.csdn.net/czp11210/article/details/47808211)，简单一些，试试看吧，在pom添加插件配置如下
还有[一个](http://blog.csdn.net/cdl2008sky/article/details/6756177)备用的

	<build>  
        <plugins>  
            <plugin>  
                <artifactId>maven-assembly-plugin</artifactId>  
                <configuration>  
                    <archive>  
                        <manifest>  
                            <mainClass>com.allen.capturewebdata.Main</mainClass>  
                        </manifest>  
                    </archive>  
                    <descriptorRefs>  
                        <descriptorRef>jar-with-dependencies</descriptorRef>  
                    </descriptorRefs>  
                </configuration>  
            </plugin>  
        </plugins>  
    </build>
然后使用命令

	mvn assembly:assembly —DskipTests

然后报错噜，因为没有回到工程根目录，返回后重新执行，成功打包，看了看大小，118M。。

显然没有idea下手动挑的那样集约，那么看一下依赖树，使用命令

	mvn dependecy:tree

发现稍微仔细看一看，也还是比较清晰明了的，看了看发现自己的pom写的还是有问题，比如从dl4j-spark排除的spark-core重新引入后scope写成了complie，实际应该写成provided。另外个这个命令还可以方便地看出dl4j其他链式依赖的scope值。而dl4j-spark的scope写成了provided，显然应该作为compile，当然jcommander也相同，因为集群上没有。仔细按需检查修改之后，重新打包，jar包大小变为了263M。。。

很奇怪为何引入的东西变多了？检查了一下hadoop都完整包含，应该是自己加入的spark依赖有问题，不应该是provided？那刚才的118M是怎么来的。。

查阅一番发现这个问题并不是那么简单，[这里](http://stackoverflow.com/questions/1459021/excluding-provided-dependencies-from-maven-assembly)指出了一种解决方案，时间还是挺久远的，语法可能都变了。。看了半天，决定好好了解一下assembly，[主页](http://maven.apache.org/plugins/maven-assembly-plugin/),总体来说就是，
1. 对maven-dependency-plugins进行配置，这里能够实现scope的过滤，这个配置比较简单，简单列出（按照plugins主页修改，后来发现前面stackoverflow上的写法也没有过时嘛
	
<plugin>
<artifactId>maven-dependency-plugin</artifactId>
<version>${dependency.plugin.version}</version>
<executions>
<execution>
<id>copy-dependencies</id>
<phase>process-resources</phase>
<goals>
<goal>copy-dependencies</goal>
</goals>
<configuration>
<outputDirectory>${project.build.directory}/lib</outputDirectory>
<excludeTransitive>false</excludeTransitive>
<stripVersion>true</stripVersion>
<excludeScope>provided</excludeScope>
</configuration>
</execution>
</executions>
</plugin>
2. 				为工程配置assembly.xml指定要打包的jar来源目录，使其指向maven-dependency-plugins的输出目录即可，这需要自定义assembly描述符文件，按照plugins主页的方式，新建文件
	
	src/assembly/src.xml

输入一下内容并保存

	<assembly xmlns="http://maven.apache.org/ASSEMBLY/2.0.0"
	    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	    xsi:schemaLocation="http://maven.apache.org/ASSEMBLY/2.0.0 http://maven.apache.org/xsd/assembly-2.0.0.xsd">
	  <id>distribution</id>
	  <formats>
	    <format>jar</format>
	  </formats>
	  <fileSets>
	    <fileSet>
	      <directory>${project.basedir}/lib</directory>
	      <outputDirectory>${project.basedir}/target</outputDirectory>
	      <includes>
        	<include>*.*</include>
		  </includes>
	    </fileSet>
	  </fileSets>
	</assembly>

然后assembly plugins的插件字段配置为

	<plugin>
        <artifactId>maven-assembly-plugin</artifactId>
        <version>${assembly.plugins.version}</version>
		<executions>
            <execution>
              <id>make-assembly</id> <!-- this is used for inheritance merges -->
              <phase>package</phase> <!-- bind to the packaging phase -->
              <goals>
                <goal>single</goal>
              </goals>
            </execution>
        </executions>
        <configuration>
          <!--<archive>
            <manifest>
              <mainClass>com.allen.capturewebdata.Main</mainClass>
            </manifest>
          </archive>-->
          <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
          </descriptorRefs>
          <descriptors>
            <descriptor>src/assembly/src.xml</descriptor>
          </descriptors>
        </configuration>
      </plugin>

这个实际上是将assembly绑定到了package上一同执行,后来发现字段位置写错了，果然还是用ide来写会比较好，这样容易发现一些。注意上面的descriptorRefs和descriptort是并列关系，两个都写的话，就都会生成，可以按需要取舍。
然后在工程根目录输入如下命令

	mvn clean package -DskipTests

然后通过查看依赖树，将所有集群上已经有的依赖在executions copy-dependencies中进行了过滤，然后打包，发现这个包里的内容全是jar，上传集群，然后在spark-submite命令中加入依赖，并不能找到了类，而这个类确实存在于这个jar中了。思考是不是应该先把jar进行unpack，然后再打入总jar，但是发现descriptor中fileSets内并没有提供unpack字样，dependencySets才有，那好吧，单独配一个dependency的descriptor是不是就可以不通过package进行了，于是决定使用dependencySets尝试进行打jar包的配置，而且这个里面也提供了一些过滤的设置字段。

嗯，为了批量将原来的groupID生成符合的exclude形式，写了一个sh

	#!/bin/sh

	while read line
	do
		echo -e '<exclude>'${line/,/}'：*</exclude>' >> $2 # 文件$1中每行去掉逗号之后，用exclude标志框起来，增加通配，然后追加到文件$2中去
	done < $1
最后我把descriptor写成了下面的样子，过滤还是可以的，虽然没有比dependency-plugins更先进。

	<assembly xmlns="http://maven.apache.org/ASSEMBLY/3.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/ASSEMBLY/3.0.0 http://maven.apache.org/xsd/assembly-3.0.0.xsd">
	  <id>distribution</id>
	  <formats>
	    <format>jar</format>
	  </formats>
	  <includeBaseDirectory></includeBaseDirectory>
	  <baseDirectory>/</baseDirectory>
	  <dependencySets>
	    <dependencySet>
	      <useProjectArtifact>false</useProjectArtifact> <!-->Whether include the project's own artifact<-->
	      <outputDirectory>${project.build.directory}</outputDirectory>
	      <unpack>true</unpack>
	      <excludes>
	        <exclude>org.apache.spark:*</exclude>
	        <exclude>org.apache.commons:*</exclude>
	        <exclude>org.scalanlp:*</exclude>
	        <exclude>org.scalatest:*</exclude>
	        <exclude>org.specs2:*</exclude>
	        <exclude>org.slf4j:*</exclude>
	        <exclude>org.xerial.snappy:*</exclude>
	        <exclude>org.spire-math:*</exclude>
	        <exclude>org.spark-project:*</exclude>
	        <exclude>org.tukaani:*</exclude>
	        <exclude>org.scalamacros:*</exclude>
	        <exclude>org.jpmml:*</exclude>
	        <exclude>org.apache.parquet:*</exclude>
	        <exclude>net.sf.opencsv:*</exclude>
	        <exclude>com.github.rwl:*</exclude>
	        <exclude>com.sun.jersey:*</exclude>
	        <exclude>com.sun.xml.bind:*</exclude>
	        <exclude>org.codehaus:*</exclude>
	        <exclude>org.codehaus.jackson:*</exclude>
	        <exclude>org.hamcrest:*</exclude>
	        <exclude>org.codehaus.janino:*</exclude>
	        <exclude>commons-codec:*</exclude>
	        <exclude>org.scalaz:*</exclude>
	        <exclude>*:commons.io</exclude>
	        <exclude>*:junit</exclude>
	        <exclude>*:datavec-api</exclude>
	        <exclude>*:datavec-data-image</exclude>
	        <exclude>*:jackson-databind</exclude>
	        <exclude>*:jsr305</exclude>
	        <exclude>*:ffmpeg-linux-ppc64le</exclude>
	        <exclude>*:ffmpeg-macosx-x86_64</exclude>
	        <exclude>*:ffmpeg-windows-x86_64</exclude>
	        <exclude>*:leptonica-windows-x86_64</exclude>
	        <exclude>*:leptonica-macosx-x86_64</exclude>
	      </excludes>
	    </dependencySet>
	  </dependencySets>
	</assembly>

然后输入命令

	mvn clean assembly:single(为何一定要使用goal的名字，总觉得execution—id更合适。)
可以成功打包，但是包里面的目录结构和刚才一样很奇怪，多了一系列project的绝对路径前缀，而且并不能够使得spark程序通过--jars找到类。
那么遗留了两个问题，怎么去掉那个奇怪的前缀路径？为什么还是找不到类，是不是没有像idea里面那样extracted的效果？综合来看，现在还是把依赖都通过idea打到应用jar包里算了。Maven算是入了一点点门，疼死了。

