---
layout: post
title: "muduo中buffer的一个问题"
date: 2015-12-05 08:20
comments: true
categories: Software_development
tags: 
---


今天在压测系统性能时，遇到了一个现象：当短时间发送大量消息(每秒16万条消息)时，内存占用过多(增长到5.4G)，而且之后没有再释放。排查了一下这个问题，找到原因是：muduo中TcpConnection的发送缓冲区增长得太快，而缓冲区是自动增长的，没有自动缩容的功能，导致缓冲区占用的空间越来越大。


<!--more-->

### 疑似内存泄露
刚开始以为是内存泄露引起的，然后检查相关代码，涉及了动态分配内存的部分都仔细看了一遍，没有问题。用valgrind工具检测了一下，没有内存泄露。


### 排查发送数据的流程
仔细了检查了下所有相关的代码，感觉问题只会出在send()消息的过程中。send过程是：在业务线程中，将发送任务转到主线程(用runInLoop)，然后再转到网络IO线程(用runInLoop)。[网络IO线程处理发送任务](https://github.com/chenshuo/muduo/blob/master/muduo/net/TcpConnection.cc)：若当前这个fd不在写 且 发送缓冲区为空，则直接发送数据; 否则，将数据append到发送缓冲区，并注册可写事件。等下次写事件到来时，将发送缓冲区中数据一齐发出去。其中，只有两个地方会不断增长，一个是runInLoop时保存回调函数的vector的增长，一个是发送缓冲区的增长。

 <br /> 
 
####runInLoop的实现

runInLoop的实现在[EventLoop.h/cc文件](https://github.com/chenshuo/muduo/blob/master/muduo/net/EventLoop.cc)中，它的实现原理是：如果调用方的正处在此线程时，则直接执行这个函数; 否则，将回调函数(boost::function对象)存入vector中，并唤醒目标线程，之后目标线程从epoll中唤醒后，执行回调函数。保存回调函数的vector原型如下：

```cpp
typedef boost::function<void()> Functor;
std::vector<Functor> pendingFunctors_;
```

可以看出，这个vector只会自动增长，不会减小。如果短时间时，要注册大量的线程函数时，vector会增长得很大。

在程序代码中，一个runInLoop的参数是一个指针，另一个runInLoop的参数是string对象。总共发了5000W条消息，粗略计算一下vector占用空间：`50,000,000 * (8 + 8 + 8 + 20) = 2.048GB左右`。实际情况会少一半多，因为通知事件会触发目标线程来消费vector，会不停复用之前的空间。vector的占用空间估计在1GB左右。还在可控范围内，所以不做优化。

 <br /> 
 
####muduo中的buffer不支持自动缩容
muduo的[buffer](https://github.com/chenshuo/muduo/blob/master/muduo/net/Buffer.h)设计地很精巧，它将数据区域分成prependable,readable,writable三个段，用两个index来指向三个段之间的分界线。相邻两个段的空间可互相利用，可通过指针、数据腾挪来修改相邻两个段的大小。  
buffer的图示：

```cpp
/// A buffer class modeled after org.jboss.netty.buffer.ChannelBuffer
///
/// +-------------------+------------------+------------------+
/// | prependable bytes |  readable bytes  |  writable bytes  |
/// |                   |     (CONTENT)    |                  |
/// +-------------------+------------------+------------------+
/// |                   |                  |                  |
/// 0      <=      readerIndex   <=   writerIndex    <=     size
///
```

定义：

```cpp
  std::vector<char> buffer_;
  size_t readerIndex_;
  size_t writerIndex_;
```

**支持自动增长**  
buffer在append数据到writable段的时候，会检查当前vec.capacity()中writable段的空间是否足够大，如果不够，则分配相应的空间，然后把数据copy进去。

```cpp
  void append(const char* /*restrict*/ data, size_t len)
  {
    ensureWritableBytes(len);
    std::copy(data, data+len, beginWrite());
    hasWritten(len);
  }

  void ensureWritableBytes(size_t len)
  {
    if (writableBytes() < len)
    {
      makeSpace(len);
    }
  }

  void makeSpace(size_t len)
  {
    if (writableBytes() + prependableBytes() < len + kCheapPrepend)
    {
      buffer_.resize(writerIndex_+len);
    }
    else
    {
      // move readable data to the front, make space inside buffer
      size_t readable = readableBytes();
      std::copy(begin()+readerIndex_,
                begin()+writerIndex_,
                begin()+kCheapPrepend);
      readerIndex_ = kCheapPrepend;
      writerIndex_ = readerIndex_ + readable;
    }
  }
```

**但不支持自动缩容**  
在《Linux 多线程服务端编程 - 使用muduo网络库》7.4节中，指明如果buffer中空闲空间太多，不会自动缩小空间，需要手动调shrink()。

**验证，找出解决方法**  
在程序中加了一个telnet监控函数，在运行一个高峰的发送过程后，打印了相关TcpConnection当前的 接收、发送缓冲区大小，与程序总体消耗的内存空间一致。

解决方法有两个：  

- 在发送数据的时候，检查一下此连接的发送缓冲区的空闲空间是否过大，如果是，则shrink到一定大小。  
- 由于对端server处理得慢，本端server可调整下发送速度，发送速度与对端的处理速度一致即可。

*注：C++ STL 的 vector 容器在 clear() 之后不会释放内存，需要 swap(empty vector)，也可使用C++11 中的 shrink_to_fit() 函数。*





------------------------------------------------
##参考

[Muduo 设计与实现之一：Buffer 类的设计](http://blog.csdn.net/solstice/article/details/6329080)

