---
layout: single
author_profile: true
comments: true
title: LeetCode BFS(上)
tagline: easy部分
category: Algorithm
tags: [LeetCode, 面试]
---
这里是LeetCode BFS类型前半部分题目：  

* [Binary Tree Zigzag Level Order Traversal ](#ch1)
* [Binary Tree Level Order Traversal ](#ch2)
* [Binary Tree Level Order Traversal II ](#ch3) 
* [Clone Graph](#ch4) 

#1.Binary Tree Zigzag Level Order Traversal 
[ref](https://oj.leetcode.com/problems/binary-tree-zigzag-level-order-traversal/)
<a name="ch1"></a>
> Given a binary tree, return the zigzag level order traversal of its nodes' values. (ie, from left to right, then right to left for the next level and alternate between).

For example:
Given binary tree {3,9,20,#,#,15,7},

		3  
	   / \  
	  9  20  
		/  \  
	   15   7  

return its zigzag level order traversal as:

	[  
	  [3],  
	  [20,9],  
	  [15,7]  
	]  

{% highlight c++  %}
struct ZigZag{
	TreeNode *node;
	int level;
	ZigZag(TreeNode *p, int n) : node(p), level(n){}
};
class Solution {
public:
	vector<vector<int> > zigzagLevelOrder(TreeNode *root) {
		vector <vector<int> > v;
		ZigZag pz(NULL,0);
		stack<ZigZag> sz;
		int child_level = 0;
		if (root)
		{
			sz.push(ZigZag(root,0));
		}
		while (!sz.empty())
		{
			pz = sz.top();
			sz.pop();
			if (v.size() < pz.level + 1)
			{
				vector<int> tmp_v;
				v.push_back(tmp_v);
			}
			if (pz.level % 2 == 1)
			{//from right to left
				v[pz.level].push_back(pz.node->val);
			}
			else
			{//from left to right
				v[pz.level].insert(v[pz.level].begin(), pz.node->val);
			}

			if (pz.node->left)
			{
				sz.push(ZigZag(pz.node->left, pz.level + 1));
			}
			if (pz.node->right)
			{
				sz.push(ZigZag(pz.node->right, pz.level + 1));
			}

		}
		return v;
	}
};
{% endhighlight %}

zigzag遍历很简单，但是我封装了TreeNode，新造了一个数据结构ZigZag来保存每一个节点的层次，不够优雅，有没有什么方法不用封装呢，看下面。

#2.Binary Tree Level Order Traversal 
[ref](https://oj.leetcode.com/problems/binary-tree-level-order-traversal/)
<a name="ch2"></a>
> Given a binary tree, return the level order traversal of its nodes' values. (ie, from left to right, level by level).

For example:
Given binary tree {3,9,20,#,#,15,7},

		3  
	   / \  
	  9  20  
		/  \  
	   15   7  

return its level order traversal as:

	[  
	  [3],  
	  [9,20],  
	  [15,7]  
	] 

{% highlight c++  %}
vector<vector<int> > levelOrder(TreeNode *root) 
{
	vector<vector<int> > v;
	typedef pair<TreeNode*, int> PAIR;
	queue<PAIR> q;

	if (root)
	{
		q.push(PAIR(root, 0));
	}
	while (!q.empty())
	{
		TreeNode *node = q.front().first;
		int level = q.front().second;
		q.pop();
		if (v.size() < level + 1)
		{
			vector<int> item;
			v.push_back(item);
		}
		v[level].push_back(node->val);

		if (node->left)
		{
			q.push(PAIR(node->left, level + 1));
		}
		if (node->right)
		{
			q.push(PAIR(node->right, level + 1));
		}
	}
	return v;
}
{% endhighlight %}
这里用pair<TreeNode*, int>来封装TreeNode，和新定义一个struct差不多，不过程序看起来更优美了。


#3.Binary Tree Level Order Traversal II 
[ref](https://oj.leetcode.com/problems/binary-tree-level-order-traversal-ii/)
<a name="ch3"></a>
> Given a binary tree, return the bottom-up level order traversal of its nodes' values. (ie, from left to right, level by level from leaf to root).

For example:
Given binary tree {3,9,20,#,#,15,7},

		3  
	   / \  
	  9  20  
		/  \  
	   15   7  
	   
return its bottom-up level order traversal as:

	[
	  [15,7],  
	  [9,20],  
	  [3]  
	]
将[Binary Tree Level Order Traversal ](#ch2)返回的数组reverse一下就可以了。
{% highlight c++  %}
vector<vector<int> > levelOrderBottom(TreeNode *root) 
{
	vector<vector<int> > v;
	typedef pair<TreeNode*, int> PAIR;
	queue<PAIR> q;
	if (root)
	{
		q.push(PAIR(root, 0));
	}
	while (!q.empty())
	{
		TreeNode *node = q.front().first;
		int level = q.front().second;
		q.pop();
		if (v.size() < level + 1)
		{
			vector<int> item;
			v.push_back(item);
		}
		v[level].push_back(node->val);

		if (node->left)
		{
			q.push(PAIR(node->left, level + 1));
		}
		if (node->right)
		{
			q.push(PAIR(node->right, level + 1));
		}
	}
	//reverse v
	int n = v.size();
	for (int i = 0; i < n / 2; i++)
	{
		v[i].swap(v[n - 1 - i]);
	}
	return v;
}
{% endhighlight %}


#4.Clone Graph

[ref](https://oj.leetcode.com/problems/clone-graph/)
<a name="ch4"></a>

在DFS系列里已经解决了这个问题==[Clone Graph](http://onestraw.net/algorithm/leetcode-dfs-medium2/#ch5)
