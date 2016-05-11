---
layout: single
author_profile: true
comments: true
title: 二维数组传参及引用格式简化
categories: [cprogram]
tags: [C/C++]
---
<div id="8T4a1XSWMlM27hn6F4+NUM91vbq8rwQeJYZJKQuQjhI=_142a03ca008:63e3c99:7fe12541_entryContent" class="content">

C语言中，二维数组的参数传递是一个比较复杂繁琐的问题，尤其是动态申请的二维数组。

动态申请二维数组、传递参数及引用的一般过程如下

1、在函数M中动态申请一个数组matrix[row][col]

2、函数M调用函数Func，并用matrix作为参数，形如Func(matrix[0], row, col)

3、在函数Func(int *m, int row, int col)中这样使用二维数组

读取m[i][j]，只能用*(m + i*col +j)来引用

下面我用一种方法让数组matrix在函数Func中一样如matrix[i][j]的简便形式引用。
<pre class="lang:c decode:true">int **citePointer(int *matrix, int row, int col)
{
	int i, **m;
	m = (int**)malloc(sizeof(int)*row);
	for(i=0; i&lt;row; i++)
		m[i] = (matrix +i*col);
	return m;
}
void initMatrix(int *matrix, int row, int col)
{
	int **m;
	m = citePointer(matrix, row, col);
	int i, j;
	for(i=0; i&lt;row; i++)
		for(j=0; j&lt;col; j++)
			m[i][j] = 119; 
}</pre>
用这种方法，空间开销多了一个指针数组，大小是二维数组的行数。

</div>
