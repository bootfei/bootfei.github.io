---
title: zookeeper-02-Leader的选举机制
date: 2021-02-16 19:48:27
tags:
---

原文链接：https://blog.csdn.net/weixin_41947378/article/details/107062257



## 1.配置维护


分布式系统中，很多服务都是部署在集群中的，即多台服务器中部署着完全相同的应用，起着完全相同的作用。当然，集群中的这些服务器的配置文件是完全相同的。

若集群中服务器的配置文件需要进行修改，那么我们就需要逐台修改这些服务器中的配置文件。如果我们集群服务器比较少，那么这些修改还不是太麻烦，但如果集群服务器特别多，比如某些大型互联网公司的 Hadoop 集群有数千台服务器，那么纯手工的更改这些配置文件几乎就是一件不可能完成的任务。即使使用大量人力进行修改可行，但过多的人员参与，出错的概率大大提升，对于集群所形成的危险是很大的。

实现原理

zk 可以通过“发布/订阅模型”实现对集群配置文件的管理与维护。“发布/订阅模型”分为推模式（Push）与拉模式（Pull）。zk 的“发布/订阅模型”采用的是推拉相结合的模式。

和Nacos、Spring Cloud Config、携程的阿波罗 作用一样

其实现的具体步骤为：

Step1：发布者应用程序作为 zk 客户端首先需要在 zk 中创建一个节点，该节点的数据内容即为当前被监控集群主机的配置文件。
Step2：被监控集群主机在启动时首先需要从 zk 的节点上读取数据内容，即配置文件内容。
Step3：读取过数据内容后，再向 zk 的该节点注册数据内容变更的 watcher 监听。
Step4：发布者将更新过的配置文件内容更新到 zk 的对应节点数据内容上。此时 zk 会引发相应 watcher 事件，然后向每一个被监控主机推送 watcher 事件。
Step5：被监控集群主机在接收到 watcher 事件后，会触发本地 watcher 调用执行回调方法，回调方法会从 zk 中拉取节点的数据内容，即更新过的配置文件内容。
## 2.命名服务


命名服务是指可以为一定范围内的元素命名一个唯一标识，以与其它元素进行区分。在分布式系统中被命名的实体可以是集群中的主机、服务地址等。

2.2 实现原理

通过利用zk 中节点路径不可重复的特点来实现命名服务的。当然，也可以配带上顺序节点的有序性来体现唯一标识的顺序性。

具体实现步骤：

Step1：生成器在启动时首先需要在 zk 中创建一个根节点，例如/app
Step2：根据具体业务需求，在根节点/app 下创建多级子节点，每一级子节点名称使用对应级别的模块名称。例如，/app/一级模块名称/二级模块名称
Step3：再在模块节点下创建顺序节点，而节点名称可以根据业务需求指定。在生成时会自动为该名称添加上序号。此时该顺序节点的全路径即为生成的唯一标识

## 3.集群监控

对于集群，我们总是希望能够随时获取到当前集群中各个主机的运行时状态、当前集群中主机的存活状况等信息。通过 zk 可以实现对集群的随时监控。

3.1 基本原理
zk 进行集群管理的基本原理如下图所示。


图解：
监控系统启动的时候先在ZK中注册/clusterManager根节点，并注册子节点列表变更Watcher监听

被监控集群的主机一启动，就在/clusterManager根节点下创建相应的临时子节点（临时节点好处就是被监听的服务器如果挂了，会话就没了，临时节点就没了）

一但被监控集群有节点新加入或者挂了，就会触发子节点列表变更事件，监控系统就会触发Watcher的回调，更新信息，例如把这些子节点列表都读取过来，在界面上显示出来，即显示存活状态

除了显示存活状态，还可以让被监控的主机定时向自己的节点里面更新其他状态数据，这样监控系统可以随时获取这些状态数据

具体实现步骤是：

