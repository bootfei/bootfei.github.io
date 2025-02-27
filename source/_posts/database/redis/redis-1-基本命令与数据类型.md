---
title: redis-1-基本命令与数据类型
date: 2021-01-08 15:59:03
tags: [redis]
---



# Redis普通数据类型

官方命令大全网址：http://www.redis.cn/commands.html
Redis 中存储数据是通过 key-value 格式存储数据的，其中 value 可以定义五种数据类型：

- String（字符类型）
- Hash（散列类型） 
- List（列表类型）
- Set（集合类型） 
- SortedSet（有序集合类型，简称zset）

注意：在 redis 中的命令语句中，命令是忽略大小写的，而 key 是不忽略大小写的。

## string类型 

### 命令

赋值

```
SET key value

set test 123
```

取值

```
127.0.0.1:6379> get test "123“
```

取值并赋值

```
GETSET key value
getset s2 222
```

#### 数值增减 

注意事项:

> 1、 当value为整数数据时，才能使用以下命令操作数值的增减。 
>
> 2、 数值递增都是【原子】操作。 
>
> 3、 redis中的每一个单独的命令都是原子性操作。当多个命令一起执行的时候，就不能保证原子性，不过我们可以使 用事务和lua脚本来保证这一点。 <!--不要以为redis对于内存数据的操作是单线程的，就没有线程安全了-->

非原子性操作示例：

```
int i = 1; 
i++; 
System.out.println(i) 
```

##### 递增数字

```
INCR key 
```

##### 增加指定的整数

```
INCRBY key increment
```

##### 递减数值

```
DECR key
```

##### 减少指定的整数

```
DECRBY key decrement
```

#### 仅当不存在时赋值 

```
setnx key value
```

使用该命令可以实现【分布式锁】的功能，后续讲解！！！

#### 其它命令

##### 向尾部追加值 

`APPEND` 命令，向键值的末尾追加 value 。
如果键不存在则将该键的值设置为 value ，即相当于 

```
SET key value 
```

返回值是追加后字符串的总长度。

##### 获取字符串长度

 `STRLEN` 命令，返回键值的长度，如果键不存在则返回0。

##### 同时设置/获取多个键值

```
MSET key value [key value …] 
MGET key [key …]
```

### 应用场景之订单号自增主键 

- 需求：商品编号、订单号采用 INCR 命令生成。
- 设计： key 命名要有一定的设计
- 实现：定义商品编号 key ： items:id 

```
192.168.101.3:7003> INCR items:id 
(integer) 2
192.168.101.3:7003> INCR items:id 
(integer) 3
```

## hash类型

hash 类型也叫散列类型，它提供了字段和字段值的映射。字段值只能是字符串类型，不支持散列类型、集合类型等其它类型。

![avatar][hash字段和字段值映射示意图]

### 命令

赋值

HSET 命令不区分插入和更新操作，当执行插入操作时 HSET 命令返回 1 ，当执行更新操作时返回 0 。 

设置一个字段值

```
HSET key field value

127.0.0.1:6379> hset user username zhangsan
(integer) 1
```

设置多个字段值

```
HMSET key field value [field value ...]
127.0.0.1:6379> hmset user age 20 username lisi
OK
```

当字段不存在时赋值

类似 `HSET` ，区别在于如果字段存在，该命令不执行任何操作

```
HSETNX key field value
127.0.0.1:6379> hsetnx user age 30 # 如果user中没有age字段则设置age值为30，否则不做任何操作 
(integer) 0
```

#### 取值

##### 获取一个字段值

```
 HGET key field
 127.0.0.1:6379> hget user username "zhangsan“
```

##### 获取多个字段值

```
HMGET key field [field ...]
127.0.0.1:6379> hmget user age username 
1) "20" 
2) "lisi"
```

##### 获取所有字段值

```
HGETALL key
HGETALL user
```



#### 删除字段

可以删除一个或多个字段，返回值是被删除的字段个数

