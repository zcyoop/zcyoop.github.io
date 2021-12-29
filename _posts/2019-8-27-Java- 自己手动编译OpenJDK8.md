---
title:  'Java- 自己手动编译OpenJDK8'
date:  2019-8-27
tags: java
categories: [编程语言,java]
---

[TOC]

#### Ubuntu下载

Ubuntu版本：16.04

下载链接：http://releases.ubuntu.com/16.04/ubuntu-16.04.6-desktop-amd64.iso

虚拟软件：VMWare

#### OpenJDK源码下载

据原文说法，[OpenJDK](http://openjdk.java.net/) 使用Mercurial进行版本管理。另外一个名叫[AdoptOpenJDK project](https://adoptopenjdk.net/about.html).提供了OpenJDK的镜像，可以让我们用git下载。

站点的官网如下：https://adoptopenjdk.net/about.html 

主页上说他们的目标就是：

Provide a reliable source of OpenJDK binaries for all platforms, for the long term future.

据我的使用体验来说，之前编译过一次OpenJDK，各种报错，各种改源码才能编译通过。这次确实编译很顺，代码一句没改。

下载代码（第一次需要安装git）

```bash
git clone --depth 1 -b master https://github.com/AdoptOpenJDK/openjdk-jdk8u.git

```

#### 下载Boot JDK 

编译openJDK任然需要使用JDK来编译

这边使用的Oracle的1.7

链接：https://www.oracle.com/technetwork/java/javase/downloads/java-archive-downloads-javase7-521261.html

下载完后解压，然后配置环境变量

```bash
export JAVA_HOME=/usr/local/jdk1.7.0_80(替换成自己的jdk路径)
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=${JAVA_HOME}/bin:$PATH
```

#### 安装依赖

```bash
sudo apt install \
        libx11-dev \
        libxext-dev \
        libxrender-dev \
        libxtst-dev \
        libxt-dev \
        libcups2-dev \
        libfreetype6-dev \
        libasound2-dev \
        libfontconfig1-dev
```

#### 配置编译脚本

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/2019-8-27/jdk1.png)

解压下载需要编译的openJDK，并进入其解压后的路径

build.sh

```bash
bash ./configure --with-target-bits=64 --with-boot-jdk=/usr/local/jdk1.7.0_80/ --with-debug-level=slowdebug --enable-debug-symbols ZIP_DEBUGINFO_FILES=0
make all ZIP_DEBUGINFO_FILES=0
```

运行build.sh

```bash
chmod +x build.sh
./build.sh
```

#### 编译成功

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/2019-8-27/jdk2.png)

切换到指定路径下查看编译后的结果

```
cd ~/jdk/openjdk-jdk8u/build/linux-x86_64-normal-server-slowdebug/jdk/bin
./java -version
```

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/2019-8-27/jdk3.png)

**参考**

- [源码编译OpenJdk 8，Netbeans调试Java原子类在JVM中的实现（Ubuntu 16.04）](https://www.cnblogs.com/grey-wolf/p/10971741.html#_label1_0)

