---
layout: post
title: "shell练习：批量修改目录下的文件后缀名"
date: 2014-03-23 14:20
comments: true
categories: Software_development Shell
tags: shell 文件后缀名
---



上个星期组长给我分配了一个任务，要将一个目录下的所有子目录的文件名转换成小写，文件的内容中有路径的也将转换成小写，功能可以用任何编程语言实现。我想了一下，这个用shell实现最方便，只需要一个函数遍历子目录，另一个函数做文件名替换的工作。

<!--more-->

具体的功能要求：  
1. 遍历一个文件夹中的所有子目录，将有大写后缀名的文件改成小后缀名的文件。  
2. 遍历一个文件夹中的所有子目录，将目录中的文件内容(一般存有一个文件的路径)中含有大写后缀名，转换成小写。  

### 修改文件后缀名，文件内容

``` sh 遍历当前目录下的文件，将文件中含有大写后缀名的内容，转换成小写
modify_file()
{
    cd $1
    for file in *;
	  do
		if [ -f $file ]
		then
			cat $file | sed 's/\..*/\L&/g' > "$file".tmp
			mv -f "$file".tmp $file
		fi
     done
}
```


``` sh 遍历当前目录下的文件，将文件的后缀名从大写改成小写
renamefile()
{
    cd $1
    for file in *;
	  do
		if [ -f $file ]
		then
			mv $file $(echo $file|sed 's/\..*/\L&/g')
		fi
     done
}
```
这里面没有复杂的东西，只是那个sed表达式`sed 's/\..*/\L&/g'`折腾了好久。

### 遍历目录下的所有子目录
``` sh 递归遍历一个目录下的所有子目录(深度优先遍历）
recusivedir()
{
	modify_file "./"
	
	for dir in `ls .`
	do
	   if [ -d $dir ]
	   then
		 echo $dir
		 cd $dir
		 
		 recusivedir
		 cd ..
	   fi
	done
}
```
递归遍历当前目录的所有子目录，函数执行时先调modify_file()，renamefile()，然后深度优先遍历各个子目录。

具体的可以看一下这两个示例代码，[代码1][link3]，[代码2][link4]。

[link3]:  https://github.com/baozh/code-snippets/blob/master/modify_file.sh
[link4]:  https://github.com/baozh/code-snippets/blob/master/rename.sh



















