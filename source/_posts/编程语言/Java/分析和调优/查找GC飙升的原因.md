---
title: 查找GC飙升的原因
date: 2020-11-05 09:24:42
tags: [java, jstack]
---





ps 命令找到对应进程的 pid(10进制) <!--nid是16进制-->

```java
ps -ef | grep "进程名称pid"
```



```
jstat -gc [pid] 1000
```

命令来对 gc 分代变化情况进行观察，1000 表示采样间隔(ms)，

S0C/S1C、S0U/S1U、EC/EU、OC/OU、MC/MU 分别代表两个 Survivor 区、Eden 区、老年代、元数据区的容量和使用量。

YGC/YGT、FGC/FGCT、GCT 则代表 YoungGc、FullGc 的耗时和次数以及总耗时。

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/WwPkUCFX4x4q4SxZeO5N1RicXwYTjxYs9MKs9HzKAziao22GmRcTGHArZ0vdmRianvicN58y2sM2Ne3mhZfb3Vg0oA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



> 如果YoungGC频率很高，因为大多数对象都是朝生夕死，所以要让他们在YongGC之前就死亡，那么应该增加Eden区的容量，减少YongGC发生的时间间隔





### [YGC 频繁](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

直接查看 gc log 并不直观，我们可以借用一些可视化工具来帮助我们分析， `[gceasy](https://gceasy.io/)` 是个挺不错的网站，我们把 gc log 上传上去后， gceasy 可以帮助我们生成各个维度的图表帮助分析。

查看 gceasy 生成的报告，发现我们服务的 gc 吞吐量是 95%，它指的是 JVM 运行业务代码的时长占 JVM 总运行时长的比例，这个比例确实有些低了，运行 100 分钟就有 5 分钟在执行 gc。幸好这些 GC 中绝大多数都是 YGC，单次时长可控且分布平均，这使得我们服务还能平稳运行。

解决这个问题要么是减少对象的创建，要么就增大 young 区。前者不是一时半会儿都解决的，需要查找代码里可能有问题的点，分步优化。

而后者虽然改一下配置就行，但以我们对 GC 最直观的印象来说，增大 young 区，YGC 的时长也会迅速增大。

其实这点不必太过担心，我们知道 YGC 的耗时是由 `GC 标记 + GC 复制` 组成的，相对于 GC 复制，GC 标记是非常快的。而 young 区内大多数对象的生命周期都非常短，如果将 young 区增大一倍，GC 标记的时长会提升一倍，但到 GC 发生时被标记的对象大部分已经死亡， GC 复制的时长肯定不会提升一倍，所以我们可以放心增大 young 区大小。

### [分代调整](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

除了 GC 太频繁之外，GC 后各分代的平均大小也需要调整。

![Image](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZffvuL0XJJboo4ouy9mNX1iaUdW1Fu9hNnP8UgA5Fy5Me51KgFxx16AQfe4tfEG2icI43eicHCn6kHH0A/640)

我们知道 GC 的提升机制，每次 GC 后，JVM 存活代数大于 `MaxTenuringThreshold` 的对象提升到老年代。当然，JVM 还有动态年龄计算的规则：按照年龄从小到大对其所占用的大小进行累积，当累积的某个年龄大小超过了 survivor 区的一半时，取这个年龄和 MaxTenuringThreshold 中更小的一个值，作为新的晋升年龄阈值，但看各代总的内存大小，是达不到 survivor 区的一半的。

![Image](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZffvuL0XJJboo4ouy9mNX1iaUtjnuXmDx8BicpSpteU9T1XJUwwtZn8zCXgSlzoAIzaiaWG3whgj5h2hA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)图片

所以这十五个分代内的对象会一直在两个 survivor 区之间来回复制，再观察各分代的平均大小，可以看到，四代以上的对象已经有一半都会保留到老年区了，所以可以将这些对象直接提升到老年代，以减少对象在两个 survivor 区之间复制的性能开销。

所以我把 MaxTenuringThreshold 的值调整为 4，将存活超过四代的对象直接提升到老年代。

### [偏向锁停顿](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

还有一个问题是 gc log 里有很多 18ms 左右的停顿，有时候连续有十多条，虽然每次停顿时长不长，但连续多次累积的时间也非常可观。

1.8 之后 JVM 对锁进行了优化，添加了偏向锁的概念，避免了很多不必要的加锁操作，但偏向锁一旦遇到锁竞争，取消锁需要进入 `safe point`，导致 Stop The World。

解决方式很简单，JVM 启动参数里添加 `-XX:-UseBiasedLocking` 即可。

### [结果](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

调整完 JVM 参数后先是对服务进行压测，发现性能确实有提升，也没有发生严重的 GC 问题，之后再把调整好的配置放到线上机器进行灰度，同时收集 gc log，再次进行分析。

由于 young 区大小翻倍了，所以 YGC 的频率减半了，GC 的吞量提升到了 97.75%。平均 GC 时长略有上升，从 60ms 左右提升到了 66ms，还是挺符合预期的。

由于 CMS 在进行 GC 时也会清理 young 区，CMS 的时长也受到了影响，CMS 的最终标记和并发清理阶段耗时增加了，也比较正常。

另外我还统计了对业务的影响，之前因为 GC 导致超时的请求大大减少了。