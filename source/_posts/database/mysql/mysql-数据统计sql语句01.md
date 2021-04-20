---
title: 'mysql:工作中常用的组合sql语句'
date: 2020-11-23 19:03:17
tags:
---



# 统计销量最高的员工

```sql
SELECT orderinfo.UserID,sum(OrderAmount) from orderinfo LEFT JOIN userinfo

on orderinfo.UserID = userinfo.userid

GROUP BY orderinfo.UserID ORDER BY sum(OrderAmount) desc LIMIT 10 ;
```

求出最畅销的十个商品 所属的品牌中各个不同尺码的销售额(锻炼sql，分步写，思路清晰)

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





# select结果update到表中

```mysql
UPDATE sale
INNER JOIN (
	SELECT
		sale.FNo,
		sale.FEntryID,
		(finishin.FQty) AS qty
	FROM
		sale,
		finishin
	WHERE
		sale.FNo = finishin.FNo
	AND sale.FEntryID = finishin.FEntryID
	ORDER BY
		sale.FNo
) sale2 ON sale2.FNo = sale.FNo
AND sale2.FEntryID = sale.FEntryID
SET sale.FqtyIn = sale2.qty
```



# select结果insert到表中

```mysql

A中有3例，B表中你只能获得2列，可以用常量占位解决
insert into tableA (列1，列2，列3) select 列1，列2，常量  from tableB WHERE 条件表达式;
```





# Get Last Record In Each Group

**我现在需要取出每个分类中最新的内容**
select * from test group by category_id order by `date`
结果如下
![这里写图片描述](https://img-blog.csdn.net/20160830142144744)
明显。这不是我想要的数据，原因是msyql已经的执行顺序是：

> 写的顺序：select … from… where…. group by… having… order by..
> 执行顺序：from… where…group by… having…. select … order by…

所以在order by拿到的结果里已经是分组的完的最后结果。



```
select product_sales.* from product_sales,
           (select product,max(order_date) as order_date
                from product_sales
                group by product) max_sales
             where product_sales.product=max_sales.product
             and product_sales.order_date=max_sales.order_date;
```

This will return the posts with the latest record in each group.



# Combine results of 2 queries to multiple columns

```
非常基础的sql

select a.id, bi.name from (select id from a) a, (select name from b) b
```

