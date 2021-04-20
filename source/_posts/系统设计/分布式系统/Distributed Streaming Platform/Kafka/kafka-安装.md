---
title: kafka-安装
date: 2021-02-13 17:05:46
tags:

---

# 集群架构

在生产环境中为了防止单点问题，Kafka 都是以集群方式出现的。下面要搭建一个 Kafka集群，包含三个 Kafka 主机，即三个 Broker

# **安装并配置第一台主机**

## **Kafka** 的下载

- 上传并解压。 

  - ```shell
    wget http://mirrors.shuosc.org/apache/kafka/1.0.0/kafka_2.11-1.0.0.tgz #去kafka官网找镜像
    ```

  - 将下载好的 Kafka 压缩包上传至 CentOS 虚拟机，并解压。

    ```shell
    #使用-C命令解压到/opt/apps目录下
    tar -xzf kafka_2.12-2.7.0.tgz -C /opt/apps
    ```

    

- 创建软链接。为了屏蔽版本信息和目录，使得操作简单

  ```shell
  ln -s kafka_2.11/ kafka
  ```

- 修改配置文件。 在 kafka 安装目录下有一个 config/server.properties 文件，修改该文件。

  ```properties
  broker.id=0  #集群中kafka的broker唯一标识，默认0
  listeners=PLAINTEXT://192.168.59.152:9092 #写当前主机IP和端口。broker之间通信用的。不配置也行，kafka会自动配置
  log.dirs=xxx
  num.partitions=1  #消费的主题是要存储在partition中的，如果不设立，那么默认存储在1 partition中
  zookeeper.connect=192.168.10.32:2181  #consumer和broker是zookeeper管理的
  ```

  > 以 kafkaOS1 为母机再克隆两台 Kafka 主机。在克隆完毕后，需要修改 server.properties中的 broker.id、listeners 与 advertised.listeners。id要改为不一样的（+1就行）,listeners同样也不一样（ip改为各自的主机ip）

## **kafka** 的启动与停止 

- 启动 **zookeeper**

  - ```shell
    zkServer.sh start
    ```

- 启动 kafka，

  - 在命令后添加-daemon 参数，可以使 kafka 以守护进程方式启动，即不占用窗口。

  - ```
    bin/kafka-server-start.sh -daemon config/server.properties
    ```

- 停止 **kafka**

  - 先停kafka，再停zk

  - ```
    bin/kafka-server-stop.sh
    ```

# 验证

## kafka是否启动

```shell
jps -lmv #查看程序是否允许，如果没有，可能是因为user权限不足，必须使用root权限查看
ps -ef | grep kafka #若启动成功，一定能看到
```

## kafka服务器防火墙是否关闭

```shell
curl server地址：9092 #如果没反应，关闭服务器防火墙
systemctrl stop firewalld.service
```

> 防火墙不关闭，导致kafka broker与kafka controller之间没办法通信

## kafka是否有消息日志

```shell
ll /tmp/kafka-logs #检查是否有topic-partition文件，如果创建topic的时候partition只有一个，那么在集群中好好找找，只有一个broker有这个文件		
```

> 必须向topic发送过消息，才能有日志。否则，只有zk存储有topic，但是kafka没有存储消息日志