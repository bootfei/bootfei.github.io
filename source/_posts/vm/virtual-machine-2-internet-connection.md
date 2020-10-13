---
title: virtual machine - 2. internet connection
date: 2020-10-08 17:56:19
tags: virtual machine
---

Ref:

1. https://cnblogs.com/wish123/p/9277995.html



1. NAT模式
   1. 原理:
      1. 虚拟机和主机之间有NAT Engine
      2. 虚拟机发送请求，实际上是传给NAT Engine，由它利用主机进行网络访问
      3. 虚拟机收到的请求,
   2. 特点:
   3. 配置: