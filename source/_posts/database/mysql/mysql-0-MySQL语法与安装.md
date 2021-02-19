---
title: mysql-MySQL语法与安装
date: 2021-02-19 16:52:09
tags:
---

# MYSQL安装



查看MySQL软件

```shell
yum remove -y mysql mysql-libs mysql-common #卸载mysql
rm -rf /var/lib/mysql #删除mysql下的数据文件
rm /etc/my.cnf #删除mysql配置文件
yum remove -y mysql-community-release-el6-5.noarch #删除组件
```

安装MySQL

```shell
#下载rpm文件
wget http://repo.mysql.com/mysql-community-release-el6-5.noarch.rpm #执行rpm源文件
rpm -ivh mysql-community-release-el6-5.noarch.rpm
#执行安装文件
yum install mysql-community-server
```

启动MySQL

```shell
systemctl start mysqld
```

设置root用户密码
例如:为 root 账号设置密码为 root :

```shell
/usr/bin/mysqladmin -u root password 'root'
#没有密码 有原来的密码则加
/usr/bin/mysqladmin -u root -p '123' password 'root'
```



登录mysql

```
mysql -uroot -proot
```

配置mysql

```
vim /etc/my.cnf
```

MySQL远程连接授权

```mysql
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'root' WITH GRANT OPTION;
FLUSH PRIVILEGES;--刷新权限
```

- ALL PRIVILEGES :表示授予所有的权限，此处可以指定具体的授权权限。
- *.* :表示所有库中的所有表
- 'root'@'%' : myuser是数据库的用户名，%表示是任意ip地址，可以指定具体ip地址。
-  IDENTIFIED BY 'mypassword' :mypassword是数据库的密码。



# **DDL**语句

创建数据库

```
create database 数据库名 character set 字符集;
```

...

# DML语句

更新记录、插入、删除:update

```
update 表名 set 字段名=值,字段名=值 where 条件;
```

# DQL语句

查询

```
select pname,price from product;
```

**聚合函数(组函数)**

  特点:只对单列进行操作

常用的聚合函数:

```text
sum():求某一列的和 
avg():求某一列的平均值 
max():求某一列的最大值 
min():求某一列的最小值 
count():求某一列的元素个数
```



# SQL解析顺序

```mysql
-- 行过滤
FROM <left_table>
ON <join_condition>
<join_type> JOIN <right_table> 
WHERE <where_condition>
GROUP BY <group_by_list>
HAVING <having_condition> 

--列过滤
SELECT
DISTINCT <select_list>

--排序
9 ORDER BY <order_by_condition> -- MySQL附加
10 LIMIT <limit_number>
```



- 如果使用 LEFT JOIN ，则主表在它左边
-  如果使用 RIGHT JOIN ，则主表在它右边
- 查询结果以主表为主，从表记录匹配不到，则补 null

```mysql
FROM(将最近的两张表，进行笛卡尔积)---VT1
ON(将VT1按照它的条件进行过滤)---VT2
LEFT JOIN(保留左表的记录)---VT3
WHERE(过滤VT3中的记录)--VT4...VTn
GROUP BY(对VT4的记录进行分组)---VT5
HAVING(对VT5中的记录进行过滤)---VT6
SELECT(对VT6中的记录，选取指定的列)--VT7
ORDER BY(对VT7的记录进行排序)--VT8
LIMIT(对排序之后的值进行分页)--MySQL特有的语法
```

