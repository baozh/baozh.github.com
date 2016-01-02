---
layout: post
title: "分布式存储 理论与实践 初探 (二)"
date: 2015-12-29 08:20
comments: true
categories: Distributed_system
tags: 
---

## 数据分布

分布式存储系统需要将数据分布到多个节点，并在多个节点之间实现负载均衡。常见的数据分布的方式有两种：一种是哈希分布，如一致性哈希，典型的系统是Amazon Dynamo系统；另一种是顺序分布，即将表格上的数据按主键排序，并切分成多块数据，每个数据存到不同的节点中，典型的系统是Google Bigtable, Taobao Oceanbase。

<!--more-->

将数据分散存储到多台机后，要尽量保证每台的存储量、访问压力等是均衡的，一般需要一个总控节点定时收集所有工作节点的负载信息，然后将负载高的节点的数据迁移到负载低的节点，实现自动负载均衡。

1. hash分布

	哈希取模的方式很常见，例如将集群中的服务器按0到N-1编号(N为服务器的数量)，根据数据的主键(hash(key)%N)或用户id(hash(user_id)%N)计算哈希值，来决定将数据映射到哪一台服务器。但是，通常找到一个散列分布均衡的哈希函数是很困难的。因为，如果按照主键散列，那同一个用户id的数据可能会分散到多台服务器，使得存取同一个用户的多条记录会比较困难; 如果按照用户id进行散列，容易出现"热点"问题，即某个用户的数据量比较大，访问频率比较高。
	
	传统的哈希分布算法有个问题：当服务器上线或下线，或N值发生变化，数据的映射关系会完全打乱，造成大量数据需要迁移。一般有两种解决思路，一种是将哈希值与服务器的对应表专门交给一个中间服务器来管理，访问数据时，先计算哈希值，再从中间服务器获得对应的存储节点。这种方法会涉及到中间服务器的高可用，一般需要布多台，并考虑机器宕机等异常处理。而且在访问数据时，增加了一次与中间服务器的往返时间RTT，增加了时延。另一种方法是用一致性哈希算法(DHT)，它给集群中每个节点分配一个token，这些token构成一个哈希环。存取数据时，先计算key的哈希值，然后从哈希环中顺时针找大于等于此哈希值的token，这个token对应的节点即key实际存取的物理节点。DHT的优点是节点增删只影响哈希环中相邻的节点，使得需要迁移的数据量较小。

	哈希分布破坏了数据的有序性，只支持随机存取，不支持顺序扫描。如果涉及跨节点的数据存取操作，一般需要在应用层做处理。
	
2. 按范围顺序分布

	按范围顺序分布一般在分布式表格系统(如Bigtable, Hbase, Oceanbase)中较常见，一般是将表格的数据按主键排序，然后根据主键切分一个个范围(称为一个tablet)，由总控节点按照一定策略将一个个tablet分配到多台存储节点中。顺序分布一般采用B+树实现，每个tablet相当于叶子节点，随着数据的插入和删除，某些tablet会变大或变小，之后需要对大的tablet进行分裂，对小的tablet进行合并，且分裂合并过程中不能停 读写服务，实现难度较大。

## 复制

为了保证可靠性，数据一般要复制多份并存储到多个物理节点中。一般有三种复制模式：异步、强同步、半同步。

1. 异步

	异步复制指用户的写请求在没同步到slave节点时，就可以返回给客户端。同时，master有一个线程不断扫描新的commit log，然后新的修改记录(commit log)发送给slave节点，slave有线程接受更新操作，并回放commit log，然后通知master更新成功。如果slave宕机，重启后向master注册，将slave最新的日志点告诉master，master会为这个slave启一个同步线程从最新的日志点开始不断地传输更新操作。异步复制是Mysql默认的模式，实现较简单，但是如果master宕机，马上切换到slave节点，可能会丢失最近的一段数据。

2. 同步

	同步复制指每一个写操作需要先写入slave的磁盘成功后，然后写入master的磁盘成功后，才能返回客户端。这种模式会增加写操作的时延，且当slave宕机时，写操作会失败。有一种折衷的方法，master保存一个slave列表，每个写操作都需要同步到slave列表的所有机器，如果发现某个slave连不上，就从slave列表上删除，下次写操作时就不同步到这个节点上了。
	
3. 半同步

	半同步复制指N个slave节点，数据写入到其中的K个slave并写入master本地，就可以返回客户端。只要K>=1，就可以保证当master宕机时，可以通过分布式选举算法(如Paxos,raft)选择一个slave节点充当master继续提供服务。
	
**复制带来的问题：一致性、可用性、性能之间的权衡**

复制到多个副本，就要考虑多副本之间的一致性。如果采用强同步复制，可以保证多副本的一致性，但是写性能较差，且主副本之间出现网络问题或其它故障时，写操作将阻塞。如果采用异步模式，保证了系统的可用性，但是无法做到多副本的强一致性。所以在设计存储系统时，需要在一致性、性能、可用性之间权衡，在适当的场景下，采用合理的策略。也可以做一些折衷处理，强调其中某一个特性，适当兼顾另外两者。

## 可扩展性

**如何衡量可扩展性？**  
可扩展性不能简单地通过系统是否为P2P架构 或 是否能将数据分布到多个存储节点来衡量，应该综合考虑，下面列出几点：

