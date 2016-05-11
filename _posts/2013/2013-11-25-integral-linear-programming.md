---
layout: single
author_profile: true
comments: true
title: 整数线性规划-ILP
categories: [Algorithm]
tags: [C/C++]
---

**ILP问题**是0-1背包问题扩展  

在约束条件(W0X0 + W1X1 +… WnXn) < C, Xi的三个可能的取值 {0, 1, 2 }

求(P0X0 + P1X1 +… PnXn)的最大值

它比0-1背包问题中的xi多出一个2，可以称为0-1-2背包问题，因此在递推关系式中多出一个比较项，仅此而已。


##递推关系

使用max[i][j]表示前 i 个整数，在不超过 j 的前提下，最大的PiXi值. 

		max[i][j] = Max(max[i-1][j], p[i] + max[i-1][j-w[i]], 2*p[i] +max[i-1][j-2*w[i]] );

##CODE

<pre class="lang:c decode:true">
/*integral linear programming*/
#include&lt;stdio.h&gt;
#include&lt;stdlib.h&gt;

/*整数线性规划*/
void ILP(int *p, int *w, int n, int c, int *x)
{
	//定义一个数组max[i][j]表示前 i 个元素，在c=j的前提下的最大值
	int i, j; 
	int **max = (int**)malloc(sizeof(int*)*(n+2));
	for(i=0; i&lt;=n; i++)
	{
		max[i] = (int *) malloc(sizeof(int)*(c+2));
	} 
	for(i=0; i&lt;=n; i++)
		max[i][0] = 0;
	for(i=0; i&lt;=c; i++)
		max[0][i] = 0;
	
	for(i=1; i&lt;=n; i++)
	{
		for(j=1; j&lt;=c; j++)
		{
			max[i][j] = max[i-1][j];
			if(j&gt;=w[i] &amp;&amp; max[i][j] &lt; p[i] + max[i-1][j-w[i]])
				max[i][j] = p[i] + max[i-1][j-w[i]];
			if(j&gt;=2*w[i] &amp;&amp; max[i][j] &lt; 2*p[i] + max[i-1][j-2*w[i]])
				max[i][j] = 2*p[i] + max[i-1][j-2*w[i]];
		}
	}
	
	int cleft= c;
	for(i=n; i&gt;0; i--)
	{
		if(max[i][cleft] == max[i-1][cleft])
			x[i] = 0;
		else if(max[i][cleft] == max[i-1][cleft - w[i]] + p[i])
			x[i] = 1, cleft -= w[i];
		else if(max[i][cleft] == max[i-1][cleft- 2*w[i]] + 2*p[i])
			x[i] = 2, cleft -= 2*w[i];
	}
	printf("max value :%d\n", max[n][c]);
}
/*program entry*/
int main()
{
	int c =22;
	int p[] = {0,10,15,6,4};
	int w[] = {0,4,6,3,2};
	int n = sizeof(p)/sizeof(p[0]) -1;
	int *x = (int*)malloc(sizeof(int)*(n+2));
	ILP(p, w, n, c, x);
	int i;
	for(i=1; i&lt;=n; i++)
		printf("x%d=%d\n",i, x[i]);
	return 0;
}</pre>
