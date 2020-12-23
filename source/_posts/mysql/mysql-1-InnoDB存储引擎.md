---
title: mysql - 1.InnoDB存储引擎
date: 2020-10-08 15:59:03
tags: [mysql]
---



# InnoDB存储引擎

## 1. InnoDB架构图

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1ghu7wvkdxyj30jg0eyq6j.jpg" style="zoom: 67%;" />

上图详细显示了InnoDB存储引擎的体系架构，从图中可见，InnoDB存储引擎由**内存池，后台线程和磁**

**盘文件**三大部分组成。接下来我们就来简单了解一下内存相关的概念和原理。

## 2. InnoDB磁盘文件

> InnoDB的主要的磁盘文件主要分为三大块：**系统表空间**，**用户表空间**，redo日志文件和归档文件。
>
> 二进制文件(binlog)等文件是MySQL Server层维护的文件，所以未列入InnoDB的磁盘文件中。

### 2.1 系统表空间

<img src="https://imgedu.lagou.com/131123de763c46a1b03b95c650ba32f0.jpg" style="zoom: 67%;" />

<img src="https://imgedu.lagou.com/6feb838308f34b0791c07a2b2b157fcf.jpg" style="zoom: 50%;" />

系统表空间存储哪些数据？

> 系统表空间是一个共享的表空间，因为它是被多个表共享的。
>
> InnoDB系统表空间包含InnoDB数据字典(元数据以及相关对象)、double write buffer、change buffer、undo logs的存储区域。
>
> 系统表空间也默认包含任何用户在系统表空间创建的表数据和索引数据。

系统表空间配置解析

> 系统表空间是由一个或者多个数据文件组成。
>
> 默认情况下，一个初始大小为10MB，名为ibdata1的系统数据文件在MySQL的data目录下被创建。
>

用户可以使用 innodb_data_file_path 对数据文件的大小和数量进行配置。

innodb_data_file_path 的格式如下：`innodb_data_file_path=datafile1[,datafile2]...`

示例(这里将**/db/ibdata1**和**/dr2/db/ibdata2**两个文件组成系统表空间)： `innodb_data_file_path=/db/ibdata1:1000M;/dr2/db/ibdata2:1000M:autoextend`

<!--如果这两个文件位于不同的磁盘上，磁盘的负载可能被平均，因此可以提高数据库的整体性能。-->

<!--两个文件的文件名之后都跟了属性，表示文件ibdata1的大小为1000MB，文件ibdata2的大小为1000MB，而且用完空间之后可以自动增长(autoextend)。-->

### 2.2 用户表空间

用户表空间只存储该表的

> - 数据
> - 索引
> - 插入缓冲BITMAP等信息
> - 其余信息还是存放在默认的系统表空间中。

如何使用用户表空间？

> 如果设置了参数**innodb_file_per_table**，则用户可以将每个基于InnoDB存储引擎的表产生一个独立的用户表空间。用户表空间的命名规则为：表名.ibd。通过这种方式，用户不用将所有数据都存放于默认的系统表空间中。

### 2.3 redo日志文件

#### 哪些文件是重做日志文件？

> 默认情况下，在InnoDB存储引擎的数据目录下会有两个名为**ib_logfile0**和**ib_logfile1**的文件，这就是InnoDB的**重做日志文件** **(redo log file)**，它记录了对于InnoDB存储引擎的事务日志。
>

#### 重做日志文件的作用

- **当InnoDB的数据存储文件发生错误时，重做日志文件就能派上用场**。InnoDB存储引擎可以使用重做日志文件将数据恢复为正确状态，以此来保证数据的正确性和完整性。

- 为了得到更高的可靠性，用户可以设置多个镜像日志组，将不同的文件组放在不同的磁盘上，以此来提高重做日志的**高可用性**。

#### 重做日志文件组如何写入数据

每个InnoDB存储引擎至少有1个重做日志文件组(group)，每个文件组下至少有2个重做日志文件，如默认的`ib_logfile0`和`ib_logfile1`。

> 在日志组中每个重做日志文件的**[大小一致]**，并以**[循环写入]**的方式运行。InnoDB存储引擎先写入重做日志文件1，当文件被写满时，会切换到重做日志文件2; 当重做日志文件2也被写满时，再切换到重做日志文件1。
>