- 扩展的机器数是否有瓶颈？比如只能扩展到500台，或者上万台。
- 扩容的自动化程度
- 扩容需要的耗时
- 扩展的灵活性。比如一次扩展需要扩一倍机器，还是任意数量的机器。

**若采用master/slave架构，master节点会成为扩展的瓶颈吗？**  
一般主流的分布式存储系统都是master/slave架构，且能够支持上万台的集群规模，一般不会成为扩展的瓶颈。

master节点用于管理所有存储节点，执行数据分布、异常处理、负载均衡等任务，它需要缓存很多元信息，内存容量可能会成为瓶颈。在设计时，可做一些优化措施，来适当减少master的负载。比如在Google GFS中，舍弃了对小文件的支持(减少了元数据的存储量)，把数据读写的控制权下放到chunkServer，通过客户端缓存元数据减少对master节点的访问等。

如果master节点成为瓶颈，可以在master节点和slave节点之间加一个中间层节点，每个中间层结点只维护一部分元数据，这样master只需要维护很少的中间层结点的信息，不会成为瓶颈。

**传统数据库扩容的缺点**  
传统数据库扩容的方法有：通过主从复制、读写分离 提高系统的读取能力，通过垂直拆分和水平拆分将数据分布到多个存储节点，通过主从复制将系统扩展到多个数据中心。当主节点出现故障时，可将服务切换到从节点。另外，当数据库整体服务能力不足时，可以根据业务的特点重新拆分数据进行扩容。但是在扩容时，会面临如下问题：

- 扩容不够灵活。一般采用双倍扩容的做法，很难做到按需扩容。
- 扩容耗时长、不够自动化。扩容时，需要迁移大量的数据，整个过程时间较长，容易出现异常情况。且数据分布的规则往往和业务相关，难以做到自动化扩展。
- 难以支持跨行/跨表、范围查询等操作。

**异构系统**

同构系统指同一存储组内的所有存储节点服务相同的分片数据。当某一个机器宕机需要增加一台备机，或者为了增加读性能而扩展一台存储机器时，需要向其它同组的结点拷贝整个结点的数据。由于数据量很大，耗时长，在拷贝过程中容易减少 被拷贝节点 的服务能力，会增加丢失整个结点数据的风险。

异构系统将数据划分成很多大小接近的分片，每个分片的多个副本分布到集群中的任何一个存储节点，同一个存储组内的多个结点存储服务不同的分片。这样，如果发生故障，原有的服务将由整个集群而不是某几个固定的存储节点来恢复，这样分担了压力，也减少了丢失整块数据的风险。

## 容错

####故障检测

**心跳**  
心跳是很容易想到的一种方法。master节点每隔一段时间向slave节点发心跳包，然后slave回一个心跳包。如果过了一定的间隔，master收不到slave的回复，则重试几次。若重试几次后，仍收不到slave的回复，则认为与slave的网络断开、或者slave宕机、停止服务。一般收不到心跳回复有几种原因：与slave的网络断开，slave操作系统崩溃、程序崩溃、slave工作繁忙无法回复。所以这带来了一个问题：如何确定这台机器已经宕机，或停止服务了？

**租约(lease)**  
在实践中，可以通过租约机制来进行故障检测。租约是一种带有超时时间的授权。假设机器A要检测机器B是否发生故障，机器A可以给机器B发放租约，机器B只能在租约有效期内提供服务，否则主动停止服务。机器B在租约快到期的时候，向机器B重新申请租约。当机器B出现故障时 或 与机器A之间的网络发生故障时，机器B的租约过期，从而机器A确定机器B不再向外提供服务，可以将机器B的服务迁移到其它机器上。

####故障恢复

**多副本的数据**  
分布式存储系统保存了数据的多个副本(例如，GFS缺省保存3份)，当某个副本失效后，分布式文件系统的master会在适当的时机启动副本复制，使得数据的副本数保持设定的数量，保证了数据的安全。

**worker故障**  
分布式存储系统的worker可能出现故障，master通过内置的heartbeat/lease监控所有worker的状态，一旦确认某个worker故障，master会把该worker保存的数据的副本个数减一，以便系统在适当时机启动副本复制以保证数据不会丢失。

**master故障**  
为了避免master成为系统的单点，master也有多个副本：其中一个是主master，其余为辅master，主master承担着master的职责，例如应答用户和worker的请求，记录操作日志等；辅master通过操作日志保持与主master的准同步。当主master发生故障后，在分布式选举协议作用下，一个辅master会升级成为主master，保证系统的继续运行。

**应用程序容错**  
出于容错和故障恢复的原因，分布式存储系统的上层应用程序不能假设它正在或将要使用哪个worker，也不能假设数据存储在或将要存储到哪个worker上，当应用程序需要使用数据时，客户端库将询问系统的master获得数据副本所在的位置，并向其中一个副本(通常是与该客户端网络“距离”最近的)发出数据请求，如果该worker在开始或者中途出现故障或因为其他原因无法完成该请求，则客户端库会自动转向另外一个副本，这对上层应用是完全透明的。


------------------------------------------------
##参考

《分布式系统工程实践》  
[杨传辉的博客](http://www.nosqlnotes.net/)  
[阳振坤的博客](http://blog.sina.com.cn/s/articlelist_1070095910_0_1.html)  
《大规模分布式存储系统》  
《大数据日知录：架构和算法》  
《分布式系统设计模式》PPT  汪源

