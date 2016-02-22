---
layout: post
title: "实现了一个Zookeeper C++客户端"
date: 2016-02-22 16:20
comments: true
categories: zookeeper
tags: zookeeper
---

最近学习了下开源的分布式协调系统zookeeper，在学习使用的过程中，感觉原生zookeeper c api比较难用、不方便，遂花时间实现了一个[C++客户端](https://github.com/baozh/zookeeper_cpp_client)，这篇文章主要记述实现过程中的一些思考。

<!--more-->

## 原生 Zookeeper C API 的不便之处

1. 不支持永久的Watcher注册  
zookeeper server为了性能的考虑，在触发了一次Watcher后，这个Watcher就失效了，需要客户端再重新注册Watcher。

2. session不支持重连  
如果发生网络原因，session会超时，需要客户端重连。

3. 节点数据的序列化/反序列化  
C API只支持存储string串，需要客户端处理对象的序列化/反序列化问题。

4. 不支持递归创建父子结点、递归删除分支结点。  

5. 接口、错误码不友好。  
Get接口把获取数据和注册Wacher合并在了一个API(Zookeeper为了处理异常，有意为之)，使得获取数据的错误码、注册的错误码混在一块，使用比较麻烦。


## CppClient 解决的问题

Java开源界有一些比较好的客户端ZkClient、Curator，但是搜罗了一下，C++方面没有好用的客户端，所以花时间实现了一个，主要解决以下几个问题：

1. 支持Watcher的永久注册  
C++ Client收到Watcher通知后，会再向Zookeeper注册Watcher。并且，也提供了接口 `取消Watcher的重注册`。

2. Client支持自动重连  
当session超时后，Client会启动一个定时器定时重连(默认支持重连)。并且，也提供了接口 `不支持重连`。

3. 提供接口，支持递归创建父子结点、递归删除分支结点。  
创建结点的时候 会判断 父结点是否存在，如果不存在，会先创建父结点。
删除结点的时候 会判断 是否有子结点，如果存在，会先删除子结点。

4. 接口友好、错误码归类  
分离了获取数据、注册Watcher的接口，并只对用户提供用得到的参数，对各种错误码做了归类、删减。

关于对象数据的序列化/反序列化，我参看了一些C++开源的序列化工具(如MessagePack)，由于C++在语言上不支持反射(Java利用反射的特性 在语言层面 支持了对象数据的序列化/反序列化)，需要在代码层面 在对象的定义中 嵌入一些宏定义，是侵入式的，所以考虑了下，让用户自己处理会比较好。


## 实现细节 

1. 线程模型  
全局创建三个线程，一个线程写日志，一个线程定时(1~10ms)检测session是否超时（如果超时，启动重连机制），一个线程做一些琐碎耗时的操作（如重注册Watcher、重连session、递归创建分支结点等）。

2. Client对象 使用shared_ptr的形式 来对外提供访问。  
由于Client对象 客户端库和用户代码都会访问，它的生命期比较模糊，所以内部、外部都使用boost::shared_ptr来访问Client。

3. 借鉴Curator，将Watcher归类成两种：NodeWatcher, ChildWatcher，并将Get接口分离成`注册Watcher`和`获取数据`两种接口。  
NodeWatcher监听节点的变更(节点删除，节点创建，节点数据变更)事件，ChildWatcher监听子节点的变更(增加、删除子结点)事件，且回调Watcher时会 提供最新的结点值 或 子结点列表。

4. 回调函数（Watcher回调、数据回调）采用了boost::function/boost::bind形式，并提供了同步、异步两种操作接口。

5. 抽象出了结点的version，提供 对指定version的结点的CAS操作。  
在Get结点数据时，会获取到结点的Stat结构，从其中抽取了version，之后在Set,Delete时，可根据指定的version做设置、删除操作。

## 使用注意事项

1. 调异步操作接口时，是立即返回的，若返回为true不能说明操作是成功的，必须要等回调返回的errcode参数来确定 操作是否成功。
	如果操作失败，则不会通知Watcher的。

2. 要注意 数据回调、Watcher回调的线程执行语境，它必须是线程安全的、不要执行耗时的操作。

3. 编译多线程版本客户端程序，需添加编译选项-DTHREADED，同时链接zookeeper_mt库。

## FAQ

1. Watcher本地被触发 到 再注册Watcher到Zookeeper，有一个时间差，如果这个时间段内，发生了数据修改、节点变更事件，是不会再触发Watcher了。  
这个问题Zookeeper的开发人员已经考虑（参考《Zookeeper : Distributed System Coordination》第22页）了，它提供的Get API合并了`注册Watcher`、`获取数据`两个操作，客户端虽然丢失了那个时间段的通知，但是能获取最新的数据值。  
注：若CppClient回调了ChildWatcher时，errcode指示 父结点已删除，这时在用户回调函数中，**必须创建父结点**，否则后续CppClient重注册Watcher会持续注册失败。

2. session断开又重连成功后，zookeeper会自动再注册之前的Watcher、比对版本，再通知。不需要客户端再注册。

3. 由于Zookeeper限制 分支结点必须是持久型的，所以在递归创建父子结点时，父路径涉及的结点都会创建成持久型的。

3. 什么时候会发生session expired错误？  
关于session event的各种state需要说明一下，当发生session event的时候可能是connected, connecting, associating,expired状态。  
（1） 当遇见connecting/associating状态时，说明连接正在建立还没完成，客户端必须检测这个connecting持续的时间，如果超过了会话的过期时间，那么就没必要建下去了，因为服务端一定也认为这个会话超时了。那么这个会话超时时间是客户端向服务端协商来的，仅仅在connected的时候调用zoorecvtimeout才能获取到，这种情景是客户端主动意识到session expired。  
（2）client先前建立了session，然后与zookeeper断开后一段时间都连接不上zookeeper，并且假设客户端没有主动意识到session expired，突然client又连上了zookeeper并试图恢复之前的session，被zookeeper告知session过期了，这时候会被watch通知一个expired的state，这是被动意识到session expired。   
之所以要实现主动意识expired，是因为如果client一直连不上zk，那么就永远不会触发watch的session expired stat，所以我们必须自己加一个定时检测： 其中session timeout在connected stat时记录下来，在触发session connecting的watch时，记录下连接建立的开始时间，并定期检测client处于connecting/associating状态下的持续时间，如果超过session timeout，那么可以认为session过期。  

4. Watcher触发的时机 要熟悉，可参考[这篇文章](http://tech.uc.cn/?p=1189)。  
注意：  
1）节点数据的变更 是指 **节点数据的版本** 变化，而不是 节点的**数据值**变化。只要成功执行了setData()方法，无论内容是否和之前一致，都会触发NodeWatcher。  
2）同一个zookeeper客户端对某一个节点注册相同的watch，只会收到一次通知。即Watcher对象只会保存在客户端，不会传递到服务端。  


-----------------------

## 参考：

[Zookeeper开发常见问题](http://tech.uc.cn/?p=1189)  
[Zookeeper C 客户端使用情景分析](http://tech.uc.cn/?p=974)  
《Zookeeper : Distributed System Coordination》  
《从Paxos到Zookeeper:从分布式一致性原理与实践》  
Java 客户端 [ZkClient](https://github.com/adyliu/zkclient), [Curator](http://curator.apache.org/)



