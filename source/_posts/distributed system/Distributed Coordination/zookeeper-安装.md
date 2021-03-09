---
title: zookeeper-安装
date: 2021-02-16 09:57:32
tags:

---

# 集群架构

3个zk节点

# 安装第一台zk

## 下载

```
wget https://mirrors.bfsu.edu.cn/apache/zookeeper/zookeeper-3.6.2/apache-zookeeper-3.6.2-bin.tar.gz
```

## 解压到/opt/apps目录下

```shell
tar -xzf apache-zookeeper-3.6.2-bin.tar.gz -C /opt/apps
```

## 创建软链接以屏蔽冗长版本号

```shell
ln -s /opt/apps/zookeeper-3.6.2 /opt/apps/zk
```

## **复制配置文件**

复制 Zookeeper 安装目录下的 conf 目录中的 zoo_sample.cfg 文件，并命名为 zoo.cfg。

## 修改配置文件zoo.cfg

### 新建数据目录

```
dataDir=/opt/data/zookeeper
```

### 创建目录

```
mkdir -p /opt/data/zk
```



## **注册** **bin** **目录**

为了保证zk目录下的命令全局可用，注册到bin目录

```shell
vim /etc/profile

export ZK_HOME=/opt/apps/zk
export PATH=$ZK_HOME/bin/:$PATH

source /etc/profile
```



## **操作** **zk**

```
#开启 zk
zkServer.sh start

#查看状态
zkServer.sh status

#关闭zk
zkServer.sh restart
```

### 

