---
title: 'apache http client引起的线程假死or卡死问题'
date: 2020-11-04 13:47:46
tags: [java, net]
---

它山之石可以攻玉

### ref links:

1. 如何一步一步找到问题的原因是假死
   https://blog.csdn.net/baomw/article/details/84070428?utm_medium=distribute.pc_relevant_bbs_down.none-task--2~all~first_rank_v2~rank_v28-3.nonecase&depth_1-utm_source=distribute.pc_relevant_bbs_down.none-task--2~all~first_rank_v2~rank_v28-3.nonecase

2. 详细指令的参数说明，并且有详细脚本：
   https://www.cnblogs.com/wyb628/p/8566337.html
3. jstack返回的详细解释：
   https://blog.csdn.net/weixin_38451161/article/details/98035500
4. http client的死锁原因深究，以及如何查看git官网的bug修复通知
   https://www.cnblogs.com/549294286/p/11241277.html
5. http client 无限卡死的原因
   https://www.oschina.net/question/2298861_240814
   https://www.oschina.net/question/2272252_2176034
6. http client 无限卡死的解决方法：设置3个超时时间，并且在spirng中的设置; 
   https://www.jianshu.com/p/403d75d310e8



思考：

1. 什么是假死？
2. Socket timeout 会stuck，而且如果不设置timeout, 那么不抛出异常