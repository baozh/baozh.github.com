---
layout: post
title: "如何记忆"
date: 2014-01-30 02:22
comments: true
categories: Miscellaneous
tags: 记忆 记笔记 Linux命令
---


关于记忆有很多方法论，这里只是记录一下我的一些胡思乱想。

### 工作上的问题
最近工作上老是健忘，很多内容想不起来，比如一个项目中某个程序段的位置，很久以前改的bug的位置，某个linux命令，gdb调试命令，telnet调试命令，wireShark过滤表达式的语法，正则表达式的语法......这些内容中常用的需要记忆的知识其实不多，自己花时间强记一下，应该不会太难。可是最近工作时，老是想不起来，经常要翻笔记，才能慢慢回忆起来。

<!--more-->

自己反思了一下，主要有几个原因：  

- **学得越多，忘得越快。**  
记得之前微博上[一位朋友说道][link_weibo]：
> 累到极点，头脑麻痹的时候，连自己叫什么，电话是什么都会忘记。我上周经历了一次，怎么都想不起来自己的电话号码，幸好还记得自己叫什么，记得老婆的电话号码。  

	我回复道：

	> 搞编程的好像经常会这样。大脑的存储容量有限，就像缓存一样，会造成局部信息的换入换出。所以，可以把一些重要信息(电话号码、帐号...)记录下来，减少记忆负担。把工作中经历的难题记录下来，减少再回忆时的时间。没时间记录的话，可简单做一个索引类的标识，以便以后查阅。

	去年下半年刚工作，需要学习的知识特别多。我喜欢在周末或者晚上有空的时候充电，学习一些自己感兴趣、与工作无关的一些知识。这样导致工作的内容经常会忘掉，或者要花很长一段时间才能回忆起来。我问了下我周围的同事，他们都没有这个问题。我比较要好的一位同事(兼好朋友)说他业余时间基本不学习，只是在工作的时候学习，尽量保证在工作的8小时内高效学习、高效工作，他思路很清晰，很少忘记知识。  
	
	人的大脑的记忆容量是有限的，当超出了记忆容量，大脑会把之前记忆的一部分内容清空，腾出空间记忆新的内容。而记忆又分短期记忆和长期记忆，短期记忆是刚接触的、刚记忆的新内容，还不稳固。长期记忆是经过不停地重复，在大脑已形成根深蒂固的印象的内容。比如自己的名字，电话号码。而大脑容量不够时，经常会换出短期记忆，然后将新学的知识填入腾出的空间。  
	
	下半年刚工作，工作上学习的内容基本是刚接触，还不稳固，属于短期记忆。而业余学的知识也特别多特别新，不稳固，也属于短期记忆。所以当大脑容量不够时，两块内容不停地在大脑中换入换出，导致短期记忆形成不了长期记忆，记忆效率降低了，工作效率也降低了。  
	
	后来我改变了学习策略，业余时间只学习自己感兴趣、与工作相关比较多的知识。这样在学习时，大脑不需要切换思维，学习效率高。而且工作上的内容也不容易忘记。等我工作久了，工作上的内容已经烂熟于心了(形成了长期记忆)，再考虑业余时间学习与工作无关的内容。试了一下，感觉效果还不错。  

- **过于依赖笔记，依赖记录的文本内容。导致了记忆惰性。**  
之前养成了学习记笔记的习惯，应该说这是一个好习惯，但也有一些副作用。在工作的时候，总是把一些遇到的问题记在本子上，不用心记住，总是觉得下次遇到，不停地重复，会自然而然记住的。然后每次Team leader过来问我的时候，自己总是支支吾吾，因为很多内容没记住，当时回答不出来。后来想了下，可能是因为平时养成了记笔记的习惯，导致不用心记忆内容，记忆能力得不到训练，记忆力下降了。后来，我摸索的解决方法是：工作上的内容一定要强行记忆，就算那块内容不重要、或者遇到的机会不多，也要强行记住。工作中交给的任务、尝试的方法、自己的思考也要记住，因为Team leader会问到。  

- **年纪大了，记忆力没以前好了。**  
这个原因应该每个人都有。大脑像一台精密的机器，当运行久了，会磨损、效率降低。所以运行一段时间后要停下来休息，加加油，润滑机器。当然，也不能停下来太长时间。大脑是需要训练的，越训练能力越大，速度越快。  


### 记忆的小技巧
我想分享一下关于记忆的小技巧。  

