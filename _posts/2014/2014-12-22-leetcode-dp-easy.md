---
layout: single
author_profile: true
comments: true
title: LeetCode DP(上)
tagline: easy部分
category: Algorithm
tags: [LeetCode, 面试]
---
LeetCode上动态规则类题目有23个，其中

* Easy: 1
* Medium: 11
* Hard: 11

动态规划通常适用于有重叠子问题和最优子结构性质的问题，关于这两个性质有两篇文章写的不错：

1. <a href="http://www.geeksforgeeks.org/dynamic-programming-set-1/" target="_blank">重叠子问题</a>
2. <a href="http://www.geeksforgeeks.org/dynamic-programming-set-1/" target="_blank">最优子结构</a>

本篇是LeetCode DP类别的简单(Easy + Medium)题目：  

1. [Climbing Stairs](#ch1)
2. [Triangle](#ch2)
3. [Best Time to Buy and Sell Stock](#ch3) 
4. [Minimum Path Sum](#ch4)
5. [Maximum Subarray](#ch5)
6. [Maximum Product Subarray](#ch6)
7. [Word Break](#ch7)
8. [Decode Ways](#ch8)

#1.Climbing Stairs
[re](https://oj.leetcode.com/problems/climbing-stairs/)
<a name="ch1"> </a>
> You are climbing a stair case. It takes n steps to reach to the top.  
  Each time you can either climb 1 or 2 steps. In how many distinct ways can you climb to the top?

组合数学中有很多这种问题，用f(n)表示前n步不同的走法，最后一步可能是一步走过来的，也可能是一次跨越两步到的，所以

> f(n) = f(n-2) + f(n-1)

{% highlight c++ %}
int climbStairs(int n)
{
	if (n < 3){
		return n;
	}
	vector<int> v = { 1, 2 };
	for (int i = 2; i < n; i++)
	{
		v.push_back(v[i - 2] + v[i - 1]);
	}
	return v[n - 1];
}
{% endhighlight %}
优化一下  
{% highlight c++ %}
int climbStairs(int n)
{
	if (n < 3){
		return n;
	}
	int a = 1;
	int b = 2;
	for (int i = 2; i < n; i++){
		int c = b;
		b = a + b;
		a = c;
	}
	return b;
}
{% endhighlight %}


#2.Triangle
[re](https://oj.leetcode.com/problems/triangle/)
<a name="ch2"> </a>
> Given a triangle, find the minimum path sum from top to bottom. Each step you may move to adjacent numbers on the row below.

For example, given the following triangle 

  [  
       [2],  
      [3,4],  
     [6,5,7],  
    [4,1,8,3]  
  ]  

The minimum path sum from top to bottom is 11 (i.e., 2 + 3 + 5 + 1 = 11).

Note:  

> Bonus point if you are able to do this using only O(n) extra space, where n is the total number of rows in the triangle.

这个题目可以自顶向下，也可以自底向上来解 
###O(n^2)解法
用一个二维数组path[i][j]存储(0,0)到(i,j)的最小路径和，递推关系如下：

> path[i][j] = min(path[i - 1][j - 1], path[i - 1][j]) + triangle[i][j];

{% highlight c++ %}
int minimumTotal(vector<vector<int> > &triangle)
{
	int row = triangle.size();
	if (row < 1)
	{
		return 0;
	}
	vector < vector<int> > path(row, vector<int>(row, 0));
	path[0][0] = triangle[0][0];
	for (int i = 1; i < row; i++)
	{
		path[i][0] = path[i - 1][0] + triangle[i][0];
		for (int j = 1; j < i; j++)
		{
			path[i][j] = min(path[i - 1][j - 1], path[i - 1][j]) + triangle[i][j];
		}
		path[i][i] = path[i - 1][i - 1] + triangle[i][i];
	}
	for (int i = 1; i < row; i++)
	{
		if (path[row - 1][0] > path[row - 1][i])
		{
			path[row - 1][0] = path[row - 1][i];
		}
	}
	return path[row - 1][0];
}
{% endhighlight %}

###O(n)解法
借鉴[Word ladder II](http://onestraw.net/#ch3)的思想，用两个一维数组交换保存当前行和上一行的最小路径和,从n*(n+1)/2的空间优化到了2*n。
{% highlight c++ %}
int minimumTotal(vector<vector<int> > &triangle)
{
	int row = triangle.size();
	if (row < 1)
	{
		return 0;
	}
	vector < vector<int> > path(2, vector<int>(row, 0));
	int pre = 1;
	int cur = 0;
	path[0][0] = triangle[0][0];
	for (int i = 1; i < row; i++)
	{
		pre = !pre;
		cur = !cur;
		path[cur][0] = path[pre][0] + triangle[i][0];
		for (int j = 1; j < i; j++)
		{
			path[cur][j] = min(path[pre][j - 1], path[pre][j]) + triangle[i][j];
		}
		path[cur][i] = path[pre][i - 1] + triangle[i][i];
	}
	for (int i = 1; i < row; i++)
	{
		if (path[cur][0] > path[cur][i])
		{
			path[cur][0] = path[cur][i];
		}
	}
	return path[cur][0];
}
{% endhighlight %}

#3.Best Time to Buy and Sell Stock
[re](https://oj.leetcode.com/problems/best-time-to-buy-and-sell-stock/)
<a name="ch3"> </a>
> Say you have an array for which the ith element is the price of a given stock on day i.  
  If you were only permitted to complete at most one transaction (ie, buy one and sell one share of the stock), design an algorithm to find the maximum profit.

prices[i]表示股票在第i天的价格，你只能买一支股票，卖一支股票，问怎么才能获得最大收益？  

* minPrice[i]表示第0天到第i的最低价格，maxProfit[i]表示前i天能获得的最大收益；
* minPrice[i] = min(minPrice[i-1], prices[i]);
* 那么prices[i] - minPrice[i]则为在第i天卖出的最大收益；
* maxProfit[i] = max(prices[i] - minPrice[i], maxProfit[i-1]);
下面是优化后的算法，仅使用O(1)的存储空间 
{% highlight c++ %}
int maxProfit(vector<int> &prices)
{
	if (prices.size() < 2)
	{
		return 0;
	}
	int maxProfit = 0;
	int minPrice = prices[0];
	for (int i = 1; i < prices.size(); i++)
	{
		minPrice = min(minPrice, prices[i]);
		maxProfit = max(maxProfit, prices[i] - minPrice);
	}
	return maxProfit;
}
{% endhighlight %}

#4.Minimum Path Sum
[re](https://oj.leetcode.com/problems/minimum-path-sum/)
<a name="ch4"> </a>
> Given a m x n grid filled with non-negative numbers, find a path from top left to bottom right which minimizes the sum of all numbers along its path.

Note: You can only move either down or right at any point in time.

* minSum[i][j]表示(0,0)到(i,j)的最小路径和
* (0,0)到(i,j)的最小路径是从左边(i,j-1)过来到(i,j)，还是从上面(i-1, j)过来，取决于minSum。递推关系如下

> minSum[i][j] = min(minSum[i - 1][j], minSum[i][j - 1]) + grid[i][j];

{% highlight c++ %}
int minPathSum(vector<vector<int> > &grid)
{
	if (grid.size() < 1)
	{
		return 0;
	}
	int row = grid.size();
	int col = grid[0].size();
	vector<vector<int> > minSum(row, vector<int>(col));

	minSum[0][0] = grid[0][0];

	for (int i = 1; i < col; i++)
	{
		minSum[0][i] = minSum[0][i - 1] + grid[0][i];
	}

	for (int i = 1; i < row; i++)
	{
		minSum[i][0] = minSum[i - 1][0] + grid[i][0];
		for (int j = 1; j < col; j++)
		{
			minSum[i][j] = min(minSum[i - 1][j], minSum[i][j - 1]) + grid[i][j];
		}
	}
	return minSum[row - 1][col - 1];
} 
{% endhighlight %}


#5.Maximum Subarray
[re](https://oj.leetcode.com/problems/maximum-subarray/)
<a name="ch5"> </a>
> Find the contiguous subarray within an array (containing at least one number) which has the largest sum.   
  For example, given the array [−2,1,−3,4,−1,2,1,−5,4],  
  the contiguous subarray [4,−1,2,1] has the largest sum = 6. 

* maxSum[i] 表示0...i的最大连续子数组和(包括a[i]);
* 如果maxSum[i]>=0, 
  * 则maxSum[i+1] = maxSum[i] + a[i+1]
  * 否则maxSum[i+1] = a[i+1]
  
{% highlight c++ %}
int maxSubArray(int A[], int n)
{
	vector<int> maxSum(n);
	maxSum[0] = A[0];
	for (int i = 1; i < n; i++)
	{
		if (maxSum[i - 1] < 0)
		{
			maxSum[i] = A[i];
		}
		else
		{
			maxSum[i] = maxSum[i - 1] + A[i];
		}
		if (maxSum[i] > maxSum[0])
		{
			maxSum[0] = maxSum[i];
		}
	}
	return maxSum[0];
}
{% endhighlight %}


#6.Maximum Product Subarray
[ref](https://oj.leetcode.com/problems/maximum-product-subarray/)
<a name="ch6"> </a>
> Find the contiguous subarray within an array (containing at least one number) which has the largest product.

> For example, given the array [2,3,-2,4],  
  the contiguous subarray [2,3] has the largest product = 6.


这个题目和[Maximum Subarray](#ch5)很类似，该题目除了记录最大整数(乘积)之外，还要记录最小负数乘积。

* positive[i]表示0...i最大连续正数乘积(包括i) 
* negative[i]表示0...i最大负数连续乘积(包括i)

{% highlight c++ %}
int maxProduct(int A[], int n)
{
	vector<int> positive(n, 0);
	vector<int> negative(n, 0);
	positive[0] = negative[0] = A[0];

	for (int i = 1; i < n; i++)
	{
		if (A[i] >= 0)
		{
			positive[i] = max(A[i], A[i] * positive[i - 1]);
			if (negative[i - 1] < 0)
			{
				negative[i] = A[i] * negative[i - 1];
			}
		}
		else
		{
			negative[i] = min(A[i], A[i] * positive[i - 1]);
			if (negative[i - 1] < 0)
			{
				positive[i] = A[i] * negative[i - 1];
			}
		}
	}
	int maxProduct = INT_MIN;
	for (int i = 0; i < n; i++)
	{
		if (maxProduct < positive[i])
			maxProduct = positive[i];
	}
	return maxProduct;
}
{% endhighlight %}


#7.Word Break
[ref](https://oj.leetcode.com/problems/word-break/)
<a name="ch7"> </a>
> Given a string s and a dictionary of words dict, determine if s can be segmented into a space-separated sequence of one or more dictionary words.

> For example, given  
  s = "leetcode",  
  dict = ["leet", "code"].  

Return true because "leetcode" can be segmented as "leet code".

* vbreak[i]表示s[i]...s[end]是否可以break
* 如果s[i...end]在dict中，或者存在k使得s[i...k-1] in dict && vbreak[i+k] == true，那么vbreak[i]=true;

{% highlight c++ %}
bool wordBreak(string s, unordered_set<string> &dict)
{
	int len = s.size();
	vector<bool>  vbreak = vector<bool>(len, false);
	
	for (int i = len - 1; i >= 0; i--)
	{
		if (dict.find(s.substr(i)) != dict.end())
		{
			vbreak[i] = true;
		}
		else
		{
			for (int j = 1; j < len - i; j++)
			{
				if (vbreak[i + j] && dict.find(s.substr(i, j)) != dict.end())
				{
					vbreak[i] = true;
					break;
				}
			}
		}
	}
	return vbreak[0];
}
{% endhighlight %}

#8.Decode Ways
[ref](https://oj.leetcode.com/problems/decode-ways/)
<a name="ch8"> </a>
> A message containing letters from A-Z is being encoded to numbers using the following mapping:

  'A' -> 1
  'B' -> 2
  ...
  'Z' -> 26

Given an encoded message containing digits, determine the total number of ways to decode it. 

For example,  
Given encoded message "12", it could be decoded as "AB" (1 2) or "L" (12).  

The number of ways decoding "12" is 2.   
####思路
对于字符s[i]，有三种解码方式

	 1. 单独解码，除了'0'字符
	 2. 和前一个字符s[i-1]组合在一块解码，要求s[i-1]=='1' or '2'
	 3. 和后一个字符s[i+1]组合在一块解码，要求s[i]=='1' or '2'

用way1[],way2[],way3[]记录三种方式，则

	 way1[i] = way1[i-1]+way2[i-1]
	 way2[i] = way3[i-1]
	 way3[i] = way1[i-1]+way2[i-1]

最后
	 wayNum = way1[n] + way2[n]

使用迭代进行优化。
	 
{% highlight c++ %}
int numDecodings(string s)
{
	int n = s.size();
	if (n > 0 && s[0] == '0')
	{
		return 0;
	}
	if (n < 2)
	{
		return n;
	}
	int way1 = 1;
	int way2 = 0;
	int way3 = s[0] <= '2' ? 1 : 0;
	for (int i = 1; i < n; i++)
	{
		int temp = way1 + way2;
		way1 = s[i] != '0' ? temp : 0;
		way2 = (s[i - 1] == '1' || (s[i - 1] == '2' && s[i] <= '6')) ? way3 : 0;
		way3 = temp;

	}
	return way1 + way2;
}
{% endhighlight %}