```
HDEL key field [field ...]

127.0.0.1:6379> hdel user age 
(integer) 1 
127.0.0.1:6379> hdel user age name 
(integer) 0 
127.0.0.1:6379> hdel user age 
username (integer) 1
```

#### 增加数字

```
HINCRBY key field increment

127.0.0.1:6379> hincrby user age 2 # 将用户的年龄加2 
(integer) 22 
127.0.0.1:6379> hget user age # 获取用户的年龄 
"22“
```

#### 其它命令

##### 判断字段是否存在

```
HEXISTS key field

127.0.0.1:6379> hexists user age 查看user中是否有age字段 
(integer) 1 
127.0.0.1:6379> hexists user name 查看user中是否有name字段 
(integer) 0
```

##### 只获取字段名或字段值

```
HEXISTS key field

127.0.0.1:6379> hexists user age 查看user中是否有age字段 
(integer) 1 
```

##### 获取字段数量

```
HLEN key

127.0.0.1:6379> hlen user 
(integer) 2
```

##### 获取所有字段

获得 hash 的所有信息，包括 key 和 value

```
hgetall key
```

### string类型和hash类型的区别 

hash类型适合存储那些对象数据，特别是对象属性经常发生【增删改】操作的数据。 string类型也可以存储对象数据，将java对象转成json字符串进行存储，这种存储适合【查询】操作。

### 应用之存储商品信息

- 商品信息字段

  ```
  [商品id、商品名称、商品描述、商品库存、商品好评]
  ```

- 定义商品信息的key

  ```
   商品ID为1001的信息在 Redis中的key为：[items:1001]
  ```

- 存储商品信息

  ```
  192.168.101.3:7003> HMSET items:1001 id 3 name apple price 999.9 
  OK
  ```

- 获取商品信息

  ```
  192.168.101.3:7003> HGET items:1001 id "3" 
  192.168.101.3:7003> HGETALL items:1001 
  1) "id" 
  2) "3" 
  3) "name" 
  4) "apple" 
  5) "price" 
  6) "999.9"
  ```

  

## list类型 

### list类型介绍

Redis 的列表类型（ list 类型）可以 存储一个有序的字符串列表 ，常用的操作是向列表两端添加元素，或者获得列表的某一个片段。

列表类型内部是使用双向链表（ double linked list ）实现的，所以向列表两端添加元素的时间复杂度为0(1) ，获取越接近两端的元素速度就越快。这意味着即使是一个有几千万个元素的列表，获取头部或尾部的10条记录也是极快的。

### 命令

#### LPUSH/RPUSH

```
LPUSH key value [value ...] 
RPUSH key value [value ...]

127.0.0.1:6379> lpush list:1 1 2 3 
(integer) 3 
127.0.0.1:6379> rpush list:1 4 5 6 
(integer) 3
```

#### LRANGE

```
LRANGE key start stop

127.0.0.1:6379> lrange list:1 3
```

#### LPOP/RPOP

从列表两端弹出元素 

从列表左边弹出一个元素，会分两步完成：

- 第一步是将列表左边的元素从列表中移除
- 第二步是返回被移除的元素值。

```
LPOP key 
RPOP key
```

#### LLEN

获取列表中元素的个数 

```
llen key

127.0.0.1:6379> llen list
(integer) 2
```

#### 其它命令

##### LREM

删除列表中前 count 个值为 value 的元素，返回实际删除的元素个数。根据 count 值的不同，该命令的执行方式会有所不同：

\- 当count>0时， LREM会从列表左边开始删除。 

\- 当count<0时， LREM会从列表后边开始删除。 

\- 当count=0时， LREM删除所有值为value的元素。 

```
LREM key count value
```



##### LINDEX

获得指定索引的元素值 

```
LINDEX key index

127.0.0.1:6379>lindex list 2 
"1"
```

##### LSET

设置指定索引的元素值

```
LSET key index value
```

##### LTRIM