- **分清什么是需要快速强行记住的，什么是可以偷懒不记忆的。**  
这个很重要。也就是分清记忆的轻重缓急，急的重要的要快速强行记住，不重要的不紧急的可以先记下笔记，以后遇到了不停地重复，会慢慢记住。

- **减少记忆负担。**  
一直感觉程序员是特别累的，要学那么多知识，要记忆那么多知识，所以应该给自己适当减负。减负的方式有多种：比如砍掉不需要记忆或者不重要的内容；用一些电子工具(比如手机,ipad)把灵光一现的想法随时记录起来；养成记笔记的习惯；通过一些方式简化记忆的内容，让它容易记住。

- **记笔记。**  
这是一个久试不爽的秘诀。烂笔头胜过好记性。对于一些重要的、但是以后重复机会不多的知识，要记笔记。以后再遇到的话，只要查阅一下笔记就行了，减少再回忆的时间。怎么做笔记也是一门学问。如果时间比较充裕的话，我会记录得比较详细。时间比较少的情况下，会减少记笔记的量，只做索引。一般来说，时间都是比较紧的。我会把资料保存成文本文件，在文件中做一些批注、重要的加一些颜色标识，摘录一些重要的句子保存下来。看书时就在书上写写划划，在旁白做批注。把重要的点在目录或首页空白处中标注、记录。

	另外，不要依赖于笔记，小心记笔记导致的记忆惰性。

- **练习记忆，训练自己的记忆能力。**  
好刀经常不用会生锈，好的大脑经常不使用也会退化。平时应该有心培养自己的思考能力、记忆能力。比如通过演讲、多沟通、把自己学到的时候用简单的语言复述给别人听，通过多重复多训练，提高自己的记忆能力。

其实记忆是很不可靠的，几乎所有几年前的记忆都会忘掉，而平时记笔记的时间又很少，所以如何有效做知识的索引，很重要。就像刘未鹏所说的，[**你所拥有的知识并不取决于你记得多少，而在于它们能否在恰当的时候被回忆起来。**][link1]当你想用到之前学过的某块知识的时候，能够快速找到，很重要。

[link1]: http://mindhacks.cn/2009/03/28/effective-learning-and-memorization/

### 如何快速找到项目中的某个目标代码段
很羡慕一些老员工快速找代码、快速定位问题的能力。我自己找某块代码时，总是要翻好几个文件，折腾一阵才能找到。自己摸索了一下，摸到了一些小窍门：首先要对项目的每个文件非常熟悉，然后找到目标文件，从目标文件中搜索关键词(比如函数名，容易记的标记名，bug号之类)。平时用心记忆很重要，要记住项目的特征，重要代码段的关键词是什么。

### 如何记忆Linux命令
Linux上的工具都是命令行工具，各个工具都有自己的一套命令，记忆这些命令是比较头疼的。我摸索了一些方法，分享一下。

- **记笔记。**  
我平时有个小本子，专门记录常用的linux命令，比如各种shell命令，vim命令，gcc命令，gdb命令，make命令，git命令...平时遇到生僻的命令就记在本子上，使用到就查查。刚开始记这些笔记是比较花时间的，以后就是修修补补，不用花多少时间。

- **用alias做成一个好记的别名。**  
有的shell命令的参数特别难记，特别多，我记不住。就用alias命令将它重命名成一个好记的别名。比如tar(打包工具)经常要与gz,bz(压缩工具)结合使用，有一堆难记的参数，我就重命名成`tar_gz`,`tar_ungz`,`tar_bz`,`tar_unbz`，这样好记些。

- **写成shell脚本。**  
有的shell命令太长了，没法用alias重命名。就把常用的shell代码段用shell脚本包装一下，然后直接调shell脚本就行了。比如工作中会将编辑好的文件挂载到linux机器上进行编译调试，这个"挂载"命令太长了，我包装了一下，加了几个参数，以后直接调脚本方便多了，不用再记忆额外的东西了。




### 推荐阅读：

[1] [暗时间][link2]，刘未鹏  
[2] [学习密度与专注力][link3]，刘未鹏  
[3] [如何有效地记忆与学习][link4]，刘未鹏  
[4] [《把时间当作朋友》][link5],李笑来  

[link2]:http://mindhacks.cn/2009/12/20/dark-time/
[link3]:http://mindhacks.cn/2007/05/24/learn-to-focus/
[link4]:http://mindhacks.cn/2009/03/28/effective-learning-and-memorization/
[link5]:http://book.douban.com/subject/3609132/
[link_weibo]:http://www.weibo.com/2652916941/AsBgWdArl



