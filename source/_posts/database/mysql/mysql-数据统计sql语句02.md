---
title: 'mysql:工作中常用的组合sql语句'
date: 2020-11-23 19:03:17
tags:
---



## 存在则更新否则插入

这里有个系统设计的难点，就是[保障“先检查，后更新”这个复合操作一定是原子性的]()

如果使用分布式锁，来保证原子性（类似cas）,会出现严重性能问题，因为所有的服务获取锁失败后，仍然不断尝试获取锁，浪费cpu。

所有，原子操作要交给数据库解决。

## 需要插入数据中有主键or唯一索引

### insert ignore into

即插入数据时，如果数据存在，则忽略此次插入，前提条件是插入的数据字段设置了主键或唯一索引，测试SQL语句如下，当插入本条数据时，MySQL数据库会首先检索已有数据（也就是idx_username索引），如果存在，则忽略本次插入，如果不存在，则正常插入数据：

![img](https://pic3.zhimg.com/80/v2-b64c607f476438fcf61476541f5282f2_1440w.jpg)

### on duplicate key update

即插入数据时，如果数据存在，则执行更新操作，前提条件同上，也是插入的数据字段设置了主键或唯一索引，测试SQL语句如下，当插入本条记录时，MySQL数据库会首先检索已有数据（idx_username索引），如果存在，则执行update更新操作，如果不存在，则直接插入：

![img](https://pic2.zhimg.com/80/v2-f19efc8f2d231e0d78c674b4f687698d_1440w.jpg)

```xml
mybatis的写法 
<insert id="batchAdd" parameterType="java.util.List">
       INSERT INTO t_student(uid,student_id,study_days)
       VALUES
       <foreach collection="list" item="item" index="index" separator=",">
           (#{item.uid},#{item.studentId},#{item.studyDays})
       </foreach>
       ON DUPLICATE KEY UPDATE
       uid = values(uid),
  		 student_id = values(student_id)
   </insert>
```



###  replace into

即插入数据时，如果数据存在，则删除再插入，前提条件同上，插入的数据字段需要设置主键或唯一索引，测试SQL语句如下，当插入本条记录时，MySQL数据库会首先检索已有数据（idx_username索引），如果存在，则先删除旧数据，然后再插入，如果不存在，则直接插入：

![img](https://pic2.zhimg.com/80/v2-7c30f2c0e1930f14020d5041379ca525_1440w.jpg)

## 不需要插入数据中有主键or唯一索引

###  insert if not exists

即insert into … select … where not exist ... ，这种方式适合于插入的数据字段没有设置主键或唯一索引，当插入一条数据时，首先判断MySQL数据库中是否存在这条数据，如果不存在，则正常插入，如果存在，则忽略：

![img](https://pic4.zhimg.com/80/v2-f44696934ba126b67e6ab5477c9eaed3_1440w.jpg)

目前，就分享这4种MySQL处理重复数据的方式吧，前3种方式适合字段设置了主键或唯一索引，最后一种方式则没有此限制，只要你熟悉一下使用过程，很快就能掌握的，



