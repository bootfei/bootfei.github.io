---
title: 分布式集群下生成唯一ID
date: 2020-11-10 13:54:56
tags: []
Categories: [分布式]
---

它山之石可以攻玉

### ref links:

1. 介绍了唯一ID的使用场景以及常见的解决思路
   https://www.javazhiyin.com/73643.html

2. 如何利用UUID作为唯一ID存储在mysql中，并提高mysql性能 => 存储为number
   https://stackoverflow.com/questions/16122934/best-way-to-handle-large-uuid-as-a-mysql-table-primary-key

   https://stackoverflow.com/questions/10950202/how-to-store-uuid-as-number

3. 深入分析mysql为什么不推荐使用uuid或者雪花id作为主键
   https://www.cnblogs.com/wyq178/p/12548864.html





思路:

1. 为什么需要唯一ID?
   对数据和信息要有唯一标识，比如分库分表后，订单要有唯一标识
2. 对ID有什么样的要求呢
   1. 