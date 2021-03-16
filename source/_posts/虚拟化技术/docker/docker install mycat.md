---
title: docker install mycat
date: 2021-01-26 11:01:27
tags: [vm]
---



# [MyCat官网docker](https://github.com/MyCATApache/Mycat-Server/wiki/2.1-docker%E5%AE%89%E8%A3%85Mycat)

```shell
yum install wget ##如果没有wget命令的话，使用yum下载
```



# [tar -C命令详解](https://blog.51cto.com/funtime/1708746)

```shell
##如果不使用-C后缀命令
tar -zxvf mycat1.6.7.6.tar.gz
##那么解压后的目录就是/root/data/mycat/mycat/;
##而且有其他目录，比如
##/root/data/mycat/mycat/lib/,
##/root/data/mycat/mycat/conf/,
##/root/data/mycat/mycat/bin/,
##/root/data/mycat/mycat/logs/,
##/root/data/mycat/mycat/catlet/


##使用-C命令，
tar -zxvf mycat1.6.7.6.tar.gz -C /root/data/ mycat/conf
###那么解压后的目录就是/root/data/mycat/
##而且只有一个目录，比如
##/root/data/mycat/conf/

rm -rf dir ##强制删除目录
```

