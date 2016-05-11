---
layout: single
author_profile: true
comments: true
title: C语言命令行输出彩色闪烁字体
categories: [cprogram]
tags: [C/C++]
---

##1.终端的color命令 
-----------

在终端中输入`color ?` 查看命令选项如下

		设置默认的控制台前景和背景颜色。
		
		COLOR [attr]
		
		attr        指定控制台输出的颜色属性
		
		颜色属性由两个十六进制数字指定 — 第一个为背景，第二个则为
		前景。每个数字可以为以下任何值之一:
		0 = 黑色       8 = 灰色
		1 = 蓝色       9 = 淡蓝色
		2 = 绿色       A = 淡绿色
		3 = 浅绿色     B = 淡浅绿色
		4 = 红色       C = 淡红色
		5 = 紫色       D = 淡紫色
		6 = 黄色       E = 淡黄色
		7 = 白色       F = 亮白色

		如果没有给定任何参数，该命令会将颜色还原到 CMD.EXE 启动时
		的颜色。这个值来自当前控制台窗口、/T 命令行开关或
		DefaultColor 注册表值。

在C程序中用system(“COLOR 02″); 调用color命令实现输出彩色文字。

但是color命令，改变终端所有文字的颜色，包括之前的输出和以后的输出，这并不是我们想要的结果。

##2.使用windows API
------------

		SetConsoleTextAttribute(HANDLE hConsoleOutput, WORD wAttributes);

设置标准输出stdout的文字及背景颜色，它改变的是当前和此后的输出文字颜色。  

在输出指定的内容后，再调用一次SetConsoleTextAttribute()，将文字颜色和背景改变为默认的灰色前景和黑色背景。  

这样我们可以改变某一行的文字颜色，而使其它输出不变。  

利用printf(“\r”)覆盖当前行和Sleep()延时 实现输出文字的闪烁效果  

##3. 测试代码
----------------

封装了两个函数，printColorStr()输出彩色文字，printTwinkleStr()输出彩色文字并闪烁  

		#include<stdio.h>
		#include <windows.h>
		/*
		*功能：
		    输出彩色字符串
		*输入参数：
		    color:颜色选择，1 red, 2 green, 3 yellow, default white
		    str: 彩色字符串
		*/
		void printColorStr(int color, char *str)
		{
		    HANDLE hd;
		
		    hd = GetStdHandle(STD_OUTPUT_HANDLE);
		    switch(color)
		    {
		        case 1: //red
		            SetConsoleTextAttribute(hd,
		                                    FOREGROUND_RED |
		                                    FOREGROUND_INTENSITY);
		            break;
		        case 2: //green
		             SetConsoleTextAttribute(hd,
		                                FOREGROUND_GREEN |
		                                FOREGROUND_INTENSITY);
		            break;
		        case 3: //yellow
		            SetConsoleTextAttribute(hd,
		                                FOREGROUND_RED | 
		                                FOREGROUND_GREEN |
		                                FOREGROUND_INTENSITY);
		            break;
		        default: //white
		            SetConsoleTextAttribute(hd,
		                                FOREGROUND_RED | 
		                                FOREGROUND_GREEN | 
		                                FOREGROUND_BLUE);
		    }
		
		    printf("%s",str);
		}
		/*
		*功能:
		    输入彩色闪烁字符串
		*输入参数：
		    count:闪烁次数
		    str:修饰的字符串
		*/
		void printTwinkleStr(int count, char *str)
		{
		    int i,len;
		    char *cstr;
		    len = strlen(str);
		    cstr = (char*)malloc(sizeof(char)*(len+2));
		    sprintf(cstr, "%s\r", str);
		    if(count%4==0)
		        count++;
		    for(i=1; i<=count; i++)
		    {
		        printColorStr(i%4, cstr);
		        Sleep(1000);
		    }
		    printColorStr(4,"\n");
		}
	
		int main()
		{
		    printTwinkleStr(6, "---->文字闪烁<----");
		    return 0;
		}
	
