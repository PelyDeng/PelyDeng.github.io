title: MyBatis 源码加中文注释
author: Peilin Deng
tags:
  - MyBatis
categories:
  - 摘抄笔记
date: 2021-08-20 19:35:00
---
# MyBatis如何给源码加中文注释？

我们在看框架源码的时候，如果没有注释，看起来会比较吃力。所以如果能够一边看源码一边自己加中文注释，下次阅读的时候就会轻松很多。

问题是：通过maven下载的jar，查看源码，实际上看到的是经过反编译的class文件，是不能够修改的（提示：file is read only）。

如果把当前maven下载的jar包强行关联到自己下载的源码，又有可能会出现字节码跟源码文件不一致的情况（提示：Library source does not match the bytecode for class），导致debug的时候无法进入代码。

如果要保证源码和字节码一致，最好的办法当然是在本地把下载的源码编译生成jar包，上传到本地maven仓库，再引用这个jar。

以MyBatis为例，如果我们要给MyBatis源码加上中文注释（以IDEA操作为例）：

## 1. 配置Maven
因为需要用Maven打包编译源代码，所以第一步是检查Maven的配置。

第一个是环境变量，需要在系统变量中添加MAVEN_HOME，配置Maven主路径，例如“E:\dev\apache-maven-3.5.4”，确保mvn命令可以使用。

第二个是检查Maven的配置。Maven运行时，默认会使用conf目录下的settings.xml配置，例如：E:\dev\apache-maven-3.5.4\conf\settings.xml。

为了保证下载速度，建议配置成国内的aliyun中央仓库（此处需要自行搜索）。

并且，settings.xml中的localRepository应该和IDEA中打开的项目设置中的Local repository保持一致（例如：E:\repository）。否则项目引入依赖时，无法读取到编译后的jar包。

![](/images/img-156.png)

## 2. 下载编译MyBatis源码
因为MyBatis源码编译依赖parent项目的源码，所以第一步是编译parent项目。
先从git clone两个工程的项目（截止2020年4月，最新版本是3.5.4）。

以在E盘根目录下载为例。
```shell
git clone https://github.com/mybatis/parent
git clone https://github.com/mybatis/mybatis-3
```

打开mybatis-3中的pom.xml文件，查看parent的版本号，例如：
```xml
  <parent>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis-parent</artifactId>
    <version>31</version>
    <relativePath />
  </parent>
```

确定parent版本是31（记住这个数字）。
把mybatis版本号改成自定义的版本号，避免跟官方版本号冲突（加上了-snapshot）：

```xml
  <artifactId>mybatis</artifactId>
  <version>3.5.4-snapshot</version>
  <packaging>jar</packaging>
```

进入parent目录，切换项目分支（不能在默认的master分支中编译），工程名后面的数字就是前面看到的parent版本号。

开始编译parent项目：
```shell
cd parent
git checkout mybatis-parent-31
mvn install
```

![](/images/img-157.png)


接下来编译mybatis工程，进入mybatis-3目录，切换到最新3.5.4分支（不能在默认的master分支中编译）。

```shell
cd ../mybatis-3
git checkout mybatis-3.5.4
mvn clean
mvn install -DskipTests=true -Dmaven.test.skip=true -Dlicense.skip=true
```

![](/images/img-158.png)

编译完毕，本地仓库就会出现一个编译后的jar包，例如：E:\repository\org\mybatis\mybatis\3.5.4-snapshot\mybatis-3.5.4-snapshot.jar

在我们的项目中就可以引入这个jar包了（version是自定义的version）。
```xml
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.5.4-snapshot</version>
</dependency>
```

## 3. 关联jar包到源码

本地编译的jar包已经有了，接下来是把jar包和源码关联起来。

Project Structure —— Libries —— Maven: org.mybatis:mybatis:3.5.4-snapshot —— 在原来的Sources上面点+（加号） —— 选择到下载的源码路径，例如：E:\mybatis-3\src\main\java，点击OK

![](/images/img-159.png)

关联好之后，开始打断点debug，就会进入到本地的源码，可以给本地的源码加上注释了。

![](/images/img-160.png)

## 4. 注意

1、如果之前打开过类的字节码文件，本地可能有缓存，一样会有“Library source does not match the bytecode for class”的提示。

	解决办法：File —— Invalidate Caches and Restart（IDEA会重启）。

2、如果添加注释导致了debug的当前行跟实际行不一致，再把mybatis3工程编译一次即可。















