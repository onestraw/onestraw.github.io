---
layout: single
author_profile: true
comments: true
title: LeetCode DFS(中下)
tagline: medium部分下
category: Algorithm
tags: [LeetCode, 面试]
---
下面是DFS Medium级别题目的后5道，列表目录：  

* [Convert Sorted Array to Binary Search Tree](#ch1)
* [Convert Sorted List to Binary Search Tree](#ch2)
* [Construct Binary Tree from Preorder and Inorder Traversal](#ch3) 
* [Construct Binary Tree from Inorder and Postorder Traversal](#ch4)
* [Clone Graph](#ch5)

#1.Convert Sorted Array to Binary Search Tree
[ref](https://oj.leetcode.com/problems/convert-sorted-array-to-binary-search-tree/)
<a name="ch1"></a>

> Given an array where elements are sorted in ascending order, convert it to a height balanced BST.

将一个升序数组转换成一个高度平衡的二分查找树BST，即[AVL树](http://zh.wikipedia.org/wiki/AVL%E6%A0%91)，递归方法很easy.

{% highlight c++ %}
TreeNode *BST(vector<int> &num, int start, int end)
{
	if (start > end)
	{
		return NULL;
	}
	int mid = (start + end) / 2;
	TreeNode *root = new TreeNode(num[mid]);
	root->left = BST(num, start, mid - 1);
	root->right = BST(num, mid + 1, end);
}

TreeNode *sortedArrayToBST(vector<int> &num)
{
	if (num.size() < 1)
	{
		return NULL;
	}
	return BST(num, 0, num.size() - 1);
}
{% endhighlight %}

#2.Convert Sorted List to Binary Search Tree
[ref](https://oj.leetcode.com/problems/convert-sorted-list-to-binary-search-tree/)
<a name="ch2"></a>
> Given a singly linked list where elements are sorted in ascending order, convert it to a height balanced BST.

###方法一
将有序链表转换成有序数组，再按照[Convert Sorted Array to Binary Search Tree](#ch1)方法来建BST树。  
缺点就是消耗了O(n)的额外空间，下策。
{% highlight c++  %}
TreeNode *buildTree(vector<int> &arr, int si, int ei)
{
	if (si > ei)
	{
		return NULL;
	}
	else
	{
		int mi = (si + ei) / 2;
		TreeNode *node = new TreeNode(arr[mi]);
		node->left = buildTree(arr, si, mi - 1);
		node->right = buildTree(arr, mi + 1, ei);
	}
}

TreeNode *sortedListToBST(ListNode *head)
{
	vector<int> arr;
	for (ListNode*p = head; p; p = p->next)
	{
		arr.push_back(p->val);
	}

	TreeNode *root = buildTree(arr, 0, arr.size() - 1);
	return root;
}
{% endhighlight %}

###方法二
我们多花点时间来查找链表的中间结点吧，时间复杂度还是O(n)，空间复杂度降到O(1). 
{% highlight c++  %}
TreeNode *buildTree(ListNode *head, int len)
{
	if (len < 1)
	{
		return NULL;
	}
	else
	{
		int mid = 0;
		ListNode *p;
		for (p = head; mid < len / 2; p = p->next, mid++);

		TreeNode *node = new TreeNode(p->val);
		node->left = buildTree(head, mid);
		node->right = buildTree(p->next, len - mid - 1);
	}
}
TreeNode *sortedListToBST(ListNode *head)
{
	int n = 0;
	for (ListNode*p = head; p; p = p->next, n++);
	return buildTree(head, n);
}
{% endhighlight %}

#3.Construct Binary Tree from Preorder and Inorder Traversal
[ref](https://oj.leetcode.com/problems/construct-binary-tree-from-preorder-and-inorder-traversal/)
<a name="ch3"></a>
> Given preorder and inorder traversal of a tree, construct the binary tree.
  Note:
  You may assume that duplicates do not exist in the tree.

根据先根遍历可以确定根结点，有了根结点，就可以分割中根遍历序列，左边是左子树，右边是右子树。所以题目中的假设前提很重要，没有重复的结点！

{% highlight c++  %}
TreeNode *build(vector<int> &preorder, int psi, int pei,
	vector<int> &inorder, int isi, int iei)
{
	if (psi > pei || isi > iei)
	{
		return NULL;
	}
	int imi = isi;
	for (imi; imi <= iei && preorder[psi] != inorder[imi]; imi++);
	assert(imi <= iei);
	int psep = imi - isi + psi;
	TreeNode *root = new TreeNode(preorder[psi]);
	root->left = build(preorder, psi + 1, psep, inorder, isi, imi - 1);
	root->right = build(preorder, psep + 1, pei, inorder, imi + 1, iei);
	return root;
}

TreeNode *buildTree(vector<int> &preorder, vector<int> &inorder)
{
	if (preorder.size() < 1)
	{
		return NULL;
	}
	return build(preorder, 0, preorder.size() - 1, inorder, 0, inorder.size() - 1);
}
{% endhighlight %}

#4.Construct Binary Tree from Inorder and Postorder Traversal
[ref](https://oj.leetcode.com/problems/construct-binary-tree-from-inorder-and-postorder-traversal/)
<a name="ch4"></a>

> Given inorder and postorder traversal of a tree, construct the binary tree.
  Note:
  You may assume that duplicates do not exist in the tree.

和[前一题目](#ch3)类似，后根遍历序列用来确定根结点，由这个根结点划分中根遍历序列，从而确立左右子树。  

{% highlight c++  %}
 TreeNode *build(vector<int> &inorder, int isi, int iei,
	vector<int> &postorder, int psi, int pei)
{
	if (psi > pei || isi > iei)
	{
		return NULL;
	}
	int imi = isi;
	for (imi; imi <= iei && postorder[pei] != inorder[imi]; imi++);
	assert(imi <= iei);
	int psep = imi - isi + psi - 1;
	TreeNode *root = new TreeNode(postorder[pei]);
	root->left = build(inorder, isi, imi - 1, postorder, psi, psep);
	root->right = build(inorder, imi + 1, iei, postorder, psep + 1, pei - 1);
	return root;
}


TreeNode *buildTree(vector<int> &inorder, vector<int> &postorder)
{
	if (inorder.size() < 1)
	{
		return NULL;
	}
	return build(inorder, 0, inorder.size() - 1, postorder, 0, postorder.size() - 1);
}
{% endhighlight %}

#5.Clone Graph
[ref](https://oj.leetcode.com/problems/clone-graph/)
<a name="ch5"></a>

> Clone an undirected graph. Each node in the graph contains a label and a list of its neighbors.  

> OJ's undirected graph serialization:  
  Nodes are labeled uniquely.  
  We use # as a separator for each node, and , as a separator for node label and each neighbor of the node.  
  As an example, consider the serialized graph {0,1,2#1,2#2,2}.  
  The graph has a total of three nodes, and therefore contains three parts as separated by #.  
  
> First node is labeled as 0. Connect node 0 to both nodes 1 and 2.  
  Second node is labeled as 1. Connect node 1 to node 2.  
  Third node is labeled as 2. Connect node 2 to node 2 (itself), thus forming a self-cycle.  
  Visually, the graph looks like the following:  
  
       1
      / \
     /   \
    0 --- 2
         / \
         \_/

邻接表法表示的图结构，每个node的标签是唯一的。主要的问题是避免重复clone结点。  
clone过程分两步  

1. 创建新节点，BFS图，用一个映射表map<int, UndirectedGraphNode*> gmap;记录创建过的结点；  
2. 连接新节点，BFS图，用set<int> visited;记录连接过的顶点标签;  
{% highlight c++  %}

struct UndirectedGraphNode {
	int label;
	vector<UndirectedGraphNode *> neighbors;
	UndirectedGraphNode(int x) : label(x) {};
};


class Solution {
public:
	UndirectedGraphNode *cloneGraph(UndirectedGraphNode *node) {
		//1. 遍历所有顶点，并创建新节点
		queue<UndirectedGraphNode*> gqueue;
		map<int, UndirectedGraphNode*> gmap;
		UndirectedGraphNode *p;
		if (node)
		{
			gqueue.push(node);
		}
		while (!gqueue.empty())
		{
			p = gqueue.front();
			gqueue.pop();

			if (gmap.find(p->label) != gmap.end())
			{//already exists
				continue;
			}

			gmap[p->label] = new UndirectedGraphNode(p->label);

			for (int i = 0; i < p->neighbors.size(); i++)
			{
				gqueue.push(p->neighbors[i]);
			}
		}
		
		//2. 将gmap中的节点连接起来
		set<int> visited;
		if (node)
		{
			gqueue.push(node);
		}
		while (!gqueue.empty())
		{
			p = gqueue.front();
			gqueue.pop();
			if (visited.find(p->label) != visited.end())
			{//already visited
				continue;
			}
			visited.insert(p->label);

			for (int i = 0; i < p->neighbors.size(); i++)
			{
				gqueue.push(p->neighbors[i]);
				
				gmap[p->label]->neighbors.push_back(gmap[p->neighbors[i]->label]);
			}
		}
		//
		if (node)
		{
			return gmap[node->label];
		}
		return node;
	}
}; 
{% endhighlight %}

###完
到此，LeetCode Esay和Medium级别的DFS类型题目已经完了，还剩下3道Hard。
