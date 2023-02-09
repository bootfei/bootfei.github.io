---
title: RocketMQ-04-开源贡献
date: 2021-05-31 09:10:13
tags:
---

原文链接：https://mp.weixin.qq.com/s/o917lCT1kHPYpAD63bvv7w

**我们希望能够有一个运维命令，能够获取当前集群中所有的客户端连接信息。**

那我们能否定制化一个运维命令实现获取所有生产者的连接信息，既然官方提供了producerConnection命令获取单个生产者组信息，服务端应该是会存储所有生产者组的连接信息，只是官方没有提供相关的命令而已，为了验证，我们可以去看一下producerConnection命令在服务端的处理逻辑。

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFsujHP5KXGTMibVaYCfRrxjY0rLzcpmfQFqLt3qrcG8ib8TLYuuURDAss2YIdKsiaryAcgxMSgxooBaA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



从服务端代码可知，在服务端会使用一个Map存储所有的生产者客户端连接，其键为生产者组名称，其值为Map键值对(通道对象(Channel),客户端信息(ClientChannelInfo))，其中客户端信息就包含了客户端ID，客户端使用的语言以及版本等信息。

经过上面分析，实现一个获取全部生产者连接信息将变得非常简单。

本文由于篇幅问题，不展示全部代码，主要总结一下开发一个RocketMQ的几个核心步骤，代码已提交到github仓库：

链接：https://github.com/apache/rocketmq/pull/2940

首先我们看一下该命令在IDEA中的测试效果，运行命令如下：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFsujHP5KXGTMibVaYCfRrxjYD6ILnKgAVhW3zTdiaeUCp34fMLRtCIvZhh3GReZIpbgzCxngMouCJ8w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


命令运行输出为：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFsujHP5KXGTMibVaYCfRrxjYzDaIoJLJtCibP3D6agiaNNRfm51IMRGXGTqXjMtk9liadibdbp3iaxNyK0g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


打包部署后命令的使用说明如下：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFsujHP5KXGTMibVaYCfRrxjY3GvNu1nSZjccZ8tekt01O0cOhD6GxEmwt5jmbADJw9lQGUYqagkTlA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

其中-b broker地址与-c cluster集群名称两个参数二选一，主用用于路由。

编写一个RocketMQ运维命令通常具备如下几个要点：

- 创建自己的命令处理类，并在MQAdminStartup中调用initCommand初始化命令。
- 定义命令请求CODE,RocketMQ为每一个请求定义了对应的RequestCode。
- 在Broker端的AdminBrokerProcessor中添加对该请求Code的处理逻辑

为了更加形象完整的展示开发一个运维命令，我将本次提交记录截图展示给大家：

## ![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFsujHP5KXGTMibVaYCfRrxjYeQkUFTqXWDs5kJNyqib9xWAaibibpBcxHdV3VbW8o08WO0A9EFFics8JQQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

