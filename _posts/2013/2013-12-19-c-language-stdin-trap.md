---
layout: single
author_profile: true
comments: true
title: C语言标准输入流stdin陷阱
categories: [cprogram]
tags: [C/C++]
---
在C语言中，在程序开始运行时，系统自动打开3个标准文件：标准输入、标准输出、标准出错输出，对应的文件指针分别为 stdin、stdout、 stderr，这3个文件都与终端（命令行）相联系，所以在使用stdin、stdout、stderr时不需要进行打开操作，直接使用就行。

使用scanf或fscanf(stdin, , )从命令行读取用户输入时，由于stdin机制，常常出现意想不到的错误。

## 一、循环输入字符造成跳跃性提示输入

示例程序

<pre class="lang:c decode:true">	int i, n=5;
	char char_sel;
	for(i=0; i&lt;n; i++)
	{
		printf("%d\tinput your choice: [(Y)es/(N)o]?  ",i+1);
		//rewind(stdin);
		//fseek(stdin, 0, 0);
		//fseek(stdin, 0, SEEK_SET);
		//fflush(stdin);

		//scanf("%c", &amp;char_sel);
		fscanf(stdin,"%c", &amp;char_sel);
		printf("#####%c####\n", char_sel);
	}</pre>
	
输入一个字符时，最后按回车，导致stdin中多输入换行'\n'字符，这时stdin中有两个字符，当执行到scanf或fscanf时，它首先 检查stdin中的还有没有数据，有就直接读入变量，没有就提示用户输入。上面程序，输入一次，它其实读取了两个字符，所以跳过一次用户输入。

<strong>验证：</strong>第一次输入4个字符asdf，最后按回车，stdin其实读入5个字符，结果是直接循环5次结束。  


<strong>解决方法</strong>是在scanf或fscanf语句前加入下面3个中的1个。  

1、fflush(stdin)
刷新标准输入缓冲区，把输入缓冲区里的东西丢弃

2、rewind(stdin)
rewind()函数的功能将文件内部的位置指针重新指向一个流（数据流/文件）的开头

3、fseek(stdin, 0, SEEK_SET)
int fseek(FILE *stream, long offset, int fromwhere);
设置文件指针stream的位置，移动到fromwhere基址的offset偏移处
SEEK_SET表示文件头，也可以用0替换

## 二、输入int整数避免此类错

输入字符导致多输入一个换行符，它们都是1个字节，所以每次用户输入之后，第二次该用户输入时，程序会在stdin流中找到一个1字节数据'\n'，所以跳过。

那么如果我们输入的是一个int整型数据（或者任何大于1个字节的数据），那么附加输入的换行就不起作用了，因为换行符只有1字节，而我们要读入的是大于1字节的数据，所以程序每一次都会提示用户输入。

示例程序
<pre class="lang:c decode:true">	int i, n=5;
	int int_sel;
	for(i=0; i&lt;n; i++)
	{
		printf("%d\tinput your choice: [(Y)es/(N)o]?  ",i+1);
		//scanf("%d", &amp;int_sel);
		fscanf(stdin,"%d", &amp;int_sel);
		printf("#####%d####\n", int_sel);
	}</pre>
而如果我们一次性输入5个数字，中间用空格隔开，那么程序也会跳过后面的提示输入
如输入： 1 2 3 4 5