只保留列表指定片段,指定范围和LRANGE一致

```
LTRIM key start stop
```

##### LINSERT

向列表中插入元素。 

该命令首先会在列表中从左到右查找值为pivot的元素，然后根据第二个参数是BEFORE还是AFTER来决定将value插 入到该元素的前面还是后面。 

```
LINSERT key BEFORE|AFTER pivot value

127.0.0.1:6379> lrange list 0 -1 
1) "3" 
2) "2" 
3) "1" 
127.0.0.1:6379> linsert list after 3 4 
(integer) 4 
127.0.0.1:6379> lrange list 0 -1 
1) "3" 
2) "4" 
3) "2" 
4) "1"
```

##### RPOPLPUSH

将元素从一个列表转移到另一个列表中 

```
RPOPLPUSH source destination
```

### 应用之商品评论列表

- 需求：
  用户针对某一商品发布评论，一个商品会被不同的用户进行评论，存储商品评论时，要按时间顺序排序。 用户在前端页面查询该商品的评论，需要按照时间顺序降序排序
- 分析：
  使用list存储商品评论信息，KEY是该商品的ID，VALUE是商品评论信息列表
- 实现：
  192.168.101.3:7001> LPUSH items:comment:1001 '{"id":1,"name":"商品不错，很 好！！","date":1430295077289}' 

## set类型

set 类型即集合类型，其中的数据是不重复且没有顺序。

集合类型的常用操作是向集合中加入或删除元素、判断某个元素是否存在等，由于集合类型的 Redis 内部是使用值

为空的散列表实现，所有这些操作的时间复杂度都为 0(1) 。 

Redis 还提供了多个集合之间的交集、并集、差集的运算。

### 命令

#### SADD/SREM

添加元素/删除元素

```
SADD key member [member ...] SREM key member [member ...]

127.0.0.1:6379> sadd set a b c 
(integer) 3 
127.0.0.1:6379> sadd set a 
(integer) 0 
127.0.0.1:6379> srem set c d 
(integer) 1
```

#### SMEMBERS

获得集合中的所有元素

```
 SMEMBERS key
```

#### SISMEMBER

判断元素是否在集合中 

### 集合运算命令 



## zset类型 （sortedset） 

### zset介绍 

在 set 集合类型的基础上，有序集合类型为集合中的每个元素都 关联一个分数 ，这使得我们不仅可以完成插入、删除和判断元素是否存在在集合中，还能够获得分数最高或最低的前N个元素、获取指定分数范围内的元素等与分数有关的操作。

在某些方面有序集合和列表类型有些相似：

- 都是有序的，列表是插入有序，集合是排序
- 都可以获得某一范围的元素

不同：

- 列表的实现是double-linkedlist, 有序集合是hashtable

### 命令

#### ZADD

```
ZADD key score member [score member ...]

127.0.0.1:6379> zadd scoreboard 80 zhangsan 89 lisi 94 wangwu 
(integer) 3 
127.0.0.1:6379> zadd scoreboard 97 lisi 
(integer) 0

```

#### ZRANGE

## 通用命令

### keys

返回满足给定pattern 的所有key 

```
keys [key]
redis 127.0.0.1:6379> keys mylist* 
1) "mylist" 
2) "mylist5" 
3) "mylist6" 
4) "mylist7" 
5) "mylist8"
```

### del

```
del [key]
```

### exists

```
exists [key]
```

### expire

```
expire [key] seconds
```

# Redis特殊数据类型**(重点)**

## BitMap

BitMap 就是通过一个 bit 位来表示某个元素对应的值或者状态, 其中的 key 就是对应元素本身 。Redis 从 2.2 版本之后新增了setbit, getbit, bitcount 等几个 bitmap 相关命令。

我们再来重申下：**Redis 其实只支持 5 种数据类型，并没有 BitMap 这种类型，BitMap 底层是基于 Redis 的字符串类型实现的。**

### 底层结构

