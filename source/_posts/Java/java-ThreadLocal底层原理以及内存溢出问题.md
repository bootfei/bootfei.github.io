---
title: java ThreadLocal底层原理以及内存溢出问题
date: 2020-10-17 19:52:16
categories: [java,oom]
tags:
---

它山之石可以攻玉

### ref links:

1. https://zhuanlan.zhihu.com/p/129607326



首先从内存模型的角度讨论ThreadLocal的存在价值

1. 多线程可以通过jvm的堆，同时读写共享变量，但是会出现并发问题
2. 为了解决1中的并发问题，可以使用局部变量，因为局部变量会存储在线程中的栈帧中，但是局部变量不能被一个线程中的多个方法共享
3. 为了解决2中的多个方法不能共享局部变量问题，可以使用ThreadLocal



然后讨论ThreadLocal的底层原理

1. ThreadLocal的set(T value)和get()方法，本质上是要获取当前线程的ThreadLocalMap (其实就是该线程的 ThreadLocal[]  数组)
2. jdk使用ThreadLocal的内部类, ThreadLocalMaps对 ThreadLocal[ ]数组进行了封装
3. ThreadLocalMap内部维护了Entry[ ] 数组, Entry是ThreadLocalMap的内部类