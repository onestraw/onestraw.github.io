---
layout: single
author_profile: true
comments: true
title: 并查集-UnionFind
categories: [Algorithm]
tags: [C/C++]
---
<strong>并查集是一种树型的数据结构，用于处理一些不相交集合的合并及查询问题。</strong>

Union-Find算法有两个操作

<strong>Find操作：</strong>

确定元素属于哪一个子集。它可以被用来确定两个元素是否属于同一子集。

<strong>Union操作：</strong>

将两个子集合并成同一个集合。

1、Union-Find简单实现

对于一个数组a[N], 用一个辅助数组id[N]表示数组a中每一个元素所属的集合，如果a[i]和a[j]属于同一个集合，那么id[i] = id[j].
<pre class="lang:c decode:true">//返回元素x所属集合
int find(int a[N], int id[N], int x)
{
	for(int i=0; i&lt;N; i++)
		if(a[i]==x)
			return id[i]
}
//判断两个元素是否属于同一集合
int sameSet(int id[N], int p, int q)
{
	return id[p]==id[q];
}
//合并第p个和第q个元素所属集合
void union(int id[N], int p, int q)
{
	int pid = id[p];
	for(int i=0; i&lt;N; i++)
		if(id[i]==id[q])
			id[i]=pid;
}</pre>
2、方法1在union操作时太慢，对每一个集合设置一个root，合并时，只需把一个集合的root作为另一个集合的子节点挂上去。但是这样会生成一棵很高的不平衡的树，导致向上搜索root时时间复杂度太高，加入路径压缩技术(path compression).
<pre class="lang:c decode:true">//将id数组改成parent数组更形象，返回一个元素所属集合
int root(int parent[N], int i)
{
	while(i!=parent[i])
	{
		parent[i] = parent[parent[i]];//路径压缩
		i = parent[i];
	}
	return i;
}
int sameSet(int parent[N], int p, int q)
{
	return root(parent,p)==root(parent, q);
}

void union(int parent[N], int p, int q)
{
	int proot = root(parent, p);
	int qroot = root(parent, q);
	parent[qroot] = proot;
}</pre>
（推荐一个普林斯顿大学的U<a title="UnionFind" href="https://www.google.com.hk/url?sa=t&amp;rct=j&amp;q=&amp;esrc=s&amp;source=web&amp;cd=4&amp;ved=0CEQQFjAD&amp;url=%68%74%74%70%3a%2f%2f%77%77%77%2e%63%73%2e%70%72%69%6e%63%65%74%6f%6e%2e%65%64%75%2f%7e%72%73%2f%41%6c%67%73%44%53%30%37%2f%30%31%55%6e%69%6f%6e%46%69%6e%64%2e%70%64%66&amp;ei=BkKSUqqmAqm1iQfvgoCICg&amp;usg=AFQjCNGWVIf2OqoCcaa9dusBziCBHAwI4Q" target="_blank">nionFind算法</a>教程）
