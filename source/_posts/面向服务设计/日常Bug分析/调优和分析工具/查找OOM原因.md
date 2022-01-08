---
title: 查找OOM原因
date: 2021-04-28 12:01:43
tags:
---

当前的 OOM 进行归因，主要分为下面几类：

- JAVA堆内存超过限制
  - 堆内存，单次分配内存过大
  - 堆内存，多次分配累计过大，如StringBuilder或者Array
  - 堆内存累计分配触顶
- 文件描述符(fd)数目超过限制
- pthread_create
  - 线程数超过限制
  - 虚拟内存不足

其中 `pthread_create` 问题占到了总比例大约在百分之 50，Java 堆内存超限为百分之 40 多，剩下是少量的 fd 数量超限。其中 `pthread_create` 和 fd 数量不足均为 native 内存限制导致的 Java 层崩溃，我们对这部分的内存问题也做了针对性优化，主要包括：





