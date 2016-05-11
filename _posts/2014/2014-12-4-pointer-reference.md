---
layout: single
author_profile: true
comments: true
title: 指针引用应用场景
tagline: leetcode实例
category: cprogram
tags: [C/C++, LeetCode]
---
在leetcode刷题时，有一个链表加法的问题，我用指针引用技术减少了代码冗余。

#Add Two Numbers

原题==>[Add Two Numbers](https://oj.leetcode.com/problems/add-two-numbers/)  

题目比较简单，在函数中我不想修改参数指针l1,l2,所以在3个地方用到了创建新节点，并插入到结果链表尾部：

1. 链表l1和l2和重叠部分；  
2. 较长的链表尾部；
3. 最高位可能有进位；

对应着下面代码的1，2，3位置的insert()函数调用，很明显代码冗余，所以剥离出一个insert函数。我写的第一个版本的insert函数是

	void insert(ListNode *head, ListNode *p, int value, int &r);

leetcode上提交是Wrong Answer, 实际输出为空，也就是说*addTwoNumbers()返回了个空指针。 
我很快定位到了insert函数的两个指针参数问题，ListNode *head 这种格式的参数在函数中只能通过 *head =x；来修改head指针所指向的内存中的值，
不能修改head本身的值，而insert函数中我恰恰修改了指针本身的值(head=p=node)，解决方法就是将指针参数改成指针引用参数：

	void insert(ListNode* &head, ListNode* &p, int value, int &r)

改成下面形式就更好理解了

	typedef ListNode*	ListNodePtr;
	void insert(ListNodePtr &head, ListNodePtr &p, int value, int &r)

{% highlight C++ linenos %}

struct ListNode {
	int val;
	ListNode *next;
	ListNode(int x) : val(x), next(NULL) {}
};

class Solution {
public:

	void insert(ListNode* &head, ListNode* &p, int value, int &r)
	{
		ListNode *node = new ListNode(value % 10);
		r = value / 10;
		if (!p)
		{
			head = p = node;
		}
		else
		{
			p->next = node;
			p = p->next;
		}
	}

	ListNode *addTwoNumbers(ListNode *l1, ListNode *l2) {
		ListNode *head = NULL;
		ListNode *p=head;
		ListNode *p1 = l1;
		ListNode *p2 = l2;
		ListNode *pt = NULL;
		int r = 0;
		int s = 0;
		//1
		for (; p1 && p2; p1 = p1->next, p2 = p2->next)
		{
			s = r + p1->val + p2->val;
			insert(head, p, s, r);
		}
		//2
		if (p1)
		{
			pt = p1;
		}
		else
		{
			pt = p2;
		}
		for (; pt; pt = pt->next)
		{
			s = r + pt->val;
			insert(head, p, s, r);
		}
		//3
		if (r > 0)
		{
			insert(head, p, r, r);
		}
		return head;
	}
};

{% endhighlight %}

#指针引用

将指针看作一种数据类型

	int*  
	char*  
	...

如果看着还是不顺眼，就用typedef定义指针类型  

	typedef int*  int_ptr;  
	typedef char* char_ptr;  
	... 
  
普通的数据类型引用格式是
  
	int &  
	char &  
	...  

指针引用和它们一样

	int_ptr &  
	char_ptr &  
	...
  
等价于
  
	int *&
	char *&  
	...

引用的重要用途是函数传参，避免参数拷贝，可以直接对源数据进行修改，在函数调用结束后，保留修改。下面两种方式的区别：

{% highlight c linenos %}

  void func1(int *p)
  {
    *p = 10;
  }
  void func2(int &p);
  {
    p = 10;
  }
  
  int main()
  {
    int x = 1000;
    int *p = &x; 
    func1(p);   //<==>func1(&x);
    func2(x);
  }
  
{% endhighlight %}
  
从上面代码可以看出，指针传参，在函数内部修改是的指针所指向的内容，不是指针本身；引用传参修改的就是引用的数据变量。  

那么问题来了，如果我们要修改指针本身呢，就像第一部分中，要通过一个函数修改链表指针参数。  

答案很简单，

	int &x; 修改x的值  
	char &ch; 修改ch的值  
 	...  
	int_ptr &p; 也就会修改p的值  
  
int_ptr &p;  等价于  int *&p;