底层也是通过对字符串的操作来实现

<img src="https://easyreadfs.nosdn.127.net/bFxn73tfrlnNzJDOe-0WzA==/8796093023252283210" alt="img" style="zoom:50%;" />

Redis 中字符串的最大长度是 512M，所以 BitMap 的 offset 值也是有上限的，其最大值是：

```
Copy8 * 1024 * 1024 * 512  =  2^32
```

### 常用命令

```shell
SETBIT key offset value #其中 offset 必须是数字，value 只能是 0 或者 1

# 获取指定范围内值为 1 的个数
# start 和 end 以字节为单位
bitcount key start end


# BitMap间的运算
# operations 位移操作符，枚举值
  AND 与运算 &
  OR 或运算 |
  XOR 异或 ^
  NOT 取反 ~
# result 计算的结果，会存储在该key中
# key1 … keyn 参与运算的key，可以有多个，空格分割，not运算只能一个key
# 当 BITOP 处理不同长度的字符串时，较短的那个字符串所缺少的部分会被看作 0。返回值是保存到 destkey 的字符串的长度（以字节byte为单位），和输入 key 中最长的字符串长度相等。
bitop [operations] [result] [key1] [keyn…]

# 返回指定key中第一次出现指定value(0/1)的位置
bitpos [key] [value]
```

### 应用场景

**1. 用户签到**

