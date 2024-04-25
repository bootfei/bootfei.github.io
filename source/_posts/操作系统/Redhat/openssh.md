---
title: 查找CPU飙升的原因
date: 2020-11-05 09:24:42
tags: [java, jstack]
---





```shell
#生成密钥对，在家目录保存
Ssh-keygen
#检查家目录，是否有该密钥
ls /root/.ssh
cat /root/.ssh/id_rsa

#把当前server的公钥，发给远程服务器
ssh-copy-id root@serverb
#检查serverb，是否接受并信任了当前机器
cat /root/.ssh/authorized_keys
```



