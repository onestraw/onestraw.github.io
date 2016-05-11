---
layout: single
author_profile: true
comments: true
title: 堆溢出
categories: [Windows]
tags: [C/C++]
---

堆是用于动态内存分配的内存空间，malloc和new函数都是在堆上分配的内存。堆的另一含义就是数据结构中的大根堆、小根堆（也叫优先级队列）。
缓冲区溢出分为栈溢出和堆溢出。使用malloc等在堆上申请内存时，分配两部分，一部分是堆头信息，另一部分是数据区。

## 一个堆溢出的小程序
程序中对user和pass申请10个字节，其实分配了16个字节，测试中user[0]和pass[0]的地址相距24个字节，pass[0]前面8个 字节是堆头信息，这个信息对读写pass指向的数据没有影响，用于释放内存管理，free(pass)。在输入user时

1. 如果输入1-15个字符，不会出现溢出，程序可正常执行并在结束正常释放内存；
2. 如果输入16-23个字符，溢出覆盖了pass堆的头部信息，但不会覆盖pass指向的数据区，这样在最后free(pass)时，程序崩溃；
3. 如果输入超过24个字符，溢出覆盖了pass堆头部信息和数据区域，程序异常终止。

{% highlight c %}
#include<stdio.h>
#include<stdlib.h>
#include<string.h> 

int main(){
    char *user;
    char *pass;
    int n=10;
    user=(char*)malloc(sizeof(char)*n);
    pass=(char*)malloc(sizeof(char)*n);
    printf("user addr:0x%p\n",&user[0]);
    printf("pass addr:0x%p\n",&pass[0]);
    strcpy(pass,"ONESTRAW");
    printf("input username:\n");
    scanf("%s",user);
    printf("username: %s\n",user);
	printf("password: %s\n",pass); 
	free(user);
	free(pass);
	return 0;
}
{% endhighlight %}
