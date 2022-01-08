---
title: ss占用vray端口解决方案
date: 2021-02-17 22:17:44
tags:
---

之前安装过ShadowsocksX-NG一直被封端口，决定换v2ray

但是使用v2ray死活上不去网，就提示以下这个问题

![img](https://ivpsr.com/wp-content/uploads/2020/11/1605841745-8a993752e9658af.jpg)

打开mac的终端，找到电脑上Launchpad类似火箭一样的图标，点击Terminal打开终端

使用命令lsof -i:1086查看被占用的端口，看到PID:438这个进程

![img](https://ivpsr.com/wp-content/uploads/2020/11/1605842167-1a785dbb9ad6943.jpg)

使用launchctl list | grep 438看启动项

![img](https://ivpsr.com/wp-content/uploads/2020/11/1605842328-c20ef55b3b45374.jpg)

```
# 关闭服务
launchctl stop com.qiuyuzhou.shadowsocksX-NG.local
# 移除服务
 launchctl unload ~/Library/LaunchAgents/com.qiuyuzhou.shadowsocksX-NG.local.plist

 rm -rf ~/Library/LaunchAgents/com.qiuyuzhou.shadowsocksX-NG.*
 rm -rf ~/Library/Preferences/com.qiuyuzhou.ShadowsocksX-NG.plist
 rm -rf ~/Library/ApplicationSupport/ShadowsocksX-NG

重启电脑
```

再用lsof -i:1086

重新启动v2,已经看到v2ray占用了1086正在运行了

