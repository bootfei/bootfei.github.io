---
title: 查找CPU飙升的原因
date: 2020-11-05 09:24:42
tags: [java, jstack]
---



## 第一步：找到耗费CPU的进程

首先，因为CPU飙升了，所以要查看CPU的相关信息，所以需要使用TOP命令

<img src="/Users/qifei/Library/Application Support/typora-user-images/image-20201202174921436.png" alt="image-20201202174921436" style="zoom:50%;" />

## 第二步：找到耗费CPU的线程（10进制）

根据1st step中的返回内容，看到进程号116664的CPU很高。所以，需要进一步锁定该进程内部的线程耗费CPU, 所以需要使用top -H -p [pid]命令查看线程， (也可使用shift -h进行切换)

![image-20201202175704489](/Users/qifei/Library/Application Support/typora-user-images/image-20201202175704489.png)



## 第三步：找到耗费CPU的线程（16进制）

根据2nd step中的返回内容，可以看到线程号117296(10进制)的java线程耗费CPU，所以需要查看该线程的详细信息，所以需要是一个jstack命令，但是，记住jstack中的nid（Native Thread ID)是系统线程id, 为16进制，需要使用top -Hp pid找到该线程的10进制pid，然后使用下边的命令打印出16进制线程nid

![image-20201202180132749](/Users/qifei/Library/Application Support/typora-user-images/image-20201202180132749.png)

```shell
  printf  "%x\n" 10进制nid
```

## 第四步：打印CPU的线程(16进制)栈信息

根据3rd step中的16进制线程号，使用 <!--注意：linux下进程和线程都用10进制pid表示-->

```
jstack pid | grep [nid]
```

命令，查询该线程的详细信息

![image-20201202180409819](/Users/qifei/Library/Application Support/typora-user-images/image-20201202180409819.png)

## 第五步：分析栈信息中的线程状态

当然更常见的是我们对整个 jstack 文件进行分析，通常我们会比较关注 WAITING 和 TIMED_WAITING 的部分，BLOCKED 就不用说了。我们可以使用命令

```shell
cat jstack.log | grep "java.lang.Thread.State" | sort -nr | uniq -c
```

来对 jstack 的状态有一个整体的把握，如果 WAITING 之类的特别多，那么多半是有问题啦。

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/WwPkUCFX4x4q4SxZeO5N1RicXwYTjxYs9zpsDC09P5ww0K7GTAYZbhxfc6VfyucR5Lf7TGY2mbfBN14UicSbOPIQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)





