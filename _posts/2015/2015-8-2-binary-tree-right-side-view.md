---
layout: single
author_profile: true
comments: true
title: Binary Tree Right Side View
tagline: 
category: Algorithm
tags: [LeetCode, 面试]
---

LeetCode好久没有总结过了，今天来看一道二叉树`Medium`题目

      
      Given a binary tree, imagine yourself standing on the right side of it, return the values of the nodes you can see ordered from top to bottom.
      
      For example:
      Given the following binary tree,
      
         1            <---
       /   \
      2     3         <---
       \     \
        5     4       <---
      
      You should return [1, 3, 4]. 


这道题目并不难，找出`层次遍历`（BFS）每一层最后一个节点，有趣的地方在于`如何优雅准确的一次通过`。

**这里用两个队列，并且使用翻转技巧**   

    /**
     * Definition for a binary tree node.
     * struct TreeNode {
     *     int val;
     *     TreeNode *left;
     *     TreeNode *right;
     *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
     * };
     */
    class Solution {
    public:
        vector<int> rightSideView(TreeNode* root) {
            vector<int > img;
            queue<TreeNode *> q[2];
            bool qi=false;
            if(root){
                q[qi].push(root);
            }
            while(true){
                if(q[qi].size()==0){
                    break;
                }
                TreeNode *t;
                while(q[qi].size() > 0){
                    t = q[qi].front();
                    q[qi].pop();
                    if(t->left){
                        q[!qi].push(t->left);
                    }
                    if(t->right){
                        q[!qi].push(t->right);
                    }
                }
                img.push_back(t->val);
                qi=!qi;
            }
            return img;
        }
    };
    
END `geeksword@onestraw.net`
