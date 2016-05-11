---
layout: single
author_profile: true
comments: true
title: 函数指针的用途
categories: [cprogram]
tags: [C/C++]
---

> “函数只不过是变量…….把函数主体看成值，函数名则为变量名称”——《head first javascript》
> 

函数名是一个32位的地址，它指示着函数的入口地址，故本质上函数名是一个32位的变量。可通过

	printf(“0x%x”, function_name);

或者

	printf(“0x%x”, &function_name);

获取函数的地址。
执行一个函数需要先把函数的入口地址放到EIP寄存器中。

# 函数指针

函数指针就是一个指向函数入口的指针，它存储了函数的入口地址，在使用中可以代替函数名。所有被同一指针运用的函数必须具有相同的参数和返回类型。   

我们也可以用一个unsigned int 型的整数获得函数的入口地址，为什么不能执行这个整数来执行特定函数呢？我尝试过了，只要你把这个整数放到EIP寄存器中，它就可以执行(暂不考虑有参数的复杂情形)。使用函数指针，编译器就帮你完成这个工作了。函数指针的声明及使用方法： 

{% highlight c %}
void hello(char *name)
{
	printf(“hello, %s\n”, name);
}

void (*p)(char*); //声明函数指针
p = hello; //或者 p = &hello; 作用一样
p(“OneStraw”); //或者  (*p)(“OneStraw”);    效果也一样。

{% endhighlight %}

# 函数指针的用途

1、简化代码，假如有三个函数，我们要循环执行20次，不使用函数指针和使用函数指针的代码如下。 

{% highlight c %}
#include<stdio.h>
int n = 20;
void zero()
{
	printf("function zero\n");
}
void one()
{
	printf("function one\n");
}
void two()
{
	printf("function two\n");
}

void simpleWay()
{
	char *func[]={&amp;zero, &amp;one, &amp;two};
	void (*p)();
	int i;
	for(i=0;i&lt;n;i++)
	{
		p = func[i%3];
		p();
	}
}
void complexWay()
{
	int i;
	for(i=0;i&lt;n;i++)
	{
		switch(i%3)
		{
			case 0:
				zero();
				break;
			case 1:
				one();
				break;
			case 2:
				two();
				break;
		}
	}
}
int main()
{
	simpleWay();
	printf("---------------\n");
	complexWay();
	return 0;
}
{% endhighlight %}

2、回调函数，函数指针作为参数，封装一个函数，函数指针作为参数，提高了灵活性。  
如C库函数qsort()声明如下  

	void qsort (void* base, size_t num, size_t size, int (*compar)(const void*,const void*));

其中compar就是一个函数指针，可以自定义比较函数，不限于一种类型的比较，通过控制compar可以递增排序，也可以递减排序。
数据包捕获函数pcap_loop()也使用了一个回调函数，原型如下：   

	int pcap_loop(pcap_t * p,int cnt, pcap_handler callback, uchar * user);

### 一个实例：

{% highlight c %}
#include<stdio.h>
/*这两个函数是我们自己定义的better标准*/
int smallIsBetter(int a, int b)
{
	return (a<b)?a:b;
} 
int largeIsBetter (int a, int b)
{
	return (a>b)?a:b;
}

/*假定这是别人封装好的函数*/
void complexFunction(int a, int b, int(*better)(int,int))
{
	int t;
	t = better(a,b);
	/*一系列复杂的过程，如快速排序*/
	t *=t*t;
	printf("%d\n",t);
}
/*program entry*/
int main()
{
	complexFunction(5,10,smallIsBetter);
	complexFunction(5,10,largeIsBetter);
	return 0;
}
{% endhighlight %}

# 区别于指针函数

指针函数是指返回值是指针的函数，即本质是一个函数，如

	float *max(float *arr, int n)


