---
layout: single
author_profile: true
comments: true
title: 检测链表中存在的环
categories: [Algorithm]
tags: [C/C++]
---
“最后的答案： 首先，排除一种特殊的情况，就是3个元素的链表中第2个元素的后面是第1个元素。设置两个指针p1和p2，p1指向第1个元素，p2指向第3个元素，看看 它们是否相等。如果相等就属于上述这种特殊情况。如果不等，把p1向后移一个元素，p2向后移两个元素。检查两个指针的值，如果相等，说明链表中存在循环。如果不相等，继续按照前述方法进行。如果出现某个指针是NULL的情况，说明链表中不存在循环。如果链表中存在循环，用这种方法肯定能够检测出来，因 为其中一个指针肯定能够追上另一个（两个指针具有相同的值），尽管有可能需要对这个链表经过几次遍历才能检测出来。”

——题目来自《C专家编程》附录A.2

##编程挑战

“证明上面最后一种方法可以检测到链表中可能存在的任何循环。在链表中设置一个循环，演练一下你的代码；把循环变得长一些，继续演练你的代码，重复进行，直到初始条件不满足为止。同样，确定当链表中不存在循环时算法可以终止。”

## 证明

这是初中小学数学中的追逐问题。

如果一个链表存在环，那么指针p2先进入环，p1后进入环，设环的长度（结点个数为L），当指针p1进入环时，p2领先p1的结点个数为k。假设 p1和p2可以相遇，即该方法能检测出循环。假定从指针p1进入环开始，到p1和p2相遇，p1移动了x次，那么p2移动了2x次。p2比p1多跑了n 圈，指针p2和p1走的“距离”有以下关系

	2*x+k = x + n*L  
	x = n*L – k  有解  

## 代码验证

不构造那个特殊的只有3个结点的链表，并且第二个节点的next指向第一具结点 .

{% highlight c linenos %}

#include<stdio.h>
#include<stdlib.h>

typedef struct Node{
	char value;
	struct Node *next;
}Node;

void constructCycleLinlist(Node *head)
{
	int i;
	Node *newNode, *p;
	p = head;
	for(i=0; i<10000; i++)
	{
		newNode = (Node*)malloc(sizeof(Node));
		newNode->value = 'A';
		newNode->next = NULL;
		p->next = newNode;
		p = p->next;
	}
	//构造环，注释掉可测试无循环的情形 
	p = head;
	for(i=0; i<10; i++)
		p = p->next;
	newNode->next = p;
}

void detectCycle(Node *head)
{
	Node *p1, *p2;
	p1 = head;
	p2 = head->next->next;
	
	while(p1 != p2)
	{
		if(p1 && p1->next)
			p1=p1->next;
		else
			p1=NULL;
		
		if(p2 && p2->next && p2->next->next)
			p2 = p2->next->next;
		else
			p2 = NULL;
	}
	if(p1==NULL)
		printf("no cycle\n");
	else
		printf("find cycle\n");
}

int main()
{
	Node *head;
	head = (Node*)malloc(sizeof(Node));
	constructCycleLinlist(head);
	detectCycle(head);
	return 0;
}

{% endhighlight %}
