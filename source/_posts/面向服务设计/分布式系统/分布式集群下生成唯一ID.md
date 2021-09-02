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



## 简单分析一下需求

所谓全局唯一的 id 其实往往对应是**生成唯一记录标识的业务需求**。

这个 id 常常是数据库的主键，数据库上会建立聚集索引（cluster index），即在物理存储上以这个字段排序。这个记录标识上的查询，往往又有分页或者排序的业务需求。所以往往要有一个time字段，并且在time字段上建立普通索引（non-cluster index）。

普通索引存储的是实际记录的指针，其访问效率会比聚集索引慢，如果记录标识在生成时能够基本按照时间有序，则可以省去这个time字段的索引查询。

这就引出了记录标识生成的两大核心需求：

- 全局唯一
- 趋势有序

## 常见生成策略的优缺点对比

### 用数据库的 auto_increment 

**优点：**

- 此方法使用数据库原有的功能，所以相对简单
- 能够保证唯一性
- 能够保证递增性
- id 之间的步长是固定且可自定义的

**缺点：**

- 可用性难以保证：数据库常见架构是 一主多从 + 读写分离，生成自增ID是写请求 主库挂了就玩不转了
- 扩展性差，性能有上限：因为写入是单点，数据库主库的写性能决定ID的生成性能上限，并且 难以扩展

**改进方案：**

- 冗余主库，避免写入单点
- 数据水平切分，保证各主库生成的ID不重复

