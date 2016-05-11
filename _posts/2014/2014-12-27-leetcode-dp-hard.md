---
layout: single
author_profile: true
comments: true
title: LeetCode DP(下)
tagline: hard部分
category: Algorithm
tags: [LeetCode, 面试]
---
本篇是LeetCode DP类别的高级(Hard)题目： 

1. [Palindrome Partitioning II](#ch1)
2. [Word Break II](#ch2)
3. [Wildcard Matching](#ch3) 
4. [Regular Expression Matching](#ch4)
5. [Maximal Rectangle](#ch5)
6. [Scramble String](#ch6)
7. [Interleaving String](#ch7)

#1.Palindrome Partitioning II
<a name="ch1"></a>
[LeetCode回文数题目集锦](http://onestraw.net/algorithm/leetcode-palindrome/)

#2.Word Break II
[ref](https://oj.leetcode.com/problems/word-break-ii/)
<a name="ch2"></a>
> Given a string s and a dictionary of words dict, add spaces in s to construct a sentence where each word is a valid dictionary word.  
  Return all such possible sentences.   
  For example, given  
  s = "catsanddog",  
  dict = ["cat", "cats", "and", "sand", "dog"].  
  A solution is ["cats and dog", "cat sand dog"].

[Word Break解题思路](http://onestraw.net/algorithm/leetcode-dp-easy/#ch7)  
本题大意是给定一个字符串和一个字典，将字符串分割成一个句子，使得句子中的所有单词都在字典中，返回所有的不同句子。  
####思路

* 首先遍历字符串，记录每一个存在于字典中的子串;
* vbreak[i][j]=true表示单词s[i...j] 在字典中，否则不在字典中;
* DFS构造所有的句子;

{% highlight c++ %}
class Solution {
public:
	vector<string> sen;
	void sentence(string &s, vector<vector<bool> > &vbreak, int end, string word)
	{
		if (end < 0){
			sen.push_back(word);
			return;
		}
		for (int i = end; i >=0; i--){
			if (vbreak[i][end]){
				string w = s.substr(i, end - i + 1);
				if (word.size()>0){
					w = w + " " + word;
				}
				sentence(s, vbreak, i - 1, w);
			}
		}
	}
	vector<string> wordBreak(string s, unordered_set<string> &dict)
	{
		int n = s.size();
		vector<vector<bool> >  vbreak(n, vector<bool>(n, false));
		for (int i = 0; i < n; i++){
			for (int j = i; j < n; j++){
				if (dict.find(s.substr(i, j - i + 1)) != dict.end()){
					vbreak[i][j] = true;
				}
			}
		}
		sentence(s, vbreak, n - 1, "");
		return sen;
	}
};
{% endhighlight %}

#3.Wildcard Matching
[ref](https://oj.leetcode.com/problems/wildcard-matching/)
<a name="ch3"></a>
> Implement wildcard pattern matching with support for '?' and '*'.

    '?' Matches any single character.
    '*' Matches any sequence of characters (including the empty sequence).
    
    The matching should cover the entire input string (not partial).
    
    The function prototype should be:
    bool isMatch(const char *s, const char *p)
    
    Some examples:
    isMatch("aa","a") → false
    isMatch("aa","aa") → true
    isMatch("aaa","aa") → false
    isMatch("aa", "*") → true
    isMatch("aa", "a*") → true
    isMatch("ab", "?*") → true
    isMatch("aab", "c*a*b") → false

通配符(wildcard)匹配问题，这个题目有多种不同的解法，可以用回溯法，可以用动态规划，也可以用贪心，尝试了两个动态规划算法， 不是Time Limit Exceeded就是Memory Limit Exceeded.  
####回溯 思路

* 这个问题的难点就是处理*,它可以匹配0个，也可以匹配到结尾;
* 从前向后进行匹配，记录*最后出现的位置star，以便后面发生匹配时，进行回溯;
* 为准确回溯，除了记录*的位置，还要记录搜索字符串匹配到*的位置ss;
* *先匹配0个，每次回溯增加一个;

{% highlight c++ %}
bool isMatch(const char *s, const char *p) {
	const char *star = NULL;
	const char *ss = s;
	while (*s){
		if (*s == *p || *p == '?'){
			s++;
			p++;
			continue;
		}
		if (*p == '*'){
			ss = s;
			star = p++;
			continue;
		}
		if (star){
			s = ++ss;
			p = star + 1;
			continue;
		}
		return false;
	}
	while (*p == '*'){
		p++;
	}
	return !*p;
}
{% endhighlight %}
<a href="http://yucoding.blogspot.com/2013/02/leetcode-question-123-wildcard-matching.html" target="_blank">参考</a>

#4.Regular Expression Matching
[ref](https://oj.leetcode.com/problems/regular-expression-matching/)
<a name="ch4"></a>
> Implement regular expression matching with support for '.' and '*'.

    '.' Matches any single character.
    '*' Matches zero or more of the preceding element.
    
    The matching should cover the entire input string (not partial).
    
    The function prototype should be:
    bool isMatch(const char *s, const char *p)
    
    Some examples:
    isMatch("aa","a") → false
    isMatch("aa","aa") → true
    isMatch("aaa","aa") → false
    isMatch("aa", "a*") → true
    isMatch("aa", ".*") → true
    isMatch("ab", ".*") → true
    isMatch("aab", "c*a*b") → true

正则表达式匹配，它和[通配符匹配](#ch3)的不同在于*是匹配前一个字符0次或者多次，这里的'.'相当于前一题的'?'.这个题目同样有回溯和动态规划两种解法。

{% highlight c++ %}
bool isMatch(const char *s, const char *p) {
	if (!*p){
		return !*s;
	}
	if (*(p + 1) != '*'){
		if (*p == *s || (*p == '.' && *s != 0)){
			return isMatch(s + 1, p + 1);
		}
		return false;
	}
	else{
		while (*p == *s || (*p == '.' && *s != 0)){
			if (isMatch(s, p + 2)){
				return true;
			}
			s++;
		}
		return isMatch(s, p + 2);
	}
}
{% endhighlight %}
<a href="http://blog.csdn.net/hopeztm/article/details/7992253" target="_blank">参考</a>  

#5.Maximal Rectangle
[ref](https://oj.leetcode.com/problems/maximal-rectangle/)
<a name="ch5"></a>
> Given a 2D binary matrix filled with 0's and 1's, find the largest rectangle containing all ones and return its area.

###朴素思路

这个问题有点难，将问题分解成n个子问题`Largest Rectangle in Histogram`


{% highlight c++ %}
int maximalRectangle(vector<vector<char> > &matrix)
{
	if (matrix.size() < 1){
		return 0;
	}
	int m = matrix.size();
	int n = matrix[0].size();
	vector<vector<int> > area(m, vector<int>(n, 0));
	vector<vector<int> > up(m, vector<int>(n, 0));
	vector<vector<int> > down(m, vector<int>(n, 0));
	//get largest continuous '1' in horizontal
	for (int i = 0; i < m; i++){
		area[i][0] = matrix[i][0] - '0';
		for (int j = 1; j < n; j++){
			if (matrix[i][j] == '1'){
				area[i][j] = area[i][j - 1] + 1;
			}
		}
	}
	//get largest Rectangle Area in vertical direction
	//refer solution of `Largest Rectangle in Histogram `
	for (int i = 0; i < n; i++){
		for (int j = 0; j < m; j++){
			int k = j;
			while (k > 0 && area[j][i] <= area[k - 1][i]){
				k = up[k - 1][i];
			}
			up[j][i] = k;
		}

		for (int j = m - 1; j >= 0; j--){
			int k = j;
			while (k < m - 1 && area[j][i] <= area[k + 1][i]){
				k = down[k + 1][i];
			}
			down[j][i] = k;
		}
	}
	int maxrect = 0;
	for (int i = 0; i < m; i++){
		for (int j = 0; j < n; j++){
			maxrect = max(maxrect, (down[i][j] - up[i][j] + 1)*area[i][j]);
		}
	}
	return maxrect;
} 
{% endhighlight %}

#6.Scramble String
[ref](https://oj.leetcode.com/problems/scramble-string/)
<a name="ch6"></a>
> Given a string s1, we may represent it as a binary tree by partitioning it to two non-empty substrings recursively.

Below is one possible representation of s1 = "great":  

        great
       /    \
      gr    eat
     / \    /  \
    g   r  e   at
               / \
              a   t
          
To scramble the string, we may choose any non-leaf node and swap its two children.  

For example, if we choose the node "gr" and swap its two children, it produces a scrambled string "rgeat".  

        rgeat
       /    \
      rg    eat
     / \    /  \
    r   g  e   at
               / \
              a   t
          
We say that "rgeat" is a scrambled string of "great".  

Similarly, if we continue to swap the children of nodes "eat" and "at", it produces a scrambled string "rgtae".  

        rgtae
       /    \
      rg    tae
     / \    /  \
    r   g  ta  e
           / \
          t   a
      
We say that "rgtae" is a scrambled string of "great".  

Given two strings s1 and s2 of the same length, determine if s2 is a scrambled string of s1.  

###一般思路

- 将字符串s1分成左子树s1ltree和右子树s1rtree，将字符串s2分成左子树s2ltree和右子树s2rtree;
- return (isScramble(s1ltree, s2ltree) && isScramble(s1rtree, s2rtree)) || 
		(isScramble(s1ltree, s2rtree) && isScramble(s1rtree, s2ltree));
- 这里有一个问题，ltree和rtree的长度并不一定相等，所以进行isScramble(s1ltree, s2ltree)和isScramble(s1ltree, s2rtree)时要进行两次不同的划分，确保isScramble的参数长度相等，就像例子中的eat子树和tae子树一样;

注意题目第一句话`Given a string s1, we may represent it as a binary tree by partitioning it to two non-empty substrings recursively.`，最初对题目理解偏差，以为是将字符串划分成尽可能的平衡树，实际上是划分成任意长度的子树(non-empty substrings)

###递归方法(超时)

{% highlight c++ %}
typedef string::iterator Iterator;
bool isScramble(Iterator first1, Iterator last1, Iterator first2, Iterator last2) {
	int len1 = distance(first1, last1);
	int len2 = distance(first2, last2);
	if (len1 != len2){
		return false;
	}
	if (len1 == 1){
		return *first1 == *first2;
	}
	for (int split = 1; split < len1; split++){
		if ((isScramble(first1, first1 + split, first2, first2 + split) && isScramble(first1 + split, last1, first2 + split, last2)) ||
			(isScramble(first1, first1 + split, last2 - split, last2) && isScramble(first1 + split, last1, first2, last2 - split))){
			return true;
		}
	}
	return false;
}

bool isScramble(string s1, string s2) {
	return isScramble(s1.begin(), s1.end(), s2.begin(), s2.end());
}
{% endhighlight %}   

简化的递归树如下 
![Scramble string](/assets/images/scramble_string.png)

从上图看出，有很多重复的子问题，下面用动态规划来解决。
使用了一个三维数组bool ss[len][len][len],其中第一维为子串的长度，第二维为s1的起始索引，第三维为s2的起始索引。  
ss[k][i][j]表示s1[i...i + k-1]是否可以由s2[j...j + k-1]变化得来。  

{% highlight c++ %}
bool isScramble(string s1, string s2) {
	int n = s1.size();
	if (n != s2.size()){
		return false;
	}
	vector<vector<vector<bool>>> ss(n + 1, vector<vector<bool>>(n, vector<bool>(n, false)));
	for (int i = 0; i < n; i++){
		for (int j = 0; j < n; j++){
			ss[1][i][j] = (s1[i] == s2[j]);
		}
	}

	for (int len = 1; len <= n; len++){
		for (int i = 0; i <= n - len; i++){
			for (int j = 0; j <= n - len; j++){
				for (int k = 1; k < len; k++){
					if ((ss[k][i][j] && ss[len - k][i + k][j + k]) ||
						(ss[k][i][j + len - k] && ss[len - k][i + k][j])){
						ss[len][i][j] = true;
						break;
					}
				}
			}
		}
	}
	return ss[n][0][0];
} 
{% endhighlight %}

#7.Interleaving String
[ref](https://oj.leetcode.com/problems/interleaving-string/)
<a name="ch7"></a>
> Given s1, s2, s3, find whether s3 is formed by the interleaving of s1 and s2.  
  For example,  
  Given:  
  s1 = "aabcc",  
  s2 = "dbbca",  
  When s3 = "aadbbcbcac", return true.  
  When s3 = "aadbbbaccc", return false.  

> 设状态 f[i][j] ，表示 s1[0, i] 和 s2[0, j] ，匹配 s3[0, i + j] 。如果 s1 的最后一个字符等
	于 s3 的最后一个字符，则 f[i][j] = f[i - 1][j] ；如果 s2 的最后一个字符等于 s3 的最后一个字符，则 f[i][j] = f[i][j - 1] 。
	因此状态转移方程如下：  
	f[i][j] = (s1[i - 1] == s3[i + j - 1] && f[i - 1][j])|| (s2[j - 1] == s3[i + j - 1] && f[i][j - 1]);
	
{% highlight c++ %}
bool isInterleave(string s1, string s2, string s3) {
	if (s1.size() + s2.size() != s3.size()){
		return false;
	}
	if (s3.size() == 0){
		return true;
	}
	vector<vector<bool>> f(s1.size() + 1, vector<bool>(s2.size() + 1, false));

	for (int i = 1; i <= s1.size(); i++){
		f[i][0] = (s1[i - 1] == s3[i - 1]);
	}
	for (int i = 1; i <= s2.size(); i++){
		f[0][i] = (s2[i - 1] == s3[i - 1]);
	}
	for (int i = 1; i <= s1.size(); i++){
		for (int j = 1; j <= s2.size(); j++){
			f[i][j] = (s1[i - 1] == s3[i + j - 1] && f[i - 1][j])
				|| (s2[j - 1] == s3[i + j - 1] && f[i][j - 1]);
		}
	}
	return f[s1.size()][s2.size()];
}
{% endhighlight %}  

[reference](https://github.com/soulmachine/leetcode)
