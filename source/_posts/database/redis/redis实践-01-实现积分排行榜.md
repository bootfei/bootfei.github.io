---
title: redis实践-01-实现积分排行榜
date: 2021-06-15 08:22:22
tags:
---

ZSet数据结构就很适合用于实现排行榜，但如何实现相同积分情况下再支持按时间排序呢？

## 实现思路

> 分布式ID雪花算法的原理，即用一个long类型变量存储多个信息。一个long类型长度为8个字节（64bit），雪花算法使用其中41bit记录时间戳，其余bit位存储机房id、机器id、序列号。

Redis的ZSet支持分值为double类型，也是8字节，那么我们也可以使用41位存储时间戳，其实位存储用户的实际积分。

> 在雪花算法中最高位是不用的，目的是不允许生成负数ID

而在实现排行榜中没有这个限制，因为我们最终要的只是用户的积分，而不是加上时间戳的分值。只不过ZSet已经支持倒序获取，最高位是0是1无所谓，所以最高位我们依然不使用。

除去最高位和存储时间戳的41位后，剩余22位表示积分，这时我们还需要结合业务考虑，如果觉得22bit不够表示积分，那么还可以继续压缩时间戳占用的bit。

由于排行榜是周期性的，如周榜、月榜，所以我们没必要存储完整的时间戳，可以取当前时间与周期开始时间相差的毫秒数，这样就可以将41bit压缩到32bit、16bit、或者更低

如果是用41bit表示时间戳，22bit表示积分的话，那么score的组成就是这样的：
0（最高位不用）`|`0000000 00000000 0000000（22bit表示积分）`|`0
00000000 00000000 00000000 00000000 00000000（41bit表示时间戳）

因为排序首先按积分排再按时间排，所以积分在高位，时间戳在低位。

在积分相同的情况下，是不是时间戳越大64bit表示的数值就越大？而我们需要的是按时间升序排，也就是最先达到xx积分的用户排在最前面，所以我们不能单纯的使用41bit存储时间戳，而应该是存储一个随时间流逝而变小的数值。

由于排行榜都会有一个周期，如周榜是一周，月榜是一个月，所以我们使用41bit存储的是一个周期的结束时间`yyy-MM-dd 23:59:59`对应的时间戳与用户积分更新时间的时间戳的差值，这个值会随着时间的推移而变小，而且不会出现负数的情况，刚好能够达到目的。

## 实现

1.实现积分+时间戳差值转score

```
// periodEndTimestamp: 当前周期结束时间的时间戳 
// 需确保point不会超过22bit所能表示的数值：2097151
private static long toScore(int point, long periodEndTimestamp) {
    long score = 0L;
    score = (score | point) << 41;
    score = score | (periodEndTimestamp - TimestampUtils.currentTimeMillis());
    return score;
}
```

2.实现从score中获取积分

```
private static int getPoint(long score) {
     return (int) (score >> 41);
}
```

3.更新积分

```
@Override
public void updateRanking(Integer periodId, Long accountId, Integer addPoint) {
    String key = String.format(RankingCacheKeys.REALTIME_POINT_RANKING_KEY, periodId);
    Double score = redisTemplate.opsForZSet().score(key, String.valueOf(accountId));
    score = (score == null) ? 0d : score;
    int curPoint = getPoint(score.longValue());
    long newScore = toScore(curPoint + addPoint, getCurPeriodEndDateTimestamp(periodId));
    redisTemplate.opsForZSet().add(key, String.valueOf(accountId), newScore);
}
```

## 总结

基于Redis ZSet实现积分排行榜（倒序）并支持按时间（升序）排序原理与注意事项：

- 先按积分排序，再按时间排序，所以需要高位存储积分，低位存储时间戳，这样才能保证积分越高对应score越大；
- 同分值情况下按时间升序排序，必然让达到当前积分时间最早的score越大；
- 由于每次更新用户积分都需要重新计算score，updateRanking方法不能保证ZSet读和写复合操作命令的原子性，因此可能存在并发数据一致性问题

## 关于实现排行榜用到ZSet的几个命令

1.获取倒序排名：

```
ZREVRANGEBYSCORE key min max offset count
```

2.获取某个用户的score：

```
ZSCORE key member
```

3.获取参与排名的总用户数（本期参与人数）：

```
ZCARD key
```

4.获取某个用户的当前排名(倒序排序，从大到小的排名)：

```shell
ZREVRANK key member
```