---
layout: single
author_profile: true
comments: true
title: LeetCode DFS(中)
tagline: medium部分上
category: Algorithm
tags: [LeetCode, 面试]
---
DFS Medium级别题目有10道，本篇解题如下：  

* [Path Sum II](#ch1)
* [Sum Root to Leaf Numbers](#ch2)
* [Flatten Binary Tree to Linked List](#ch3) 
* [Validate Binary Search Tree](#ch4)
* [Populating Next Right Pointers in Each Node](#ch5)

#1.Path Sum II
[ref](https://oj.leetcode.com/problems/path-sum-ii/)    
[PathSum题解](http://onestraw.net/algorithm/leetcode-dfs-easy/)  

<a name="ch1"></a>

> Given a binary tree and a sum, find all root-to-leaf paths where each path's sum equals the given sum.  
  For example:  
  Given the below binary tree and sum = 22,  

                5
               / \
              4   8
             /   / \
            11  13  4
           /  \    / \
          7    2  5   1
    return  
    [  
     [5,4,11,2],  
     [5,8,4,5]  
    ] 
  
{% highlight C++ %}
  bool hasPathSum(TreeNode *root, int sum, vector<vector<int> > &v, vector<int> arr)
	{
		if (!root)
		{
			return false;
		}
		arr.push_back(root->val);
		if (!root->left && !root->right && root->val == sum)
		{
			v.push_back(arr);
			return true;
		}
		bool left_sum = hasPathSum(root->left, sum - root->val, v, arr);
		bool right_sum = hasPathSum(root->right, sum - root->val, v, arr);
		return  left_sum || right_sum;
	}

	vector<vector<int> > pathSum(TreeNode *root, int sum) 
	{
		vector<vector<int> > v;
		hasPathSum(root, sum, v, vector<int>());
		return v;
	}
{% endhighlight %}

#2.Sum Root to Leaf Numbers
[ref](https://oj.leetcode.com/problems/sum-root-to-leaf-numbers/)  

<a name="ch2"></a>  

> Given a binary tree containing digits from 0-9 only, each root-to-leaf path could represent a number.  
  An example is the root-to-leaf path 1->2->3 which represents the number 123.  
  Find the total sum of all root-to-leaf numbers.  
  
  For example,  

      1
     / \
    2   3
  The root-to-leaf path 1->2 represents the number 12.  
  The root-to-leaf path 1->3 represents the number 13.  
  Return the sum = 12 + 13 = 25.  
  
{% highlight C++ %}
  void findLeaf(TreeNode *root, int &sum, int value)
	{
		if (root)
		{
			int curValue = value * 10 + root->val;
			if (root->left || root->right)
			{
				findLeaf(root->left, sum, curValue);
				findLeaf(root->right, sum, curValue);
			}
			else
			{//leaf
				sum += curValue;
			}
		}
	}
	int sumNumbers(TreeNode *root) 
	{
		int sum = 0;
		findLeaf(root, sum, 0);
		return sum;
	}
{% endhighlight%}

#3.Flatten Binary Tree to Linked List 
[ref](https://oj.leetcode.com/problems/flatten-binary-tree-to-linked-list/)  

<a name="ch3"></a>

> Given a binary tree, flatten it to a linked list in-place.  
  For example,  
  Given
  
           1
          / \
         2   5
        / \   \
       3   4   6
     
  The flattened tree should look like:
  
     1
      \
       2
        \
         3
          \
           4
            \
             5
              \
               6
               
{% highlight c++  %}
 	TreeNode *rightmost(TreeNode*root)
	{
		TreeNode *p = root;
		while (p && p->right)
		{
			p = p->right;
		}
		return p;
	}

	void flatten(TreeNode *root) 
	{
		if (root)
		{
			flatten(root->left);
			flatten(root->right);
			if (root->left)
			{
				TreeNode *p = rightmost(root->left);
				p->right = root->right;
				root->right = root->left;
				root->left = NULL;
			}
		}
	}
{% endhighlight %}

#4.Validate Binary Search Tree
[ref](https://oj.leetcode.com/problems/validate-binary-search-tree/)  
<a name="ch4"></a>

> Given a binary tree, determine if it is a valid binary search tree (BST).  
  Assume a BST is defined as follows:  
    
  ·The left subtree of a node contains only nodes with keys less than the node's key.  
  ·The right subtree of a node contains only nodes with keys greater than the node's key.  
  ·Both the left and right subtrees must also be binary search trees.  

{% highlight C++ %}
  TreeNode *leftmost(TreeNode*root)
	{
		TreeNode *p;
		for (p = root; p->left; p = p->left);
		return p;
	}
	TreeNode *rightmost(TreeNode*root)
	{
		TreeNode *p;
		for (p = root; p->right; p = p->right);
		return p;
	}
	bool isValidBST(TreeNode *root) 
	{
		if (!root)
		{
			return true;
		}
		bool lflag = true;
		bool rflag = true;
		if (root->left)
		{
			lflag = root->val > rightmost(root->left)->val && isValidBST(root->left);
		}
		if (root->right)
		{
			rflag = root->val < leftmost(root->right)->val && isValidBST(root->right);
		}
		return rflag && lflag;
	}
{% endhighlight %}

#5.Populating Next Right Pointers in Each Node
[ref](https://oj.leetcode.com/problems/populating-next-right-pointers-in-each-node/)  
<a name="ch5"></a>  

> Given a binary tree  
    struct TreeLinkNode {  
      TreeLinkNode *left;  
      TreeLinkNode *right;  
      TreeLinkNode *next;  
    }  
  Populate each next pointer to point to its next right node. If there is no next right node, the next pointer should be set to NULL.  
  Initially, all next pointers are set to NULL.  
  
  Note:  
  
  You may only use constant extra space.  
  You may assume that it is a perfect binary tree (ie, all leaves are at the same level, and every parent has two children).  
  For example,  
  Given the following perfect binary tree,  
 
           1
         /  \
        2    3
       / \  / \
      4  5  6  7
  
  After calling your function, the tree should look like:
  
           1 -> NULL
         /  \
        2 -> 3 -> NULL
       / \  / \
      4->5->6->7 -> NULL

###1.思路一
层次遍历二叉树，入队列时，对每一个节点标记上层级；出队列时，按层级保存到一个二维数组中；最后按层级连接前后结点next指针。
 
{% highlight c++  %}
 struct TLN{
	TreeLinkNode *node;
	int level;
	TLN(TreeLinkNode *tnode, int l) :node(tnode), level(l){}
};

class Solution {
public:
	void connect(TreeLinkNode *root) 
	{
		if (!root)
		{
			return;
		}
		TLN *p;
		queue<TLN *> q;
		vector<vector<TreeLinkNode*>> v;
		q.push(new TLN(root,0));
		while (!q.empty())
		{
			p = q.front();
			q.pop();

			if (v.size() < p->level + 1)
			{
				vector<TreeLinkNode*> item;
				v.push_back(item);
			}
			v[p->level].push_back(p->node);

			if (p->node->left)
			{
				q.push(new TLN(p->node->left, p->level + 1));
			}
			if (p->node->right)
			{
				q.push(new TLN(p->node->right, p->level + 1));
			}
		}
		//connect
		for (int i = 0; i < v.size(); i++)
		{
			for (int j = 0; j < v[i].size() - 1; j++)
			{
				v[i][j]->next = v[i][j + 1];
			}
		}
	}
};
{% endhighlight %}

###2.思路二
还是层次遍历的思想，用两个数组模拟队列，队列q中的数据始终是同一层级的结点，qt作为临时缓冲队列。  

{% highlight c++  %}
 	void connect(TreeLinkNode *root)
	{
		vector<TreeLinkNode*> q;
		vector<TreeLinkNode*> qt;
		q.push_back(root);
		while (!q.empty())
		{
			qt.clear();
			for (int i = 0; i < q.size();i++)
			{
				if (q[i])
				{
					if (i>0)
					{
						q[i - 1]->next = q[i];
					}
					qt.push_back(q[i]->left);
					qt.push_back(q[i]->right);
				}
			}
			q = qt;
		}
	}
{% endhighlight %}

###3.思路三
该题是DFS系列的题目哎，而思路一和二明明都是BFS的解法啊。  
注意Note中提示，假定这是一棵perfect binary tree，所有的叶子节点在同一高度，每一个父结点都有两个子结点。题目提示用O(1)的空间。
如果不占用额外空间的话，就需要记住每个结点的右兄弟结点，具体算法如下：  
{% highlight c++  %}
 void link(TreeLinkNode *lnode, TreeLinkNode* rnode)
	{
		if (lnode)
		{
			lnode->next = rnode;
			link(lnode->left, lnode->right);
			if (rnode)
			{
				link(lnode->right, rnode->left);
			}
			else
			{
				link(lnode->right, NULL);
			}
		}
	}
	void connect(TreeLinkNode *root)
	{
		link(root, NULL);
	}
{% endhighlight %}

对于[二叉树](#ch5)，它对应的递归树为：  
<img src="/assets/images/Populating-Next-Right-Pointers-in-Each-Node.png" alt="" />
