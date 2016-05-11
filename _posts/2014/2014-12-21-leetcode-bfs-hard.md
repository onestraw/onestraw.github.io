---
layout: single
author_profile: true
comments: true
title: LeetCode BFS(下)
tagline: hard部分
category: Algorithm
tags: [LeetCode, 面试]
---
这里是LeetCode BFS类型后半部分题目：  

* [Surrounded Regions ](#ch1)
* [Word Ladder](#ch2)
* [Word Ladder II](#ch3) 

#1.Surrounded Regions
[ref](https://oj.leetcode.com/problems/surrounded-regions/)
<a name="ch1"></a>
> Given a 2D board containing 'X' and 'O', capture all regions surrounded by 'X'.  
  A region is captured by flipping all 'O's into 'X's in that surrounded region.
  For example,   
  
	X X X X
	X O O X
	X X O X
	X O X X

> After running your function, the board should be:

	X X X X
	X X X X
	X X X X
	X O X X

题目大意是将X包围的O全部改为X。 
###思路一
对矩阵中的任一位置的O，判断它能否找到一个全O路径(上下左右移动)，通向外界。OK, 这里DFS和BFS算法均可，下面是一个BFS代码
{% highlight c++ %}
class Solution {
	int m;
	int n;
	
	bool isClosure(vector<vector<char>> &board, int x, int y)
	{
		vector<vector<bool>> visited(m, vector<bool>(n, false));
		queue<pair<int, int>> q;
		pair<int, int> pos = { x, y };
		q.push(pos);
		visited[x][y] = true;
		while (!q.empty())
		{
			pos = q.front();
			q.pop();
			x = pos.first;
			y = pos.second;
			
			if (x == 0 || x == m - 1 || y == 0 || y == n - 1)
			{
				return false;
			}
			//up, down, left, right
			if (x > 0 && board[x - 1][y] == 'O' && !visited[x - 1][y])
			{
				q.push(pair<int, int>(x - 1, y));
				visited[x - 1][y] = true;
			}
			if (x < m - 1 && board[x + 1][y] == 'O' && !visited[x + 1][y])
			{
				q.push(pair<int, int>(x + 1, y));
				visited[x + 1][y] = true;
			}
			if (y > 0 && board[x][y - 1] == 'O' && !visited[x][y - 1])
			{
				q.push(pair<int, int>(x, y - 1));
				visited[x][y - 1] = true;
			}
			if (y < n - 1 && board[x][y + 1] == 'O' && !visited[x][y + 1])
			{
				q.push(pair<int, int>(x, y + 1));
				visited[x][y + 1] = true;
			}
		}
		return true;
	}
public:
	//判断 O 能不能找到一路径出去
	//如果不能出去就置 X 
	void solve(vector<vector<char>> &board) 
	{
		m = board.size();
		if (m < 1)
		{
			return;
		}
		n = board[0].size();
		for (int i = 1; i < m - 1; i++)
		{
			for (int j = 1; j < n - 1; j++)
			{
				if (board[i][j] == 'O' && isClosure(board, i, j))
				{
					board[i][j] = 'X';
				}
			}
		}
	}
}; 
{% endhighlight %}
但是当矩阵规模200x200以上时会超时Time Limit Exceeded。显然有更好的算法。

###思路二
上述问题之所以超时，是有太多的重复搜索，我考虑过在搜索完成一次，如果没有找到路径，就将访问过的位置全部置X，可是当矩阵规模达到250x250，还是超时。  
从外向里搜索，对于O的位置，其周围的O位置保持，其它未搜索到的O就反转为X，这样就避免了重复搜索。
{% highlight c++ %}
class Solution {
public:
	//from outside to inside
	void solve(vector<vector<char>> &board) 
	{
		int m = board.size();
		if (m < 1)
		{
			return;
		}
		int n = board[0].size();
		typedef pair<int, int> POS;
		queue<POS> q;
		POS pos;
		vector<vector<bool>> xo(m, vector<bool>(n, true));//true for x
		//left and right
		for (int i = 0; i < m; i++)
		{
			if (board[i][0] == 'O')
			{
				q.push(POS(i, 0));
				xo[i][0] = false;
			}
			if (board[i][n - 1] == 'O')
			{
				q.push(POS(i, n - 1));
				xo[i][n - 1] = false;
			}
		}
		//up and down
		for (int i = 0; i < n; i++)
		{
			if (board[0][i] == 'O')
			{	
				q.push(POS(0, i));
				xo[0][i] = false;
			}
			
			if (board[m - 1][i] == 'O')
			{
				q.push(POS(m - 1, i));
				xo[m - 1][i] = false;
			}	
		}

		while (!q.empty())
		{
			pos = q.front();
			q.pop();
			if (pos.first > 0 && board[pos.first - 1][pos.second] == 'O' && xo[pos.first - 1][pos.second])
			{
				xo[pos.first - 1][pos.second] = false;
				q.push(POS(pos.first - 1, pos.second));
			}
			if (pos.first < m - 1 && board[pos.first + 1][pos.second] == 'O' && xo[pos.first + 1][pos.second])
			{
				xo[pos.first + 1][pos.second] = false;
				q.push(POS(pos.first + 1, pos.second));
			}
			if (pos.second > 0 && board[pos.first][pos.second - 1] == 'O' && xo[pos.first][pos.second - 1])
			{
				xo[pos.first][pos.second - 1] = false;
				q.push(POS(pos.first, pos.second - 1));
			}
			if (pos.second < n - 1 && board[pos.first][pos.second + 1] == 'O' && xo[pos.first][pos.second + 1])
			{
				xo[pos.first][pos.second + 1] = false;
				q.push(POS(pos.first, pos.second + 1));
			}
		}

		for (int i = 0; i < m; i++)
		{
			for (int j = 0; j < n; j++)
			{
				board[i][j] = xo[i][j] ? 'X' : 'O';
			}
		}
	}
}; 

{% endhighlight %}
#2.Word Ladder
[ref](https://oj.leetcode.com/problems/word-ladder/)
<a name="ch2"></a>
> Given two words (start and end), and a dictionary, find the length of shortest transformation sequence from start to end, such that:  
  
  1. Only one letter can be changed at a time  
  2. Each intermediate word must exist in the dictionary  
  
  For example,  

	Given:  
	start = "hit"  
	end = "cog"  
	dict = ["hot","dot","dog","lot","log"]  
	As one shortest transformation is "hit" -> "hot" -> "dot" -> "dog" -> "cog",  
	return its length 5.  
 
Note:
  
  * Return 0 if there is no such transformation sequence.
  * All words have the same length.
  * All words contain only lowercase alphabetic characters.

这道题目我又是先想了一个简单而超时的解法，还是分开说一下吧
###思路一
1. 将ladder当前的单词记为word，将word可能变化成的单词记为chword，word初始为start，chword与word只相差一个字母;
2. 遍历word所有字符位置，依次改变一个位置，对每一个位置有25种变化可能;
3. 如果chword等于end，直接返回;
4. 如果chword存在于字典dict中，则加入队列q;
5. 队列中的每一个单词有一个ladder length，表示从start单词变化到此的ladder长度，初始length=1; 

{% highlight c++  %}
int ladderLength(string start, string end, unordered_set<string> &dict)
{
	if (end.compare(start) == 0)
	{
		return 1;
	}
	int length = 0;
	string word;
	string nword;
	typedef pair<string, int> PAIR;
	queue<PAIR > q;
	q.push(PAIR(start, 1));
	while (!q.empty())
	{
		word = q.front().first;
		length = q.front().second;
		q.pop();

		//change position in start word
		for (int i = 0; i < start.size(); i++)
		{
			//26 alpha
			for (int j = 0; j < 26; j++)
			{
				if (word[i] == 'a' + j)
				{
					continue;
				}
				nword = word;
				nword.replace(i, 1, 1, char('a' + j));
				//cout << nword << endl;
				if (end.compare(nword) == 0)
				{
					return length + 1;
				}
				if (dict.find(nword) != dict.end())
				{
					q.push(PAIR(nword, length + 1));
				}
			}
		}
	}
	return 0;
}
{% endhighlight %}
可惜又超时了。

###思路二
分析一下超时的原因，看上面代码，有两个for循环，第一个遍历单词的所有位置，不可或缺！第二个for循环遍历所有字母，有很多无用功(根本不在dict中)，看起来可以优化。
1. compare()函数，没必要每次变化出一个单词就比较，可以在第一个for循环前面加上一个判断，看看word和end之间是不是只差一个字母。
2. hot -> dot 之后，在搜索dot时，第一个字母不应该再变化了，因为dot变换首字母的情况和hot变换首字母的情况重复了，而思路一明显没有考虑这种情况，可能会造成无穷循环。解决办法是

> 增加一个unordered_set<string> used; 用于在插入队列时判断这个单词是否已经入过队列；或者直接在dict中删除。


{% highlight c++  %}
int diffnum(string s1, string s2)
{
	int n = 0;
	for (int i = 0; i < s1.size(); i++)
	{
		if (s1[i] != s2[i])
		{
			n++;
		}
	}
	return n;
}

int ladderLength(string start, string end, unordered_set<string> &dict)
{
	if (end.compare(start) == 0)
	{
		return 1;
	}
	bool cmpflag = false;
	int length = 0;
	string word;
	string nword;
	typedef pair<string, int> PAIR;
	queue<PAIR > q;
	unordered_set<string> used;
	q.push(PAIR(start, 1));
	while (!q.empty())
	{
		word = q.front().first;
		length = q.front().second;
		q.pop();
		//first judge the diff letter num
		cmpflag = diffnum(word, end) == 1 ? true : false;
		//change position in start word
		for (int i = 0; i < start.size(); i++)
		{
			for (char c = 'a'; c <= 'z'; c++)
			{
				if (word[i] == c)
				{
					continue;
				}
				nword = word;
				nword[i] = c;

				//cout << nword << endl;
				if (cmpflag && end.compare(nword) == 0)
				{
					return length + 1;
				}
				if (dict.find(nword) != dict.end() && (used.find(nword) == used.end()))
				{
					q.push(PAIR(nword, length + 1));
					used.insert(nword);
				}
			}
		}
	}
	return 0;
}
{% endhighlight %}


#3.Word Ladder II
[ref](https://oj.leetcode.com/problems/word-ladder-ii/)
<a name="ch3"></a>
> Given two words (start and end), and a dictionary, find all shortest transformation sequence(s) from start to end, such that:  
  
  1. Only one letter can be changed at a time  
  2. Each intermediate word must exist in the dictionary  
  
  For example,

  Given:  
  start = "hit"  
  end = "cog"   
  dict = ["hot","dot","dog","lot","log"]  
  Return
  [  
    ["hit","hot","dot","dog","cog"],  
    ["hit","hot","lot","log","cog"]  
  ]  

  Note:

  * All words have the same length.
  * All words contain only lowercase alphabetic characters.
  
下面是参考一个大牛的代码，其主要思路是把每个单词的前驱记录下来，用vector<string>保存，他没有直接使用队列，是用两个数组模拟队列，两个数组交替表示当前层和上一层。  

<img src="/assets/images/wordladder.png" />  

{% highlight c++ %}
class Solution {
public:
	vector<vector<string> > findLadders(string start, string end, unordered_set<string> &dict) {
		pathes.clear();
		dict.insert(start);
		dict.insert(end);
		vector<string> prev;
		unordered_map<string, vector<string> > traces;
		for (unordered_set<string>::const_iterator citr = dict.begin();citr != dict.end(); citr++) {
			traces[*citr] = prev;
		}

		vector<unordered_set<string> > layers(2);
		int cur = 0;
		int pre = 1;
		layers[cur].insert(start);
		while (true) {
			cur = !cur;
			pre = !pre;
			for (unordered_set<string>::const_iterator citr = layers[pre].begin();
				citr != layers[pre].end(); citr++)
				dict.erase(*citr);
			layers[cur].clear();
			for (unordered_set<string>::const_iterator citr = layers[pre].begin();
				citr != layers[pre].end(); citr++) {
				for (int n = 0; n<(*citr).size(); n++) {
					string word = *citr;
					int stop = word[n] - 'a';
					for (int i = (stop + 1) % 26; i != stop; i = (i + 1) % 26) {
						word[n] = 'a' + i;
						if (dict.find(word) != dict.end()) {
							traces[word].push_back(*citr);
							layers[cur].insert(word);
						}
					}
				}
			}
			if (layers[cur].size() == 0)
				return pathes;
			if (layers[cur].count(end))
				break;
		}
		vector<string> path;
		buildPath(traces, path, end);

		return pathes;
	}

private:
	void buildPath(unordered_map<string, vector<string> > &traces,
		vector<string> &path, const string &word) {
		if (traces[word].size() == 0) {
			path.push_back(word);
			vector<string> curPath = path;
			reverse(curPath.begin(), curPath.end());
			pathes.push_back(curPath);
			path.pop_back();
			return;
		}

		const vector<string> &prevs = traces[word];
		path.push_back(word);
		for (vector<string>::const_iterator citr = prevs.begin();
			citr != prevs.end(); citr++) {
			buildPath(traces, path, *citr);
		}
		path.pop_back();
	}

	vector<vector<string> > pathes;
};
{% endhighlight %}
参考
[Word Ladder II](http://blog.csdn.net/niaokedaoren/article/details/8884938)  
