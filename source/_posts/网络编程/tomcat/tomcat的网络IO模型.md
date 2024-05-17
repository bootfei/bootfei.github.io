---
title: tomcat的网络IO模型
date: 2024-04-27 00:19:51
tags: [tomcat],[IO]

---



# 背景介绍

io模型是解决高速CPU与低速的外接设备之间数据如何传递

# IO模型

## Unix的IO模型

只比java多一种信号量驱动

## Java的IO模型

对于一个网络 I/O 通信过程，比如网络数据读取，会涉及两个对象，一个是调用这个 I/O 操作的用户线程，另外一个就是操作系统内核。

当用户线程发起 I/O 操作后，网络数据读取操作会经历两个步骤：

- 用户线程等待内核将数据从网卡拷贝到内核空间。
- 内核将数据从内核空间拷贝到用户空间。

各种 I/O 模型的区别就是：它们实现这两个步骤的方式是不一样的。

### 同步阻塞IO

### 同步非阻塞IO

### IO多路复用

### 异步IO



# tomcat网络模型





# 参考文献

1. https://tomcatist.gitee.io/2021/02/02/%E6%B7%B1%E5%85%A5%E6%8B%86%E8%A7%A3Tomcat%E5%92%8CJetty%E4%B9%8B%E8%BF%9E%E6%8E%A5%E5%99%A8/
2. https://blog.csdn.net/chuixue24/article/details/123428143