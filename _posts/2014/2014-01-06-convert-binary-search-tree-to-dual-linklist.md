---
layout: single
author_profile: true
comments: true
title: 将二叉查找树转换成有序的双向链表
categories: [cprogram]
tags: [C/C++, 笔试]
---
<h1><strong>题目</strong></h1>
输入一棵二元查找树，将该二元查找树转换成一个排序的双向链表。

要求不能创建任何新的结点，只调整指针的指向。

	10
	/  \
	6   14
	/ \   / \
	4  8 12 16

转换成双向链表

4=6=8=10=12=14=16。

<h1><strong>分析</strong></h1>

二叉查找树有一个性质：

一个节点的左子树上所有节点的值均小于该节点的值，并且在图上观看就是左子树上有最大值的节点在最右侧

一个节点的右子树上的所有节点的值均大于该节点的值，并且在图上观看就是右子树上有最小值的节点在最左侧

用递归的思想，将一个节点A的左右子树分别变成双向链表，返回左子树的最大值节点Aleft和右子树的最小值节点Aright，最后将A、Aleft和Aright的指针连接起来。
<h1><strong>代码</strong></h1>

<pre class="lang:c decode:true">/*
author:SwordStraw

http://onestraw.net

*/
#include&lt;stdio.h&gt;
#include&lt;stdlib.h&gt;
//搜索二叉树节点
typedef struct BSTreeNode{
	int cargo;
	BSTreeNode *left;
	BSTreeNode *right;
}BSTreeNode;

//深度优先建树，假定该树的所有节点值均为正数, 输入负数和0表示回上一层父结点
BSTreeNode* createBSTree(int high)
{
	BSTreeNode *root;
	root = NULL;
	int value;
	printf("第%d层, 请输入节点的值: ", high);
	scanf("%d", &amp;value);
	if(value &gt; 0)
	{
		root = (BSTreeNode*)malloc(sizeof(BSTreeNode));
		root-&gt;cargo = value;
		root-&gt;left = createBSTree(high+1);
		root-&gt;right = createBSTree(high+1);
	}
	return root;
}

//返回二叉树的最右侧节点
BSTreeNode *rightOfTree(BSTreeNode *root)
{
	BSTreeNode *p;
	p = root;
	while(p &amp;&amp; p-&gt;right)
		p = p-&gt;right;
	return p;
}
//返回二叉树的最左侧节点
BSTreeNode *leftOfTree(BSTreeNode *root)
{
	BSTreeNode *p;
	p = root;
	while(p &amp;&amp; p-&gt;left)
		p = p-&gt;left;
	return p;
}
//把二叉搜索树转换成双向链表
//left指针指向驱节点，right指针指向后继节点
void ConvertToDulLinkList(BSTreeNode *cur)
{
	BSTreeNode *lchild, *rchild;
	lchild = rchild = NULL;
	if(cur-&gt;left)
	{
		ConvertToDulLinkList(cur-&gt;left);
		lchild = rightOfTree(cur-&gt;left);//左子树最大的
	}
	if(cur-&gt;right)
	{
		ConvertToDulLinkList(cur-&gt;right);
		rchild = leftOfTree(cur-&gt;right);//右子树最小的
	}
	if(lchild)
	{
		lchild-&gt;right= cur;
		cur-&gt;left=lchild;
	}
	if(rchild)
	{
		rchild-&gt;left=cur;
		cur-&gt;right=rchild;
	}
}
//遍历双向链表
//direction表示方向，0表示从后向前，1表示从前向后
void traverseTree(BSTreeNode *head, int direction)
{
	while(head)
	{
		printf("%d-&gt;",head-&gt;cargo);
		if(direction == 1)
			head = head-&gt;right;
		else if(direction == 0)
			head = head-&gt;left;
	}
	printf("\n");
}
//测试程序 
int main(int argc, char *argv[])
{
	printf("[深度优先建树, 假定该树的所有节点值均为正数, 输入负数和0表示回上一层父结点]\n");
	BSTreeNode *root;
	/*构造如下一棵二叉搜索树
	10
   /  \
  6    14
 / \   / \
4   8 12 16
	
	如果嫌输入麻烦，可直接将下面粘贴到命令行：10 6 4 0 0 8 0 0 14 12 0 0 16 0 0
	*/
	root = createBSTree(0);
	BSTreeNode *head, *tail;//链表的头和尾
	head = leftOfTree(root);
	tail = rightOfTree(root);

	ConvertToDulLinkList(root);
	printf("\n双向遍历\n");
	traverseTree(head,1);
	traverseTree(tail,0);
	return 0;
}</pre>
