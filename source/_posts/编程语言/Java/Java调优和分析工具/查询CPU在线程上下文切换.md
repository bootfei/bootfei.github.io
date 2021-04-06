---
title: 查询CPU在线程上下文切换
date: 2020-11-05 09:24:42
tags: [java, jstack]
---

# [如何查看系统的上下文切换情况](https://cloud.tencent.com/developer/article/1669005?from=information.detail.linux%20%E6%9F%A5%E7%9C%8B%E4%B8%8A%E4%B8%8B%E6%96%87%E5%88%87%E6%8D%A2)

## vmstat 

- 使用 vmstat 这个工具，来查询系统的上下文切换情况
- vmstat 是一个常用的系统性能分析工具，主要用来分析系统的**内存使用情况**，也常用来**分析 CPU 上下文切换和中断的次数**

## 了解 vmstat 输出的参数含义

每隔 2s 输出一次结果

```javascript
vmstat 2
```

![img](https://ask.qcloudimg.com/http-save/7256485/2lw9uksb2i.png?imageView2/2/w/1620)

这里我们只了解必备参数，后面有单独一篇文章展开来讲解 vmstat 命令

#### 参数分析

- **cs（context switch）：**每秒上下文切换的次数
- **in（interrupt）：**每秒中断的次数
- **r（Running or Runnable）：**就绪队列的长度，也就是正在运行和等待 CPU 的进程数
- **b（Blocked）：**处于不可中断睡眠状态的进程数

vmstat 只给出了系统总体的上下文切换情况，如何查看每个进程详细情况？答案是通过 pidstat

## 通过 pidstat 查看进程上下文切换的情况

加上 -w 选项，每 3s 输出一次结果，共输出 3 次

```javascript
pidstat -w 3 3
```

![img](https://ask.qcloudimg.com/http-save/7256485/ai3ti74rxs.png?imageView2/2/w/1620)

#### 结果分析

- **cswch：**每秒自愿上下文切换
- **nvcswch：**每秒非自愿上下文切换的次数

#### 自愿上下文切换

- 进程无法获取所需自愿，导致的上下文切换
- **栗子：**I/O、内存等系统**资源不足**时，就会发生

#### 非自愿上下文切换

- 非自愿上下文切换，则是指进程由于时间片已到等原因，**被系统强制调度**，进而发生的上下文切换
- **栗子：**大量进程都在**争抢 CPU** 时，就容易发生非自愿上下文切换

# 通过栗子去看上下文切换

### 前期准备

- 安装 sysbench：上面有提到了
- 安装 sysstat：参考这篇文章，https://www.cnblogs.com/poloyy/p/13325507.html
- 需要有一个虚拟机，我自己的虚拟机是 4核的哈
- 等下会通过远程连接工具来远程虚拟机，然后需要三个终端均访问我的虚拟机

### sysbench 介绍

- 一个**多线程**的基准测试工具（前面讲的 stress 是多进程）
- 一般用来评估不同系统参数下的数据库负载情况
- 在接下来的案例中，主要是当成一个异常进程来看，作用是**模拟上下文切换过多的问题**

### 空闲系统的上下文切换次数

输入以下命令，每 1 秒输出一次结果，输出 5 次

```javascript
vmstat 1 5
```

![img](https://ask.qcloudimg.com/http-save/7256485/s4t2xa6zmn.png?imageView2/2/w/1620)

#### 结果分析

- 现在的上下文切换次数 **cs** 是 200-300左右，而中断次数 **in** 是 200 左右，r 和 b 都是 0。
- 因为这会儿并没有运行其他任务，所以它们就是空闲系统的上下文切换次数

### 第一个终端运行 sysbench

输入以下命令，以 10 个线程运行 5 分钟的基准测试，模拟多线程切换的问题

```javascript
sysbench --threads=10 --time=300 threads run
```

### 第二个终端通过 vmstat 查看上下文切换

```javascript
vmstat 1
```

![img](https://ask.qcloudimg.com/http-save/7256485/lggkm26f86.png?imageView2/2/w/1620)

#### 结果分析

- **cs 列：**上下文切换次数从之前 200 骤然上升到了 160w+...
- **r 列：**就绪队列的长度最大到 8了，大于我们的 CPU 个数 4，所以会**存在大量的 CPU 竞争**
- **us、sy 列：**两列的 CPU 使用率加起来上升到了 80-90，其中系统 CPU 使用率都是 60%+，说明 **CPU 主要是被内核占用了**
- **in 列：**中断次数已经达到 8w 了...说明**中断处理也是个潜在的问题**

#### 总结下

- 系统的就绪队列过长，也就是正在运行和等待 CPU 的进程数过多，导致了大量的上下文切换，而上下文切换又导致了 CPU 使用率升高
- 一环扣一环的，先有因后有果，别搞乱了顺序

#### 提出疑问

到底是什么进程导致了这些问题呢？

### 第三个终端通过 pidstat 来看进程的上下文切换次数

输入以下命令，-w 输出进程切换指标，-u 输出 CPU 使用情况

```javascript
pidstat -w -u 1
```

![img](https://ask.qcloudimg.com/http-save/7256485/e8z8nx0oqx.png?imageView2/2/w/1620)

#### 结果分析

- sysbench 进程 CPU 使用率很高，已经差不多占用了 4 个 CPU 了
- 但上下文切换次数多主要是其他进程，包括内核线程 kworker
- 貌似所有进程加起来的上下文切换次数也就几百，远不如 vmstat 看到的上百万，咋肥事！

### 分析下为什么上下文切换次数会这么少

- 首先，Linux 调度的基本单位是线程
- sysbench 是模拟线程的调度问题

#### 查看 pidstat 命令的作用

```javascript
man pidstat
```

![img](https://ask.qcloudimg.com/http-save/7256485/mhbes5imwo.png?imageView2/2/w/1620)

有那么一句英文，可以看到，pidstat 默认显示**进程**级别的指标数据

![img](https://ask.qcloudimg.com/http-save/7256485/m2hzvsw3pi.png?imageView2/2/w/1620)

- 然后往下翻，可以看到 **-t** 参数
- 它可以显示与选定任务关联的**线程**的统计信息

#### 第三个终端重新执行 pidstat 命令

```javascript
pidstat -wt 1 10
```

![img](https://ask.qcloudimg.com/http-save/7256485/7jqwie1x0p.png?imageView2/2/w/1620)

#### 结果分析

sysbench 的多个线程的上下文切换次数有非常多，终于找到罪魁祸首了

### 分析为什么中断次数也颇高

前面也说到 in 值达到了 8w，那是什么导致中断次数如此之高呢，接下来瞧一瞧

#### 首先

中断处理，它只发生在内核态，而 pidstat 只是一个**进程的性能分析工具**，并不提供任何关于中断的详细信息

#### 如何查看中断发生的类型

从  /proc/interrupts  这个只读文件中读取

 /proc  实际上是 Linux 的一个虚拟文件系统，用于**内核空间与用户空间之间的通信**

#### 继续在第三个终端执行命令

```javascript
watch -d cat /proc/interrupts
```

![img](https://ask.qcloudimg.com/http-save/7256485/n428w3q0ak.png?imageView2/2/w/1620)

#### 结果分析

- 观察一段时间，可以发现变化速度最快的是**重调度中断**（RES），表示唤醒空闲状态的 CPU 来调度新的任务运行
- 这是**多处理器系统**（SMP）中，调度器用来分散任务到不同 CPU 的机制，通常也被称为**处理器间中断**（Inter-Processor Interrupts， IPI）

#### 总结

中断次数升高还是因为**多任务的调度问题**，和前面线程上下文切换次数的分析结果是一致的

# 每秒上下文切换多少次才算正常？

- 这个数值其实取决于系统本身的 CPU 性能
- 如果系统的上下文切换次数比较稳定，那么**数百到一万以内**，都是正常的
- 但当上下文切换次数超过一万次，或者切换次数出现数量级的增长时，就很可能已经出现了性能问题

## 深入分析

根据上下文切换的类型，具体分析

1. 自愿上下文切换多了，说明进程都在**等待资源**，有可能发生了 I/O 等其他问题
2. 非自愿上下文切换多了，说明进程都在**被强制调度**，也就是都在争抢 CPU，说明 CPU 的确成了瓶颈
3. 中断次数变多了，说明 CPU 被**中断处理程序占用**，还需要通过 /pro/interrupts文件来分析具体的中断类型

## 全文总结-如何查看分析上下文切换

- 通过 **vmstat** 确认**系统**的当前的上下文切换（cs）、中断次数（in）、就绪队列（r）、CPU 使用率（us、sy）
- 若上下文切换次数和 CPU 使用率过高，通过 pidstat 查看是哪个**进程或线程**的切换次数过高，CPU 使用率过高
- 然后确认是自愿上下文切换还是非自愿上下文切换，从而深入分析是否存在其他系统瓶颈问题
- 若中断次数过高，通过 /pro/interrupts分析是哪种中断类型 



