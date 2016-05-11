---
layout: single
author_profile: true
comments: true
title: INI配置文件介绍及编程使用
categories: [Windows]
tags: [C/C++]
---

.ini 文件是Initialization File的缩写，即初始化文件，是windows系统配置文件所采用的存储格式.

## ini文件格式：段+键+值

	[section]
	keyname1=value1
	keyname2=value2


## VC中ini文件操作API

常用的主要有三个函数，一个写入和两个读出函数

	BOOL WritePrivateProfileString();
	DWORD GetPrivateProfileString();
	UINT GetPrivateProfileInt();

具体参数可查MSDN，使用参见下面程序举例

## ini操作编程实战

{% highlight c %}
#include<windows.h>
#include<iostream>  
using namespace std;

int main(int argc, char *argv[])   
{        
	/*
		. 表示当前目录，两个反斜杠，有一个表示转义，另外一个表示路径分隔
		所以  .\\test.ini 表示当前目录下的test.ini文件
	*/
	/*
		User: 段名
		Name: 键名
		OneStraw.Net: 键值
		.\\test.ini: ini文件名
	*/
	WritePrivateProfileString ("User", "Name", "OneStraw.Net", ".\\test.ini");   
	WritePrivateProfileString ("User", "Age", "23", ".\\test.ini");   
    
	char name[100];
	int age = 0;  
	/*
	* 在test.ini文件中的[User]段中,查找键名为Name 的值，
	* 如果没有找到这个键，把第三个参数“Not Find”赋给name，
	* name是一个大小为100的缓冲区
	*/
	GetPrivateProfileString ("User", "Name", "Not Find", name, 100, ".\\test.ini");  
	/*
	* 类似地，此函数将找到的值返回给age
	*如果没有找到键名为Age的参数，把22作为默认值返回
	*/
	age = GetPrivateProfileInt("User", "Age", 22, ".\\test.ini");  
	
	printf("Name=%s\nAge=%d\n",name, age);

	return 0;
}
{% endhighlight %}
