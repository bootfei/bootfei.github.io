---
title: mysql - 常见事务问题解决方案
date: 2022-03-08 15:59:03
tags: [mysql,transaction]
---

## 1 数据库如何保证先查询，后插入/更新 原子性？

ref: https://blog.51cto.com/u_7592962/2543362

当操作积分用户表时，如果accountId在表中没有数据，那新增一条数据用户积分。如果accountId在表中有数据，更新用户积分。

