---
layout: single
author_profile: true
comments: true
title: LeetCode DFS(上)
tagline: easy部分
category: Algorithm
tags: [LeetCode, 面试]
---

LeetCode上目前关于DFS的题目有19个，其中有18道是二叉树相关的，只有一道是图方面的。DFS系列解题报告按难度分为上中下三部分，下面是第一部分Easy.

TreeNode结构

{% highlight c++  %}
  struct TreeNode {
	  int val;
	  TreeNode *left;
	  TreeNode *right;
	  TreeNode(int x) : val(x), left(NULL), right(NULL) {}
  };
{% endhighlight %}

#1.Symmetric Tree
[ref](https://oj.leetcode.com/problems/symmetric-tree/)  

<a name='ch1'></a>

> Given a binary tree, check whether it is a mirror of itself (ie, symmetric around its center).  
  For example, this binary tree is symmetric:
  
	      1  
	     / \  
	    2   2 
	   / \ / \  
	  3  4 4  3  
  
  But the following is not:
 
      1  
     / \  
    2   2  
     \   \  
     3    3  
     
###Recursive Solution

{% highlight c++  %}
	bool mirror(TreeNode *l, TreeNode *r)
	{
		if (!l || !r)
		{
			return l == r;
		}
		return l->val == r->val && mirror(l->left, r->right) && mirror(l->right, r->left);
	}
	bool isSymmetric(TreeNode *root)
	{
		if (!root)
		{
			return true;
		}
		return mirror(root->left, root->right);
	}
{% endhighlight %}

###Iterative Solution

{% highlight c++  %}
bool isSymmetric(TreeNode *root)
{
	if (!root)
	{
		return true;
	}
	TreeNode *lc;
	TreeNode *rc;
	stack<TreeNode*> ls; // store root's left subtree
	stack<TreeNode*> rs; // store right's right subtree
	ls.push(root->left);
	rs.push(root->right);

	while (!ls.empty() && !rs.empty())
	{
		lc = ls.top();
		ls.pop();
		rc = rs.top();
		rs.pop();

		if (((!lc || !rc) && lc != rc) || (lc && rc && lc->val != rc->val))
		{
			return false;
		}
		if (lc)
		{//first push left
			ls.push(lc->left);
			ls.push(lc->right);
		}
		if (rc)
		{//first push right
			rs.push(rc->right);
			rs.push(rc->left);
		}
	}
	if (!ls.empty() || !rs.empty())
	{
		return false;
	}
	return true;
}
{% endhighlight %}

#2.Same Tree
[ref](https://oj.leetcode.com/problems/same-tree/)  

> Given two binary trees, write a function to check if they are equal or not.  
  Two binary trees are considered equal if they are structurally identical and the nodes have the same value.
  
###Recursive Solution

关于二叉树的一些题目，我的一个心得就是递归，简单明了，迭代算法相对繁琐，如[Symmetric Tree](#ch1)，迭代算法用了40行代码，而递归仅仅用了15行代码。这个题目用递归核心就3行代码，不再给出迭代解法。   

{% highlight c++ %}
	bool isSameTree(TreeNode *p, TreeNode *q) 
	{
		if (!p || !q)
		{
			return p == q;
		}
		return p->val == q->val && isSameTree(p->left, q->left) && isSameTree(p->right, q->right);
	}
{% endhighlight %}


#3.Balanced Binary Tree
[ref](https://oj.leetcode.com/problems/balanced-binary-tree/)  

> Given a binary tree, determine if it is height-balanced.  
  For this problem, a height-balanced binary tree is defined as a binary tree in which the depth of the two subtrees of every node never differ by more than 1.
  
下面这个解法还可以优化，通过封装TreeNode结点，添加一个表示结点高度的变量，可以避免子问题重复求解；或者用一个map<TreeNode*, int>来保存每个节点的高度。    

{% highlight c++  %}
  int height(TreeNode *root)
	{
		if (!root)
		{
			return 0;
		}
		return max(height(root->left), height(root->right)) + 1;
	}
	bool isBalanced(TreeNode *root) 
	{
		if (!root)
		{
			return true;
		}
		if (abs(height(root->left) - height(root->right)) > 1)
		{
			return false;
		}
		return isBalanced(root->left) && isBalanced(root->right);
	}
{% endhighlight %}

#4.Minimum Depth of Binary Tree
[ref](https://oj.leetcode.com/problems/minimum-depth-of-binary-tree/)  

> Given a binary tree, find its minimum depth.  
  The minimum depth is the number of nodes along the shortest path from the root node down to the nearest leaf node. 
  
{% highlight c++  %}
  int minDepth(TreeNode *root) 
	{
		if (!root)
		{
			return 0;
		}
		// root must not NULL
		if (!root->left && !root->right)
		{
			return 1;
		}
		int left_min = root->left ? minDepth(root->left) : INT_MAX;
		int right_min = root->right ? minDepth(root->right) : INT_MAX;

		return min(left_min, right_min) + 1;
	}
{% endhighlight %}

#5.Maximum Depth of Binary Tree
[ref](https://oj.leetcode.com/problems/maximum-depth-of-binary-tree/)  

> Given a binary tree, find its maximum depth.  
  The maximum depth is the number of nodes along the longest path from the root node down to the farthest leaf node.
  
{% highlight c++  %}
  int maxDepth(TreeNode *root) 
	{
		if (!root)
		{
			return 0;
		}
		return max(maxDepth(root->left), maxDepth(root->right)) + 1;
	}
{% endhighlight %}

#6.Path Sum
[ref](https://oj.leetcode.com/problems/path-sum/)  

> Given a binary tree and a sum, determine if the tree has a root-to-leaf path such that adding up all the values along the path equals the given sum.
  For example:  
  Given the below binary tree and sum = 22,  
  
	              5
	             / \
	            4   8
	           /   / \
	          11  13  4
	         /  \      \
	        7    2      1

	return true, as there exist a root-to-leaf path 5->4->11->2 which sum is 22.  
	
{% highlight c++  %}
  bool hasPathSum(TreeNode *root, int sum)
	{
		if (!root)
		{
			return false;
		}
		if (root && !root->left && !root->right && root->val == sum)
		{
			return true;
		}
		return hasPathSum(root->left, sum - root->val) || hasPathSum(root->right, sum - root->val);
	}
{% endhighlight %}
