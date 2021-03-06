---
layout: post
title: "Linux多线程编程tips"
date: 2015-11-29 16:20
comments: true
categories: Linux C/C++
tags: Linux 多线程
---

这是我学习Linux多线程编程、实际编程时 遵守的一些小tips，也算是阅读《Linux多线程服务端编程》的读书笔记吧。

<!--more-->

1. 用流水线，生产者消费者、任务队列这些有规律的机制，最低限度地共享数据，减少需要同步的场合。  
	比如可使用线程局部存储、原子计数类型来避免共享对象。
2. 若需要多个线程同时操作 new出来的对象指针时，应使用shared_ptr/scoped_ptr来管理对象的生命期，而不是用原始指针。
3. 尽量使用更高阶的并发编程构件，如TaskQueue、Producer-Consumer Queue、CountDownLatch等。
4. 不得已必须使用底层同步原语时，只使用非递归的互斥器和条件变量，慎用读写锁，不用信号量。
5. 关于互斥器(mutex)的建议：  
	- 用RAII手法封装mutex的创建、销毁、加锁、解锁四个操作。
	- 思考一路上( 调用栈上)已经持有的锁，防止因加锁顺序不同而导致死锁。
6. 多线程程序中不要用sleep/usleep()/nanosleep函数。  
	生产代码中线程的等待只有两种：  
	- 等待资源可用。要么等待在select/poll/epoll_wait上，要么等待在条件变量上。
	- 等着进入临界区(等在mutex)上以便读写共享数据。  
	
	等待某个事件的发生，正确的做法是用select等价物或condition，抑或高层同步工具。在用户态做轮询是低效的。  
7. 熟悉读写锁的特点、适用场景。  
	特点：rwlock比mutex 加锁的代价更高(因为它要更新当前reader的计数)。通常writer lock操作优先于reader lock，所以wlock会阻塞后来的rlock。  
	适用场合：读的频率大大超过写的频率。也对数据更新的时效性要求较高(要求读到的是最近最新的数据)。  
	如果对读取的时效性要求较高，且对写更新的及时性要求不高，则应该选择mutex。  
8. 要理解整个程序的线程模型，理解各个回调函数所处的线程语境，避免在回调函数中调用 破坏共享数据、或者调用 非线程安全的函数、或者执行耗时的操作导致其它任务的阻塞。
9. 熟悉glibc提供的函数是否是线程安全的，要使用线程安全版本的函数，如asctime_r/gmtime_t，strtok_r，gethostbyname_r等。
10. 善用\_\_thread关键字。经常使用它来缓存线程数据 来避免系统调用。  
	\_\_thread是GCC内置的线程局部存储设施，它的实现非常高效，比pthread\_key\_t快很多。使用规则：只用修饰POD类型(struct，内置类型)，不能修饰class类型(因为无法调用构造和析构函数)。可用修饰全局变量、函数内的静态变量。\_\_thread变量的初始化只能用编译期常量。  
11. 线程创建和销毁的守则：
	- 尽量用相同的方式创建线程，例如boost::thread库,muduo::thread库。
	- 在进入main()函数之前不应该启动线程。
	- 线程的创建最好在初始化阶段全部完成。
	线程的创建和销毁是有代价的，一个程序最好在一开始创建所需的线程，并一直反复使用。
	- 不要从外部强行终止线程(即不使用pthread_cancel)。因为强制线程的话，没有机会清理资源(内存、锁...)。
	- 如果确定要在外部终止一个线程，可以使用一个线程退出标志，当设置为true时，线程自然退出。  
	
12. 多线程与IO(网络IO、文件IO)：
	- 不要多个线程同时操作(读写、关闭、打开)同一个fd(socket fd,file fd,timefd...)。一个fd只能由一个线程拥有并操作。
	- 应该把对同一个epoll fd的操作(添加、删除、修改、等待)放到同一个线程中。  
		思考：当一个线程正阻塞epoll_wait()上，另一个线程往此epoll fd添加一个新的监视fd，会发生什么？
13. 用RAII来包装socket fd，来避免发生网络的串话。
	因为TcpConnection是网络库和用户代码共有的，它的生命期难以确定。muduo中使用shared_ptr来管理TcpConnection的生命期。
14. 尽量不要在多线程程序中调用fork()。因为Linux的fork()只克隆当前线程的thread of control，不克隆其他线程。即fork成的子进程，只有当前线程，其它线程都消失了。
15. 要在设计程序的时候，就确定线程模型。明确每个线程的职责，例如IO线程处理IO事件等，计算线程负责计算，数据库线程操作DB等。
16. 要在设计程序的时候，就考虑清楚有哪些共享对象，每个共享对象会暴露给哪些线程，每个线程是读、还是写、还是读写都会进行？


--------------

##参考

《Linux多线程服务端编程 - 使用muduo C++网络库》前4章  



