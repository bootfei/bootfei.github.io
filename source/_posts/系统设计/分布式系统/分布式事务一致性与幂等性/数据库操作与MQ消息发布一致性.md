---
title: 数据库操作与MQ消息分布式操作的一致性
date: 2021-04-09 09:15:10
tags:
---



## 活动中心场景介绍 

------

在电商系统上线初期，往往会进行一些“拉新”活动，例如活动部门提出**新用户注册送积分、送优惠券活动**。

基于分布式、微服务的设计理念，通常的架构设计（子系统交互）如下图所示：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFvz5BhRHmnGstLtUaJHNRLicv3lkl4PYzkxwxUUT8cvHPic1EVzs5Ib9p3DmbMu4CR2ejNV7lQ0biavg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


其核心系统介绍如下：

- 账户中心
  提供用户登录、用户注册等服务，一个新用户注册时，向MQ服务器中的USER_REGISTER主题发送一条消息，主流程结束，与送积分，送优惠券等过程解耦。
- 优惠券（券系统）
  提供发放优惠券、使用优惠券等与券相关的基础服务。
- 积分中心
  提供积分相关的服务，例如积分赠送、积分消费、积分查询等基础服务。
- 送积分服务（消费者）
  订阅MQ，按照规则决定是否需要赠送积分，如果需要则调用积分相关的基础接口，完成积分的发放。
- 送优惠券（消费者）
  订阅MQ，按照规则决定是否需要赠送优惠券，如果需要则调用券系统相关的基础接口，完成优惠券的发放。

问题：**如果新用户注册成功，但消息发送到MQ失败，或者消息成功发送到MQ，但发送完MQ后系统出现异常导致用户注册失败又该如何呢？**

上面的问题其实就是**典型的分布式事务问题**：即如何保证**用户注册(数据库操作)与MQ消息发送**这两个分布式操作的一致性。

## 事务消息实现原理

------

**解决思路：二阶段提交与事务状态回查**，其具体实现流程如下图所示：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFvz5BhRHmnGstLtUaJHNRLicvRbyskeRHIO7ocUm8qibhfS71gPKlmv1cicXz41hPegy67sRhiboKBibtg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


其核心设计理念：

- 应用程序开启一个本地事务
  - 事务中包含数据库操作
  - 事务中发送一条PREPARE消息，PREPARE消息发送成功后通知应用程序记录本地事务状态，然后提交本地事务。
- RocketMQ在收到类型为PREPARE的消息时，首先**备份消息的原主题与原消息消费队列**，然后将消息存储在主题为RMQ_SYS_TRANS_HALF_TOPIC的消息队列中，[故PREPARE的消息是不会被客户端消费的]()。
- 定时服务器开启一个定时任务处理RMQ_SYS_TRANS_HALF_TOPIC中的消息，会每隔指定时间向应用程序发起**事务状态查询请求** ,询问应用程序本地事务是否成功，然后根据回查状态决定是提交还是回滚，即对处于PREPARE状态进行提交或回滚操作。
  - 定时服务器如果明确得知事务成功，则可以返回COMMIT，服务端会提交该条消息，具体操作是恢复原消息的主题与队列，重新发送到Broker，消费端感知后消费。
  - 定时服务器如果无法明确得知事务状态，则返回UNOWN，此时服务端会等待一定时间后再次向发送者询问，默认询问15次。
  - 定时服务器如果非常明确得知事务失败，则可以返回ROLLBACK。

在具体实践中，[定时服务器]()在无法获取事务状态时不要武断的返回ROLLBACK，而是要返回UNOWN，让服务端定时重试回查，说明如下：

<img src="https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFvz5BhRHmnGstLtUaJHNRLicibtwsHP1sxDH14bE3W4Q7NhEIziam8g1PjXUeicchCT6VbpzovIvXl0xw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1" alt="Image" style="zoom:50%;" />


