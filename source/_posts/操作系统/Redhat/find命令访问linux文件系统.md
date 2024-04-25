---
title: 查找CPU飙升的原因
date: 2020-11-05 09:24:42
tags: [java, jstack]
---



```shell
#根据名称查找
find -name
find /root -name a.txt
#根据文件类型
find -type
```



```shell
#根据数据库索引，查找文件, 比find文件
locate a.txt
#更新linux的数据库索引,  把linux文件放到数据库索引中
updatedB
```



### 实际练习

```shell
#查找student用户的所有文件并将其副本放入/root/findfiles目录
mkdir /root/findfiles
find / -user student -exec cp -a {} /root/findfiles \;
```