#### 设置重做日志文件大小

用户可以使用`innodb_log_file_size`来设置重做日志文件的大小，这对InnoDB存储引擎的性能有着非常大的影响。

> 如果重做日志文件设置的**太大**，数据丢失时，恢复时可能需要很长的时间；
>
> 另一方面，如果设置的**太小**，重做日志文件太小会导致依据`checkpoint`的检查需要频繁刷新脏页到磁盘中，导致性能的抖动。
>

### 2.4 InnoDB逻辑存储结构

InnoDB存储引擎逻辑存储结构可分为五级：表空间、段、区、页、行。

<img src="https://static001.geekbang.org/infoq/71/715646b8db39a9537925e985c5140c70.png" style="zoom: 80%;" />

#### 1. 表空间

从InnoDB存储引擎的逻辑存储结构看，所有数据都被逻辑地存放在一个空间中，称之为表空间(tablespace)。

从功能上来看，InnoDB存储引擎的表空间分为系统表空间，独占表空间，通用表空间，临时表空间，Undo表空间。InnoDB 存储引擎有一个共享表空间，叫做系统表空间，在磁盘上的文件名为ibdata1。

- 如果设置了独立表空间innodb_file_per_table=1，每张表的数据都会存储到一个独立的表空间，即一个单独的.ibd文件。

- 如果设置了参数innodb_file_per_table=0，关闭了独占表空间，则所有基于InnoDB存储引擎的表数据都会记录到系统表空间。

#### 2. 段

表空间是由各个段组成的，常见的段有数据段、索引段、回滚段等。

如果开启了独立表空间innodb_file_per_table=1，每张表的数据都会存储到一个独立的表空间，即一个单独的.ibd文件。

一个用户表空间里面由很多个段组成，创建一个索引时会创建两个段：数据段和索引段。

- 数据段存储着索引树中叶子节点的数据。

- 索引段存储着索引树中非叶子节点的数据。

一个段的空间大小是随着表的大小自动扩展的：表有多大，段就有多大。

一个段会包含多个区，至少会有一个区，段扩展的最小单位是区。

#### 3. 区

一个区由64个**连续**的页组成，一个区的大小=1M=64个页(16K)。**为了保证区中页的连续性**，区扩展时InnoDB 存储引擎会一次性从磁盘申请4 ~ 5个区。

#### 4. 页

![](https://img-blog.csdnimg.cn/20200705151425989.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3p3eDkwMDEwMg==,size_16,color_FFFFFF,t_70)

InnoDB 每个页默认大小时是 16KB，页是 InnoDB管理磁盘的最小单位，也InnoDB中磁盘和内存交互的最小单位。

```mysql
show global variables like 'innodb_page_size';
```

索引树上一个节点就是一个页，MySQL规定一个页上最少存储2个数据项。

- 如果向一个页插入数据时，这个页已将满了，就会从区中分配一个新页。

- 如果向索引树叶子节点中间的一个页中插入数据，如果这个页是满的，就会发生页分裂。

操作系统管理磁盘的最小单位也是页，是操作系统读写磁盘最小单位，Linux中页一般是4K，可以通过命令查看。

```shell
#默认 4096 4K 
\> getconf PAGE_SIZE
```

所以InnoDB从磁盘中

- 读取一个数据页时，操作系统会**分4次**从磁盘文件中读取数据到内存。
- 写入也是一样的，需要**分4次**从内存写入到磁盘中。

#### 5. 行

InnoDB的数据是以行为单位存储的，1个页中包含多个行。在MySQL5.7中，InnoDB提供了4种行格式：Compact、Redundant、Dynamic和Compressed行格式，Dynamic为MySQL5.7默认的行格式。InnoDB行格式官网：https://dev.mysql.com/doc/refman/5.7/en/innodb-row-format.html

创建表时可以指定行格式：

```mysql
CREATE TABLE t1 (c1 INT) ROW_FORMAT=DYNAMIC; 

ALTER TABLE tablename ROW_FORMAT=行格式名称; 

\#修改默认行格式 

SET GLOBAL innodb_default_row_format=DYNAMIC; 

\#查看表行格式 

SHOW TABLE STATUS LIKE 'student'\G; 
```

