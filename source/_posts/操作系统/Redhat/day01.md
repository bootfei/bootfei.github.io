---
title: 系统日志
date: 2020-11-05 09:24:42
tags: [java, jstack]
---



## 文本日志

日志文件的位置：系统日志保存在/var/log

- /var/log/messages：大多数日志



日志系统也是通过软件服务生成的

- Rpm -qac | grep rsyslog
- -c：show配置文件

配置文件的设置

AA BB  CC

- AA表示设备or类比
- BB表示日志级别
- CC表示



配置文件生效需要重启

```
systemctl restart rsyslog.config
```

校验配置文件是否生效

```
logger -p user.debug "test qf message"
```





### 考试试验1：配置rsyslog服务，产生自定义日志

```shell
#1. 打开日志服务(rsyslog)的配置文件
rpm -qac | grep rsyslog #查看配置文件的路径
vim /etc/rsyslog.conf #打开配置文件
#2. 添加自定义配置, 类别=所有类型，日志级别=debug以上，文件位置
*.debug   /etc/log/user.debug.message
#3. 重新加载配置
systemctl reload /etc/rsyslog.conf
#4. 测试配置是否生效
logger -p user.debug "debug message"  #产生一条类别=user，日志级别=debug以上
tail -fn 200 /etc/rsyslog.conf #检查文件位置，是否有"debug message"
```



### production问题：查询某个时间段下的日志

```shell
cat -n /var/log/user.debug.message | grep "23:[30-59]:[0-59]"
```





## 二进制日志



```shell
#查看操作系统的进程tree			
pstree -d 
#查看服务
systemctl start system
#存储位置
cd /run/log/journal/**

```

###  systemd日志



### 考试实验1：永久存储二进制日志

```shell
#1.查找服务的配置
mandb  #刷新mandb数据库
man -k journal #查询帮助手册，发现需要查询配置【5】
man 5 journal
#2.修改配置文件,

#3.重新开启配置
systemctl restrat systemd-journald
#4. 创建日志目录
ll -d /var/log/journal/ #查看文件是否存在和权限
cp -a /run/log/journal /var/log/journal/ #考试要求使用二进制服务的默认权限，那么取巧，复制目录的权限属性


```



## 时间同步

### 考试实验

```shell
#查看时间
timedatectl  | date

#修改时间同步配置文件
man chron自动补全 #查看配置文件
vim /etc/chrony.conf
server 172.25.254.254 iburst #以172.25.254.254为时间服务器做同步
#重启服务
systemctl restart chronyd
#开机自启
systemctl enable --now chronyd #enable开机自动 now启动服务
```





## nmctl管理网络

同一网段（子网掩码） + 同一个主机（主机位）=32

```
192.168.1.0/24
192.168.2.0/24
两者不同网段，不能互通

```



### 网关链接不同网段

```
主机1：192.168.1.3/24
主机2：192.168.2.3/24

主机1 -> （网关192.168.1.0）路由器 （网关：192.168.2.0）<- 主机2
```

### 网络接口名称

- 一般，Linux的网络接口列举为eth0\eth1\eth2
- RHEL8是以网络硬件的名称列举,  ens88
  - en:
  - s:

```shell
#查看网络接口的命令
ip address show eth0
ifconfig
```

### 端口和服务

```
lsof -i:22
netstat
```

### 网络管理工具network management

一个网卡可以有多个conn, 一个conn只能属于一个网卡

#### 考试实验1：nmcli命令添加、修改、启动网卡

```

```

#### 考试实验2：nmtui图形化工具

#### 考试实验3：更改配置文件
