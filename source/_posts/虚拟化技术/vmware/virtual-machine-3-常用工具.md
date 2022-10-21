---
title: virtual machine -3-常用工具
date: 2020-10-08 17:56:19
tags: virtual machine
---



# jdk

```shell
yum search java
yum install -y java-1.8.0-openjdk-devel.x86_64
```

```shell
java -version #

1rpm -ql java-1.8.0-openjdk #查看安装目录
```



# ifconfig网络工具



```
yum search ifconfig
```

***![img](https://www.linuxidc.com/upload/2018_10/181011090563194.png)***

以上运行结果，我们只要分析最好一行就可以。**Matched:** ifconfig 这个 分割行 是用来显示 匹配结果的。

最后一行 中 冒号（：）前面的数据， （**net-tools.x86_64** ） 是匹配的软件包；冒号（：）后面的数据，（**Basic networking tools** ） 是对前面包的描述。

结合上面的信息， 安装ifconfig 包 只需要安装 net-tools.x86_64 即可。

所以，我们执行 

```
yum install net-tools.x86_64
```

```shell
ifconfig
```





# vim编辑器

```
yum -y install vim*
```

