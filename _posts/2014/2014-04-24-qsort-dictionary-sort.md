---
layout: single
author_profile: true
comments: true
title: qsort实现字典序排序和字符串排序
categories: [cprogram]
tags: [C/C++]
---
<h1>问题一</h1>
一个物体有价格、重量和体积三个属性，均为整数，现在对n个物体进行从大到小的排序，规则是先按价格排序，如果价格相等，则比较重量，如果重量还相等，最后比较体积。

利用库函数qsort实现如下

<pre class="">
#include&lt;stdio.h&gt;
#include&lt;stdlib.h&gt;
typedef struct Node{
	int a[3];
}Node;
void printNodeArr(Node *node, int n)
{
	int i,j;
	for(i=0; i&lt;n; i++)
	{
		for(j=0; j&lt;sizeof(Node)/sizeof(int); j++)
			printf("\t%d",node[i].a[j]);
		printf("\n");
	}
	printf("\n---------------------------\n");
}
int cmpNode(const void *a, const void *b)
{
	Node x,y;
	x = *(Node*)a;
	y = *(Node*)b;
	int i;
	for(i=0; i&lt;sizeof(Node)/sizeof(int) &amp;&amp;x.a[i]==y.a[i]; i++);
	return x.a[i] - y.a[i];
}
//用qsort实现对多个字段的数据字典序排序 
void testSort()
{
	int i, j, n;
	n = 20;
	Node node[20];
	for(i=0; i&lt;n; i++)
		for(j=0; j&lt;sizeof(Node)/sizeof(int); j++)
			node[i].a[j] = rand()%10;
	
	printNodeArr(node,n);
	qsort(node, n, sizeof(Node), cmpNode);
	printNodeArr(node,n);	
}

int main()
{
	testSort();
	return 0;
}</pre>

<h1>问题二</h1>

对一组字符串进行排序，也利用qsort快速实现
<pre>int cmpStr(const void*a, const void*b)
{
	return -strcmp((char*)a, (char*)b);
}
//实现字符串排序 
void strSort()
{
	int maxLen = 10;
	int n = 5;
	char s[][10]={"bcdf","abcd","abcde","bcde","bbbbbbb"};
	qsort(s, n, sizeof(char)*maxLen,cmpStr);
	
	for(n=n-1; n&gt;-1; n--)
		printf("%s\n",s[n]);
}</pre>
