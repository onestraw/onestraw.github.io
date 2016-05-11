---
layout: single
author_profile: true
comments: true
title: 翻转句子中单词的顺序
categories: [cprogram]
tags: [C/C++, 笔试]
---
<h1><strong>题目</strong></h1> 

输入一个英文句子，翻转句子中单词的顺序，但单词内字符的顺序不变。  
句子中单词以空格符隔开。为简单起见，标点符号和普通字母一样处理。  
例如输入“I am a student.”，则输出“student. a am I”。

<h1><strong>分析</strong></h1>

翻转单词的顺序，却不改变字符顺序，把每个单词看作一个整体，该题就是明显的考查栈的题目——后进先出，从左向右第一个读到单词，最后一个输出。

如果用栈来解决的话，该如何在栈中存储单词呢？

指针!

为节省空间，不再用新的内存来存储字符串，而是只存储每个单词在字符串中的开始位置和结束位置。

定义一个结构体Word，它有两个指针，用来保存单词在句子中的开始和结束位置

<pre>typedef struct Word{
	char *phead;
	char *ptail;
}Word;</pre>

也可以不用char*指针，用两个整数保存数组下标。

<pre class="lang:c decode:true">#include&lt;stdio.h&gt;
#include&lt;stdlib.h&gt;
#include&lt;string.h&gt;
#define MaxSize 20

typedef struct Word{
	char *phead;
	char *ptail;
}Word;

typedef struct Stack{
	Word *w;
	int top;
}Stack;

void initStack(Stack *s)
{
	s-&gt;w = (Word *)malloc(sizeof(Word)*MaxSize);
	s-&gt;top = -1;
}

void push(Stack *s, Word word)
{
	if((s-&gt;top + 1) &lt; MaxSize)
		s-&gt;w[++s-&gt;top] = word;
	else
		printf("Stack full\n");
}

Word *pop(Stack*s)
{
	if(s-&gt;top&lt;0)
		printf("Stack empty\n");
	else
		return &amp;s-&gt;w[s-&gt;top --];
}

void printWord(Word *w)
{
	for(; w-&gt;phead !=w-&gt;ptail; w-&gt;phead++)
		printf("%c", *w-&gt;phead);
	printf("%c ", *w-&gt;ptail);
}

void reverseSentence(char *str, int len)
{
	int i;
	Stack *s;
	s = (Stack*)malloc(sizeof(Stack));
	initStack(s);
	Word *w;
	w= (Word*)malloc(sizeof(Word));
	for(i=0; i&lt;len; i++)
	{
		if((i==0 || str[i-1]==' ')&amp;&amp; str[i]!=' ')
			w-&gt;phead = (str+i);
		if((i==len-1 || str[i+1]==' ') &amp;&amp; str[i]!=' ')
		{
			w-&gt;ptail = (str+i);
			push(s, *w);
		}
	}
	while(s-&gt;top &gt; -1)
	{
		w = pop(s);
		printWord(w);
	}
	printf("\n");
	free(s);
	free(w);
}

int main()
{
	char *sentence="I am SwordStraw; welcome to http://onestraw.net";
	int len = strlen(sentence);
	reverseSentence(sentence, len);
	return 0;
}</pre>
