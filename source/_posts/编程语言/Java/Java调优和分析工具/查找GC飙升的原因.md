---
title: 查找GC飙升的原因
date: 2020-11-05 09:24:42
tags: [java, jstack]
---



## 第一步：找到目标进程pid

ps 命令找到对应进程的 pid

```java
ps -ef | grep "进程名称"
```



## 第二步：找到目标

当然我们还是会使用 jstack 来分析问题，但有时候我们可以先确定下 gc 是不是太频繁，使用jstat -gc pid 1000命令来对 gc 分代变化情况进行观察，1000 表示采样间隔(ms)，S0C/S1C、S0U/S1U、EC/EU、OC/OU、MC/MU 分别代表两个 Survivor 区、Eden 区、老年代、元数据区的容量和使用量。YGC/YGT、FGC/FGCT、GCT 则代表 YoungGc、FullGc 的耗时和次数以及总耗时。如果看到 gc 比较频繁，再针对 gc 方面做进一步分析，具体可以参考一下 gc 章节的描述。

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/WwPkUCFX4x4q4SxZeO5N1RicXwYTjxYs9MKs9HzKAziao22GmRcTGHArZ0vdmRianvicN58y2sM2Ne3mhZfb3Vg0oA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

