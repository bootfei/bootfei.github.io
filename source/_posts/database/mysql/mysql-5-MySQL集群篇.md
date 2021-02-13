---
title: mysql-5-MySQL集群篇
date: 2021-02-13 12:17:56
tags:
---

# **集群搭建之主从复制**

## **主从复制原理**

![img](https://img-blog.csdn.net/20180313225542529)

- 主从机制下，主机负责读写，从机只负责读或者禁止读。
  后面的读写分离，其实就是为了优化主机的性能，让主机只负责写，从机负责读。
- 主机
  - bin log:
    - bin log是由MySQL server层去记录处理的 数据修改并进行事务提交之后，会写入binlog日志。
    - binlog是不断增大的，如果不清理，他是一直存在的。
    - bin log一般都建议打开。因为它存储了整个表的数据变更操作。
  -  log dump 线程：主机有binlog日志了之后，生成 log dump 线程，去将binlog日志远程发送给从机
- 从机
  - 有两个重要线程:
    - IO线程 ，是一直去请求主机，去要binlog日志。
    - SQL线程，SQL线程从relay log里面去读取数据，然后在从机里面的MySQL服务器执行相应的语句。
  - relay log:中继日志，类似于接力
     

## binlog介绍和relay日志

查看bin log和relay log日志:

```
mysqlbinlog --base64-output=decode-rows -v -v mysql-bin.000058 > binlog
```

## **主从复制实践**

###  **关闭主从机器的防火墙**

```shell
systemctl stop iptables(需要安装iptables服务) 
systemctl stop firewalld(默认)
systemctl disable firewalld.service(设置开启不启动)
```

### **主服务器配置** 

第一步:修改my.conf文件

在[mysqld]段下添加:

```shell
log-bin=mysql-bin #启用二进制日志
server-id=133 #服务器唯一ID，一般取IP最后一段
binlog-do-db=kkb2 #指定复制的数据库(可选) 
replicate-ignore-db=kkb #指定不复制的数据库(可选,，mysql5.7) 
replicate-ignore-table=db.table1 #指定忽略的表(可选，mysql5.7) 
```

**第二步:重启**mysql服务

```
systemctl  restart  mysqld
```

第三步:主机给从机授备份权限

注意:先要登录到MySQL命令客户端

```
mysql>GRANT REPLICATION SLAVE ON *.* TO '从机MySQL用户名'@'从机IP' identified by '从机MySQL密码';

---示例
GRANT REPLICATION SLAVE ON *.* TO 'root'@'%' identified by '123456'; 

---注意事项:
一般不用root帐号，“%”表示所有客户端都可能连，只要帐号，密码正确，此处可用具体客户端IP代替，如192.168.145.226，加强安全。
```

**第四步:刷新权限**

```
mysql> FLUSH PRIVILEGES;
```

**第五步:查询**master的状态

```
mysql> show master status;
```

### **从服务器配置** 

**第一步:修改**my.conf文件

```
[mysqld]
server-id=135
```

**第二步:删除**UUID

文件 如果出现此错误:

```
fatal error: The slave I/O thread stops because master and slave have equal
MySQL server UUIDs; these UUIDs must be different for replication to work.
```

因为是mysql是克隆的系统所以mysql的uuid是一样的，所以需要修改。

解决方案：

```
删除/var/lib/mysql/auto.cnf文件，重新启动MySQL服务。
```

