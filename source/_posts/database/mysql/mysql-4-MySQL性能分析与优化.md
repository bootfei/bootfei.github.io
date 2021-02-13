---
title: mysql - 4.MySQL性能分析与优化
date: 2021-01-08 15:59:03
tags: [mysql]
---

# **性能分析的思路**

- 首先需要使用【慢查询日志】功能，去获取所有查询时间比较长的SQL语句 
- 其次【查看执行计划】查看有问题的SQL的执行计划 explain

- 最后可以使用【show profile[s]】 查看有问题的SQL的性能使用情况

## **慢查询日志**

修改my.cnf开启:

```mysql
slow_query_log=ON
long_query_time=3
slow_query_log_file==/var/lib/mysql/slow-log.log
```

### 慢查询日志介绍

### 开启慢查询功能

### 分析慢查询日志的工具 

使用mysqldumpslow工具

- mysqldumpslow是MySQL自带的慢查询日志工具。 
- 可以使用mysqldumpslow工具搜索慢查询日志中的SQL语句

使用percona-toolkit工具

- percona-toolkit是一组高级命令行工具的集合，
- 可以查看当前服务的摘要信息，磁盘检测，分析慢查询日志，查找重复索引，实现表同步等等。



## profile分析语句

Query Profiler是MySQL自带的一种**query**诊断分析工具**，通过它可以分析出一条SQL语句的**硬件性能 瓶颈**在什么地方。比如CPU，IO等，以及该SQL执行所耗费的时间等。不过该工具只有在MySQL 5.0.37 以及以上版本中才有实现。默认的情况下，MYSQL的该功能没有打开，需要自己手动启动。

**开启**Profile功能

Profile 功能由MySQL会话变量 : 

- **profiling**控制,默认是**OFF**关闭状态。 
- 查看是否开启了Profile功能:

```mysql
select @@profiling;
show variables like ‘%profil%’;
set profiling=1; --1是开启、0是关闭
show profiles;
show profile for query 6;  --查询query 6的cpu,IO等消耗时间
show profile cpu,block io,swaps for query 6;
```



# **服务器层面优化** 

## **服务器硬件优化**

提升硬件设备，例如选择尽量高频率的内存(频率不能高于主板的支持)、提升网络带宽、使用SSD高 速磁盘、提升CPU性能等。

CPU的选择:

- 对于数据库并发比较高的场景，CPU的数量比频率重要。 
- 对于CPU密集型场景和频繁执行复杂SQL的场景，CPU的频率越高越好。

## CentOS系统针对mysql的参数优化（交给专业DBA）

### 内核相关参数(/etc/sysctl.conf)

以下参数可以直接放到sysctl.conf文件的末尾。 

1.增加监听队列上限:

```
net.core.somaxconn = 65535 
net.core.netdev_max_backlog = 65535 
net.ipv4.tcp_max_syn_backlog = 65535
```

2.加快TCP连接的回收:

```
net.ipv4.tcp_fin_timeout = 10 
net.ipv4.tcp_tw_reuse = 1 
net.ipv4.tcp_tw_recycle = 1
```

3.TCP连接接收和发送缓冲区大小的默认值和最大值:

```
net.core.wmem_default = 87380 
net.core.wmem_max = 16777216 
net.core.rmem_default = 87380 
net.core.rmem_max = 16777216
```

4.减少失效连接所占用的TCP资源的数量，加快资源回收的效率:

```
net.ipv4.tcp_keepalive_time = 120 
net.ipv4.tcp_keepalive_intvl = 30 
net.ipv4.tcp_keepalive_probes = 3
```

5.单个共享内存段的最大值: kernel.shmmax = 4294967295

1. 这个参数应该设置的足够大，以便能在一个共享内存段下容纳整个的Innodb缓冲池的大小。 
2. 这个值的大小对于64位linux系统，可取的最大值为(物理内存值-1)byte，建议值为大于物理

内存的一半，一般取值大于Innodb缓冲池的大小即可。

6.控制换出运行时内存的相对权重:

vm.swappiness = 0

这个参数当内存不足时会对性能产生比较明显的影响。(设置为0，表示Linux内核虚拟内存完全被占 用，才会要使用交换区。)

Linux系统内存交换区: 在Linux系统安装时都会有一个特殊的磁盘分区，称之为系统交换分区。
使用 free -m 命令可以看到swap就是内存交换区。 作用:当操作系统没有足够的内存时，就会将部分虚拟内存写到磁盘的交换区中，这样就会发生内存交换。

