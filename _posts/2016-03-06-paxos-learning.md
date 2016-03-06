---
layout: post
title: "Paxos协议 学习小结"
date: 2016-03-6 16:20
comments: true
categories: paxos
tags: zookeeper paxos
---



一直对Paxos协议比较感兴趣，之前对这个算法 有耳闻 但只是了解皮毛，最近在学Zookeeper，趁着这股新鲜劲，就花时间研究了下Zookeeper的选举算法实现，再重新学习了Paxos算法，这篇文章算是我的学习总结吧。

<!--more-->

## Basic Paxos 协议

Paxos 协议是一个解决分布式系统中 多个节点之间就某个值（提案value）达成一致（决议）的通信协议，也就是说，每个节点 提出的提案 会 对 同一个 提案内容 达成一致(`知悉 且 要commit成功`)。下面简称`提案ID`为 `ProposalID`，`提案内容`为`value`。

### 算法(提案的提出与批准)主要分为两个阶段:

**第一阶段 Prepare**  
**P1a：Proposer 发送 Prepare请求**  
Proposer 生成全局唯一且递增的ProposalID，向 Paxos 集群的所有机器发送 Prepare请求，这里不携带value，只携带 ProposalID 。  

**P1b：Acceptor 应答 Prepare**  
Acceptor 收到 Prepare请求 后，判断：收到的ProposalID 是否 比 之前已响应的所有提案的ProposalID 大：  

* 如果是，则：  
    (1) 在本地持久化 ProposalID，可记为`Max_ProposalID`。  
    (2) 回复请求，并带上 已Accept的提案中 ProposalID 最大的 `value`（若此时还没有已Accept的提案，则返回value为空）。  
    (3) 做出承诺：不会Accept 任何 小于 `Max_ProposalID`的提案。  
* 如果否：不回复。  

**第二阶段 Accept**  
**P2a：Proposer 发送 Accept**  
经过一段时间后，Proposer 收集到一些 Prepare 回复，有下列几种情况：  
(1)  回复数量 > 一半的Acceptor数量，且 所有的回复的 value 都为空，则 Porposer发出accept请求，并带上 `自己指定的value`。  
(2)  回复数量 > 一半的Acceptor数量，且 有的回复 value 不为空，则 Porposer发出accept请求，并带上 `回复中 ProposalID最大 的value`(作为自己的提案内容)。  
(3)  回复数量 <= 一半的Acceptor数量，则 尝试 更新生成更大的 ProposalID，再转 P1a 执行。

**P2b：Acceptor 应答 Accept**  
Accpetor 收到 Accpet请求 后，判断：  
(1) 收到的ProposalID >= Max_ProposalID (一般情况下是 `等于`)，则 回复 提交成功，并 持久化 ProposalID 和value。  
(2) 收到的ProposalID < Max_ProposalID，则 不回复 或者 回复 `提交失败`。  

**P2c: Proposer 统计投票**  
经过一段时间后，Proposer 收集到一些 Accept 回复`提交成功`，有几种情况：  
(1) 回复数量 > 一半的Acceptor数量，则表示 提交value成功。此时，可以发一个广播给所有Proposer、Learner，通知它们 已commit的value。  
(2) 回复数量 <= 一半的Acceptor数量，则 尝试 更新生成更大的 ProposalID，再转 P1a 执行。  
(3) 收到 一条`提交失败`的回复，则 尝试 更新生成更大的 ProposalID，再转 P1a 执行。  

**最后，经过多轮投票后，达到的结果是：**  
(1) 所有Proposer都 提交提案成功了，且提交的value是同一个value。  
(2) 过半数的 Acceptor都提交成功了，且提交的是 同一个value。  

### Paxos 协议 的几个约束：

* P1: 一个Acceptor必须接受(accept)第一次收到的提案;  
* P2a: 一旦一个具有value v的提案被批准(chosen)，那么之后任何Acceptor 再次接受(accept)的提案必须具有value v;  
* P2b: 一旦一个具有value v的提案被批准(chosen)，那么以后任何 Proposer 提出的提案必须具有value v;  
* P2c: 如果一个编号为n的提案具有value v，那么存在一个多数派，要么他们中所有人都没有接受(accept)编号小于n的任何提案，要么他们已经接受(accpet)的所有编号小于n的提案中编号最大的那个提案具有value v;  


### 常见的疑问、及异常处理：  

1. Paxos算法的核心思想是什么？  
(1) 引入了 多个Acceptor，避免 单个Acceptor成为单点。  
     Proposer用 更大ProposalID 来抢占 临时的访问权，避免 其中一个 Proposer崩溃宕机 导致 死锁。  
