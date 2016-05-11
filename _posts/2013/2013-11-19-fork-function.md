---
layout: single
author_profile: true
comments: true
title: fork函数
date: 2013-11-19 19:08
comments: true
categories: Linux
tags: [Linux, C/C++, 面试]
---
> “一个现有的进程可以调用fork函数创建一个新进程，fork函数被调用一次，返回两次，两次返回唯一的区别是子进程返回值是0，而父进程的返回值则是新子进程的进程ID”——《APUE》

网上有一个面试题目，考fork运行机制的，一个简单的题目，扯那么多。先提高一下这个题目的难度，然后用一棵简单的二叉树给出答案，根据只有一个，就是开头那句话。
分析下面程序，一共有几行输出，它们的含义是什么（自己假定进程号）？

{% highlight c  %}

#include<stdio.h>
#include<unistd.h>

int main()
{
	pid_t pid1, pid2, pid3;
	pid1 = fork();
	pid2 = fork();
	pid3 = fork();
	printf("pid1=%4u\tpid2=%4u\tpid3=%4u\n",pid1,pid2,pid3);
	sleep(1);
	return 0;
}
{% endhighlight %}

##一、进程二叉树
<img src="/assets/images/fork.png" alt="fork" />

<strong>树中每一个非叶子节点表示一个进程在执行fork函数，往左儿子节点走，表示进程不变，往右儿子节点走，表示进入新创建的子进程。总共8个叶子节点，8个不同的进程，到叶子节点8条不同的路径，程序输出8行。</strong>

##二、Ubuntu下运行结果验证

<img src="/assets/images/fork-3.png" alt="fork-3" />  

（2014-7-3添加）注：要真正弄懂fork函数如何做到返回两次，去看Linux源码。