如果Linux系统上完全禁用交换分区，带来的风险: 

1. 降低操作系统的性能

2. 容易造成内存溢出，崩溃，或都被操作系统kill掉

### 增加资源限制(/etc/security/limit.conf)

打开文件数的限制(以下参数可以直接放到limit.conf文件的末尾):

```
* soft nofile 65535 
* hard nofile 65535
```

*:表示对所有用户有效 soft:表示当前系统生效的设置(soft不能大于hard ) hard:表明系统中所能设定的最大值 nofile:表示所限制的资源是打开文件的最大数目 65535:限制的数量

以上两行配置将可打开的文件数量增加到65535个，以保证可以打开足够多的文件句柄。 注意:这个文件的修改需要重启系统才能生效。

### 磁盘调度策略

1.cfq (完全公平队列策略，Linux2.6.18之后内核的系统默认策略) 

该模式按进程创建多个队列，各个进程发来的IO请求会被cfq以轮循方式处理，对每个IO请求都是公平的。该策略适合离散读的应用。

2.deadline (截止时间调度策略)

deadline，包含读和写两个队列，确保在一个截止时间内服务请求(截止时间是可调整的)，而默认读 期限短于写期限。这样就防止了写操作因为不能被读取而饿死的现象，deadline对数据库类应用是最好 的选择。

3.noop (电梯式调度策略) 

noop只实现一个简单的FIFO队列，倾向饿死读而利于写，因此noop对于闪存设备、RAM及嵌入式系统是最好的选择。

4.anticipatory (预料I/O调度策略)

本质上与deadline策略一样，但在最后一次读操作之后，要等待6ms，才能继续进行对其它I/O请求进 行调度。它会在每个6ms中插入新的I/O操作，合并写入流，用写入延时换取最大的写入吞吐量。 anticipatory适合于写入较多的环境，比如文件服务器。该策略对数据库环境表现很差。

```
查看调度策略的方法:cat /sys/block/devname/queue/scheduler
修改调度策略的方法:echo > /sys/block/devname/queue/scheduler
```

## **MySQL**数据库配置优化

```shell
vim /etc/my.cnf
```

### 将数据保存在内存中，保证从内存读取数据

设置足够大的 innodb_buffer_pool_size ，将数据读取到内存中。建议innodb_buffer_pool_size设置为总内存大小的3/4或者4/5.

怎样确定 innodb_buffer_pool_size 足够大。数据是从内存读取而不是硬盘?

```mysql
mysql> show global status like 'innodb_buffer_pool_pages_%';
```

### **降低磁盘写入次数**

- 用来控制redo log刷新到磁盘的策略**innodb_flush_log_at_trx_commit**。(0性能最好，主从机制的机器，从机器是不负责写操作， 所以不牵扯事务提交，所以为了从数据库的性能提升，需要将此参数设置为0)
- 每提交1次事务同步写到磁盘中**sync_binlog**，可以设置为n。(主从机制的机器，从机器是不负责写操作，所 以不牵扯事务提交，也不需要再重复写binlog，所以将此处设置为0)
- 使用足够大的写入缓存 **innodb_log_file_size**，过小导致造成日志切换频繁。推荐值为20%~25%。
- 脏页占innodb_buffer_pool_size的比例**innodb_max_dirty_pages_pct**时，触发刷脏页到磁盘。 推荐值为25%~50%。
- 指定innodb共享表空间文件的大小**innodb_data_file_path**。(分磁盘，这样可以减轻磁盘的IO操作性能压力)

### 减少与业务无关从操作

- 全量日志建议关闭。**general_log**=0
- 慢查询日志，不要随意开启，开启的时候，要合理设置阈值

# **数据库设计层面优化**

设计表的时候，添加如下几列

- 业务字段(预留字段)
- 系统字段(修改人、修改时间)
- 流程字段(状态)

具体优化方案如下: 

- **设计中间表**，一般针对于**统计分析**功能，或者实时性不高的需求(OLTP、OLAP)
- 为减少关联查询，创建合理的**冗余字段**(考虑数据库的三范式和查询性能的取舍，创建冗余字段还 需要注意**数据一致性问题**)逆范式
- 对于字段太多的大表，考虑**拆表**(比如一个表有100多个字段)(查询的时候，是以整行为单位去 加载的。)
- Blob Clob Text类型的字段，建议拆到另一张表中。(商品表10-15列，**商品详情介绍****[text]-**--建议 将商品详情介绍拆成单独的表)
- 对于表中经常不被使用的字段或者存储数据比较多的字段，考虑拆表(比如商品表中会存储商品介绍，此时可以将商品介绍字段单独拆解到另一个表中，使用商品ID关联)
- 每张表建议都要有一个主键(**主键索引**)，而且主键类型最好是**int**类型**，建议自增主键(**如果是分布式系统的情况，采用雪花算法)。