Step1：监控系统在启动时会在 zk 中创建一个根节点，例如/clusterManager
Step2：当集群主机应用启动后，就会自动在 zk 的监控系统根节点下创建一个对应的临时子节点，并将自己的运行状态定时写入到该临时节点，或根节点的数据内容中，例如主机当前正在处理的连接请求有多少，当前主机的权重等。写入到这两个节点的效果是不同的：
写到临时节点：临时节点消失后，从监控系统中根本就查找不到任何该临时节点对应主机的信息。
写入到根节点：可以获取到所有曾经存在过的节点信息。
Step3：监控系统在根节点/clusterManager 上注册一个 watcher 监听。一旦集群中增减主机，就会引发子节点数量变更的 watcher 事件。然后 zk 会将事件推送给监控系统
Step4：监控系统在接收到 zk 发送的事件后，调用相应的 watcher 对象回调，将变化情况显示到监控平台。
Step5：若集群主机状态信息是写入到根节点数据内容的，那么监控系统需要在根节点上再注册一个数据内容变更的 watcher 监听，以实时获取到集群主机的状态数据。
Step6：若集群主机状态信息是写入到对应临时节点的，那么监控系统需要在每个主机临时节点上注册数据内容变更的 watcher 监听，以实时获取到集群主机的状态数据。
3.2 分布式日志收集系统
下面以分布式日志收集系统为例来分析 zk 对于集群的管理。

（1） 系统组成
首先要清楚，分布式日志收集系统由四部分组成：日志源集群、日志收集器集群，zk集群，及监控系统。


（2） 系统工作原理


图解：
sourcehost这些源节点，设置为临时节点，监控存活状态很简单

收集器collector1/2/3…这些节点只能设置为持久节点，监控存活状态比较困难，怎么处理？

可以在collector持久节点下再创建收集器自己的临时节点，如果该临时节点没了，代表其收集器主机挂了

但是如果收集器的临时节点挂了，其收集器的持久节点，还有对应的源主机临时节点还在怎么办？

这个时候就需要将这些源主机的临时节点分配给其他收集器主机，怎么分配？

负载均衡，谁的压力小给谁，按照每个收集器收集的主机数量或者每个主机的日志产生量等判断
可以为每一个收集器主机分配一个负载量，然后把挂掉的collector对应的主机按照我们自己的分配方式，进行分配，比如找到压力最大的前几个收集器排除掉，然后把需要分配的主机分配给剩下的收集器。

如果要扩容，可以把压力最大的收集器，或者压力最大的前几个里面，挑出来分给新加的收集器

分布式日志收集系统的工作步骤有以下几步：

A、收集器的注册
在 zk 上创建各个收集器对应的节点。
B、 任务分配
系统根据收集器的个数，将所有日志源集群主机分组，分别分配给各个收集器。
C、 状态收集
这里的状态收集指的是两方面的收集：
日志源主机状态，例如，日志源主机是否存活，其已经产生多少日志等
收集器的运行状态，例如，收集器本身已经收集了多少字节的日志、当前 CPU、内存的使用情况等
D、任务再分配 Rebalance
当出现收集器挂掉或扩容，就需要动态地进行日志收集任务再分配了，这个过程称为Rebalance。只要发现某个收集器挂了，则系统进行任务再分配。
## 4.DNS 服务

zk 的 DNS 服务的功能主要是实现消费者与提供者的解耦合，防止提供者的单点问题，实现对提供者的负载均衡。


图解：
比如消费者想要调用service1，就先从ZK中把注册表（即Service1服务节点下所有主机列表）读到，然后内部根据负载均衡策略选一个主机调用服务
此时ZK提供的功能就是把服务提供者注册到ZK里面，然后消费者通过读取的注册表，负载均衡调用服务，即提供者写，调用者读。Dubbo就是这样实现的。

什么是 DNS
DNS，Domain Name System，域名系统，即可以将一个名称与特定的主机 IP 加端口号进行绑定。zk 可以充当 DNS 的作用，完成域名到主机的映射。

4.2 基本 DNS 实现原理
假设提供者应用程序 app1 与 app2 分别用于提供 service1 与 service2 两种服务，现要将其注册到 zk 中，具体的实现步骤如下图所示。


具体实现步骤：

Step1：在 zk 上为当前 DNS 功能创建一个根节点，例如/DNS。
Step2：以该提供者的服务名称为名在应用根节点下创建子节点，该节点即为域名节点，例如/DNS/ service1。
Step3：为域名节点添加数据内容，数据内容为当前服务的所有提供者主机地址集合，即多个提供者地址间使用逗号分隔。
4.3 具有状态收集功能的 DNS 实现原理


以上模型存在一个问题，如何获取各个提供者主机的健康状态、运行状态呢？可以为每一个域名节点再添加一个状态子节点，而该状态子节点的数据内容则为开发人员定义好的状态数据。这些状态数据是如何获取到的呢？是通过状态收集器（开发人员自行开发的）定期写入到 zk 的该节点中的。

一些其他情况的处理：

