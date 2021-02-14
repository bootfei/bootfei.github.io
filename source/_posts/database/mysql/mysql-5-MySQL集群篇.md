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

### **主服务器配置** 

 **关闭主从机器的防火墙**

```shell
systemctl stop iptables(需要安装iptables服务) 
systemctl stop firewalld(默认)
systemctl disable firewalld.service(设置开启不启动)
```



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

解决方案：删除[/var/lib/mysql/auto.cnf](/var/lib/mysql/auto.cnf)文件，重新启动MySQL服务。

第三步:重启并登录到MySQL进行配置从服务器

```mysql
mysql>change master to
 master_host='192.168.211.128',
 master_port=3306,
 master_user='root',
 master_password='123456',
 master_log_file='mysql-bin.000001',
 master_log_pos=397
```

> 语句中间不要断开， master_port 为mysql服务器端口号(**无引号**)， master_user 为执行同步操作的数据库账户， “410” 无单引号(此处的 410 就是 show master status 中看到的 position 的值，这里的mysql-bin.000001 就是 file 对应的值)。

第四步:启动从服务器复制功能

```
mysql>start slave;
```

第五步:检查从服务器复制功能状态

```mysql
mysql> show slave status \G;
........................(省略部分)
Slave_IO_Running: Yes //此状态必须YES 
Slave_SQL_Running: Yes //此状态必须YES 
........................(省略部分)
```



## 主从同步延迟的原因及解决办法

mysql 用主从同步的方法进行读写分离，减轻主服务器的压力的做法现在在业内做的非常普遍。 主从同 步基本上能做到实时同步。

在配置好了， 主从同步以后， 主服务器会把更新语句写入binlog, 从服务器的IO 线程(这里要注意， 5.6.3 之前的IO线程仅有一个，5.6.3之后的有多线程去读了，速度自然也就加快了)回去读取主服务器的 binlog 并且写到从服务器的Relay log 里面，然后从服务器的 的SQL thread 会一个一个执行 relay log 里面的sql ， 进行数据恢复。

**1**、主从同步的延迟的原因

一个服务器开放N个链接给客户端来连接的， 这样有会有大并发的更新操作, 但是从服务器的里面读取binlog 的线程仅有一个， 当某个SQL在从服务器上执行的时间稍长 或者由于某个SQL要 进行锁表就会导致，主服务器的SQL大量积压，未被同步到从服务器里。这就导致了主从不一致， 也就 是主从延迟。



**2**、主从同步延迟的解决办法

实际上主从同步延迟根本没有什么一招制敌的办法， 因为所有的SQL必须都要在从服务器里面执行一 遍，但是主服务器如果不断的有更新操作源源不断的写入， 那么一旦有延迟产生， 那么延迟加重的可 能性就会原来越大。 当然我们可以做一些缓解的措施。

a. 我们知道因为主服务器要负责更新操作， 他对安全性的要求比从服务器高， 所有有些设置可以修 改，比如[sync_binlog=1，innodb_flush_log_at_trx_commit = 1](https://blog.csdn.net/sinat_37903468/article/details/108469482) 之类的设置，而slave则不需要这么高 的数据安全，完全可以讲sync_binlog设置为0或者关闭binlog，innodb_flushlog， innodb_flush_log_at_trx_commit 也 可以设置为0来提高sql的执行效率 这个能很大程度上提高效率。 另外就是使用比主库更好的硬件设备作为slave。

b. 就是把，一台从服务器当度作为备份使用， 而不提供查询， 那边他的负载下来了， 执行relay log 里 面的SQL效率自然就高了。

c. 增加从服务器喽，这个目的还是分散读的压力， 从而降低服务器负载。



**3**、判断主从延迟的方法

MySQL提供了从服务器状态命令，可以通过 [show slave status](https://blog.csdn.net/sinat_37903468/article/details/108469482) 进行查看， 比如可以看看 Seconds_Behind_Master参数的值来判断，是否有发生主从延时。
 其值有这么几种:
 NULL - 表示io_thread或是sql_thread有任何一个发生故障，也就是该线程的Running状态是No,而非 Yes.0 - 该值为零，是我们极为渴望看到的情况，表示主从复制状态正常





# 集群搭建之读写分离

读写分离的理解

> MySQL的主从复制，只会保证主机对外提供服务，而从机是不对外提供服务的，只是在后台为主机进行备 份。
>
> MySQL的读写分离，只有主机提供更新操作，从机提供读操作

需要使用mysql-proxy操作

## **读写分离演示需求**

```
MySQL master:135 
MySQL slave :136 
MySQL proxy :137
```

## **MySQL-Proxy**安装

```shell
wget https://downloads.mysql.com/archives/get/file/mysql-proxy-0.8.5-linux-el6-x86-64bit.tar.gz
tar -xf mysql-proxy-0.8.5-linux-el6-x86-64bit.tar.gz -C /kkb
```

## **MySQL-Proxy**配置

创建mysql-proxy.cnf文件

```
[mysql-proxy]
user=root
admin-username=root
admin-password=root
proxy-address=192.168.10.137:4040
proxy-backend-addresses=192.168.10.135:3306
proxy-read-only-backend-addresses=192.168.10.136:3306
proxy-lua-script=/root/mysql-proxy/share/doc/mysql-proxy/rw-splitting.lua
log-file=/root/mysql-proxy/logs/mysql-proxy.log
log-level=debug
keepalive=true
daemon=true
```

修改mysql-proxy.cnf文件的权限

```
chmod 660 mysql-proxy.cnf
```

修改rw-splitting.lua脚本

```
max-idle-connection=2
```

## MySQL-Proxy启动域测试

- 启动命令

```
./mysql-proxy --defaults-file=mysql-proxy.cnf配置文件的地址

如果没有配置profile文件的环境变量，则需要去拥有mysql-proxy命令的目录通过./mysql-proxy 进行启动。
```

- 在其他客户端，通过mysql命令去连接MySQL Proxy机器

  - ```
    mysql -uroot -proot -h192.168.10.134 -P4040
    ```

    

  