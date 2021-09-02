---
title: zk调优与分析-01-集群无法选举出leader
date: 2021-07-05 19:01:00
tags:
---

## 前言

ZooKeeper作为dubbo的注册中心，可谓是重中之重，线上ZK的任何风吹草动都会牵动心弦。最近笔者就碰到线上ZK Leader宕机后，选主无法成功导致ZK集群拒绝服务的现象，于是把这个case写出来分享给大家(基于ZooKeeper 3.4.5)。

## Bug现场

一天早上，突然接到电话，说是ZooKeeper物理机宕机了，而剩余几台机器状态都是

```
sh zkServer.sh status
it is probably not running
```

笔者看了下监控，物理机宕机的正好是ZK的leader。3节点的ZK，leader宕了后，其余两台一直未能成为leader，把宕机的那台紧急拉起来之后，依旧无法选主，
导致ZK集群整体拒绝服务！
![Image](https://mmbiz.qpic.cn/mmbiz_png/yiaiaFLiaflYRSh5ibqDTrYjnj6t1bAh5er0m4ibVmN3CGPSxKJJdcVq2u6cHK6chnCicGLRXE5pLPt5czKt0mcPsx4w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

## 业务影响

Dubbo如果连接不上ZK，其调用元信息会一直缓存着，所以并不会对请求调用造成实际影响。麻烦的是，如果在ZK拒绝服务期间，应用无法重启或者发布，一旦遇到紧急事件而重启(发布)不能，就会造成比较重大的影响。
好在我们为了高可用，做了对等机房建设，所以非常淡定的将流量切到B机房，
![Image](https://mmbiz.qpic.cn/mmbiz_png/yiaiaFLiaflYRSh5ibqDTrYjnj6t1bAh5er04xonZeWUwegNic2KQHH3jJJGgoz1ltQRxY86ajADNBCYdaAqUd5WU5g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
双机房建设就是好啊,一键切换！
切换过后就可以有充裕的时间来恢复A机房的集群了。在紧张恢复的同时，笔者也开始了分析工作。

## 日志表现

首先，查看日志，期间有大量的client连接报错，自然是直接过滤掉，以免干扰。

```
cat zookeeper.out | grep -v 'client xxx' | > /tmp/1.txt
```

首先看到的是下面这样的日志:

### ZK-A机器日志

```
Zk-A机器:
2021-06-16 03:32:35 ... New election. My id=3
2021-06-16 03:32:46 ... QuoeumPeer] LEADING  // 注意，这里选主成功
2021-06-16 03:32:46 ... QuoeumPeer] LEADING - LEADER ELECTION TOOK - 7878'
2021-06-16 03:32:48 ... QuoeumPeer] Reading snapshot /xxx/snapshot.xxx
2021-06-16 03:32:54 ... QuoeumPeer] Snahotting xxx to /xxx/snapshot.xxx
2021-06-16 03:33:08 ... Follower sid ZK-B.IP
2021-06-16 03:33:08 ... Unexpected exception causing shutdown while sock still open
java.io.EOFException 
    at java.io.DataInputStream.readInt
    ......
    at quorum.LearnerHandler.run
2021-06-16 03:33:08 ******* GOODBYE ZK-B.IP *******
2021-06-16 03:33:27 Shutting down
```

这段日志看上去像选主成功了，但是和其它机器的通信出问题了，导致Shutdown然后重新选举。

## ZK-B机器日志

```
2021-06-16 03:32:48 New election. My id=2
2021-06-16 03:32:48 QuoeumPeer] FOLLOWING
2021-06-16 03:32:48 QuoeumPeer] FOLLOWING - LEADER ELECTION TOOK - 222
2021-06-16 03:33:08.833 QuoeumPeer] Exception when following the leader
java.net.SocketTimeoutException: Read time out
    at java.net.SocketInputStream.socketRead0
    ......
    at org.apache.zookeeper.server.quorum.Follower.followLeader
2021-06-16 03:33:08.380 Shutting down
```

这段日志也表明选主成功了，而且自己是Following状态，只不过Leader迟迟不返回，导致超时进而Shutdown

## 时序图

笔者将上面的日志画成时序图，以便分析:
![Image](https://mmbiz.qpic.cn/mmbiz_png/yiaiaFLiaflYRSh5ibqDTrYjnj6t1bAh5er0YyjZOzPPZ1H1iatv0dJIWewJmt33kt7bSrMMhiciasLBtdsniaq8HpACSg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
从ZK-B的日志可以看出，其在成为follower之后，一直等待leader，直到Read time out。
从ZK-A的日志可以看出,其在成为LEADING后，在33:08,803才收到Follower也就是ZK-B发出的包。而这时，ZK-B已经在33:08,301的时候Read timed out了。

### 首先分析follower(ZK-B)的情况

我们知道其在03:32:48成为follower,然后在03:33:08出错Read time out，其间正好是20s。于是笔者先从Zookeeper源码中找下其设置Read time out是多长时间。

```
Learner
protected void connectToLeader(InetSocketAddress addr) {
    ......
    sock = new Socket()
    // self.tockTime 2000 self.initLimit 10
    sock.setSoTimeout(self.tickTime * self.initLimit);
    ......
}
```

其Read time out是按照zoo.cfg中的配置项而设置:

```
tickTime=2000 self.tickTime
initLimit=10 self.initLimit
syncLimit=5
```

很明显的，ZK-B在成为follower后，由于某种原因leader在20s后才响应。那么接下来对leader进行分析。

### 对leader(ZK-A)进行分析

首先我们先看下Leader的初始化逻辑:

```
quorumPeer
    |->打印 LEADING
    |->makeLeader
        |-> new ServerSocket listen and bind 
    |->leader.lead()
        |->打印 LEADER ELECTION TOOK
        |->loadData
            |->loadDataBase 
                |->resore 打印Reading snapshot
            |->takeSnapshot
                |->save 打印Snapshotting
            |->cnxAcceptor 处理请求Accept
```

可以看到，在我们的ZK启动监听端口到正式处理请求之间，还有Reading Snapshot和Snapshotting(写)动作。从日志可以看出一个花了6s多,一个花了14s多。然后就有20s的处理空档期。如下图所示:
![Image](https://mmbiz.qpic.cn/mmbiz_png/yiaiaFLiaflYRSh5ibqDTrYjnj6t1bAh5er0H8iawNnVj63Ic0JWnlB7EMohfSktibyMNic1ZSfa6trDQbLLoxz3rLyPg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
由于在socket listen 20s之后才开始处理数据，所以ZK-B建立成功的连接实际还放在tcp的内核全连接队列(backlog)里面，由于在内核看来三次握手是成功的，所以能够正常接收ZK-B发送的follower ZK-B数据。在20s，ZK-A真正处理后，从buffer里面拿出来20s前ZK-B发送的数据，处理完回包的时候，发现ZK-B连接已经断开。
同样的，另一台follower(这时候我们已经把宕机的拉起来了，所以是3台)也是由于此原因gg,而leader迟迟收不到其它机器的响应，认为自己的leader没有达到1/2的票数，而Shutdown重新选举。

## Snapshot耗时

那么是什么导致Snapshotting读写这么耗时呢？笔者查看了下Snapshot文件大小,有将近一个G左右。

## 调大initLimit

针对这种情况，其实我们只要调大initLimit，应该就可以越过这道坎。

```
zoo.cfg
tickTime=2000 // 这个不要动，因为和ZK心跳机制有关
initLimit=100 // 直接调成100,200s!
```

## 这么巧就20s么？

难道就这么巧，每次选举流程都刚好卡在20s不过？反复选举了好多次，应该有一次要<20s成功吧，不然运气也太差了。如果是每次需要处理Snapshot 30s也就算了，但这个20s太接近极限值了，是否还有其它因素导致选主不成功？

## 第二种情况

于是笔者翻了下日志，还真有！这次leader这边处理Snapshot快了，但是follower又拉跨了!日志如下:

### leader(ZK-A)第二种情况

```
2021-06-16 03:38:03 New election. My id= 3
2021-06-16 03:38:22 QuorumPeer] LEADING
2021-06-16 03:38:22 QuorumPeer] LEADING - LEADER ELECTION TOOK 25703
2021-06-16 03:38:22 QuorumPeer] Reading snapshot
2021-06-16 03:38:29 QuorumPeer] Snapshotting
2021-06-16 03:38:42 LearnerHandler] Follower sid 1
2021-06-16 03:38:42 LearnerHandler] Follower sid 3
2021-06-16 03:38:42 LearnerHandler] Sending DIFF
2021-06-16 03:38:42 LearnerHandler] Sending DIFF
2021-06-16 03:38:54 LearnerHandler] Have quorum of supporters
2021-06-16 03:38:55 client attempting to establsh new session 到这开始接收client请求了
......
2021-06-16 03:38:58 Shutdown callsed
java.lang.Exception: shutdown Leader! reason: Only 1 followers,need 1
    at org.apache.zookeeper.server.quorum.Leader.shutdown
```

从日志中我们可以看到选举是成功了的，毕竟处理Snapshot只处理了13s(可能是pagecache的原因处理变快)。其它两个follower顺利连接，同时给他们发送DIFF包，但是情况没好多久，又爆了一个follower不够的报错，这里的报错信息比较迷惑。
我们看下代码:

```
Leader.lead
void lead() {
    while(true){
                 Thread.sleep(self.tickTime/2);
                 ......
                 syncedSet.add(self.getId())
                 for(LearnerHandler f:getLearners()){
                     if(f.synced() && f.getLearnerType()==LearnerType.PARTICIPANT){
                         syncedSet.add(f.getSid());
                     }
                     f.ping();
                 }
                  // syncedSet只有1个也就是自身，不符合>1/2的条件，报错并跳出
                if (!tickSkip && !self.getQuorumVerifier().containsQuorum(syncedSet)) {
                    shutdown("Only" + syncedSet.size() + " followers, need" + (self.getVotingView().size()/2));
                    return;
              } 
    }
}
```

报错的实质就是和leader同步的syncedSet小于固定的1/2集群，所以shutdown了。同时在代码里面我们又可以看到syncedSet的判定是通过learnerHander.synced()来决定。我们继续看下代码:

```
LearnerHandler
    public boolean synced(){
        // 这边isAlive是线程的isAlive
        return isAlive() && tickOfLastAck >= leader.self.tick - leader.self.syncLimit;
    }
```

很明显的，follower和leader的同步时间超过了leader.self.syncLimit也就是5 * 2 = 10s

```
zoo.cfg
tickTime = 2000
syncLimit = 5
```

那么我们的tick是怎么更新的呢,答案是在follower响应UPTODATE包,也就是已经和leader同步后，follower每个包过来就更新一次，在此之前并不更新。
![Image](https://mmbiz.qpic.cn/mmbiz_png/yiaiaFLiaflYRSh5ibqDTrYjnj6t1bAh5er0tttT7icMDVASzFJWML5XkY093c94IgvFkt8b1QaibWQMDGRlEbVIzbOA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
进一步推理，也就是我们的follower处理leader的包超过了10s，导致tick未及时更新，进而syncedSet小于数量，导致leader shutdown。
![Image](https://mmbiz.qpic.cn/mmbiz_png/yiaiaFLiaflYRSh5ibqDTrYjnj6t1bAh5er0yhedbofHiaNsJib1Lg7ePTiboiaO03k5AC5SpfOpMo2AgnL6E9kvtR5mWA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

### follower(ZK-B)第二种情况

带着这个结论，笔者去翻了follower(ZK-B)的日志(注:ZK-C也是如此)

```
2021-06-16 03:38:24 New election. My id = 3
2021-06-16 03:38:24 FOLLOWING
2021-06-16 03:38:24 FOLLOWING - LEADER ELECTION TOOK - 8004
2021-06-16 03:38:42 Getting a diff from the leader
2021-06-16 03:38:42 Snapshotting
2021-06-16 03:38:57 Snapshotting
2021-06-16 03:39:12 Got zxid xxx
2021-06-16 03:39:12 Exception when following the leader
java.net.SocketException: Broken pipe
```

又是Snapshot,这次我们可以看到每次Snapshot会花15s左右，远超了syncLimit。
从源码中我们可以得知，每次Snapshot之后都会立马writePacket(即响应)，但是第一次回包有由于不是处理的UPTODATE包,所以并不会更新Leader端对应的tick:

```
learner:
proteced void syncWithLeader(...){
outerloop:
    while(self.isRunning()){
        readPacket(qp);
        switch(qp.getType()){
            case Leader.UPTODATE
            if(!snapshotTaken){
                zk.takeSnapshot();
                ......
            }
            break outerloop;
        }
        case Leader.NEWLEADER:
            zk.takeSnapshot();
            ......
            writePacket(......) // leader收到后会更新tick
            break;
    }
    ......
    writePacket(ack,True); // leader收到后会更新tick
}
```

注意，ZK-B的日志里面表明会两次Snapshotting。至于为什么两次，应该是一个微妙的Bug,(在3.4.5的官方注释里面做了fix,但看日志依旧打了两次)，笔者并没有深究。好了，整个时序图就如下所示:
![Image](https://mmbiz.qpic.cn/mmbiz_png/yiaiaFLiaflYRSh5ibqDTrYjnj6t1bAh5er0JdoibkzrcpBvl1GH9RCIqtbDdlZepmmFBZ4N3Vdr2KSU8oWebetd7Qg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
好了，第二种情况也gg了。这一次时间就不是刚刚好出在边缘了，得将近30s才能Okay, 而synedSet只有10s(2*5)。ZK集群就在这两种情况中反复选举，直到人工介入。

## 调大syncLimit

针对这种情况，其实我们只要调大syncLimit，应该就可以越过这道坎。

```
zoo.cfg
tickTime=2000 // 这个不要动，因为和ZK心跳机制有关
syncLimit=50  // 直接调成50,100s!
```

## 线下复现

当然了，有了分析还是不够的。我们还需要通过测试去复现并验证我们的结论。我们在线下构造了一个1024G Snapshot的ZookKeeper进行测试，在initLimit=10以及syncLimit=5的情况下确实和线上出现一模一样的那两种现象。在笔者将参数调整后:

```
zoo.cfg
tickTime=2000
initLimit=100 // 200s
syncLimit=50  // 100s
```

Zookeeper集群终于正常了。

## 线下用新版本3.4.13尝试复现

我们在线下还用比较新的版本3.4.13尝试复现，发现Zookeeper在不调整参数的情况下，很快的就选主成功并正常提供服务了。笔者翻了翻源码，发现其直接在Leader.lead()阶段和SyncWithLeader阶段(如果是用Diff的话)将takeSnapshot去掉了。这也就避免了处理snapshot时间过长导致无法提供服务的现象。

```
Zookeeper 3.4.13

ZookeeperServer.java
public void loadData(){
    ...
    // takeSnapshot() 删掉了最后一行的takeSnapshot
}

learner.java
protected void syncWithLeader(...){
    boolean snapshotNeeded=true
    if(qp.getType() == Leader.DIFF){
        ......
        snapshotNeeded = false
    }
    ......
    if(snapshotNeeded){
        zk.takeSnapshot();
    }
    ......
}
```

还是升级到高版本靠谱呀，这个版本的代码顺带把那个迷惑性的日志也改了！

## 为何Dubbo-ZK有那么多的数据

最后的问题就是一个dubbo相关的ZK为什么有那么多数据了!笔者利用ZK使用的

```
org.apache.zookeeper.server.SnapshotFormatter
```

工具dump出来并用shell(awk|unique)聚合了一把，发现dubbo的数据只占了其中的1/4。
有1/2是Solar的Zookeeper(已经迁移掉，遗留在上面的)。还有1/4是由于某个系统的分布式锁Bug不停的写入进去并且不删除的(已让他们修改)。所以将dubbo-zk和其它ZK数据分离是多么的重要！随便滥用就有可能导致重大事件！

