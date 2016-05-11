---
layout: single
author_profile: true
comments: true
title: Linux终端显示下载进度
categories: [cprogram]
tags: [C/C++]
---
在Linux的终端下更新、下载、安装工具时，会有一个类似68%这样动态变化的进度值，这是如何做到的呢

我首先想到了清屏

Turbo中可以使用clrscr()，头文件canio.h

Dev-C++中可以使用windows.h中的system(“cls”);

但是终端那么多内容，不可能全部删除，再重新输出吧，效率特别低不说，执行一次cls，会将终端所有的内容清除，根本不可能再重新输出。

linux下的clear命令，是将能看到的这一“页”屏幕翻过去，你还可以通过滚动条查看历史记录。

所以我们看到的那些更新或下载进度的实现不可能是上面这种方法。

在网上查了查，原来只用一条简单的语句就可以实现，而且还是C语言最简单的函数，准确的说是一个字符”\r”

<strong>printf(“\r”);</strong>

在printf()第一个参数最后加上”\r”字符，下一次的输出就会覆盖这一行。

更新示例

<pre class="lang:c decode:true ">#include&lt;stdio.h&gt;
#include&lt;windows.h&gt;
int main()
{
	int i;
	printf("downloading SwordGame from onestraw.net...\n");
	for(i=0; i&lt;=100; i++)
	{
		printf("finish %d%%...\r",i);
		Sleep(100);
	}
	printf("\ndone\n");
	return 0;
}</pre>
