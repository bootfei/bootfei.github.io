---
title: 'mysql:工作中常用的组合sql语句'
date: 2020-11-23 19:03:17
tags:
---



### 统计销量最高的员工

```sql
SELECT orderinfo.UserID,sum(OrderAmount) from orderinfo LEFT JOIN userinfo

on orderinfo.UserID = userinfo.userid

GROUP BY orderinfo.UserID ORDER BY sum(OrderAmount) desc LIMIT 10 ;
```

先查到10个畅销商品所属于的品牌

```mysql
CREATE table top10brand(

  SELECT typeid from orderdetail left JOIN goodsinfo

  on orderdetail.GoodsID= goodsinfo.goodsid

  GROUP BY orderdetail.GoodsID

  ORDER BY sum(GoodsPrice*Amount) desc LIMIT 10

);

SELECT * from top10brand;
```



内连接各个表

```mysql
SELECT orderdetail.SizeID,sum(GoodsPrice*Amount) from goodsinfo

  INNER JOIN top10brand on top10brand.typeid=goodsinfo.typeid

  INNER JOIN orderdetail ON orderdetail.GoodsID = goodsinfo.goodsid

GROUP BY orderdetail.SizeID

ORDER BY sum(GoodsPrice*Amount) desc;
```





### select结果update到表中

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



### select结果insert到表中

```mysql

A中有3例，B表中你只能获得2列，可以用常量占位解决
insert into tableA (列1，列2，列3) select 列1，列2，常量  from tableB WHERE 条件表达式;
```





### Get Last Record In Each Group

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



### Combine results of 2 queries to multiple columns

```
非常基础的sql

select a.id, bi.name from (select id from a) a, (select name from b) b
```



### join on 多个条件中的任意一个即可

What I would need is something like this:

```sql
Select * from Table1 as t1     
LEFT JOIN Table2 as t2 
on t1.checkcode = (t2.checkcode1 OR t2.checkcode2)     
```

Try like this,

```sql
Select * from Table1 as t1  
LEFT JOIN Table2 as t2 on t1.checkcode = t2.checkcode1 OR t1.checkcode = t2.checkcode2
```



### 排序以后rank

注意：千万不要有where语句

```
select @rank := @rank + 1
from t1, (select @rank :=0) rank
order by t1.age desc
```

