---
layout: single
author_profile: true
comments: true
title: Word Search II
tagline: 
category: Algorithm
tags: [LeetCode, 面试]
---

 Given a 2D board and a list of words from the dictionary, find all words in the board.  

Each word must be constructed from letters of sequentially adjacent cell, where "adjacent" cells are those horizontally or vertically neighboring. The same letter cell may not be used more than once in a word.  

For example,  
Given words = ["oath","pea","eat","rain"] and board =  

    [
      ['o','a','a','n'],
      ['e','t','a','e'],
      ['i','h','k','r'],
      ['i','f','l','v']
    ]

Return ["eat","oath"].  

Note:  
You may assume that all inputs are consist of lowercase letters a-z.   

-----------

引用[Trie类实现](http://onestraw.net/algorithm/trie/)

    //word search II
    class Solution {
    public:
        void bt_search(vector<vector<char>> &board, vector<vector<bool>> &used, 
            int x, int y, string word){
            if(x<0 || y<0 || x==board.size() || y==board[0].size() || used[x][y]){
                return;
            }
            word += board[x][y];
            if(!trie.startsWith(word)){
                return;
            }
            if(trie.search(word)){
                result.push_back(word);
            }
            used[x][y] = true;
            bt_search(board, used, x-1, y, word);
            bt_search(board, used, x, y-1, word);
            bt_search(board, used, x+1, y, word);
            bt_search(board, used, x, y+1, word);
            used[x][y] = false;
        }
    
        vector<string> findWords(vector<vector<char>>& board, vector<string>& words) {
            for(auto w=words.begin(); w!=words.end(); ++w){
                trie.insert(*w);
            }
            vector<vector<bool>> used(board.size(), vector<bool>(board[0].size(), false));
            for(int i=0; i<board.size(); i++){
                for(int j=0; j<board[i].size(); j++){
                    bt_search(board, used, i, j, "");
                }
            }
            sort(result.begin(), result.end());
            auto it = unique(result.begin(), result.end());
            result.resize(distance(result.begin(), it));
            return result;
        }
    private:
        Trie trie;
        vector<string > result;
    };
