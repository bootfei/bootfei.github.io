---
title: jdk jstack使用教程
date: 2020-11-05 09:24:42
tags: [java, jstack]
---

它山之石可以攻玉

### ref links:

1. 基础步骤
   https://www.javatt.com/p/65620





bootFei:

1. 记住jstack中的nid（Native Thread ID)是系统线程id, 为16进制，需要使用top -Hp pid找到该线程的10进制id，然后使用下边的命令打印出16进制线程id
  ```shell
  printf  "%x  n" 10进制tid
  ```

2. jstack -l pid | grep 16进制tid

