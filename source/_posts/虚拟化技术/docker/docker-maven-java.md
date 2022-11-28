---
title: docker-maven-build-java
date: 2021-01-11 11:01:27
tags: [vm]
---

# Case1:使用基础镜像maven构建jar包

## 前期准备

- 宿主机没有maven和jdk环境
- java项目是spring-boot,需要使用spring-boot-maven插件，打成可执行的jar包
- 准备同时含有maven和jdk的docker镜像。注意：spring的pom.xml文件中一般会配置JDK的版本，这个版本号要和Maven镜像中的JDK版本一致，否则编译期间会报错；



## 技术方案



跳过宿主机，使用具有Maven和JDK环境的docker容器，构建可执行jar包

注意：docker的仓库maven官方也有相关文档：https://hub.docker.com/_/maven

#### Step1: 在docker images开源仓库，找到同时含有maven环境和jdk环境的;



<img src="/Users/qifei/Documents/blog/source/_posts/虚拟化技术/docker/maven镜像.png" alt="image-20221022170407078" style="zoom:25%;" />

下载images

```
docker pull maven:3.3-jdk-8
```



#### Step2: 启动一个基于step1中的镜像的容器，并且执行mvn build命令

```bash
docker run -it --name mvn001 -v "$PWD":/usr/src/mymaven  -w /usr/src/mymaven maven:3.3-jdk-8 mvn clean package -U -DskipTests
```

以上的命令中有下面几处需要注意：
1. –name mvn001：表示容器名称为mvn001
2. -v "$PWD":/usr/src/mymaven：表示将当前 "$PWD"目录映射到Docker容器的/usr/src/mymaven目录，也就是Spring 工程的目录
3. -w /usr/src/mymaven maven:maven:3.3-jdk-8：表示容器的工作目录为/usr/src/mymaven maven:maven:3.3-jdk-8
4. mvn clean package -U -DskipTests：表示启动容器后在工作目录下执行的命令



![image-20221022184532841](/Users/qifei/Documents/blog/source/_posts/虚拟化技术/docker/构建jar包ing.png)

可以看到 buiding spring boot项目的jar包 

![image-20221022191550102](/Users/qifei/Documents/blog/source/_posts/虚拟化技术/docker/构建jar包done.png)

build jar包 done，目录就是$PWD

#### Step3: 执行jar包



# case2:使用容器化环境允许java程序

## 技术方案

Step1：build images 包含jar包





Step2: 在container中运行jar包
