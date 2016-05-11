---
layout: single
author_profile: true
comments: true
title: 最复杂的函数指针——signal函数
categories: [cprogram]
tags: [C/C++]
---


《APUE》10.3节开头的一个函数就打击了我，这个函数是就最简单的信号函数signal。它是第一个让我连声明都看不懂的函数。

    void (*signal (int signo, void (*func) (int) ) ) (int);

1. 最里面void(*func)(int) 很简单，它声明的一个函数指针func。
2. 最外面的一层和函数指针的定义也很像啊。
令 

    p = signal (int signo, void (*func) (int) )

则上面的声明可写成如下形式

    void (*p)(int);

这毫无疑问就是个函数指针。  
3. p 是就 signal函数的返回值，signal函数有两个参数，一个是int整数，另外一个就是开始所说的函数指针。  
4. 就是这么简单，signal函数有一个参数是函数指针，它的返回值也是函数指针，并且这两个函数指针的类型一样。   

### 质疑：
-------

**一个简单的信号函数为什么要定义这么复杂呢？**

signal函数的参数有一个函数指针可以理解，当信号signo发生时，执行func函数。   
但为什么返回值也要是函数指针呢？   
unix系统头文件`signal.h`中定义signal的返回值如下：

    #define SIG_ERR (void (*)())-1
    #define SIG_DFL (void (*)())0
    #define SIG_IGN (void (*)())1

为什么不直接定义成 

    int  signal (int signo,  void (*func) (int) );

费解，实在费解！  

不过现在基本被sigaction函数取代，sigaction函数返回值类型是int.

