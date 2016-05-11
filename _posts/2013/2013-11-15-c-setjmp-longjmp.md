---
layout: single
author_profile: true
comments: true
title: C语言异常处理setjmp/longjmp
categories: [cprogram]
tags: [C/C++]
---
常见的goto语句也可以实现跳转，但是仅限于当前函数内，使用setjmp()和longjmp()函数可以实现跨函数跳转，主要用途是<strong>实现C语言的异常处理机制</strong>，这两个函数包含在头文件setjmp.h中。数据类型jmp_buf用于保存恢复调用环境所需的信息，即程序调用环境上下文，运行时的堆栈信息。

1. setjmp(jmp_buf env);  
使用env记录现在的位置，用于将来跳转回此处。该函数初始化env缓冲区，<code>env</code>将被<code>longjmp</code>使用。如果是从<code>setjmp</code>直接调用返回，<code>setjmp</code>返回值为0。如果是从<code>longjmp</code>恢复的程序调用环境返回，<code>setjmp</code>返回非零值(这个非零值可由longjmp设置)。
2. longjmp(jmp_buf env, int k); 
回到env记录的位置，让它看上去像是从setjmp函数返回一样，该函数使setjmp函数返回 k，标记是由哪个longjmp函数返回的。特别的，如果k设置为0，setjmp返回1.

## 异常处理实例

{% highlight c %}
#include<stdio.h>
#include<string.h>
#include<setjmp.h>
static jmp_buf buf;

float division(int a, int b)
{
	printf("in division()执行除法\n");
	if(b==0)
		longjmp(buf,2);
	return a/b;
} 
void copy(const char *str)
{
	printf("in copy()执行字符串拷贝\n");
	if(strlen(str)>10)
		longjmp(buf, 3);
	char buffer[10];
	strcpy(buffer,str);
}

int main(int argc, char *argv[])
{	
	switch(setjmp(buf))
	{
		printf("setjmp()\n");
		case 0:
			printf("setjmp,保存运行环境上下文\n");
			division(10, 0);
			break;
		case 2:
			printf("longjmp out division()\n");
			printf("除数为0\n");
			static char s[]="123456789abc";
			copy(s);
			break;
		case 3:
			printf("longjmp out copy()\n");
			printf("存在缓冲区溢出漏洞\n");
			break;
	}
	printf("\nbefore exiting....\n");
	return 0;
}
{% endhighlight %}

注意事项：

	char s[]="123456789abc";
	
要么声明为static，要么提到setjmp()之前声明，否则会出现编译错误。

	crosses initialization of ‘char s [13]‘

## Another Example

{% highlight c %}
#include<stdio.h>
#include<setjmp.h>
static jmp_buf env;

void processInput()
{
	printf("input a number(1-9):");
	int n;
	scanf("%d",&n);
	if(n<1 || n>9)
		longjmp(env,1);
}

int main(int argc, char *argv[])
{	
	if(setjmp(env))
	{
		printf("input error\n");
	}
	processInput();
	printf("\nbefore exiting....\n");
	return 0;
}

{% endhighlight %}
