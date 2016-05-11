---
layout: single
author_profile: true
comments: true
title: 将一棵二叉查找对转换为它的镜像
categories: [cprogram]
tags: [笔试]
---
<strong>题目：</strong>输入一颗二元查找树，将该树转换为它的镜像，

即在转换后的二元查找树中，左子树的结点都大于右子树的结点。

用递归和循环两种方法完成树的镜像转换。

例如输入：

	8
	/   \
	6    10
	/ \   /  \
	5  7  9  11

输出：

	8
	/  \
	10   6
	/ \    /\
	11 9  7  5

这个题目没什么难度，关键在于对指针的理解，转换成镜像只需要交换每个节点的左右孩子指针。

下面只把两个函数贴出来，关于创建树，以及栈的操作前面几篇文章已写。

<strong>递归算法</strong>

只需要10行，recursiveMirrorBST函数

<pre class="lang:c decode:true">//递归，将二叉查找树转换为它的镜像
void recursiveMirrorBST(BSTreeNode *root)
{
	if(root)
	{
		recursiveMirrorBST(root-&gt;left);
		recursiveMirrorBST(root-&gt;right);
		BSTreeNode *p;
		p = root-&gt;left;
		root-&gt;left = root-&gt;right;
		root-&gt;right = p;
	}
}</pre>

<strong>非递归的算法</strong>

借助栈来完成操作，定义栈需要占用一些体积

<pre class="lang:c decode:true ">//循环，将二叉查找树转换为它的镜像
void loopMirrorBST(BSTreeNode *root)
{
	BSTreeNode *p, *q;
	Stack *s;
	s = (Stack *)malloc(sizeof(Stack));
	initStack(s);
	push(root,s);
	
	while(s-&gt;top &gt; -1)
	{
		p=pop(s);
		if(p-&gt;left)
			push(p-&gt;left, s);
		if(p-&gt;right)
			push(p-&gt;right, s);
		q = p-&gt;left;
		p-&gt;left = p-&gt;right;
		p-&gt;right = q;
	}

}</pre>
