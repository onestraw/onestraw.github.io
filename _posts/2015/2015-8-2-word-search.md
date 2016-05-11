---
layout: single
author_profile: true
comments: true
title: Word Search
tagline: 
category: Algorithm
tags: [LeetCode, 面试]
---

 Given a 2D board and a word, find if the word exists in the grid.    

The word can be constructed from letters of sequentially adjacent cell, where "adjacent" cells are those horizontally or vertically neighboring. The same letter cell may not be used more than once.  

For example,  
Given board =  

    [
      ["ABCE"],
      ["SFCS"],
      ["ADEE"]
    ]

word = "ABCCED", -> returns true,   
word = "SEE", -> returns true,  
word = "ABCB", -> returns false.   

    class Solution {
    public:
        bool bt_exist(vector<vector<char>> &board, vector<vector<bool>> &used, int x, int y, string word, int wi){
            if(wi==word.size()){
                return true;
            }
            if(x<0 || y<0 || x==board.size() || y==board[0].size() || used[x][y] ||board[x][y]!=word[wi]){
                return false;
            }
            used[x][y] = true;
            bool result = 
                bt_exist(board, used, x+1, y, word, wi+1)   ||
                bt_exist(board, used, x, y+1, word, wi+1)   ||
                bt_exist(board, used, x-1, y, word, wi+1)   ||
                bt_exist(board, used, x, y1, word, wi+1);
            used[x][y] = false;
    
            return result;
        }
        bool exist(vector<vector<char>>& board, string word) {
            vector<vector<bool>> used(board.size(), vector<bool>(board[0].size(), false));
            bool result = false;
            for(int i=0; i<board.size(); i++){
                for(int j=0; j<board[i].size(); j++){
                    result = result || bt_exist(board, used, i, j, word, 0);
                }
            }
            return result;
        }
    };
