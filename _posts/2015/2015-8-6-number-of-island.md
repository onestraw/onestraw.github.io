---
layout: single
author_profile: true
comments: true
title: Number of Islands
tagline: 
category: Algorithm
tags: [LeetCode]
---

Given a 2d grid map of '1's (land) and '0's (water), count the number of islands. An island is surrounded by water and is formed by connecting adjacent lands horizontally or vertically. You may assume all four edges of the grid are all surrounded by water.  

Example 1:  

    11110
    11010
    11000
    00000

Answer: 1  
   
Example 2:   

    11000
    11000
    00100
    00011

Answer: 3  

--------

    class Solution {
    public:
        void dfs(vector<vector<char>>& grid, vector<vector<bool>>& used, int x, int y){
            if(x<0||x==grid.size()||y<0||y==grid[0].size()||grid[x][y]=='0'||used[x][y]){
                return;
            }
            used[x][y] = true;
            dfs(grid, used, x-1,y);
            dfs(grid, used, x+1,y);
            dfs(grid, used, x,y-1);
            dfs(grid, used, x,y+1);
        }
        int numIslands(vector<vector<char>>& grid) {
            if(grid.size()==0 || grid[0].size()==0){
                return 0;
            }
            vector<vector<bool > > used(grid.size(), vector<bool>(grid[0].size(),false));
            size_t n=0;
            for(size_t i=0; i<grid.size(); i++){
                for(size_t j=0; j<grid[i].size(); j++){
                    if(grid[i][j]=='1' && !used[i][j]){
                       n++;
                       dfs(grid, used, i, j);
                    }
                }
            }
            return n;
        }
    };