时间差问题：状态收集器是定时更新状态的，会导致提供者主机已经挂了，ZK还没跟新，恰好有消费者把ZK的数据内容读取到了，且调用服务的机器就是挂掉的

如何解决：方案很多，如下是为主机定义临时节点的解决方案


如果一个服务提供者全挂了怎么办？服务降级，降级点很多，例如消费者本身可以这么处理，提供者不行，用本地代码返回一些信息
降级目的就是增强用户体验

扩展-集群监控平台

PS：绿色都需要程序员来实现
阿里的 Dubbo 就是使用 Zookeeper 作为域名服务器的。



## 5.Master 选举


集群是分布式系统中不可或却的组成部分，是为了解决分布式系统中计算单元的单点问题，水平扩展计算单元的处理能力的一种解决方案。

一般情况下，会在群集中选举出一个 Master，用于协调集群中的其它 Slave 主机，对于Slave 主机的状态具有决定权。

5.2 广告推荐系统

系统会根据用户画像，将用户归结为不同的种类。系统会为不同种类的用户推荐不同的广告。每个用户前端需要从广告推荐系统中获取到不同的广告 ID。

（2） 分析
这个向前端提供服务的广告推荐系统一定是一个集群，这样可以更加快速高效的为前端进行响应。需要注意，推荐系统对于广告 ID 的计算是一个相对复杂且消耗 CPU 等资源的过程。如果让集群中每一台主机都可以执行这个计算逻辑的话，那么势必会形成资源浪费，且降低了响应效率。此时，可以只让其中的一台主机去处理计算逻辑，然后将计算的结果写入到某中间存储系统中，并通知集群中的其它主机从该中间存储系统中共享该计算结果。那么，这个运行计算逻辑的主机就是 Master，而其它主机则为 Slave。

（3） 架构

用户画像：对用户，用一堆属性进行刻画，描述，一般用户画像系统少的20 维、30维，多的上百维，用户画像系统描述信息的维度越高，系统需要的性能就要越高，否则无法运算

整个广告推荐系统是一个集群，5台机器，如果让5台机器既处理读又处理运算，会导致用户前端体验比较差，每台Slave主机运行效率都比较低

所以Master负责运行，并写入到中间存储系统，Slave负责读

整个流程：用户前端访问Slave，Slave先去中间存储系统，如果有直接返回，如果没有，请求转给Master，由Master根据用户id，从用户画像系统找到对应的画像，然后再根据广告管理系统进行运算，把运算结果，广告的id存入中间存储系统，然后推给用户

所以这个系统就需要读写分离

### Master选举方式：zk,dbms,redis

方案一：使用【 DBMS 的主键唯一】特性可以实现 Master 的选举。集群启动时，让所有集群主机向数据库某表中插入主键相同的记录。

- 优点：启动 or 重新选举时，实现master选举
- 弊端：Master down时，主键记录无法被删除，导致无法重新选举

方案二：使用 zk中【多个客户端对同一节点创建时，只有一个客户端可以成功】和【临时节点在client宕机时被zk删除】的特性实现。

- 优点：启动 or 重新选举时，实现master选举；Master down时，可以重新选举

具体来说，由三步完成：

Step1：多个客户端同时发起对同一临时节点/master-election/master 进行创建的请求，最终只能有一个客户端成功。这个成功的客户端主机就是 Master，其它客户端就是 Slave。
Step2：让 Slave 都向这个临时节点的父节点/master-election 注册一个子节点列表的watcher 监听。
Step3：一旦该 Master 宕机，临时节点就会消失，zk 服务器就会向所有 Slave 发送子节点变更事件，Slave 在接收到事件后会调用相应的回调方法，该回调方法会重新向这个父节点创建相应的临时子节点。谁创建成功，谁就是新的 Master。
## 6.分布式同步

分布式同步，也称为分布式协调，是分布式系统中不可缺少的环节，是将不同的分布式组件有机结合起来的关键。对于一个在多台机器上运行的应用而言，通常需要一个协调者来控制整个系统的运行流程，例如执行的先后顺序，或执行与不执行等。

6.2 MySQL 数据复制总线
下面以“MySQL 数据复制总线”为例来分析 zk 的分布式同步服务。

（1） 数据复制总线组成
MySQL 数据复制总线是一个实时数据复制框架，用于在不同的 MySQL 数据库实例间（Mysql本身的主从做不到）进行异步数据复制。其核心部分由三部分组成：生产者、复制管道、消费者。

