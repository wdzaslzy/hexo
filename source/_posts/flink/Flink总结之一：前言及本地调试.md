---
title: Flink总结之一：前言及本地调试
date: 2022-07-07 10:27:08
tags: [flink]
categories: Flink总结
description: 使用Flink已有三年，虽然在我们真实的业务场景下，使用到Flink的特性并不多，但不能代表未来不会使用到其它的特性。为了保证自己的知识库，所以打算写一下Flink相关的总结，以便日后翻阅。
top: 6
---

使用Flink已有三年，虽然在我们真实的业务场景下，使用到Flink的特性并不多，但不能代表未来不会使用到其它的特性。为了保证自己的知识库，所以打算写一下Flink相关的总结，以便日后翻阅。

近些年来，流计算技术发展迅速，被广泛应用于数据ETL、数据BI、实时数据仓库建设和AI等方面。Flink是继Spark之后的一个更优秀的计算引擎，自问世以来发展迅猛，其技术生态圈也日益壮大，现已成为Apache顶级开源项目中最活跃的项目之一。在国内，很多企业选择用Flink来构建其流计算体系或流批一体体系，使用on YARN或on Kubernetes部署模式来进行大规模生产。

目前，Flink已经发布1.15版本。三年前使用Flink时，还只用的1.10版。中间也变化了很多。

此文是基于Flink1.15版本的说明。后续Flink升级的新特性，会在Flink发布时补充。



## Flink环境准备

在深入理解Flink前，首先要做的一件事是，本地能够调试。因为很多Flink的功能，单纯的看源码相当费劲，但是在本地debug时，就会容易很多。



### 获取与导入Flink源代码

**第一步**

在github上下载[Flink源码](https://github.com/apache/flink)。建议使用git clone方式下载。因为可以任意切换分支。

```sh
git clone git@github.com:apache/flink.git
```



**第二步**

导入Flink源码到IDEA中。

如果想对Flink进行二次开发，或者想为开源社区贡献代码，可以配置CheckStyle。如果只是单纯的学习，则可以跳过该步骤。

在Flink 1.12时，我们有个需求：可以配置k8s上，cpu和内存的上限。当时的flink还不支持该feature。因此，我们对它进行了二次开发。配置CheckStyle方法如下：

IntelliJ IDEA通过CheckStyle-IDEA插件来支持CheckStyle。

1. 在IntelliJ IDEA的Plugins Marketplace中查找并安装CheckStyle-IDEA插件。

2. 依次选择Settings→Tools→Checkstyle并设置checkstyle。

3. 将Scan Scope设置为Only Java sources（including tests）。

4. 在Checkstyle version下拉列表中选择checkstyle版本，并单击Apply按钮。（注：官方推荐版本为8.20。）

5. 在Configuration File面板中单击“+”图标添加新配置：

   在弹窗中将Description设置为Flink；

   选中Use a local Checkstyle file，并选择Flink源代码目录下的tools/maven/checkstyle.xml文件；

   勾选Store relative to project location选项，单击Next按钮；

   将checkstyle.suppressions.file的属性设置值为suppressions.xml，单击Next按钮即完成配置。

6. 勾选刚刚添加的新配置Flink，以将其设置为活跃的配置，依次单击Apply和OK按钮，即完成Java部分CheckStyle的配置。若源代码违反CheckStyle规范，CheckStyle会给出警告。



### 编译Flink源代码

在构建源码之前，如果有特定的Flink版本，只需要切换到对应的分支即可。我这边使用Flink 1.15版本。

Flink源代码的编译与构建会因Maven版本的不同而有所差异。对于Maven 3.0.x版本、3.1.x版本、3.2.x版本，可以采用简单构建Flink的方式，在Flink源代码的根目录下运行以下命令。

```sh
$ mvn clean install -DskipTests
```

而对于Maven 3.3.x及以上版本，则要相对麻烦一点，在Flink源代码的根目录下运行下面的命令。

```sh
$ mvn clean install -DskipTests
$ cd flink-dist
$ mvn clean install
```

如果想快速的构建Flink源代码，只需要加上-Dfast即可。它会跳过测试、QA插件、Java docs。

除此之外，需要设置scala版本。推荐2.12。为了加快编译速度，可以设置多线程编译。

最终，在maven 3.3以上版本下，编译方式：

```sh
$ mvn clean install -DskipTests -Dfast -T 4 -D maven.compile.fork=true -D scala-2.12
```



PS：Window下，flink会下载一个nodeJs的插件。64位windows下无法执行。解决方式是，自己从[nodejs官网](https://nodejs.org/zh-cn/download/)下载一个64位版本的，覆盖它。或修改flink-ruuntime-web下的pom文件，修改node版本，并替换下载地址：

```xml
<configuration>
    <nodeVersion>v16.15.1</nodeVersion>
    <npmVersion>8.1.2</npmVersion>
    <downloadRoot>http://npm.taobao.org/mirrors/node/</downloadRoot>
</configuration>
```

再失败，则把该插件注释掉。

### 本地调试

Flink的examples包下，自带了一些案例，可以通过运行这些案例，来调试。



