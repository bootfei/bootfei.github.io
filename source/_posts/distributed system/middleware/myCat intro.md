---
title: mycat intro
date: 2021-01-16 13:54:56
tags: []
Categories: [distributed system]
---



# MyCat核心概念

![](https://upload-images.jianshu.io/upload_images/13553988-2a982caabfdef4bb.png?imageMogr2/auto-orient/strip|imageView2/2/w/664/format/webp)

从映射的角度理解就非常简单

- schema: myCat的逻辑数据库（相当于mySql的database)
- table: 相当于mySql的table
- dataNode: 存储数据的物理节点,  其实本质就是引导客户端session找到具体的物理节点， mapping 为 dataHost : dataBase (mySql)
- dataHost: 存储节点所在的数据库主机

MyCAT支持水平分片与垂直分片：

- 水平分片：一个表格的数据分割到多个节点上，按照行分隔。
  其实就是一张表数据太多了，放不下了，需要多张表存储，多张表在其他database上。
- 垂直分片：一个数据库中多个表格A，B，C，A存储到节点1上，B存储到节点2上，C存储到节点3上。

# MyCat安装

```shell
wget http://dl.mycat.io/1.6-RELEASE/Mycat-server-1.6-RELEASE-20161028204710- linux.tar.gz

tar -zxvf Mycat-server-1.6-RELEASE-20161028204710-linux.tar.gz

- 启动命令：./mycat start - 停止命令：./mycat stop - 重启命令：./mycat restart - 查看状态：./mycat status

#使用mysql的客户端直接连接mycat服务。默认服务端口为【8066】 
mysql -uroot -p123456 -h127.0.0.1 -P8066

```

[如果一直提示密码错误，说明mysql 8.0版本做了密码sha2加密]()

```shell
mysql -uroot -p123456 -P8066 -h192.168.199.229  --default-auth=mysql_native_password
```



# MyCat分片

## 配置文件

### Schema.xml


schema.xml作为Mycat中重要的配置文件之一，管理着Mycat的逻辑库、表、分片规则、DataNode以及DataHost之间的映射关系。弄懂这些配置，是正确使用Mycat的前提。

- schema 标签用于定义MyCat实例中的逻辑库
- Table 标签定义了MyCat中的逻辑表
- dataNode 标签定义了MyCat中的数据节点，也就是我们通常说所的数据分片。
- dataHost标签在mycat逻辑库中也是作为最底层的标签存在，直接定义了具体的数据库实例、读写分离配置和心跳语句。

```xml
<?xml version="1.0"?><!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
	<!--schema : 逻辑库 name:逻辑库名称 sqlMaxLimit：一次取多少条数据  table:逻辑表 dataNode:数据节点 对应datanode标签 rule：分片规则，对应rule.xml subTables:子表 primaryKey：分片主键 可缓存 -->
    <schema name="TESTDB" checkSQLschema="false" sqlMaxLimit="100">
        <table name="item" dataNode="dn1,dn2,dn3" rule="mod-long" primaryKey="ID"/> 
    </schema>
    <dataNode name="dn1" dataHost="localhost1" database="db1" />
    <dataNode name="dn2" dataHost="localhost1" database="db2" />
    <dataNode name="dn3" dataHost="localhost1" database="db3" />
    
	<!--dataHost : 数据主机（节点主机） balance：1 ：读写分离 0 ： 读写不分离 writeType：0 第一个writeHost写， 1 随机writeHost写 dbDriver： 数据库驱动 native：MySQL JDBC：Oracle、SQLServer switchType： 是否主动读 1、主从自动切换 -1 不切换 2 当从机延时超过slaveThreshold值时切换为主读 -->
    <dataHost name="localhost1" maxCon="1000" minCon="10" balance="0" writeType="0" dbType="mysql" dbDriver="native" switchType="1" slaveThreshold="100">
        <heartbeat>select user()</heartbeat>
        <writeHost host="hostM1" url="192.168.24.129:3306" user="root" password="root" ></writeHost>
    </dataHost>
</mycat:schema>
```

