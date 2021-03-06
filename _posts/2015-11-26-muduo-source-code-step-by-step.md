---
layout: post
title: "muduo网络库源码阅读Step by Step"
date: 2015-11-26 08:20
comments: true
categories: muduo C++
tags: muduo
---





一般写服务端程序都需要有一个称手的网络库来帮我们处理琐碎的网络通信细节，比如连接的建立、关闭，读取数据，发送数据，接收、发送缓冲区的管理等，常用的C/C++网络库有libevent，asio，libev，我们项目组使用的是[muduo网络库](https://github.com/chenshuo/muduo)。muduo是[陈硕](http://www.chenshuo.com/)写的，基于非阻塞IO和事件驱动的现代C++网络库，原生支持one loop per thread模型(即reactor模型)，它适合开发Linux下的面向业务的多线程服务端网络应用程序。<!--more-->在Linux上muduo的性能(吞吐量、高并发下的事件处理效率)比其它网络库都要好，编程接口也很友好，它使用了很多Boost C++的现代特性(比如用RAII来管理资源，用function/bind来代替虚函数作为库的回调接口，借助shared_ptr实现线程安全的对象回调等)，代码写得很精练简洁，而且源代码中附带有很多例子程序，是学习网络编程的很好范例，值得细细品读。

我从进项目组开始，花了6个月时间，把muduo相关的代码、例子程序、书都研究了一遍，感觉受益非浅，尤其是对处理网络通信的细节、异常处理了然于胸，写起代码来很顺溜。现在大概总结下阅读muduo网络库的顺序。


## Step 0       熟悉boost智能指针、function/bind的使用

在muduo库内部使用了boost::scoped_ptr,weak_ptr,shared_ptr来管理各个网络对象(channel,tcpconn,socket,acceptor, connector)的生命期，尤其是TcpConnection，是网络库和用户代码共有的，当某个连接不需要时，不能在用户代码中持有这个shared_ptr，否则会造成资源泄漏。更重要的是，在写代码时要学习、融入这种思想：用智能指针来管理动态分配的资源。如果能正确使用智能指针，在C++代码中一般不需要出现delete语句。muduo中另一个重要的思想是用function/bind来封装回调函数。function对象类似于函数指针(不过它可以带状态，即绑定一个类对象的成员函数)，在发生某些事件时，执行相关的回调函数。比如它连接断开、建立时会回调OnConnection()，数据读完后会回调OnMessage。在用户代码中，也可以模仿这种思想。

## Step 1       熟悉muduo库的编译、安装，使用
这个阶段主要摸熟muduo库的使用，即使用它封装的接口来编写网络应用程序。多敲敲代码，看书，把example目录下的所有例子程序都做一遍，这样差不多能处理常见的网络通信业务。然后考虑写更专业一点的网络服务程序，这时候会考虑：网络通信的消息格式怎么定？心跳协议怎么定？系统中 各个server如何分工，如何交互、连接？遇到些底层的网络bug怎么排查？如何做性能调优？

## Step 2       阅读muduo网络相关的代码，理解网络通信的各个细节
这个阶段主要研读muduo实现网络通信的各个细节(即net目录下的代码)。阅读时，多参考陈硕著的《Linux多线程服务端编程》第8章。跟着书、代码，走一遍，基本就能理解muduo网络通信的原理了。这个时候基本上解答了所有step 1产生的疑问，比如：从建立连接、读数据、写数据、关闭连接的流程是怎么样的？读写缓冲是是全局放一个，还是每个连接放一个？它的IO模型(即网络IO线程模型，谁来epoll_wait，谁来accept，如何分配连接到子线程)是什么样的？各个回调函数是处于主线程，还是子线程的语镜中？Connector主动发起连接要考虑哪些细节？

这个阶段就两个任务：  

- 理解muduo处理网络通信的流程，各种事件发生的时机。  
- muduo对各种网络异常的处理。  

最后要能够达到：在各种环境下，遇到网络问题，能够定位问题出在哪里。

## Step 3  阅读[muduo线程库](http://blog.csdn.net/solstice/article/details/5829421)，定时器，runInLoop实现
muduo的线程库封装了一些常用的多线程设施，比如thread,条件变量，锁，countDownLatch，线程池，线程私有变量，BlockingQueue等。没有boost thread库提供得多，但是也足够用了。有些东西封装得挺有特色，可以看看。定时器和runInLoop几乎是所有网络库的标配。muduo中定时器实现得较简单(用stl::set来存放所有timer)，用timerfd来获取定时事件的到来。具体可参考《Linux多线程服务端编程》第8.2节。runInLoop很有特色、很重要，它可以将一个线程的某些操作 转到 另一个线程来做，这样可以使很多非线程安全的函数变成线程安全的(比如定时器操作)，也可以将某些耗时的操作异步化(比如可以将数据库写操作放到另一个线程处理)。具体实现中，会将一个function对象绑定到另一个loop io线程的待执行列表中，loop io线程处理完所有网络事件、定时器事件后，就会执行所有绑定的function。里面用到了eventfd，来通知那个线程有待执行的funcion了。

可以看出，muduo使用了socket fd(网络IO事件), timerfd(定时器到来事件), eventfd(通知事件), signal fd(信号到来事件)来处理相关的所有事件，只需一个epoll循环，整个程序的处理流程很清晰简洁。

## Step 4  阅读[muduo日志库](http://blog.csdn.net/solstice/article/details/7639814)
muduo日志库的代码不多，很容易阅读。读的时候可参考《Linux C++多线程服务端编程》第5章，走一遍，大概能理解它的流程。阅读的时候，主要学习两点：

1.  从需求、现有的硬件等限制，学习如何设计并实现一个合格的日志库？  
	需求分为功能需求、性能需求。  
	- 功能需求主要考虑: 接口设计是否合理、易用？需要支持哪些功能？muduo日志库的设计原则是：尽量提供最精简的日志设施，不必要的就不提供。封闭的接口尽量便于程序员阅读、查错。
	- 性能需求考虑单位时间内可写入的最大日志量，且要求它不会阻塞正常的业务处理流程。muduo的设计原则：性能只需要“足够好”，即能达到现代硬盘的最大写入带宽即可。且在实现时，要考虑减少多线程的锁争用，尽量不阻塞正常的业务处理逻辑。

2. 如何做异常处理、性能调优？  
	日志库的实现逻辑基本大同小异，逻辑基本是：在各个业务线程中拼装日志串，然后将日志串存入一个Buffer（访问时需要加锁），另外有一个日志线程不停地从Buffer中取数据，然后将它输出到日志文件中。实现的时候会考虑：  
	- Buffer如何设计？什么时候唤醒日志线程从Buffer中取数据？
	- 如何减少 业务线程、日志线程 访问Buffer时的锁竞争？
	- 日志串如何组装，才能使它组装速度足够快、且要兼顾接口设计的易用性？
	- 要考虑线程间的竞争、写入速度等各种情况，保证不会丢失每条日志串。
	- 什么时候切换写到另一个日志文件？什么时候flush到日志文件？
	- 若日志串写入过多，日志线程来不及消费，怎么办？
	
	muduo日志库的性能很高，大概可以达到每秒200多万条，非常快。代码中做了很多性能调优，比如实现了一个memory output stream，来加快各种类型转换成字符串。利用线程私有变量缓存了一些变量值来加快日志串的组装。用双缓冲技术来减少线程之间的锁竞争、最大化一次性输出日志的吞吐量。阅读时注意每个优化细节，然后在平时做性能调优时 模仿它。

## Step 5  定制开发，修改源码。借鉴某些组件的实现，应用到自己的代码中

muduo网络库有很多功能不提供，比如SSL加密，配置业务线程的个数，设置多个监听端口。在实际应用时，可能根据需要，来修改muduo库。我们项目组做的修改有：  

1. 将各业务线程的个数、网络IO线程的个数、心跳线程的个数 等可配置化。  
	在实际应用中，会根据业务特点来决定线程模型。有的时候，会将一些耗时的操作（比如操作mysql）、压力大的操作单独配置一些线程来操作。
	
2. 一个TcpServer支持多个监听端口。  
	在实际高并发服务的项目中，一般都会有一个链接服务器，来接受并管理所有的外部链接。同时，它也需要与内部的server通信。这时候，就需要监听两个端口号。为了编程的方便，我们使muduo支持了一个TcpServer监听多个端口号。  
	*注：也可以不修改，可创建两个TcpServer，每个TcpServer监听一个端口号。*

3. TcpServer支持发起连接。  
	在实际项目中，经常需要一个服务端程序作为一个server来接受别人的连接，也需要主动发起一些连接到别的server上。为了编程的方便，我们使支持了TcpServer既可以发起连接，也可以接受连接。  
	*注：也可以不修改，创建单独的TcpServer来接受连接，创建单独的TcpClient来发起连接。*
	
4. 在thirdpartServer中利用runInLoop、定时器的实现。  
	在我们的项目中，有个server要与苹果的Apns服务通信，而Apns只接受Ssl加密通信，但是muduo不支持Ssl连接。当时考虑过，修改muduo使之支持Ssl通信，但是由于当时还不熟悉Ssl编程，直接修改muduo会带来一些不必要的风险，所以直接使用Linux上的原始函数、openSsl库来与Apns通信。通信时要用到定时器和runInLoop，我就把muduo的这部分实现代码借鉴进来，实现了这两个功能。


------------------------------------------------

##参考

《Linux多线程服务端编程 - 使用muduo C++网络库》    陈硕著  
[《Muduo 网络库：现代非阻塞C++网络编程》演讲](http://blog.csdn.net/solstice/article/details/7703959)  
[陈硕的博客](http://blog.csdn.net/solstice)  
[陈硕在Boolan上的培训](http://boolan.com/course/4)