在应用程序将PREPARE消息发送到Broker后，定时服务器发起事务查询时本地事务可能还未提交，为了避免无效的事务回查机制，RocketMQ通常至少在收到PREPARE消息6s后才会发起第一次事务回查，可通过 transactionTimeOut 配置。故客户端在实现事务回查时无法证明事务状态时不应该返回ROLLBACK，而是返回UNOWN。

## 事务消息实战

------

光说不练假把式，接下来以一个**新用户注册送优惠券的场景**来详细介绍如何使用事务消息。

项目模块职责说明如下：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFvz5BhRHmnGstLtUaJHNRLicp0FZqHBcHorVYwgyonW3l3GvxVf7JbwfPydgFfJ2W213bDhm5xAVzg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


事务消息的核心代码组装在transaction-service，其核心类图如下：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFvz5BhRHmnGstLtUaJHNRLicRycFWlp0DLnuiaRsIEghB7vlSspbOOmqa5WyiasibBRX9d1AQOJg9ffDQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


其中核心要点如下：

- UserServiceImpl
  Dubbo接口业务实现类，类似MVC的控制层，在这里做一些参数验证，但**不执行具体的业务逻辑**，只是发送一条**事务消息**到MQ。
- UserRegTransactionListener
  事务监听器，在 executeLocalTransaction 方法中执行业务逻辑，数据库本地事务加在该方法。

> 温馨提示：之所以不在UserServicveImpl中执行本地事务，是因为 executeLocalTransaction 中抛出的异常会被RocketMQ框架捕捉，及异常无法被UserServiceImpl感知，即无法实现其事务的一致性。

### 应用程序端: UserServiceImpl 

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFvz5BhRHmnGstLtUaJHNRLicHG08xaibsdPwygtzwqibIBibZxaxJmVbtZwBKvT9Iia033rprrdRlhpqNQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


UserServiceImpl 的核心要点如下：

- 首先应该对**参数进行校验、业务逻辑进行校验**
- 发送事务消息，**建议对消息设置Key**,Key的值可以用**业务处理流水号**(可唯一表示该业务操作)或者**核心业务字段**(例如订单编号)
- 业务入口类可通过**事务消息发送状态**来判断**业务是否失败**。

### MQ+定时任务: UserRegTransactionListener

事务监听器需要实现执行本地事务与事务回查两个接口。

##### 3.2.1 实现 executeLocalTransaction

首先需要实现 executeLocalTransaction 方法，执行本地事务，其代码如下图所示：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFvz5BhRHmnGstLtUaJHNRLicXGC8hJ2eFUM1iaZwNLC2yoctYXalVJzJl3a1KgVr9dUVtLsOZZWo9fw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


其中几个关键点说明如下：

- 在该方法上添加数据库事务标签。
- 执行业务逻辑，示例Demo只是将用户数据存储到数据库。
- 如果业务执行失败，可明确告知需要回滚，上层调用方也可根据ROLLBACK_MESSAGE进行相应的处理。
- 如果业务成功，**不建议直接返回COMMIT，而是建议返回UNKNOW**,因为该方法尽管在方法最后一行，但可能发生断电等异常情况，数据库并没有成功。

##### 3.2.2 实现 checkLocalTransaction

其次需要实现事务状态回查，用来RocketMQ服务端感知事务是否成功，其实现原理如下图所示：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFvz5BhRHmnGstLtUaJHNRLicOzMibUQdW16wMsd4Qia09pLKM7JWIeuOMCq87esPGNia07ib7WnRa1TWng/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


其实现关键点如下：

- 如果能明确得知本地事务成功，则返回COMMIT_MESSAGE
- 如该不能明确得知本地事务成功，**不能返回ROLLBACK_MESSAGE**,而是返回UNKNOW，等待服务端下一次事务回查(不会立即触发)，**服务端默认回查15次，如果15次都得到UNKNOW，则会回滚该消息**。



https://github.com/dingwpmz/rocketmq-learning