![img](https://spoolblog.files.wordpress.com/2011/11/unique_users.png?w=529)

很多网站都提供了签到功能，并且需要展示最近一个月的签到情况，这种情况可以使用 BitMap 来实现。
根据日期 offset = （今天是一年中的第几天） % （今年的天数），key = 年份：用户id。

如果需要将用户的详细签到信息入库的话，可以考虑使用一个一步线程来完成。

**2. 统计活跃用户（用户登陆情况）**

![img](https://spoolblog.files.wordpress.com/2011/11/union.png?w=529)

使用日期作为 key，然后用户 id 为 offset，如果当日活跃过就设置为1。

假如

20201009 活跃用户情况是： [1，0，1，1，0]
20201010 活跃用户情况是 ：[ 1，1，0，1，0 ]

统计连续两天活跃的用户总数：

```
Copybitop and dest1 20201009 20201010 
# dest1 中值为1的offset，就是连续两天活跃用户的ID
bitcount dest1
```

统计20201009 ~ 20201010 活跃过的用户：

```
Copybitop or dest2 20201009 20201010 
```

**3. 统计用户是否在线**

如果需要提供一个查询当前用户是否在线的接口，也可以考虑使用 BitMap 。即节约空间效率又高，只需要一个 key，然后用户 id 为 offset，如果在线就设置为 1，不在线就设置为 0。

**4. 实现布隆过滤器**

## **HyperLogLog**(2.8)

基数计数是用于统计一个集合中不重复的元素个数，比如日常需求场景有，统计页面的UV或者统计在线的用户数、注册IP数等。

如果让你实现这个需求，会怎么思考实现了？简单的做法就是记录集合中的所有不重复的 集合S，新来一个元素x，首先判断x在不在S中，如果不在，则将x加入到S，否则不记录。常用的SET数据结构就可以实现。

但是这样实现，如果数据量越来越大，会造成什么问题？

- 当统计的数据量变大时，相应的存储内存会线性增长。
- 当集合S越大时，判断x元素是否在集合S中的所花的成本会越大。

还有别的方案能减少上面2个问题带来的困扰吗？

概率算法，概率算法是通过牺牲准确率来换取空间，对于不要求绝对准确率的场景下，概率算法是一种不错的选择，因为概率算法不直接存储数据集合本身，通过一定的概率统计方法预估基数值，同时保证误差在一定范围内，这种方式可以大大减少内存。HyperLogLog就是概率算法的一种实现，下面重点介绍一下此算法。



HyperLogLog 原理思路是通过给定 n 个的元素集合，记录集合中数字的比特串第一个1出现位置的最大值k，也可以理解为统计二进制低位连续为零的最大个数。通过k值可以估算集合中不重复元素的数量m，m近似等于2^k。

下图来源于网络，通过给定一定数量的用户User，通过Hash得到一串Bitstring，记录其中最大连续零位的计数为4，User的不重复个数为 2 ^ 4 = 16.

![][hyperLogLog算法]


## **Geospatial**

可以用来保存地理位置，并作位置距离计算或者根据半径计算位置等。有没有想过用Redis来实现附近 的人?或者计算最优地图路径?Geo本身不是一种数据结构，它本质上还是借助于ZSET

```
GEOADD key 经度 维度 名称 
```

把某个具体的位置信息(经度，纬度，名称)添加到指定的key中，数据将会用一个sorted set存储，以便稍后能使用GEORADIUS和GEORADIUSBYMEMBER命令来根据半径来查询位置信息。

# Redis消息模式

最好交给kafka，RocketMq来做，因为它对于消息的可靠性和消息的限流不友好, 消息容易丢掉

## 队列模式

使用list类型的lpush和rpop实现消息队列

注意事项：

- 消息接收方如果不知道队列中是否有消息，会一直发送rpop命令，如果这样的话，会每一次都建立一次连接，这样显然不好。
- 可以使用brpop命令，它如果从队列中取不出来数据，会一直阻塞，在一定范围内没有取出则返回null、

## 发布订阅模式（略）

# Redis事务

弱事务，不保证ACID, 所以不重要

## 事务介绍

- Redis 的事务是通过 MULTI 、 EXEC 、 DISCARD 和 WATCH 、UNWATCH这五个命令来完成的。
- Redis 的单个命令都是原子性的，所以这里需要确保事务性的对象是命令集合。 Redis 将命令集合序列化并保处于同一事务的命令集合连续且不被打断的执行
- Redis 不支持回滚操作。

## 事务命令

### MULTI

用于标记事务块的开始。 

Redis会将后续的命令逐个放入队列中，然后使用EXEC命令原子化地执行这个命令序列。 

### EXEC

在一个事务中执行所有先前放入队列的命令，然后恢复正常的连接状态

### DISCARD

清除所有先前在一个事务中放入队列的命令，然后恢复正常的连接状态。 

### WATCH

监控某个key, 如果这个key在事务执行的期间发生改变，那么此次事务就会执行失败。

```
watch [key]
```

### UNWATCH

清楚监控

## 事务演示

```shell
127.0.0.1:6379> multi 
OK
127.0.0.1:6379> set s1 111 
QUEUED 
127.0.0.1:6379> hset set1 name zhangsan 
QUEUED 
127.0.0.1:6379> exec 
1) OK 
2) (integer) 1 
127.0.0.1:6379> multi 
OK
127.0.0.1:6379> set s2 222 
QUEUED

127.0.0.1:6379> hset set2 age 20 
QUEUED 
127.0.0.1:6379> discard 
OK
127.0.0.1:6379> exec 
(error) ERR EXEC without MULTI 

127.0.0.1:6379> watch s1 
OK
127.0.0.1:6379> multi 
OK
127.0.0.1:6379> set s1 555 
QUEUED 
127.0.0.1:6379> exec # 此时在没有exec之前，通过另一个命令窗口对监控的s1字段进行修改
(nil) 
127.0.0.1:6379> get s1 
111
```

##  事务失败处理

- Redis 语法错误（编译期）
- Redis 运行错误

Redis 不支持事务回滚（为什么呢）
1、大多数事务失败是因为语法错误或者类型错误，这两种错误，在开发阶段都是可以预见的
2、 Redis 为了性能方面就忽略了事务回滚。

# 附录

## hash字段和字段值映射示意图

[hash字段和字段值映射示意图]:data:image/png;base64,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

## hyperLogLog算法

[hyperLogLog算法]:data:image/jpeg;base64,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