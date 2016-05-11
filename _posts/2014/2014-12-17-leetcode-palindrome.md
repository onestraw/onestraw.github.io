---
layout: single
author_profile: true
comments: true
title: LeetCode回文数题目集锦
tagline: 
category: Algorithm
tags: [LeetCode, 面试]
---
做LeetCode题目有3周了，完成了113/166，感觉LeetCode最大的特点就是题目简洁，不像Codeforces, CodeChef等，在进入正题之前先说个小故事，对我这样英语不好的人，理解题意就要半天，其次LeetCode比较规范，定义好了函数接口。以前总是搞C, Python, 这段时间也熟悉了C++ STL。  
  
下面总结一下LeetCode上的回文数(Palindrome Number)题目，总共有5道，涵盖回溯、动态规划等算法。  
  
#1.Valid Palindrome
[ref](https://oj.leetcode.com/problems/valid-palindrome/) 

> Given a string, determine if it is a palindrome, considering only alphanumeric characters and ignoring cases.  
  For example,  
  "A man, a plan, a canal: Panama" is a palindrome.  
  "race a car" is not a palindrome.  
  Note:  
  Have you consider that the string might be empty? This is a good question to ask during an interview.  
  For the purpose of this problem, we define empty string as valid palindrome.  

这个题目只考虑字母和数字，忽略其它任何字符，空串是回文的。  

{% highlight c++ %}
  class Solution {
  public:
  	bool isPalindrome(string s) 
  	{
  		int si = 0;
  		int ei = s.size() - 1;
  		while (si <= ei)
  		{
  			if (!isalnum(s[si]))
  			{
  				si++;
  				continue;
  			}
  			if (!isalnum(s[ei]))
  			{
  				ei--;
  				continue;
  			}
  			if (tolower(s[si]) != tolower(s[ei]))
  			{
  				return false;
  			}
  			si++;
  			ei--;
  		}
  		return true;
  	}
  };
{% endhighlight %}

#2.Palindrome Number
[ref](https://oj.leetcode.com/problems/palindrome-number/)  
 
 >  Determine whether an integer is a palindrome. Do this without extra space.  

Some hints:  

> Could negative integers be palindromes? (ie, -1)
  If you are thinking of converting the integer to string, note the restriction of using extra space.
  You could also try reversing an integer. However, if you have solved the problem "Reverse Integer", you know that the reversed integer might overflow. How would you handle such case?
  There is a more generic way of solving this problem.
  
题目很简单：判断一个int整数是不是回文数？  
陷阱很多：负数是不是回文数？维基百科里说了，负数不是回文数。   
你是否想着把整数转化成字符串，再按[Valid Palindrome]解法来做，对不起，不能用额外内存。  
  
思路：每次比较最高位和最低位数字，主要问题就是怎么取最高位和最低位，以及比较完成之后怎么保留中间部分。如12321,232,2.  

{% highlight c++ %}
  class Solution {
  public:
  	bool isPalindrome(int x) 
  	{
  	  if (x < 0)
  	  { 
  	    return false; 
  	  }
  		int bits = 0;
  		long long big = 1;
  		int y = x;
  		for (; x / 10 != 0 || x % 10 != 0; bits++, x = x / 10, big *= 10);
  		big /= 10;
  		x = y;
  		for (int i = 0; i < bits / 2; i++)
  		{
  			if (x % 10 != x / big)
  			{
  				return false;
  			}
  			x = (x / 10) % (big/10);
  			big /= 100;
  		}
  		return true;
  	}
  };
{% endhighlight %}

#3.Longest Palindromic Substring
[ref](https://oj.leetcode.com/problems/longest-palindromic-substring/)  
<a name='ch3' ></a>

> Given a string S, find the longest palindromic substring in S. You may assume that the maximum length of S is 1000, and there exists one unique longest palindromic substring.

找出一个字符串的最大回文子串。  

思路：只有遍历完S的所有子串，才能找出最大的回文字符串，或者，按照子串长度从大到小开始遍历，一旦发现回文串，立即返回。
这里面一个子问题就是判断一个子串是不是回文的，可以按如下递归的方法解决：

{% highlight c++   %}
  bool isPalindromic(string &s, int start, int end)
  {
    if(start < end)
    {
      return (s[start]==s[end]) && isPalindromic(s, start+1, end-1);
    }
    return true;
  }
{% endhighlight %}

这里包含了重复子问题，而且具有最优子结构，可以用动态规划来解决。  
定义一个二维数组bool isp[n][n]; isp[i][j](i<=j)表示S[i]...S[j]是否是回文的。  

{% highlight c++%}
  class Solution {
  public:
  	string longestPalindrome(string s)
  	{
  		int n = s.size();
  		if (n < 2)
  		{
  			return s;
  		}
  		//isp[i][j]: s[i..j] is palindromic ?
  		bool isp[1001][1001];
  		for (int i = 0; i < n; isp[i][i] = true, i++);
  		int start=0;
  		int end=0;
  		//substring length
  		for (int L = 2; L <= n; L++)
  		{
  			for (int i = 0; i < n - L + 1; i++)
  			{
  				int j = i + L - 1;
  				if (L == 2)
  				{
  					isp[i][j] = s[i] == s[j];
  				}
  				else
  				{
  					isp[i][j] = (s[i] == s[j]) && isp[i + 1][j - 1];
  				}
  				if (isp[i][j])
  				{
  					start = i;
  					end = j;
  				}
  			}
  		}
  		return s.substr(start, end-start+1);
  	}
  };
{% endhighlight %}

#4.Palindrome Partitioning
[ref](https://oj.leetcode.com/problems/palindrome-partitioning/)  
<a name='ch4' ></a>

> Given a string s, partition s such that every substring of the partition is a palindrome.
  Return all possible palindrome partitioning of s.  
  For example, given s = "aab",  
  Return  
  [  
    ["aa","b"],  
    ["a","a","b"]  
  ]

对字符串s进行分割，使得分割的各个部分都是回文的，本题是要找出所有的划分方案。找出一种划分方案很容易想到递归的方法，
那么获得所有的划分方式就得使用回溯法。

{% highlight c++ %}
  class Solution {
  	vector<vector<string>> result;
  
  	bool palindrome(string &s, int start, int end)
  	{
      if(start < end)
      {
        return (s[start]==s[end]) && palindrome(s, start+1, end-1);
      }
      return true;
  	}
  	bool subpartition(string &s, int start, int end, vector<string> item)
  	{
  		if (start > end)
  		{
  			result.push_back(item);
  			return true;
  		}
  		vector<string> temp;
  		bool partflag = false;
  		for (int i = start; i <=end; i++)
  		{
  			temp = item;
  			if (palindrome(s, start, i))
  			{
  				temp.push_back(s.substr(start, i - start + 1));
  				partflag = subpartition(s, i + 1, end, temp);
  			}
  		}
  		return partflag;
  	}
  public:
  	vector<vector<string>> partition(string s)
  	{
  		result.clear();
  		subpartition(s, 0, s.size() - 1, vector<string>());
  		return result;
  	}
  };
{% endhighlight %}

#5.Palindrome Partitioning II
[ref](https://oj.leetcode.com/problems/palindrome-partitioning-ii/)  

> Given a string s, partition s such that every substring of the partition is a palindrome.  
  Return the minimum cuts needed for a palindrome partitioning of s.  
  For example, given s = "aab",  
  Return 1 since the palindrome partitioning ["aa","b"] could be produced using 1 cut.  

在[Palindrome Partitioning](#ch4)题目中是找出所有的划分方式，本题是在所有的划分方案中，找出一个最小划分，即在保证所有子串是回文的前提下，分割的子串最少。
很明显一个最省力(笨)的方法就是在[Palindrome Partitioning](#ch4)题目的基础上，去搜索一下长度最小的result[i]。 

我们先像[Longest Palindromic Substring](#ch3)题目那样定义一个二维数组isp[n][n]来记录每一个子串的回文情况。  
再定义一个数组 int cut[n]; cut[i]表示s[0]...s[i]所有子串都是回文时，最少的划分次数。

{% highlight c++ %}
  class Solution {
  public:
  	int minCut(string s)
  	{
  		int n = s.size();
  		if (n < 2)
  		{
  			return 0;
  		}
  		//cut[i]记录s[0...i]分割最少多少次，能保证子串是回文数
  		vector<int> cut(n, 0);
  		vector<vector<bool> > isp(n, vector< bool>(n, false));
  		for (int i = 0; i < n; isp[i][i]=true, i++);
  		//substring length
  		for (int L = 2; L <= n;L++)
  		{
  			for (int i = 0; i < n-L+1; i++)
  			{
  				int j = i + L - 1;
  				if (L==2)
  				{
  					isp[i][j] = s[i] == s[j];
  				}
  				else
  				{
  					isp[i][j] = (s[i] == s[j]) && isp[i + 1][j - 1];
  				}
  			}
  		}
  		for (int i = 0; i < n; i++)
  		{
  			if (isp[0][i])
  			{
  				cut[i] = 0;
  			}
  			else
  			{
  				cut[i] = INT_MAX;
  				for (int k = 0; k < i; k++)
  				{
  					if (isp[k + 1][i] && cut[k] + 1 < cut[i])
  					{
  						cut[i] = cut[k] + 1;
  					}
  				}
  			}
  		}
  		
  		return cut[n - 1];
  	}
  };
{% endhighlight %}  

详细分析见[geeksforgeeks](http://www.geeksforgeeks.org/dynamic-programming-set-17-palindrome-partitioning/)，这个文章中最后的O(n^2)解法中有个错误。