(2) 保证一个ProposalID，只有一个Proposer能进行到第二阶段运行，Proposer按照ProposalID递增的顺序依次运行。  
(3) 新ProposalID 的 proposer 采用 `后者认同前者`的思路运行。  
    * 在肯定 旧ProposalID 还没有生成确定的value (Acceptor 提交成功一个value)时，新ProposalID 会提交自己的value，不会冲突。  
    * 一旦 旧ProposalID 生成了确定的value，新ProposalID 肯定可以获取到此值，并且认同此值。  

    容错要求：  
    (1) 半数以内的Acceptor失效、任意数量的Proposer 失效，都能运行。  
    (2) 一旦value值被确定，即使 半数以内的Acceptor失效，此值也可以被获取，并不再修改。  

2. 工程实践中 `ProposalID` 怎么定？  
在《Paxos made simple》中提到，推荐Proposer从不相交的数据集合中进行选择，例如系统有5个Proposer，则可为每一个Proposer分配一个标识j(0~4)，则每一个proposer每次提出决议的编号可以为5*i + j(i可以用来表示提出议案的次数)。  
在实践过程中，可以用 `时间戳 + 提出提案的次数 + 机器 IP/机器ID` 来保证唯一性和递增性。  

3. 如何保证 更大的ProposalID的Proposer 不会破坏 已经达成的确定性取值value？    
在P2a阶段中，Proposer会以 `所有回复中ProposalID最大 的value` 作为自己的提案内容。  
其中，prepare阶段的目的有两个: 1) 检查是否有被批准的值，如果有，就改用批准的值。2) 如果之前的提案还没有被批准，则阻塞掉他们以便不让他们和我们发生竞争，当然最终由ProposalID 的大小决定。  

4. Paxos协议的活锁问题  
新轮次的抢占会使旧轮次停止运行，如果每一轮在第二阶段执行成功之前 都 被 新一轮抢占，则导致活锁。怎么解决？  
这个问题在实际应用会发生地比较少，一般可通过 `随机改变 ProposalID的增长幅度` 或者 `增加Proposer发送新一轮提案的间隔` 来解决。  

5. Paxos 运行过程中，半数以内的Acceptor失效，都能运行。为什么？   
(1) 如果 半数以内的Acceptor失效时 还没确定最终的value，此时，所有Proposer会竞争 提案的权限，最终会有一个提案会 成功提交。之后，会有半过数的Acceptor以这个value提交成功。  
(2) 如果 半数以内的Acceptor失效时 已确定最终的value，此时，所有Proposer提交前 必须以 最终的value 提交，因为，一个Proposer要拿到过半数的accept响应，必须同一个已提交的Acceptor存在交集，故会在P2a阶段中会继续沿用该value。   

6. 若两个Proposer以不同的ProposalID，在进行到P2a阶段，收到的prepare回复的value值都为空，则两个proposer都以自己的值作为value(提案内容)，向Acceptor提交请求，最后，两个proposer都会认为自己提交成功了吗？  
不会，因为Acceptor会根据ProposalID，批准执行`最大的ProposalID的value`，另一个会回复 `执行失败`。当proposer收到`执行失败`的回复时，就知道：当前 具有更大的ProposalID的提案提交成功了。  

7. 由于`大ProposalID` 可以 抢占 `小ProposalID` 的提交权限，如果 此时 Acceptor还没有一个确定性取值，有一个`具有最大ProposalID的proposer`进行到P2a阶段了，但这时 这个proposer挂了，会造成一种死锁状态（小ProposalID的会提交失败，但是 `具有最大ProposalID的proposer`却不能提交accept请求），如何解决这种死锁状态？  
不会产生这种死锁状态，acceptor回复`提交失败`后，proposer再生成 更大的ProposalID，下一轮可以用自己value提交成功。  


### Zookeeper做的调整及优化：

1. 刚挂掉的leader 优先 争抢 leadership。  
在运行的过程中，如果leader节点崩溃，此时，所有slave节点需要sleep几秒后，才能争抢leadership，而刚挂掉的节点重启后，可以马上争抢leadership。  
这么做，是为了`解决 leader节点由于负载过重 导致 假死`，优先选择 原来的leader节点 可以减少 同步的数据量。  

2. ProposalID以 `Zxid + 机器ID` 来确定，其中 Zxid表示 本节点 最近Commit的事务ID。  
Zookeeper会优化 选择 `拥有最新commit记录（Zxid最大）的机器`作为leader，如果Zxid相等，则选择机器ID更大的那个。  

3. Zookeeper中的节点兼具Acceptor,Proposer，而且没有明确区分Prepare阶段和Accept阶段。  
提案的ProposalID和value，用的同一个结构体Vote，每轮投票直接发Vote，其它节点收到后，如果比自己的提案大，则将 `自己的提案ProposalID和value` 更新为 `收到的Vote的值`，且广播向其它节点发送新的提案。  
经过多轮投票后，会使 每个节点的提案 趋于最终的那个值。  


