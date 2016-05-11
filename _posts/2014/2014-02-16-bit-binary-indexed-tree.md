---
layout: single
author_profile: true
comments: true
title: 树状数组/二分索引树(BIT)
categories: [Algorithm]
tags: [C/C++]
---
在CF上做题时，碰到”Time limit exceeded”错误，程序中频繁的遍历连续子序列，当序列长度增加时，程序效率急剧下降，在此学习一种高效的数据结构——<strong>树状数组</strong>。

树状数组是一种数据结构，适合解决如下问题： 

定义问题如下：我们有n个盒子，可能的操作为：   

1. 往第i个盒子增加石子  

2. 计算第k个盒子到第l个盒子的石子数量(包含第k个和第l个)  

原始的解决方案中(即用普通的数组进行存储，box[i]存储第i个盒子装的石子数)， 操作1和操作2的时间复杂度分别是O(1)和O(n)。<strong>假如我们进行m次操作，最坏情况下， 即全为第2种操作，时间复杂度为O(n*m)</strong>。使用数状数组，它在最坏情况下的时间复杂度也为O(m log n)。  

<a href="http://hawstein.com/posts/binary-indexed-trees.html" target="_blank">树状数组的全面介绍</a>

C语言实现

<pre class="lang:c decode:true ">#include&lt;stdio.h&gt;
#define N 100
int a[N], c[N];

int LowBit(int x)
{
	return x &amp; (-x);
}
//求树状数组c
void GetArrayC()
{
	int i, j;
	for(i=0; i&lt;N; i++)
	{
		c[i]=0;
		for(j = i-LowBit(i)+1; j&lt;=i; j++)
			c[i] += a[j];
	}
}
//数组a前n项和
int Sum(int n)
{
    int sum=0;
    while(n&gt;0)
    {
         sum += c[n];
         n=n - LowBit(n);
    }   
    return sum;
}
//对c[i]增加x
void Change(int i, int x)
{
     while(i&lt;=N)
     {
          c[i] = c[i] + x;
          i = i + LowBit(i);
     }
}

//test
int main()
{
	int i;
	for(i=0; i&lt;N; i++)
		a[i] = i;
	GetArrayC();
	//求51+52+...+99的值
	printf("%d\n", Sum(99)-Sum(50));
	return 0;
}</pre>
特别是当求连续子序列和比较频繁时，该数据结构特别有效。
