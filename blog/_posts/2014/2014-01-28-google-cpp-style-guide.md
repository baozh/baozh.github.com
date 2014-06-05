---
layout: post
title: "Google C++编程风格指南"
date: 2014-01-28 07:45
comments: true
categories: Software_development C/C++
---


前几个月阅读了[《从缺陷中学习C/C++》][link1]、[《C++ primer 5th》][link2]，两本书都由潘爱民老师做序，在序中推荐结合《Google C++ Style Guide》一起阅读，所以最近花了点时间读完了这份文档。

<!--more-->

这份文档相当于是Google内部对C++的编码规范，阐述了在Google的项目中如何有效使用C++，应遵守哪些规范，避免C++的各种陷阱。对我来说，最有用的莫过于Google对C++的各种特性的取舍。C++是个由各个复杂特性组成的语言，在实际项目运用的时候总会困惑，是否要使用这个特性，应该如何使用，它的应用场合是什么。比如大家比较关心的：是否要使用异常，流的运用，智能指针的使用，函数重载、操作符重载如何运用，何时禁用拷贝构造函数和赋值操作符。

明白一个C++特性的使用场合很重要。它的优点、缺点是什么，使用的度是什么。这些很难在读一些文章后完全理解，只有实践中积累经验、多思考，才能在实践运用自如。实际上，在阅读这份文档的时候，还有很多不理解的地方。比如为何要禁用拷贝构造函数和赋值操作符，为什么不使用流，为什么不使用操作符重载。现在C++的标准库，或者其它各类库都广泛运用了C++的各种特性，如果在项目中规定不使用某些特性，如何与这些库配合编程。

最近经常思考的一个问题：如果在项目的编码规范中规定不使用某些特性，而使用的库却使用这个特性，那如何与这个库配合编程？比如常遇到一个问题：项目中规定不使用异常，但是在STL中使用异常的，如果STL抛出异常，你如何处理？一个广泛的例子是C++的new操作符，如果new失败，会默认抛出bad_alloc异常。可是在项目代码中，我要么看到不检查new的返回值，不catch异常，要么检查返回的指针是否为NULL。关于这点，《Effective C++》有比较好的论述。

我们项目组使用C++的方式是偏C风格的，基本上是C+Class+STL，最多加一点继承。实际上STL用得也不多，基本上使用自己定制的一套基础库。这套编码规范在实际项目中的运用效果不错，但我感觉还是保守了一些，一些好的C++特性，可以提高生产效率的，没有使用到。相比之下，Google的编码风格更加开放一些(但是是有限制的开放)，或者说规定得更加严格、细致，尽量让各种特性的使用限制在一定范围内。

这份文档也要有借鉴地使用，比如有些使用方式是Google特有的，不适合自己的项目场景(比如一些命名规范，某些C++特性的限制使用)，尽量借鉴其安全使用C++的思格(比如不让编译器做多余的动作，把自己的意图清晰地告诉编译器；限制编译器的自动推导，自动生成的代码)，在自己实践的过程中，摸索出自己的一套编码风格。尽量运用好C++的各种特性，提高生产率，又要安全，避免C++的各种陷阱，还要与项目组的其它代码保持一致性。

在学习过程中，阅读别人写的代码也是一种好的学习方式。比如之前看过陈硕写的muduo网络库，代码写得很漂亮，很有Google的风格，它在代码中熟练运用C++的高级特性(boost库，function/bind，智能指针，其实主要是RAII、事件回调)，陈硕称之为"现代C++的编码风格"。还有Google开源了很多C++项目(protobuf,leveldb,glog,gtest)，是不错的学习对象。


[link1]:  http://book.douban.com/subject/25716166/
[link2]:  http://book.douban.com/subject/25708312/


### 推荐阅读：
[1] [Google C++ Style Guide] [link3]  
[2] [Google C++编程风格指南][link4]  

关于C++中的流：  
[3] [C++的流设计很糟糕][link5]  
[4] [C++ 工程实践(7)：iostream 的用途与局限][link6]，陈硕  

关于C++中的错误处理：  
[5] [解读google C++ code style谈对C++的理解][link7]  
[6] [错误处理(Error-Handling)：为何、何时、如何][link8]，刘未鹏  
[7] [C++的另一种错误处理策略][link9]  

关于C++特性的裁减方案：  
[8] [编程语言的层次观点——兼谈C++的剪裁方案][link10]，孟岩  
[9] [用C设计，用C++编码][link11]，孟岩  

关于C++11新特性的运用：  
[10] [C++11（及现代C++风格）和快速迭代式开发][link12]，刘未鹏  
[11] [Elements of Modern C++ Style][link13]  
[12] [C++0X的三件好东西（零）][link14]，孟岩  
[13] [C++11 中值得关注的几大变化（详解）][link15]，陈皓  

[link3]: http://google-styleguide.googlecode.com/svn/trunk/cppguide.xml
[link4]: https://github.com/brantyoung/zh-google-styleguide/
[link5]: http://www.cppblog.com/converse/archive/2010/07/06/119427.html
[link6]: http://blog.csdn.net/solstice/article/details/6612179
[link7]: http://www.cppblog.com/converse/archive/2010/05/29/116689.html
[link8]: http://blog.csdn.net/pongba/article/details/1815742
[link9]: http://blog.jobbole.com/54699/
[link10]: http://blog.csdn.net/myan/article/details/1920
[link11]: http://blog.csdn.net/myan/article/details/1778843
[link12]: http://mindhacks.cn/2012/08/27/modern-cpp-practices/
[link13]: http://herbsutter.com/elements-of-modern-c-style/
[link14]: http://blog.csdn.net/myan/article/details/5877305
[link15]: http://coolshell.cn/articles/5265.html

