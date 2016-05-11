---
layout: single
author_profile: true
comments: true
title: Hugepages 减少 TLB miss
tagline: 
category: essay
tags : [DPDK]
---

**Hugepages是DPDK使用的一项技术，它为什么能提高包处理的速度呢？**   
正确答案是：使用huge page(如ubuntu14 下2M page)代替传统的4K page，减少VPN (Virtual page number)数目，
进而减少TLB（快表）管理的VPN到PPN(Physical page number)的映射项，在TLB缓存一定的情况下，TLB命中的概率增大，所以就减少了TLB miss，也就提升了性能。

**虚拟地址到物理地址映射**

先来看看CPU寻址的过程，CPU执行一条指令
    
    addl (%ecx), %ebx

寄存器ecx存储一个变量的地址，这条指令作用是将ecx指向地址的数值加到ebx上。  
这里就涉及到`将ecx的值（虚拟地址）转换成物理地址，并物理地址中取出数值`。     

kernel将内存划分成page来管理，每个page一般是4KB大小.   

对于一个32位的虚拟地址，前10位表示页目录偏移PD offset，中间10位表示页偏移PF offset，后12位表示页内偏移 Frame offset。CR3存储着当前页目录基址。假设虚拟地址为x，物理地址的形式化计算方法如下：

    x的物理地址 = ((x>>22 + CR3) + ((x>>12) & 0x3ff)) + (x & 0x3ff);


更详细介绍参考[Linux Memory Management](http://www.tldp.org/HOWTO/KernelAnalysis-HOWTO-7.html)

**什么是TLB**   

> TLB: translation lookaside buffer，传输后备缓冲器是一个内存管理单元用于改进虚拟地址到物理地址转换速度的缓存。

TLB通俗的称为`快表`，它存储着最近的虚拟地址到物理地址的映射，也就是一个索引缓存。
准确的说TLB缓存着Virtual page number (VPN)到 Physical page number (PPN) 的映射，PPN和 page offset（虚拟地址中的最后几位）构成物理地址。  

简单来说，  

- 虚拟地址包含`VPN + 页内偏移`，每一页增大，页内偏移所占位数增加，VPN位数减少，VPN数量变少。  
- TLB包含`VPN,PPN,...`，VPN数量变少，TLB所容纳的条目一定，TLB命中的概率增大，所以就减少了TLB miss。  
   
更详细的内容参见[Virtual Memory](http://cseweb.ucsd.edu/classes/fa10/cse240a/pdf/08/CSE240A-MBT-L18-VirtualMemory.ppt.pdf)


**More**

- [分页表 ](https://zh.wikipedia.org/wiki/%E5%88%86%E9%A0%81%E8%A1%A8)
- [kernel doc](https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt)
