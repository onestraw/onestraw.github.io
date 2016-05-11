---
layout: single
author_profile: true
comments: true
title: Linux内核中的链表
tagline: 不一样的实现
category: Linux
tags : [Linux]
---

##struct list_head

内核中定义的链表结点和数据是分离的，链表结点定义如下

    struct list_head{
    	struct list_head *next;
    	struct list_head *prev;
    };

任何使用双向链表组织的数据结构包含一个struct list_head成员，如下

    struct book_list{
    	char name[256];
    	int price;
    	struct list_head list;
    };
  
遍历链表book_list肯定要通list_head，但是如何根据list_head访问struct book_list的成员呢？

**`内核的实现方法`**

**一个book_list变量的地址 = 结构成员list的地址 - list在struct book_list中的偏移地址**  

` book_list book'addr = book.list'addr - list's offset in book_list`   

下面看看具体的代码实现

##container_of

它在内核的宏定义是

    #define container_of(ptr, type, member) ({                  \
              const typeof( ((type *)0)->member ) *__mptr = (ptr);\
              (type *)( (char *)__mptr - offsetof(type,member) );})

`(type*)0`   是将0强制转换成type*类型，这里0可以换成任何数；  
`typeof` 是取得一个变量的类型，进而用它定义一个相同类型变量;   

     int a;
     typeof(a) b;

等价于

     int a, b;

`offsetof(type, member)`就是取得member相对type起始地址的偏移字节  

offsetof(struct_list, price)等于256   

所以，如果已知boot_list.list的地址addr，和list在类型struct book_list中的偏移offset，就能得到struct book_list的地址addr - offset

那么可以简单定义成如下

    #define container_of(ptr, type, member) ({                  \
              (type *)( (char *)ptr - offsetof(type,member) );})

为什么内核定义那么复杂呢？？？（在下面的测试程序中改成上述形式能够正常运行） 


##测试程序

    #include<stdio.h>
    #include<stdlib.h>
    #include<stddef.h>
    
    ////////////////////////////////////////////
    #define container_of(ptr, type, member) ({                  \
    		const typeof( ((type *)0)->member ) *__mptr = (ptr);\
    		(type *)( (char *)__mptr - offsetof(type,member) );})
    
    #define list_entry(ptr, type, member) \
    	container_of(ptr, type, member)
    
    #define list_for_each(pos, head) \
        for (pos = (head)->next; pos != (head); \
             pos = pos->next)
    
    #define list_for_each_entry(pos, head, member)              \
    	for (pos = list_entry((head)->next, typeof(*pos), member);  \
             &pos->member != (head);    \
             pos = list_entry(pos->member.next, typeof(*pos), member))
    
    ////////////////////////////////////////////
    
    struct list_head{
    	struct list_head *next;
    	struct list_head *prev;
    };
    static inline void __list_add(struct list_head *new, struct list_head*prev, struct list_head*next)
    {
    	next->prev = new;
    	new->next = next;
    	new->prev = prev;
    	prev->next = new;
    }
    static inline void list_add(struct list_head *new, struct list_head *head)
    {
    	__list_add(new, head, head->next);
    }
    
    static inline void __list_del(struct list_head * prev, struct list_head * next)
    {
    	    next->prev = prev;
    	    prev->next = next;
    }
    
    static inline void list_del(struct list_head *entry)
    {
    	    __list_del(entry->prev, entry->next);
    //	    entry->next = LIST_POISON1;
    //	    entry->prev = LIST_POISON2;
    }
    static inline void INIT_LIST_HEAD(struct list_head *list)
    {
         list->next = list;
         list->prev = list;
    }
    ////////////////////////////////////////			
    struct book_list{
    	char name[256];
    	int price;
    	struct list_head list;
    };
    static inline struct book_list* new_book()
    {
    	struct book_list *book;
    	book = (struct book_list*)malloc(sizeof(struct book_list));
    	INIT_LIST_HEAD(&book->list);
    	return book;
    }
    
    /////////////////////////////////////////
    void test()
    {
    	struct book_list *books;
    	books = new_book();
    	const int N = 3; 
    	char name[3][64]={
    		"operating system",
    		"linux programming",
    		"algorithms"
    	};
    	int i;
    	for(i=0; i<N; i++)
    	{
    		struct book_list * new = new_book();
    		strcpy(new->name, name[i]);
    		new->price = N*10;
    		list_add(&new->list, &books->list);
    	}
    	struct list_head *p;
    	struct book_list *book;
    	printf("testing list for_each()\n");
    	list_for_each(p, &books->list){
    		book = list_entry(p, struct book_list, list);
    		printf("%s -> %d\n", book->name, book->price);
    	}
    
    	printf("\ntesting list for_each_entry()\n");
    	list_for_each_entry(book, &books->list, list){
    		printf("%s -> %d\n", book->name, book->price);
    	}
    	//free the list
    	for(p = &books->list.next; p != &books->list; p = &books->list.next;)
    	{
    		book = list_entry(p, struct book_list, list);
    		list_del(p);
    		free(book);
    	}
    }
    ////////////////////////////////////////////
    int main()
    {
    	test();
    	return 0;
    }


参考《Linux内核设计与实现》6.1链表