<img src="https://img-blog.csdnimg.cn/20200701141844520.png" alt="在这里插入图片描述" style="zoom:50%;" />

> 生产者和消费者，是不同的DB厂商(图中有纰漏，比如生产者是oracle，消费者是mysql)、Mysql版本、Mys数据库实体 、数据库名、表，但是要求只拷贝某个表的某一个字段到另一个数据库表的某一个字段，即对两段的数据库没有任何要求




那么，MySQL 数据复制总线系统中哪里需要使用 zk 的分布式同步功能呢？以上结构中可以显示看到存在的问题：replicator 存在单点问题。为了解决这个问题，就需要为其设置多个热备主机。那么，这些热备主机是如何协调工作的呢？这时候就需要使用 zk 来做协调工作了，即由 zk 来完成分布式同步工作。

（2） 数据复制总线工作原理

<img src="https://img-blog.csdnimg.cn/20200701141958496.png" alt="在这里插入图片描述" style="zoom:67%;" />

> 绿色表示开发者自己实现的逻辑代码

1.【协调者】启动阶段：创建【根节点/mysql_replicator】

2. 【任务】阶段启动阶段：每一个复制任务【payRecord任务、orderRecord任务】都会在【根节点mysql_replicator】下创建一个子节点【pay_record节点, order_record节点】，每个任务的节点下都有【status节点】和【instances节点】，【协调者】会在【instanes节点】上注册子节点列表的【变更watcher】(注意：此时 host1-001, host2-002节点还没被创建）

3. 【replicator】启动阶段：对【status节点】注册【数据内容watcher监听】，紧接着在【instances节点】下创建相应的【有序临时节点】
4. 此时【协调者】对【instances节点】的【变更watcher】监听回调会被触发，回调会马上指定各个replicator主机的状态（按照自己定义的规则，例如哪个节点的序号最小就设置为RUNNING 状态，其他设置为STANDBY 状态），将状态写入【status节点】

4. status节点内容发生变更，马上触发replicator对status节点的数据内容watcher监听，回调中就会把status内容读取并解析，检测到自己状态是running就会进行复制任务

5.进行复制任务中，每复制一条就会记录RUNNING主机对Binlog的消费点，记到instances里面

6.如果运行中的replicator挂了，意味着instances列表会发生变更，会触发协调者对应的子节点列表变更watcher监听，回调中会马上读取instances节点的所有子节点，指定新的replicator是running状态，其他是STANDBY 状态，写入到status节点

7.写入status节点后，又会马上触发replicator对status节点的数据内容watcher监听，replicator解析内容检测到自己是running后又开始复制任务，复制任务会首先从instances节点中拿到Binlog的消费点，从这之后做复制。

> 思考：replicator的master选举，是否可以避开【协调者】呢？
>
> 当然可以，【5.Master选举】就给出了示例



MySQL 复制总线的工作步骤，总的来说分为三步：
A、复制任务注册

复制任务注册实际就是指不同的复制任务在 zk 中创建不同的 znode，即将复制任务注册到 zk 中。

B、 replicator 热备

复制任务是由 replicator 主机完成的。为了防止 replicator 在复制过程中出现故障，replicator 采用热备容灾方案，即将同一个复制任务部署到多个不同的 replicator 主机上，但仅使一个处于 RUNNING 状态，而其它的主机则处于 STANDBY 状态。当 RUNNING 状态的主机出现故障，无法完成复制任务时，使某一个 STANDBY 状态主机转换为 RUNNING 状态，继续完成复制任务。

C、 主备切换

当 RUNNING 态的主机出现宕机，则该主机对应的子节点马上就被删除了，然后在当前处于 STANDBY 状态中的 replicator 中找到序号最小的子节点，然后将其状态马上修改为RUNNING，完成“主备切换”。
## 7.分布式锁

分布式锁是控制分布式系统同步访问共享资源的一种方式。Zookeeper 可以实现分布式锁功能。根据用户操作类型的不同，可以分为排他锁与共享锁。

7.1 分布式锁的实现
在 zk 上对于分布式锁的实现，使用的是类似于“/xs_lock/[hostname]-请求类型-序号”的临时顺序节点。当客户端发出读写请求时会在 zk 中创建不同的节点。根据读写操作的不同及当前节点与之前节点的序号关系来执行不同的逻辑。



具体实现步骤：

Step1：当一个客户端向某资源发出读/写请求时，若发现其为第一个请求，则首先会在 zk中创建一个根节点。若节点已经存在，则无需创建。
Step2：根节点已经存在了，客户端在根节点上注册子节点列表变更的 watcher 监听。
Step3：watcher 注册完毕后，其会在根节点下人创建一个读/写操作的临时顺序节点。
Step4：节点创建完毕后，其就会马上触发客户端的 watcher 回调的执行。回调方法首先会将子节点列表读取，然后会查看序号比自己小的节点，并根据读写操作的不同，执行不同的逻辑。
如果当前节点是读，比自己小的都是读，就可以读
如果当前节点是写，只要有比自己小的，都不能写
Step5：客户端读写操作完毕，其与 zk 的连接断开，则 zk 中该会话对应的节点消失。
7.2 分布式锁的改进
前面的实现方式存在“羊群效应”，为了解决其所带来的性能下降，可以对前述分布式锁的实现进行改进。

由于一个操作而引发了大量的低效或无用的操作的执行，这种情况称为羊群效应。

当客户端请求发出后，在 zk 中创建相应的临时顺序节点后马上获取当前的/xs_lock 的所有子节点列表，但任何客户端都不再向/xs_lock 注册用于监听子节点列表变化的 watcher，而是改为根据请求类型的不同向“对其有影响的”子节点注册 watcher。

对其有影响：

如果当前节点是读，第一看自己是不是最小的，如果是肯定能执行，如果不是最小的，看我前面有没有写操作，如果有写操作，就不能读，并且只需要关注比自己小，并且最近的写节点即可,即对该节点注册 节点删除的监听Watcher，Watcher一回调就代表可以写了。
如果当前节点是写，如果不是最小的，则直接监听自己前一个节点，如果前一个节点被删除了，触发Watcher，在Watcher回调中，拉取根节点的子节点列表，判断自己前面还有没有节点，如果有，继续监听自己前面的节点，递归，直到自己前面没有节点即可写操作。
8. 分布式队列
说到分布式队列，我们马上可以想到 RabbitMQ、Kafka 等分布式消息队列中间件产品。zk 也可以实现简单的消息队列。

8.1 FIFO 队列

zk 实现 FIFO 队列的思路是：利用顺序节点的有序性，为每个数据在 zk 中都创建一个相应的节点。然后为每个节点都注册 watcher 监听。一个节点被消费，则会引发消费者消费下一个节点，直到消费完毕。

其具体的实现步骤是：

Step1：为每一个数据按照其到达的顺序为其创建顺序子节点，且将数据作为节点的数据内容。这个子节点可以是持久顺序子节点，也可以是临时顺序子节点。不同类型，后面的监听方案是不同的。
Step2：若注册的为持久顺序节点。每个消费者会向其所消费的那个节点的前一个节点注册一个“数据内容变更事件”的 watcher 监听。若其消费的是第一个节点，则无需注册监听，可以直接消费。
Step3：当一个消费者对一个数据消费过后，会马上修改该节点的数据内容。而该数据内容的变化会引发一个 watcherEvent 事件，并会将此事件发送给监听者。
Step4：监听者在接收到 watcherEvent 后，调用其回调方法。该回调方法会来读取其所要消费的节点的数据内容。该节点的数据内容被读取后，数据内容会被修改。而该修改会引发一个 watcherEvent 事件，并会将此事件发送给监听者，然后再循环执行第 4 步
8.2 分布式屏障 Barrier 队列

Barrier，屏障、障碍物。Barrier 队列是分布式系统中的一种同步协调器，规定了一个队列中的元素必须全部聚齐后才能继续执行后面的任务，否则一直等待。其常见于大规模分布式并行计算的应用场景中：最终的合并计算需要基于很多并行计算的子结果来进行。

zk 对于 Barrier 的实现原理是，在 zk 中创建一个/barrier 节点，其数据内容设置为屏障打开的阈值，即当其下的子节点数量达到该阈值后，app 才可进行最终的计算，否则一直等待。每一个并行运算完成，都会在/barrier 下创建一个子节点，直到所有并行运算完成。

其具体的实现步骤是：

Step1：创建一个/barrier 节点，其数据内容设置为屏障打开的阈值
Step2：应用程序向/barrier 注册一个 watcher 监听，监听其下的子节点数量变化
Step3：开始每一个并行计算。对于每个并行计算，每计算出一个子结果，就会在/barrier下创建一个子节点，而每增加一个节点，就会触发应用程序获取/barrier 的子节点列表，当子节点个数与阈值相等时，则会开启最终的合并计算，即打开了屏障