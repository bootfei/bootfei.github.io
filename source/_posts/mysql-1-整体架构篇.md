---
title: mysql - 1.整体架构篇
date: 2020-10-08 15:59:03
tags: mysql
---

1. mysql server 层

   1. Connection pool

      1. 连接的数量

         1. 查看mysql的链接数量
						```bash
            Show variables like ""
            ```

         2. 查看linux

   2. Sql interface

      DML, DDL, Stored procedures
   
   3. parser
   
   4. optimizer
   
   5. cacehs & buffers

