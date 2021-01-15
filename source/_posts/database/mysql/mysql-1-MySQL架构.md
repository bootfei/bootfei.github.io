---
title: mysql - 1.MySQL架构
date: 2020-10-08 15:59:03
tags: [mysql]
---

# 逻辑架构





# 物理结构

- MySQL是通过文件系统对数据和索引进行存储的。
- MySQL从物理结构上可以分为日志文件和数据索引文件。
- MySQL在Linux中的数据索引文件和日志文件都在/var/lib/mysql目录下。
- 日志文件采用顺序IO方式存储、数据文件采用随机IO方式存储。

## 日志文件



### 错误日志（errorlog）

默认是开启的，而且从5.5.7以后无法关闭错误日志，错误日志记录了运行过程中遇到的所有严重的错误信息,以及 MySQL每次启动和关闭的详细信息。

### 二进制日志（bin log）

记录数据变化

binlog记录了数据库所有的ddl语句和dml语句，但不包括select语句内容，语句以事件的形式保存，描述了数据的变更顺序，binlog还包括了每个更新语句的执行时间信息。如果是DDL语句，则直接记录到binlog日志，而DML语句，必须通过事务提交才能记录到binlog日志中。 生产中开启

数据备份、恢复、主从

### 通用查询日志（general query log）

啥都记录 耗性能 生产中不开启

### 慢查询日志（slow query log）

SQL调优 定位慢的 select

默认是关闭的。

需要通过以下设置进行开启：

```
#开启慢查询日志 
slow_query_log=ON
#慢查询的阈值 
long_query_time=3
#日志记录文件如果没有给出file_name值， 默认为主机名，后缀为-slow.log。如果给出了文件名， 但不是绝对路径名，文件则写入数据目录。 slow_query_log_file=file_name
```

记录执行时间超过long_query_time秒的所有查询，便于收集查询时间比较长的SQL语句

### 重做日志（redo log）

### 回滚日志（undo log）

### 中继日志（relay log）



看日志开启情况：

```
show variables like 'log_%';
```



## 数据文件

```
SHOW VARIABLES LIKE '%datadir%';
```



InnoDB数据文件

- .frm文件：主要存放与表相关的数据信息,主要包括表结构的定义信息

- .ibd：使用独享表空间存储表数据和索引信息，一张表对应一个ibd文件。

- ibdata文件：使用共享表空间存储表数据和索引信息，所有表共同使用一个或者多个ibdata文件。


MyIsam数据文件

- .frm文件：主要存放与表相关的数据信息,主要包括表结构的定义信息

- .myd文件：主要用来存储表数据信息。

- .myi文件：主要用来存储表数据文件中任何索引的数据树。