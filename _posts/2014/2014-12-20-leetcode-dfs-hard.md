---
layout: single
author_profile: true
comments: true
title: LeetCode DFS(下)
tagline: hard部分
category: Algorithm
tags: [LeetCode, 面试]
---
下面是DFS Hard级别题目，只有3个：  

* [Populating Next Right Pointers in Each Node II](#ch1)
* [Recover Binary Search Tree](#ch2)
* [Binary Tree Maximum Path Sum](#ch3) 

#1.Populating Next Right Pointers in Each Node II
[ref](https://oj.leetcode.com/problems/populating-next-right-pointers-in-each-node-ii/)
<a name="ch1"></a>


> Follow up for problem "Populating Next Right Pointers in Each Node".
  What if the given tree could be any binary tree? Would your previous solution still work?

> Note:
  You may only use constant extra space.
  For example,
  Given the following binary tree,
  
         1
       /  \
      2    3
     / \    \
    4   5    7

> After calling your function, the tree should look like:

         1 -> NULL
       /  \
      2 -> 3 -> NULL
     / \    \
    4-> 5 -> 7 -> NULL
  
这个题目[Populating Next Right Pointers in Each Node](http://onestraw.net/algorithm/leetcode-dfs-medium/#ch5)的思路一和思路二都适合本题，
解法一的代码可以直接用，解法二不能再由一个结点存在推断出同一层的结点都存在，需要修改。此外它们都消耗O(n)的额外空间，都不是DFS，我们考虑在思路三上改进。 
[Populating Next Right Pointers in Each Node思路三](http://onestraw.net/algorithm/leetcode-dfs-medium/#ch5)代码如下  
{% highlight c++ linenos  %}
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

二叉树可能是

         1
       /  \
      2    3
     /    / \
    4    6   7

第一个link递归需要改进：link(lnode->left, lnode->right);

二叉树可能是

         1
       /  \
      2    3
     / \    \
    4   5    7

第二个link递归需要改进：link(lnode->right, rnode->left);

如果二叉树像下面这样

           1
         /  \
        2    3
       / \    \
      4   5    7
     /          \
    8            9
  
如何连接8和9呢？节点8仅靠父结点4和叔父节点5是根本找不到9的！
错，别忘记了next指针，如果先DFS右子树，再DFS左子树，这样到最后一层时，4->5->7已经连接好了，那么就可以找到8的右兄弟节点9.

{% highlight c++  %}
void connect(TreeLinkNode *root)
{
	if (!root)
	{
		return;
	}
	//rightmost
	if (!root->next)
	{
		if (root->right)
		{
			root->right->next = NULL;
		}
		if (root->left)
		{
			root->left->next = root->right;
		}
	}
	else
	{
		TreeLinkNode *p = root;
		TreeLinkNode *right_brother = NULL;
		for (p = root->next; p; p = p->next)
		{
			if (p->left)
			{
				right_brother = p->left;
				break;
			}
			if (p->right)
			{
				right_brother = p->right;
				break;
			}
		}
		if (root->right)
		{
			root->right->next = right_brother;
			if (root->left)
			{
				root->left->next = root->right;
			}
		}
		else if (root->left)
		{
			root->left->next = right_brother;
		}
	}
	connect(root->right);
	connect(root->left);
}
{% endhighlight %}

#2.Recover Binary Search Tree
[ref](https://oj.leetcode.com/problems/recover-binary-search-tree/)
<a name="ch2"></a>

> Two elements of a binary search tree (BST) are swapped by mistake.  
  Recover the tree without changing its structure.  
  
> Note:
  A solution using O(n) space is pretty straight forward. Could you devise a constant space solution?

###O(n)解法
{% highlight c++  %}
void inorder(TreeNode *root, vector<TreeNode*> &vt)
{
	if (root)
	{
		inorder(root->left, vt);
		vt.push_back(root);
		inorder(root->right, vt);
	}
}
void recoverTree(TreeNode *root)
{
	if (!root)
	{
		return;
	}
	vector<TreeNode*> vt;
	inorder(root, vt);
	int i;
	int first = -1;
	int second = -1;
	for (i = 0; i < vt.size() - 1; i++)
	{
		if (vt[i]->val > vt[i + 1]->val)
		{
			if (first < 0)
			{
				first = i;
			}
			else
			{
				second = i + 1;
			}
		}
	}
	if (second < 0)
	{
		second = first + 1;
	}
	swap(vt[first]->val, vt[second]->val);
}
{% endhighlight %}

###O(1)解法
思路还是InOrder遍历，用两个指针记录破坏顺序了两个节点。

{% highlight c++  %}
TreeNode *first;
TreeNode *second;
TreeNode *pre;
void inorder(TreeNode *root)
{
	if (!root)
	{
		return;
	}
	inorder(root->left);
	if (pre && root->val < pre->val)
	{
		if (!first)
		{
			first = pre;
		}
		second = root;
	}
	pre = root;
	inorder(root->right);
}
void recoverTree(TreeNode *root)
{
	if (!root)
	{
		return;
	}
	first = second = pre = NULL;
	inorder(root);
	swap(first->val, second->val);
} 
{% endhighlight %}

[reference](http://yucoding.blogspot.com/2013/03/leetcode-question-75-recover-binary.html)  

#3.Binary Tree Maximum Path Sum
[ref](https://oj.leetcode.com/problems/binary-tree-maximum-path-sum/)
<a name="ch3"></a>

> Given a binary tree, find the maximum path sum.  
  The path may start and end at any node in the tree.  

> For example:
  Given the below binary tree,  

       1
      / \
     2   3
     
    Return 6.

求二叉树的最大路径和，不是根节点到某一个节点的路径，是树中任何两个节点间的最大路径和。   
这个最大路径可能不经过根，可能不包括叶子节点。    

* 先求出每个节点node到其子孙节点的最大路径和：rootMaxSum();  
* 然后求出以node为根的子树，任何两个节点的最大路径和：solve();   

{% highlight c++  %}
struct PathMax
{
	int lmax;
	int rmax;
};

class Solution {
	map<TreeNode*, PathMax> maxMap;
public:
	int rootMaxSum(TreeNode *root)
	{
		if (!root)
		{
			return 0;
		}
		PathMax pm;
		pm.lmax = rootMaxSum(root->left) + root->val;
		pm.rmax = rootMaxSum(root->right) + root->val;
		maxMap[root] = pm;
		return max(max(pm.lmax, pm.rmax), 0);
	}

	int solve(TreeNode *root)
	{
		if (!root)
		{
			return INT_MIN;
		}
		return max(maxMap[root].lmax + maxMap[root].rmax - root->val, max(solve(root->left), solve(root->right)));
	}

	int maxPathSum(TreeNode *root)
	{
		rootMaxSum(root);
		return solve(root);
	}
};
{% endhighlight %}

DFS问题全部完成了，全部列表如下：

* [LeetCode DFS(上)](http://onestraw.net/algorithm/leetcode-dfs-easy)
* [LeetCode DFS(中)](http://onestraw.net/algorithm/leetcode-dfs-medium)
* [LeetCode DFS(中下)](http://onestraw.net/algorithm/leetcode-dfs-medium2)
* [LeetCode DFS(下)](#ch1) 

最后推荐几个大牛的leetcode代码，自己做完可以对照一下，多学几种思路。

* [酷壳-陈皓](https://github.com/haoel/leetcode)
* [soulmachine](https://github.com/soulmachine/leetcode)
* [Yu's Coding](http://yucoding.blogspot.com/2014/04/leetcode-online-judge-questions-table.html)
