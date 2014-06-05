---
layout: post
title: "如何在C/C++中调用Shell脚本"
date: 2013-11-10 15:21
comments: true
categories: Software_development Shell
---

### 缘起
在Linux平台下开发程序时，经常要处理一些锁碎的事情，比如删除某个目录下符合某种特征的文件，安装程序到某个目录下，打包备份一个程序，这些在Linux中很容易用shell来处理。在开发后台程序时，也经常要处理程序的安装、升级、备份，通常这些功能用shell脚本实现。所以不可避免的，要在程序中调用shell命令或shell脚本。之前考虑过这个问题，但没有深究。最近在维护一个项目时，要在C++程序中调用shell脚本来实现程序的升级和备份，所以花时间研究了一下，遂成本文。

<!--more-->

### 方法一：使用system函数
```c++
#include <stdlib.h>
int system(const char *string);
```
`system()`会调用`fork()`产生子进程，由子进程来调用`/bin/sh-c string`来执行参数string字符串所代表的命令，此命令执行完后随即返回原调用的进程。

返回值：

- =-1:出现错误  
- =0:调用成功但是没有出现子进程  
- \>0:成功退出的子进程的id

如果`system()`在调用/bin/sh时失败则返回127，其他失败原因返回-1。若参数string为空指针(NULL)，则返回非零值。如果`system()`调用成功则最后会返回执行shell命令后的返回值，但是此返回值也有可能为`system()`调用/bin/sh失败所返回的127，因此最好能再检查errno来确认执行成功。

根据`system()`的返回值来判断shell脚本是否执行成功是一件比较繁琐的事情(参见这两篇文章：[博文一][link1]，[博文二][link2])，且无法取得shell脚本的返回值。所以通常只是用`system()`来调用一个shell命令，或较短的shell脚本。

[link1]:  http://blog.csdn.net/yangruibao/article/details/7255787
[link2]:  http://blog.csdn.net/ddkxddkx/article/details/7019408


### 方法二：使用popen函数，创建管道来连接两个程序的输入输出
```cpp
#include <stdio.h>
FILE* popen ( const char *command , const char *type );
int pclose ( FILE *stream );
```
`popen()`会调用`fork()`产生子进程，然后从子进程中调用`/bin/sh -c`来执行参数command的指令。参数type可使用r代表读取，w代表写入。依照mode值，`popen()`会建立管道连接到子进程的标准输出流或标准输入流，然后返回一个文件指针。随后进程便可利用此文件指针来读取子进程的输出流或是写入到子进程的标准输入流中。此外，所有使用文件指针(FILE*)操作的函数也都可以使用，除了`fclose()`以外。

具体来说：

`popen()`函数通过创建一个管道，调用`fork()`产生一个子进程，执行一个shell以运行命令来开启一个进程。这个进程必须由`pclose()`函数关闭，而不是`fclose()`函数。`pclose()`函数关闭标准I/O流，等待命令执行结束，然后返回shell的终止状态。如果shell不能被执行，则`pclose()`返回的终止状态与shell已执行exit一样。

type参数只能是读或者写中的一种，得到的返回值（标准I/O流）也具有和type相应的只读或只写类型。如果type是"r"，则文件指针连接到command的标准输出；如果type是"w"，则文件指针连接到command的标准输入。

`popen()`的返回值是个标准I/O流，必须由`pclose`来终止。前面提到这个流是单向的，所以向这个流写内容相当于写入该命令的标准输入；与之相反的，从流中读数据相当于读取命令的标准输出。

例：在~/myprogram/目录下有shell脚本`test.sh`(打印HOME环境变量)，在C程序中调用shell脚本，并获取返回信息。
```sh test.sh
#!bin/bash 
echo $HOME
```
在该目录下新建一个c文件`systemtest.c`，内容为：
```cpp systemtest.c
#include<stdio.h>

int main()
{
	FILE *fp;
	char buffer[80];
	
	fp=popen(“~/myprogram/test.sh”,”r”);    //以读方式，fork产生一个子进程，执行shell命令
	fgets(buffer,sizeof(buffer),fp);       //读取shell脚本中输出(stdout)的值
	printf(“%s”,buffer);
	pclose(fp);
	
	return 0;
}
```
执行结果如下：
```sh
bao@bao-Matrix:~/myprogram$vim systemtest.c
bao@bao-Matrix:~/myprogram$gcc systemtest.c -o systemtest
bao@bao-Matrix:~/myprogram$./systemtest
/home/bao
```

### 具体实现

根据程序开发需要，不仅需要执行shell脚本，而且要从脚本获取返回值。这个返回值可以是数值，也可以是字符串，或者任何标识脚本是否执行成功的值。

1. 调用`ExecuteShell()`传入脚本的路径及参数。在`ExecuteShell()`函数中，调用`popen()`执行shell脚本，通过fgets获取执行结果。执行结果中，1代表shell脚本执行成功，否则执行失败。（为什么不使用0作为成功判断，原因在于shell脚本发生任何不可控的错误时，stderr返回值都为0）
2. 在shell脚本中，输出结果用`echo/printf`输出来(echo 1表示执行成功。如果执行过程中失败，则将错误信息重定向到日志文件)。echo输出的，会通过管道，传出到stdout。因为`popen()`是以读方式打开shell脚本的。

其中，`ExecuteShell()`可以如下实现：
```c++
int ExecuteShell(LPCSTR pShellName, char *szFormat, ...)
{
	if(pShellName == NULL)
	{
		return SHELL_RET_FAIL;
	}

	char szParam[256] = {0};
	char szCommand[256*2] = {0};
	char szResult[256] = {0};
	FILE* fp = NULL;
	int dwRet = 0;

	va_list pvList;
	va_start(pvList, szFormat); 
	const u32 actLen = vsprintf(szParam, szFormat, pvList);   //把所有参数都存入szParam字符串中
	if((actLen <= 0) || (actLen >= sizeof(szParam)))
	{	
		return SHELL_RET_FAIL;
	}
	va_end(pvList);

	sprintf(szCommand, "%s %s", pShellName, szParam);     //取得执行shell脚本的命令szCommand
	fp = popen(szCommand, "r");         //以读方式，fork产生一个子进程，执行shell命令
	if(fp == NULL)
	{
		return SHELL_RET_FAIL;
	}

	fgets(szResult,sizeof(szResult),fp);                  //读取shell脚本中输出(stdout)的值，即用echo输出的东西。
	pclose(fp);

	dwRet = atol(szResult);      //如果输出不为1，则说明执行失败！
	if( dwRet != 1 )
	{
		return SHELL_RET_FAIL;		
	}

	return SHELL_RET_OK;
}
```
- 在代码中，无论失败还是成功，使用“echo”命令返回执行结果。由于`popen()`管道的存在，本来应该在标准输出显示的echo值被重定向到C/C++程序中。
- 脚本执行成功，根据约定，返回数值“1”作为执行结果。
- 程序过程中，如果发现任何错误，统一重定向到日志文件，并且echo输出到stdout。
- 脚本中使用重定向的方法，把标准输出和标准错误全部重定向到“/dev/null”。由业务判断操作成功与否，返回错误值。如下：
```sh
tar tvf $UPDATE_FILE | grep "tvm/tvm" >/dev/null 2>&1
```
具体的可以看一下这两个示例代码，[代码1][link3]，[代码2][link4]。

[link3]:  https://github.com/baozh/code-snippets/blob/master/ExecuteShell.cpp
[link4]:  https://github.com/baozh/code-snippets/blob/master/upgrade.sh