![Image](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)![Image](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbudlL2EP3TLE6MKFNCg3bJwqicUGPB3zibWNXkibQibHHBZCjhib8Pzz5u7aVnJryuiabDEKgsLiashe8Gj0w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

如上图所述，由1个写库变成3个写库，每个写库设置不同的 auto_increment 初始值，以及相同的增长步长，以保证每个数据库生成的ID是不同的（上图中DB 01生成0,3,6,9…，DB 02生成1,4,7,10，DB 03生成2,5,8,11…）

改进后的架构保证了可用性，但缺点是

- 丧失了ID生成的“绝对递增性”：先访问DB 01生成0,3，再访问DB 02生成1，可能导致在非常短的时间内，ID生成不是绝对递增的（这个问题不大，目标是趋势递增，不是绝对递增
- 数据库的写压力依然很大，每次生成ID都要访问数据库

为了解决这些问题，引出了以下方法：

### 单点批量ID生成服务

分布式系统之所以难，很重要的原因之一是“没有一个全局时钟，难以保证绝对的时序”，要想保证绝对的时序，还是只能使用单点服务，用本地时钟保证“绝对时序”。

数据库写压力大，是因为每次生成ID都访问了数据库，可以使用批量的方式降低数据库写压力。

![Image](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbudlL2EP3TLE6MKFNCg3bJwqOP9UXxwIOCibhlXXiaSSGwmSTC80GBCA8LVxLcKxH89BtaO9lqzbMfqg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



如上图所述，<!--数据库使用双master保证可用性-->数据库中只存储当前ID的最大值，例如4。

假设每次批量拉取5个ID，客户端应用访问ID生成服务，

1. ID生成服务将当前ID的最大值修改为4
2. 客户端应用访问ID生成服务索要ID
3. ID生成服务依次派发0,1,2,3,4这些ID。
4. 当ID发完后，再将ID的最大值修改为11，就能再次派发6,7,8,9,10,11这些ID了，于是数据库的压力就降低到原来的1/6。

**优点：**

- 保证了ID生成的绝对递增有序
- 大大的降低了数据库的压力，ID生成可以做到每秒生成几万几十万个

**缺点：**

- 服务仍然是单点
- 如果服务挂了，服务重启起来之后，继续生成ID可能会不连续，中间出现空洞（服务内存是保存着0,1,2,3,4，数据库中max-id是4，分配到3时，服务重启了，下次会从5开始分配，3和4就成了空洞，不过这个问题也不大）
- 虽然每秒可以生成几万几十万个ID，但毕竟还是有性能上限，无法进行水平扩展

**改进方案**

- 单点服务的常用高可用优化方案是“备用服务”，也叫“影子服务”，所以我们能用以下方法优化上述缺点：

![Image](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbudlL2EP3TLE6MKFNCg3bJwqQvubrmA8V4J1H7UzCe1647T82GToR508bgiaLKLnMQ42MBGIyob1XdA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



如上图，对外提供的服务是主服务，有一个影子服务时刻处于备用状态，当主服务挂了的时候影子服务顶上。这个切换的过程对调用方是透明的，可以自动完成，常用的技术是 vip+keepalived。另外，id generate service 也可以进行水平扩展，以解决上述缺点，但会引发一致性问题。

### uuid / guid

不管是通过数据库，还是通过服务来生成ID，业务方Application都需要进行一次远程调用，比较耗时。uuid是一种常见的本地生成ID的方法。

```
UUID uuid = UUID.randomUUID();
```

**优点：**

- 本地生成ID，不需要进行远程调用，时延低
- 扩展性好，基本可以认为没有性能上限

**缺点：**

- 无法保证趋势递增
- uuid过长，往往用字符串表示，作为主键建立索引查询效率低，常见优化方案为“转化为两个uint64整数存储”或者“折半存储”（折半后不能保证唯一性）

### 取当前毫秒数

uuid是一个本地算法，生成性能高，但无法保证趋势递增，且作为字符串ID检索效率低，有没有一种能保证递增的本地算法呢？- 取当前毫秒数是一种常见方案。（搜索公众号Java知音，回复“2021”，送你一份Java面试题宝典）

**优点：**

- 本地生成ID，不需要进行远程调用，时延低
- 生成的ID趋势递增
- 生成的ID是整数，建立索引后查询效率高

**缺点：**

- 如果并发量超过1000，会生成重复的ID
- 这个缺点要了命了，不能保证ID的唯一性。当然，使用微秒可以降低冲突概率，但每秒最多只能生成1000000个ID，再多的话就一定会冲突了，所以使用微秒并不从根本上解决问题。

### 使用 Redis 来生成 id

当使用数据库来生成ID性能不够要求的时候，我们可以尝试使用Redis来生成ID。这主要依赖于Redis是单线程的，所以也可以用生成全局唯一的ID。可以用Redis的原子操作 INCR 和 INCRBY 来实现。

**优点：**

- 依赖于数据库，灵活方便，且性能优于数据库。
- 数字ID天然排序，对分页或者需要排序的结果很有帮助。

**缺点：**

- 如果系统中没有Redis，还需要引入新的组件，增加系统复杂度。
- 需要编码和配置的工作量比较大。

### Twitter 开源的 Snowflake 算法

snowflake 是 twitter 开源的分布式ID生成算法，其核心思想为，一个long型的ID：

- 41 bit 作为毫秒数 - 41位的长度可以使用69年
- 10 bit 作为机器编号 （5个bit是数据中心，5个bit的机器ID） - 10位的长度最多支持部署1024个节点
- 12 bit 作为毫秒内序列号 - 12位的计数顺序号支持每个节点每毫秒产生4096个ID序号

![Image](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbudlL2EP3TLE6MKFNCg3bJwqoQPdvS7rxC9Z3VG6wryFPzah882ZqyIgfK9eoQsHuq9SQWdJYnibhAg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)Snowflake图示

算法单机每秒内理论上最多可以生成1000*(2^12)，也就是400W的ID，完全能满足业务的需求。

该算法 java 版本的实现代码如下：

```
public class SnowflakeIdGenerator {
   
    // 系统开始时间截 (UTC 2017-06-28 00:00:00)
    private final long startTime = 1498608000000L;
    // 机器id所占的位数
    private final long workerIdBits = 5L;
    // 数据标识id所占的位数
    private final long dataCenterIdBits = 5L;
    // 支持的最大机器id(十进制)，结果是31 (这个移位算法可以很快的计算出几位二进制数所能表示的最大十进制数)
    // -1L 左移 5位 (worker id 所占位数) 即 5位二进制所能获得的最大十进制数 - 31
    private final long maxWorkerId = -1L ^ (-1L << workerIdBits);
    // 支持的最大数据标识id - 31
    private final long maxDataCenterId = -1L ^ (-1L << dataCenterIdBits);
    // 序列在id中占的位数
    private final long sequenceBits = 12L;
    // 机器ID 左移位数 - 12 (即末 sequence 所占用的位数)
    private final long workerIdMoveBits = sequenceBits;
    // 数据标识id 左移位数 - 17(12+5)
    private final long dataCenterIdMoveBits = sequenceBits + workerIdBits;
    // 时间截向 左移位数 - 22(5+5+12)
    private final long timestampMoveBits = sequenceBits + workerIdBits + dataCenterIdBits;
    // 生成序列的掩码(12位所对应的最大整数值)，这里为4095 (0b111111111111=0xfff=4095)
    private final long sequenceMask = -1L ^ (-1L << sequenceBits);
   
    /**
     * 工作机器ID(0~31)
     */
    private long workerId;
    /**
     * 数据中心ID(0~31)
     */
    private long dataCenterId;
    /**
     * 毫秒内序列(0~4095)
     */
    private long sequence = 0L;
    /**
     * 上次生成ID的时间截
     */
    private long lastTimestamp = -1L;
 
    /**
     * 构造函数
     *
     * @param workerId     工作ID (0~31)
     * @param dataCenterId 数据中心ID (0~31)
     */
    public SnowflakeIdGenerator(long workerId, long dataCenterId) {
        if (workerId > maxWorkerId || workerId < 0) {
            throw new IllegalArgumentException(String.format("Worker Id can't be greater than %d or less than 0", maxWorkerId));
        }
        if (dataCenterId > maxDataCenterId || dataCenterId < 0) {
            throw new IllegalArgumentException(String.format("DataCenter Id can't be greater than %d or less than 0", maxDataCenterId));
        }
        this.workerId = workerId;
        this.dataCenterId = dataCenterId;
    }
    // ==================================================Methods========================================================
    // 线程安全的获得下一个 ID 的方法
    public synchronized long nextId() {
        long timestamp = currentTime();
        //如果当前时间小于上一次ID生成的时间戳: 说明系统时钟回退过 - 这个时候应当抛出异常
        if (timestamp < lastTimestamp) {
            throw new RuntimeException(
                    String.format("Clock moved backwards.  Refusing to generate id for %d milliseconds", lastTimestamp - timestamp));
        }
        //如果是同一时间生成的，则进行毫秒内序列
        if (lastTimestamp == timestamp) {
            sequence = (sequence + 1) & sequenceMask;
            //毫秒内序列溢出 即 序列 > 4095
            if (sequence == 0) {
                //阻塞到下一个毫秒,获得新的时间戳
                timestamp = blockTillNextMillis(lastTimestamp);
            }
        }
        //时间戳改变，毫秒内序列重置
        else {
            sequence = 0L;
        }
        //上次生成ID的时间截
        lastTimestamp = timestamp;
        //移位并通过或运算拼到一起组成64位的ID
        return ((timestamp - startTime) << timestampMoveBits) //
                | (dataCenterId << dataCenterIdMoveBits) //
                | (workerId << workerIdMoveBits) //
                | sequence;
    }
    // 阻塞到下一个毫秒 即 直到获得新的时间戳
    protected long blockTillNextMillis(long lastTimestamp) {
        long timestamp = currentTime();
        while (timestamp <= lastTimestamp) {
            timestamp = currentTime();
        }
        return timestamp;
    }
    // 获得以毫秒为单位的当前时间
    protected long currentTime() {
        return System.currentTimeMillis();
    }
    //====================================================Test Case=====================================================
    public static void main(String[] args) {
        SnowflakeIdGenerator idWorker = new SnowflakeIdGenerator(0, 0);
        for (int i = 0; i < 100; i++) {
            long id = idWorker.nextId();
            //System.out.println(Long.toBinaryString(id));
            System.out.println(id);
        }
    }
}
```