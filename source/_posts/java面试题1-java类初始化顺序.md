---
title: 'java面试题1: java类初始化顺序'
date: 2020-10-19 23:30:35
tags: java面试题
Categories: java
---

题目汇总：https://www.zhihu.com/question/60949531



ref

1. https://developer.ibm.com/zh/articles/j-lo-clobj-init/
2. https://zhuanlan.zhihu.com/p/65872513
3. https://blog.nowcoder.net/n/a9a1c0cf7dc841c496769865b91a3b05
4. http://baijiahao.baidu.com/s?id=1626608229788927963



类初始化过程以及该过程的意义：

1. 对于类和接口，初始化就是调用初始化方法；
2. java虚拟机层面：
3. java虚拟机是支持多线程的，所以需要处理同步
   1. 同步就需要互斥，需要锁LC; C和LC的映射关系，有jvm自行决定
   2. 如果C的Class对象显示当前C的初始化由其他线程正在执行 -> 当前线程释放LC并进入阻塞，直等到其他线程完成初始化
   3. 如果C的Class对象显示当前C的初始化由当前线程正在执行 -> do nothing, release LC





bootfei's answer:

1. 类初始化是类加载过程中的最后一步，也是真正开始调用java的字节码文件
2. 类初始化首先区分是针对接口还是类
   1. 如果是类，会首先调用父类的<cinit>,然后调用子类的<cinit>
   2. 如果是接口，因为接口是没有静态代码块的，如果没有静态变量，则不会调用父接口的<cinit>
3. 类初始化的顺序是
   1. 静态变量
   2. 静态代码块，并且按照文本顺序执行，其中不能访问其他静态变量，只能赋值