### Server.xml



```xml
<?xml version="1.0" encoding="UTF-8"?> 

<!DOCTYPE mycat:server SYSTEM "server.dtd"> 

<mycat:server xmlns:mycat="http://io.mycat/"> 

    <system> 

    	<property name="defaultSqlParser">druidparser</property> 

    </system> 

    <user name="mycat"> 

        <property name="password">mycat</property> 

        <property name="schemas">TESTDB</property> 

    </user> 

</mycat:server>
```



### Rule.xml

rule.xml里面就定义了我们对表进行拆分所涉及到的规则定义。我们可以灵活的对表使用不同的分片算法，或者对表使用相同的算法但具体的参数不同。这个文件里面主要有tableRule和function这两个标签。在具体使用过程中可以按照需求添加tableRule和function。

此配置文件可以不用修改，使用默认即可。

```shell
<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE mycat:rule SYSTEM "rule.dtd"> 

<mycat:rule xmlns:mycat=”http://io.mycat/“ > 

    <tableRule name="sharding-by-intfile"> 

        <rule>

            <columns>sharding_id</columns> 

            <algorithm>hash-int</algorithm> 

        </rule> 

    </tableRule> 

    <function name="hash-int" 

    class="io.mycat.route.function.PartitionByFileMap"> 

    	<property name="mapFile">partition-hash-int.txt</property> 

    </function> 

</mycat:rule>
```

tableRule 标签配置说明：

- **name** 属性指定唯一的名字，用于标识不同的表规则
- **rule** 标签则指定对物理表中的哪一列进行拆分和使用什么路由算法。
- **columns** 内指定要拆分的列名字。
- **algorithm** 使用 function 标签中的 name 属性。连接表规则和具体路由算法。当然，多个表规则可以连接到同一个路由算法上。 table 标签内使用。让逻辑表使用这个规则进行分片。

function 标签配置说明：

- **name** 指定算法的名字。
- **class** 制定路由算法具体的类名字。
- **property** 为具体算法需要用到的一些属性。

## 分片规则

### 连续分片（略）

### 离散分片

#### 一致性hash

参考这篇文章：https://www.huaweicloud.com/articles/1d7c24fb8e8322bb95aa2e632439b823.html

一致性hash 2的32次方 - 1 , 有效解决了分布式数据的扩容问题

### 综合分片（略）



## 实验课：测试分片

### 搭建MySQL测试的集群

我使用docker搭建的3个master,  即3个分片

### 配置MyCat分片

如果docker启动Mycat失败，参考这篇文章https://codingnote.cc/p/299914/

配置schema.xml



```xml
<dataNode name="dn1" dataHost="localhost1" database="db1" /> 

<dataNode name="dn2" dataHost="localhost1" database="db2" /> 

<dataNode name="dn3" dataHost="localhost1" database="db3" /> 

<dataHost name="localhost1" maxCon="1000" minCon="10" balance="1" writeType="0" dbType="mysql" dbDriver="native" switchType="2" 
slaveThreshold="100"> 

<heartbeat>show slave status</heartbeat> 

<writeHost host="hostM" url="192.168.25.134:3306" user="root" password="root"> 

<readHost host="hostS" url="192.168.25.166:3306" user="root" password="root" /> 

</writeHost> 

</dataHost>
```



# Mycat读写分离

MyCat的读写分离是建立在MySQL主从复制基础之上实现的，所以必须先搭建MySQL的主从复制。

## 配置MySQL主从复制

检查MySQL是否开启slave，进行主从复制

```shell
#进入mySql
mysql -uroot -proot
>show slave status
slave IO running: YES
slave SQL running: YES
```



数据库读写分离对于大型系统或者访问量很高的互联网应用来说，是必不可少的一个重要功能。对于MySQL来说，标准的读写分离是主从模式，一个写节点Master后面跟着多个读节点，读节点的数量取决于系统的压力，通常是1-3个读节点的配置