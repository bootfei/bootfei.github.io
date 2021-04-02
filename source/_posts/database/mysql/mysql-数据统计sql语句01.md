---
title: 'mysql:工作中常用的组合sql语句'
date: 2020-11-23 19:03:17
tags:
---



它山之石可以攻玉

### ref links:

1. 删除重复数据，只保留id最小的数据
   https://juejin.cn/post/6844903582114775053











# sql分析电商数据

本次使用sql进行xxx电商数据实战，来源于xxx的脱敏数据（数据相对多，部分字段未使用）。

数据介绍：

数据存放在9张表中，大致可以分为3类型：用户表（3）订单表（3）商品表（3），ER关系图如下：

![img](https://pic4.zhimg.com/80/v2-0f72fab10c5d8b06f582df2d0d48601f_1440w.jpg)仅包含部分关键字段

注意点：orderinfo 或者 useraddress表 province、city、district字段（外键）都来自

regioninfo表 regionid字段。

一、提出问题

客户角度：

1、对于店铺营业额贡献大的客户是哪些？购买数量最多的客户有哪些？

- 1.0 求出购买产品的前十名顾客（金额最多）
- 1.1、求出购买产品的前10名顾客（数量最多）

2、哪些客户留存率高，购买次数多？

- 2.0、求出购买次数最多的前10名顾客
- 2.1 求出购买产品金额最多的前十名顾客的最后登录时间



地区角度：

3、哪些城市购买力最强？

- 3.0 求出购买力最强的前十个城市
- 3.1 求出购买力最强的前十个城市以及他们所在的省份



商品角度：

4、畅销商品、畅销品牌、畅销颜色是什么？

- 4.0 求出最畅销的十个品牌（按照总销售额OR总销量）
- 4.1 求出最畅销的十种颜色
- 4.2 求出最畅销的十个商品 所属的品牌中 各个不同尺码的销售额(分步写)



二、理解数据、数据清洗

数据存在在csv中，需导入到数据库；导入数据后数据清洗，主要处理时间。（from_unixtime 将时间戳转化成日期，date_format 将文本型日期转化成日期型日期）

注意点：

update userinfo set lastlogin_ = from_unixtime(lastlogin);

pt varchar（9）导入数据，如果最后一个字段是文本型，一定要记得留一位给换行符，

那么原本8位，+1成9位，取最后文本型字段的前八位字符，使用 substring

update regioninfo set pt = substring(pt,1,8);

update regioninfo set pt_ = date_format(pt,'%y-%m-%d');

（以下直接通过mysql 5.6 客户端操作，据说使用navicat导入csv特别方便，不过我没试过。）

create database ds;

use ds;



四、构建模型（以下使用navicat操作）

客户角度：

## 求出购买产品金额最多的前十名顾客

```sql
SELECT orderinfo.UserID,sum(OrderAmount) from orderinfo LEFT JOIN userinfo

on orderinfo.UserID = userinfo.userid

GROUP BY orderinfo.UserID ORDER BY sum(OrderAmount) desc LIMIT 10 ;
```

结论：购买产品金额前10的客户，存在极大值：24k；2位客户在7k附近；其余客户大部分在3.5-5k。



### 求出购买产品数量最多的前10名顾客

```
SELECT UserID,sum(Amount) FROM orderdetail GROUP BY UserID ORDER BY sum(Amount) DESC LIMIT 10 ;
```

结论：1、2-8原则，20%的客户贡献了80%的利润，1.1的客户在1.0查询结果都未出现，虽然购买产品多，但是金额不是相对多。2、前5位客户数量购买都过100，最高1456，个人猜测应该是为为团体进行购买，可以与客户进行私下联系，保持业务往来。



### 求出购买次数最多的前10名顾客

```sql
SELECT orderinfo.UserID,count(OrderID) from orderinfo LEFT JOIN userinfo on

orderinfo.UserID = userinfo.userid

GROUP BY orderinfo.UserID ORDER BY count(OrderID) desc LIMIT 10 ;
```

结论：客户回头率最高是17，前10最低是4，这些都属于忠实客户，让我想到RFM模式，但是我还不会用RFM。



## 求出购买产品金额最多的前十名顾客的最后登录时间

```sql
SELECT orderinfo.UserID,username,sum(OrderAmount),userinfo.lastlogin_ from orderinfo LEFT JOIN userinfo on orderinfo.UserID = userinfo.userid

where username is not NULL

GROUP BY orderinfo.UserID ORDER BY sum(OrderAmount) desc LIMIT 10 ;
```

结论：lastlogin时间竟然都是20170601，区别在于这些客户都爱在12:00至pm登录，只有一位8：39 am。

个人推断：如果统计出所有的客户最后登录时间，假设发现依旧下午居多，那么可以加大下午广告投放力度，促进营销。



拓展：（以下sql未写，只是个人思考）

1、对所有客户的购买金额进行统计分段，查看哪个段位的比较多，了解客户整体大概情况。

1、userinfo表存在regtime（注册时间）、lastlogin（最后登录时间）可以分析客户留存。

2、RFM模式：R(Recency)表示客户最近一次购买的时间有多远，F(Frequency)表示客户在最近一段时间内购买的次数，M(Monetary)表示客户在最近一段时间内购买的金额；分别进行查询。利用RFM分析，我们可以做以下几件事情：

（1）建立会员金字塔，区分各个级别的会员，如高级会员、中级会员、低级会员，然后针对不同级别的会员施行不同的营销策略，制定不同的营销活动。

（2）发现流失及休眠会员，通过对流失及休眠会员的及时发现，采取营销活动，激活这些会员。

（3）在短信、EDM促销中，可以利用模型，选取最优会员。

（4）维系老客户，提高会员的忠诚度。

使用方法：可以给三个变量不同的权重或按一定的规则进行分组，然后组合使用，即可分出很多不同级别的会员。



地区角度：

## 求出总购买力最强的前十个城市

```sql
SELECT City,regionname ,SUM(OrderAmount) from orderinfo

LEFT JOIN regioninfo on orderinfo.City= regioninfo.regionid

GROUP BY City ORDER BY SUM(OrderAmount) desc LIMIT 10 ;
```

结论：1、25000以上：石家庄，广州；10000-20000:5个城市；<10000:3个城市。石家庄和广州都是物流枢纽中心，贸易发达。2、10个城市中，北方3个，南方7个，可见南方对于营业额的贡献占比多。



### 求出购买力最强的前十个城市以及他们所在的省份（其实和2一样，只是为了锻炼sql语句）

-- 城市id 和 省份id 对应的具体名字都在regioninfo表，所以套一次子查询

```
SELECT citytop10.城市名, citytop10.金额, regioninfo.regionname as 省份名

from

(SELECT regionname as 城市名,SUM(OrderAmount) as 金额,Province from orderinfo

LEFT JOIN regioninfo on orderinfo.City= regioninfo.regionid

GROUP BY City ORDER BY SUM(OrderAmount) desc LIMIT 10) as citytop10

LEFT JOIN regioninfo on citytop10.Province=regioninfo.regionid

order by 金额 DESC;
```

拓展：（以下sql未写，只是个人思考）

1、可以按照省份、区域划分，例如：各个省份的销售额是多少？每个省的各个区销售额多少？按照时间是如何变化的？是否有规律可循，例如：天气、季节........？

2、每个省、每个区域购买商品数量最多或者销售额分布情况？



商品角度：

## 求出最畅销的十个品牌（畅销定义：商品卖的总销售额或者总销量）

### 按照总销量

```sql
SELECT BrandType,sum(orderdetail.Amount)

FROM goodsinfo INNER JOIN orderdetail ON goodsinfo.goodsid=orderdetail.GoodsID

INNER JOIN goodsbrand on goodsinfo.typeid=goodsbrand.SupplierID

GROUP BY goodsinfo.typeid

order by sum(orderdetail.Amount) DESC LIMIT 10;
```

### 按照总销售额

```
SELECT BrandType as 品牌,sum(orderdetail.GoodsPrice*Amount) as 金额

FROM goodsinfo INNER JOIN orderdetail ON goodsinfo.goodsid=orderdetail.GoodsID

INNER JOIN goodsbrand on goodsinfo.typeid=goodsbrand.SupplierID

GROUP BY goodsinfo.typeid

order by sum(orderdetail.GoodsPrice*Amount) DESC LIMIT 10;       
```

结论：畅销品牌按照销售量OR销售金额排序：前7名完全一致，第一名：伊妮儿，销量：2975，销售额：179749；第七名：BKSY，销量：400，销售额：24175。第一名销量OR销售额是第二名的2倍，是主打品牌。

第8、9、10名，ABOUT ME销量第8，但是销售额未进前10，可以适当提高ABOUT ME商品单价；DOUBLE 7销售额第10，但是销量未进前10，可以降低DOUBLE 7E商品单价。

### 求出最畅销的十种颜色

-- 总数量

SELECT goodscolor.ColorNote ,SUM(orderdetail.Amount)

FROM orderdetail LEFT JOIN goodscolor ON orderdetail.ColorID=goodscolor.ColorID

GROUP BY orderdetail.ColorID

order by SUM(orderdetail.Amount) deSC LIMIT 10;

结论：黑色OR白色果然是百搭款，销量稳占前2；蓝色与粉色紧随其后，推测该店铺应该女装居多。黄色销量最少，近280。



### 求出最畅销的十个商品 所属的品牌中各个不同尺码的销售额(锻炼sql，分步写，思路清晰)

4.4.1、先查到10个畅销商品所属于的品牌

```
CREATE table top10brand(

SELECT typeid from orderdetail left JOIN goodsinfo

on orderdetail.GoodsID= goodsinfo.goodsid

GROUP BY orderdetail.GoodsID

ORDER BY sum(GoodsPrice*Amount) desc LIMIT 10

);
```

SELECT * from top10brand;

4.4.2、内连接各个表

```
SELECT orderdetail.SizeID,sum(GoodsPrice*Amount) from goodsinfo

INNER JOIN top10brand on top10brand.typeid=goodsinfo.typeid

INNER JOIN orderdetail ON orderdetail.GoodsID = goodsinfo.goodsid

GROUP BY orderdetail.SizeID

ORDER BY sum(GoodsPrice*Amount) desc;
```

