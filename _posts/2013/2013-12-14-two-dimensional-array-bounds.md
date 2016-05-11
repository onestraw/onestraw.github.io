---
layout: single
author_profile: true
comments: true
title: 二维数组越界访问
categories: [cprogram]
tags: [C/C++]
---
编程测试时的一个bug

我动态申请的一块连续的内存用作二维数组int arr[10][5]

测试时没注意维度就写了一个arr[5][5] = 100;

输出也没报错，仔细观察发现arr[6][0]的值变成了100

原因其实很简单，普遍的对于row*col的二维数组a[row][col]，访问a[i][j]等价于*(a[0]+i*col+j)，它并不检查 i 是否大于等于row，j 是否大于等于col，在C语言中是这样。

所以上述的arr[5][5]，用指针表示就是*(arr[0]+5*5+5) = *(arr[0]+30)

arr[6][0]等价于*(arr[0]+6*5+0)，也是*(arr[0]+30)，我们发现也可以用arr[0][30]来访问arr[6][0]。

静态申请的二维数组内存也是连续，也有上述“bug”.

对于内存连续的二维数组arr，我们可以这样遍历
<pre class="lang:c decode:true ">	for(i=0; i&lt;row; i++)
	{
		for(j=0; j&lt;col; j++)
			printf("%f\t", temp[0][i*col+j]);
		printf("\n");
	}</pre>