## Multi Paxos 协议

可以看出，一个Basic Paxos Instance运行完，要经过 每个Proposer的prepare阶段、accept阶段，还需要在过半数的节点上commit成功，流程太多，导致性能不高。一般在实践过程中用Multi Paxos协议。  

Multi Paxos协议省去了Basic Paxos的prepare阶段，直接由一个leader节点提交提案，并通知其它Follower节点提交提案。在Leader租约内，只有Leader可以提出提案。如果leader节点崩溃宕机、或者租约到期后没有续约，则由 其它Follower 使用 Basix Paxos协议 竞争leadship。  

在使用Multi Paxos 协议做多节点的复制时，有一些细节需要注意。详细可以参考[这篇文章](http://oceanbase.org.cn/?p=111) 和 [这篇文章](http://www.cnblogs.com/foxmailed/archive/2014/02/19/3553277.html)。  


## 解决单点问题的几种思路

最近在项目中遇到一个需求，要解决一个中心节点(下称DC节点)的单点问题，我思考了下，解决方式有三种：  

1. vip + keepalived  

    布署两台机器(一主一备)，这两台用keepalive检测心跳，对外提供vip。每台机器都设置一个crontab脚本，每5分钟检查一下Dc进程在不在，如果不在，则拉起来。  
    外部用vip访问这个节点。如果一台机器的Dc进程崩溃了，则crontab脚本会拉起来。如果整台机器都挂了，则vip会切换到另一台机器的ip上。  

2. 用Zookeeper选举一个leader来对外服务，当leader节点挂掉后，再选举另一个节点。外部通过Zookeeper来实时 获取Dc的真实ip。  
    这种方法有个缺点：要布署一个zookeeper集群、且需要zookeeper先运行、其它节点后运行。  

3. 在Dc节点中内置Basic Paxos协议 来实现选举，对外提供接口`获取Dc leader节点的ip`。  


-----------------------

## 参考：

1. 《Paxos made simple》论文  

2. [图解分布式一致性协议Paxos](http://codemacro.com/2014/10/15/explain-poxos/)  

3. [图解zookeeper FastLeader选举算法](http://codemacro.com/2014/10/19/zk-fastleaderelection/)  

4. 《从Paxos到Zookeeper--分布式一致性原理与实践》 第2、3、4、7.6章节  

5. [一步一步理解Paxos算法](http://mp.weixin.qq.com/s?__biz=MjM5MDg2NjIyMA==&mid=203607654&idx=1&sn=bfe71374fbca7ec5adf31bd3500ab95a&key=8ea74966bf01cfb6684dc066454e04bb5194d780db67f87b55480b52800238c2dfae323218ee8645f0c094e607ea7e6f&ascene=1&uin=MjA1MDk3Njk1&devicetype=webwx&version=70000001&pass_ticket=2ivcW%2FcENyzkz%2FGjIaPDdMzzf%2Bberd36%2FR3FYecikmo%3D)  
    这篇文章讲得很浅显易懂，我就是通过这篇文章开悟的！  

6. [[Paxos三部曲之一] 使用Basic-Paxos协议的日志同步与恢复](http://oceanbase.org.cn/?p=90)  
    讲解应用Basic-Paxos解决 多节点 复制日志的一致性问题。  

7. [[Paxos三部曲之二] 使用Multi-Paxos协议的日志同步与恢复](http://oceanbase.org.cn/?p=111)  

8. [架构师需要了解的Paxos原理、历程及实战](https://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=403582309&idx=1&sn=80c006f4e84a8af35dc8e9654f018ace&scene=1&srcid=0119gtt2MOru0Jz4DHA3Rzqy&key=710a5d99946419d927f6d5cd845dc9a72ff3d652a8e66f0ddf87d91262fd262f61f63660690d2d5da76a44a29e155610&ascene=0&uin=MjA1MDk3Njk1&devicetype=iMac+MacBookPro11%2C4+OSX+OSX+10.11.1+build(15B42)&version=11020201&pass_ticket=bhstP11nRHvorVXvQ4pt9fzB9Vdzj5sSRBe84783gsg%3D)  

9. 视频 [paxos和分布式系统](http://www.tudou.com/programs/view/e8zM8dAL6hM/)  
百度的高级工程师对Paxos协议的解决，很不错！  

10. [Paxos在大型系统中常见的应用场景](http://timyang.net/distributed/paxos-scenarios/)  

11. [多IDC的数据分布设计(一)](http://timyang.net/distributed/multi-idc-consensus/)  

12. [多IDC的数据分布设计(二)](http://timyang.net/data/multi-idc-design/)  