# **SQL**语句优化

## 索引优化

- 为搜索字段(**where中的条件**)、排序字段、select查询列，创建合适的索引，不过要考虑数据的 业务场景:查询多还是增删多?

- 尽量建立**组合索引**并注意组合索引的创建顺序，按照顺序组织查询条件、尽量将筛选粒度大的查询 条件放到最左边。

- **尽量使用覆盖索引**，SELECT语句中尽量不要使用*。

- order by、group by语句要尽量使用到索引

  name,age---组合索引

  select * from user where name = "james" order by age ---- 使用到索引

  select * from user order by name,age ---- 使用到索引

- 索引长度尽量短，短索引可以节省索引空间，使查找的速度得到提升，同时内存中也可以装载更多的索引键值。[不推荐UUID当索引](https://blog.csdn.net/sinat_37903468/article/details/108469482)。太长的列，可以选择建立前缀索引。比如给name列加索引，但是name列长度为300。其实最多在使用索引搜索的时候，使用到前10 个字节长度。这个时候建议建立前缀索引

- 索引更新不能频繁，更新非常频繁的数据不适宜建索引，因为维护索引的成本。

- order by的索引生效，order by排序应该遵循最佳左前缀查询，如果是使用多个索引字段进行排序，那么排序的规则必须相同(同是升序或者降序)，否则索引同样会失效。

## **LIMIT**优化

- 如果预计SELECT语句的查询结果是一条，最好使用 **LIMIT 1**，可以停止全表扫描。

  - ```
    SELECT * FROM user WHERE username=’全力詹’; -- username没有建立唯一索引 
    SELECT * FROM user WHERE username=’全力詹’ LIMIT 1;
    ```

- 处理分页会使用到 LIMIT ，当翻页到非常靠后的页面的时候，偏移量会非常大，这时LIMIT的效率 会非常差。 LIMIT OFFSET , SIZE;

  - 解决方案1:**使用**order by和索引覆盖

    - ```
      原SQL(如果 表中的记录有10020条):
      SELECT id,name FROM user  LIMIT 10000, 20;
      
      优化的SQL:
      SELECT id,name FROM user  ORDER BY id desc LIMIT 20;
      ```

  - 解决方案2:**使用子查询**

    - ```
      SELECTid,nameFROM(selectid,namefromuserorderbyidlimit 10000,20) t
      ```

  - 解决方案3:单表分页时，使用自增主键排序之后，先使用where条件 id > offset值，limit后面只 写rows

## **其他查询优化**

- 小表驱动大表，建议使用left join时，以小表关联大表，因为使用join的话，第一张表是必须全扫 描的，以少关联多就可以减少这个扫描次数。

explain---select列的信息，显示查询顺序。
 A表10万条记录
 B表一千万条记录 需求:关联A表和B表去查询数据。比如结果能匹配的也就10条记录

- 避免全表扫描，mysql在使用不等于(!=或者<>)的时候无法使用导致全表扫描。在查询的时候，如 果对索引使用不等于的操作将会导致索引失效，进行全表扫描

- 如果mysql估计使用全表扫描要比使用索引快，则不使用索引。(最典型的场景就是数据量少的时候)

- 尽量不使用count(*)、尽量使用count(主键)

  - ```
    COUNT(*)\COUNT(1)\COUNT(列)，从索引使用情况来说，是一样的。
    如果COUNT(非索引列)， 那么MySQL会选择该表中，最小的那颗索引树，去进行统计。
    COUNT(*)以行为统计，最终的结果是包含null值 
    COUNT(1)会过滤NULL值 
    COUNT(列)会过滤NULL
    ```

- JOIN两张表的关联字段最好都建立索引，而且最好字段类型是一样的。

- WHERE条件中尽量不要使用1=1、not in语句(建议使用not exists)

- 不用 MYSQL 内置的函数，因为内置函数不会建立查询缓存。

  - ```
    SQL查询语句和查询结果都会在第一次查询只会存储到MySQL的查询缓存中，如果需要获取到查询缓 存中的查询结果，查询的SQL语句必须和第一次的查询SQL语句一致。
    SELECT * FROM user where birthday = now();
    ```

