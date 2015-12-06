---
layout: post
title: "RPC优缺点"
date: 2015-11-29 08:20
comments: true
categories: Software_development
tags: 
---


最近在阅读一些开源代码，看到好多项目是用RPC方式(而不是直接用消息传递)来进行模块间的通信，遂简单思考了下RPC的优缺点。

<!--more-->

**优点**

1. 交互方式简单，一个service就是一个interface。client/server间的交互协议容易统一。  
	一般成熟的公司都自己维护的RPC框架(比如百度的[sofa-pbrpc](https://github.com/baidu/sofa-pbrpc), google的[gRpc](https://github.com/grpc/grpc)，使用它们非常简单，只需要一个proto文件就可以描述两边的协议交互。因为描述文件(proto文件)是确定的，所以两边容易保持一致，基本不会出错。而且大多可用RPC框架生成所有interface的封包、解包代码，用户只需要调调函数即可。

2. 测试方便。  
	大多RPC框架是跨语言的，所以可用更方便的脚本语言(如python)写测试程序，模拟与C/C++程序的交互。
	
**缺点**

1. 多了一层间接性，出现网络问题时，debug比较困难。

2. 交互方式单一，不能进行复杂的多模块之间的协议交互。  
	RPC的描述方式较单一，一般是一应一答。有一些场景没法用RPC描述，比如有一个服务需要协调多个模块、服务端主动发通知给客户端。
	
3. 异常处理困难。
	- 如果是同步调用，长时间阻塞怎么办？
	- 如果是异步调用，长时间没有得到结果怎么办？
	- 如果判断failure(网络不通、参数不对、心跳超时)？  
		可以重试吗？调用语意：至多一次、至少一次、恰好一次？
		
------------------------------------------------
##参考

[分布式系统的工程化开发方法](http://blog.csdn.net/solstice/article/details/5950190)

