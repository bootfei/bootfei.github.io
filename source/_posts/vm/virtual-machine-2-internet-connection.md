---
title: virtual machine - 2. internet connection
date: 2020-10-08 17:56:19
tags: virtual machine
---

Ref:

1. https://cnblogs.com/wish123/p/9277995.html



1. NAT模式
   1. 原理:
      1. 虚拟机和主机之间有NAT Engine
      2. 虚拟机发送请求，实际上是传给NAT Engine，由它利用主机进行网络访问
      3. 虚拟机收到的请求,
   2. 特点:
   3. 配置:



# 配置网络ipv4

```
vim /etc/sysconfig/network-scripts/ifcfg-xxx
```

> 我们需要修改 BOOTPROTO=dhcp 中的dhcp改为static（静态），同时在文字下方添加：
>
> IPADDR=192.168.1.152      #静态IP  
>
> GATEWAY=192.168.1.1      #默认网关  
>
> NETMASK=255.255.255.0  #子网掩码  
>
> DNS1=192.168.1.1              #DNS 配置  
>
> DNS2=8.8.8.8 

重启：service network restart 

验证：ip addr