---
layout: post
title: "关于消息格式的一些思考"
date: 2015-11-27 08:20
comments: true
categories: Software_development
tags: 
---


刚进项目组的时候，各个server之间的消息格式较复杂，有好几个段(字节段)，每个段有bin串、也有json串、C struct，还需要标识每个段的长度、消息的类型等，这使得消息解析的时候很麻烦，要处理各种case。跨平台、跨语言的交互(比如和android交互)就更麻烦，经常要沟通联调好一会，才把消息调通。后来考虑调整一下消息格式，裁减不需要的段，简化交互协议。
<!--more-->


## 好的消息格式 特点：

1. 易写。就是解包、封包 代码写得少，实现方便。
2. 简单、一致。消息格式设计得越简单，两者交互时需要约定的东西就越少。产生的歧意越少，bug就越少。
3. 可扩展性好。比如之后下一版本，要对一个信令增加一个参数，不会影响现有的对端老程序。这点很重要。这点google proto buf/json/xml的扩展性不错，但是 直接传猓数据、C struct结构体 的可扩展性很差（因为 只要一方增加了一个参数，对端的程序就会崩溃）。
4. 性能好。就是生成的二进制串/json串/xml串 尽可能少（节省传输数据量），解包、封包速度 足够快。
5. 表达力够强，至少能表示 变长数据，比如 字符串、变长数组。

## 项目组使用的消息格式

刚开始项目中使用的消息格式是这样：  

　　　　　　　　　　　　　　　|<-------bit fields, c struct------->|<----------Json string---------->|  
+------------------------------------------------------------------------------------------------------------------+  
|　　　　BaseHead　　　　　　|　　　　extraData　　　　　　|　　　　rData　　　　　　　|  
+------------------------------------------------------------------------------------------------------------------+  

其中，BaseHead是消息头部，存放消息的元信息(消息类型，各个段的长度)。extraData段存放猓数据(bin串，bit fields, c struc)，rData段存放Json串。一个消息可能只有某一个段，也可能有两个段。

####这个消息格式的缺点

1. 封包、解包的逻辑比较繁琐，各种case比较多。  

	比如测试解析消息的各种case有：  
	1）只含有有extraData的情况  
	i) extraData第一次没收全数据，下一次收全了。  
	ii) extraData第一次数据就收全了。  
	2）只含有rData的情况测试:   
	i) extraData第一次没收全数据，下一次收全了。  
	ii) extraData第一次数据就收全了。  
	3）含有extraData,rData的情况  
	i) extraData,rData第一次数据就收全了。  
	ii) extraData第一次没收全数据，下一次收全了（extra,rdata都全了）。  
	iii) extraData第一次没收全数据，下一次收全了extraData，但没收全rData。之后一次，才收全rData。  
	iiii) extraData第一次收全了，但没收全rData。下一次收全rData。  

2. 使用了猓数据(bin串，bit fields, c struc)，导致不易升级(可扩展性差)、不可跨语言。  

	- 可扩展性差  
		在做下一版本的程序时，有些信令增加一些参数/修改一些参数，如果对端没有升级，就会导致对端老程序崩溃。
	- 容易出错  
		封包的时候要小心翼翼，要注意字节序。解析的时候也要小心翼翼，要注意struct中各个字段的顺序与发送方一致。
	- 不跨语言  
		让C/C++封包的消息 使用 Java来解析，估计到费一番功夫。而且要时刻同步这些语言的封包、解包代码，以后若有修改，稍不注意就会出错。

####关于可扩展性

有些项目组使用版本号来区分各个版本的消息格式，实事证明这是非常不靠谱的。拿Google Protocal Buffers文档举的反面例子：  

``` cpp
if  (version == 3) {
	//...
} else if (version > 4) {
	if (version == 5) {
		//...
	}
	//...
}
```

时间一长，版本号一多，代码中就会一堆又臭又长的switch-case，谁也弄不清版本之间的具体差别，也不敢轻易删除旧版本的代码，历史包袱就一直背下去。

解决方法是，采用某种中间描述语言来描述消息格式，然后生成不同语言的封包、解包代码。比如文本格式可考虑XML、Json，二进制格式可考虑Google Protocal Buffer。


####修改后的消息格式


|<----消息头BaseHead---->|<------------------Json string---------------->|  
+----------------------------------------------------------------------------------+  
|　　len　　|　　type　　|　　　　　　　data　　　　　　　　　　|  
+----------------------------------------------------------------------------------+  

其中，消息头包含len,type。len是整个消息的长度，type是消息类型(uint16_t类型)。所有消息的data段均采用json格式。  

消息类型是一个全局唯一的数字，所以需要在全局商定模块间的消息号范围。我们根据两两交互模块来划分不同的消息号范围：

	#define    NONE_MODULE_MSGID_BASE          0              //无方向性的信令交互   0~499
	#define    CS_MSG_MSGID_BASE               500              //CsServer与msgServer的交互   500~999  
	#define    MSG_OFFLINE_MSGID_BASE          1000           //msgServer与offline server的交互   1000~1499  
	#define    MSG_SENDER_MSGID_BASE           1500           //msgServer与sender server的交互    1500~1999  
	#define    CS_MOBILE_MSGID_BASE            2000           //CsServer 与手机sdk的交互   2000~2499  
	#define    SDK_CS_MSGID_BASE               2500           //sdk server与 CsServer的交互            2500~2999  
	#define    ACCESS_SENDER_MSGID_BASE        3000           //access server与sender server的交互     3000~3500  
	#define    DCSERVER_CS_MSGID_BASE          3500           //DcServer与 CsServer的交互            3500~3999  
	#define    DCSERVER_MSG_MSGID_BASE         4000           //DcServer与 msgServer的交互           4000~4499  
	#define    DCSERVER_SENDER_MSGID_BASE      4500           //DcServer与 sender server的交互       4500~4999  
	#define    DCSERVER_PRELOADER_MSGID_BASE   5000           //DcServer与preloader server的交互     5000~5499  
	#define    DCSERVER_OFFLINE_MSGID_BASE     5500           //DcServer与offline server的交互       5500~5999  
	#define    DCSERVER_ADMIN_MSGID_BASE       6000          //DcServer与admin 的交互               6000~6499  
	#define    DCSERVER_MONITOR_MSGID_BASE     6500          //DcServer与monitor的交互              6500~6999  


为每两个模块的交互分配500个消息号。如果要添加别的模块交互，则直接在后面加上相关的宏定义。每两个模块的负责人 各自商定 在对应的消息号范围 的消息类型、json串格式。

这种格式简单一致，可扩展性也较好，解决了上面的所有问题。


## 陈硕书上推荐的消息格式
[陈硕](http://www.chenshuo.com/)在《Linux多线程服务端编程》第7.5节推荐的消息格式：

+-----------------------------------------------------------------------------------------------------------------------+  
|　　len　　|　　nameLen　　|　　msgName　　｜　　　　protobuf data　　　　|  　checksum　｜  
+-----------------------------------------------------------------------------------------------------------------------+  

与上面的区别是：

1. 消息类型用字符串表示(而不是数字)，所以需要一个nameLen表示字符串的长度。用字符串代替数字的好处是：
	- 不需要全局分配消息类型的值，只需要两个交互的模块商定(确保在两个模块内唯一)即可。
	- 使用tcpdump抓包，很容易根据字符串确定消息类型，方便debug。
	
	缺点是增加了消息长度(增加一个nameLen和变长字符串)。

2. 使用了checksum段校验消息的准确性，来确保网络传输的可靠性。


--------------

##参考

《Linux多线程服务端编程 - 使用muduo C++网络库》第7.5节，9.6节  








