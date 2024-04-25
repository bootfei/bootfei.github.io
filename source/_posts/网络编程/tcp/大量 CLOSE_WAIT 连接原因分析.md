---
title: 大量 CLOSE_WAIT 连接原因分析
date: 2021-05-12 00:19:51
tags:
---



# 背景介绍

CLOSE_WAIT是tcp的四次挥手阶段的服务器状态

![](https://mmbiz.qpic.cn/mmbiz_jpg/u4aVxRTn2TgUCmDdceT9LVRwWgb9icsy4AsN4tIJ7iaibx2iaSxNa59JZHQRSjqf41ibqDQia01SZ7bPOWo1Dn3qJjRg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

服务端收到FIN包后，一直没有发送FIN包

# Dubbo服务器出现大量 CLOSE_WAIT

https://mp.weixin.qq.com/s?__biz=MzU3Njk0MTc3Ng==&mid=2247486020&idx=1&sn=f7cf41aec28e2e10a46228a64b1c0a5c&scene=21#wechat_redirect
