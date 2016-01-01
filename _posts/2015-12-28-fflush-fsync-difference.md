---
layout: post
title: "[转] fflush与fsync的区别"
date: 2015-12-28 18:20
comments: true
categories: Software
tags: 
---

转自: http://blog.xiaoheshang.info/?p=274

1 来源

fflush为C标准库函数, fsync为系统函数。

<!--more-->

```cpp
	#include <stdio.h>
	int fflush(FILE *stream);
```

```cpp
	#include <unistd.h>
	int fsync(int fd);
```

2 参数

fflush的参数为FILE*，fsync为文件描述符。

3 功能

fflush: 把C标准库中的缓冲写到内核缓冲区
fsync: 将内核缓冲区的内容写入磁盘，所有的内容写入磁盘后才返回。  
C库缓冲区--fflush-->内核缓冲区--fsync-->磁盘


问：为什么要有个内核缓冲区，以前要实时写到磁盘我都是直接调用的fflush，好像没出现过问题啊？  
答：一般不会有问题，程序退出时，系统会将系统缓冲区的内容写到磁盘上。如果fflush之后突然断电，那么数据就可能丢失。  
fflush(fp); fsync(fileno(fp));可以保证数据完全写入磁盘。
















