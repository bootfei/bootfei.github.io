---
title: kafka-安装
date: 2021-02-13 17:05:46
tags:

---

# 集群架构

在生产环境中为了防止单点问题，Kafka 都是以集群方式出现的。下面要搭建一个 Kafka集群，包含三个 Kafka 主机，即三个 Broker

# **安装并配置第一台主机**

## 上传并解压

将下载好的 Kafka 压缩包上传至 CentOS 虚拟机，并解压。

```shell
wget https://mirrors.tuna.tsinghua.edu.cn/apache/kafka/2.7.0/kafka_2.12-2.7.0.tgz

#使用-C命令解压到/opt/apps目录下
tar -xzf kafka_2.12-2.7.0.tgz -C /opt/apps
```

## **创建软链接**

不想修改文件名，因为有版本号；但是使用命令的时候，不想使用这么长的文件名。所以创建软连接。

## **修改配置文件**

在 kafka 安装目录下有一个 config/server.properties 文件，修改该文件。

### 配置broker id

```config
broker.id=0 # 1,2,3根据集群数量设置
```

### 配置端口号

不配置的话，kafka自动配置

### 配置zk地址

```
zookeeper.connect=
```

### 配置消息日志（不是启动日志）

### 配置topic的Partition

```
num.partition=1 #默认创建topic时，只有一个partition
```

