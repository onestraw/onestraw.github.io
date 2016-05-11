---
layout: single
author_profile: true
comments: true
title: Linux Netfilter Hook技术
tagline: 深入学习TCP/IP协议栈
category: Linux
tags : [Linux]
---
本文的主要内容包括

1. netfilter和iptables介绍
2. netfilter hook技术
3. 编写netfilter hook函数
4. 监控数据包读写实例


##1.Netfilter and iptables

------------

`Netfilter`是Linux内核中的一个框架，它提供一个标准的接口，通过该接口能够方便的进行不同的网络操作，包括包过滤、网络地址转换和端口转换。Netfilter在内核中提供一组钩子hooks，通过这些hooks，内核模块可以向TCP/IP协议栈注册回调函数。

>	Netfilter is a framework inside the Linux kernel which offers flexibility for various networking-related operations to be implemented in form of customized handlers. 
	Netfilter offers various options for packet filtering, network address translation, and port translation. 
	Netfilter represents a set of hooks inside the Linux kernel, thus it allows specific kernel modules to register callback functions with the kernel's network stack. 

`Iptables`是一个配置IPv4包过滤和NAT的管理工具。
>	iptables is a user-space application program that allows a system administrator to configure the tables provided by the Linux kernel firewall (implemented as different Netfilter modules) and the chains and rules it stores.
	Different kernel modules and programs are currently used for different protocols; iptables applies to IPv4, ip6tables to IPv6, arptables to ARP, and ebtables to Ethernet frames.
	There are five predefined chains (mapping to the five available Netfilter hooks), though a table may not have all chains. Predefined chains have a policy, for example DROP, which is applied to the packet if it reaches the end of the chain.

iptables包含4个表，5个链。其中表是按照对数据包的操作区分的，链是按照不同的Hook点来区分的，表和链实际上是netfilter的两个维度。

4个表:

- filter：一般的过滤功能
- nat:用于nat功能（端口映射，地址映射等）
- mangle:用于对特定数据包的修改
- raw:优先级最高，设置raw时一般是为了不再让iptables做数据包的链接跟踪处理，提高性能

默认表是filter（没有指定表的时候就是filter表）。表的处理优先级：

    raw > mangle > nat > filter

5个链：

- PREROUTING: 数据包进入路由表之前
- INPUT: 通过路由表后目的地为本机
- FORWARDING: 通过路由表后，目的地不为本机
- OUTPUT: 由本机产生，向外转发
- POSTROUTIONG: 发送到网卡接口之前。
           
关于数据包的处理过程，表和链的遍历，请参考[Traversing of tables and chains](http://www.iptables.info/en/structure-of-iptables.html)



##2.Netfilter hook技术

---------------

Netfilter框架为多种协议提供了一套钩子（hooks），用一个

struct list_head nf_hooks[NPROTO][NF_MAX_HOOKS]

二维数组结构存储，一维为协议族，二维为hook点

netfilter提供5个hook点：

1. NF_IP_PRE_ROUTING：刚刚进入网络层的数据包通过此点（刚刚进行完版本号，校验和等检测）， 源地址转换在此点进行；
2. NF_IP_LOCAL_IN：经路由查找后，送往本机的通过此检查点，INPUT包过滤在此点进行；
3. NF_IP_FORWARD：要转发的包通过此检测点，FORWARD包过滤在此点进行；
4. NF_IP_POST_ROUTING：所有马上要通过NIC出去的包通过此检测点，内置的目的地址转换功能（包括地址伪装）在此点进行；
5. NF_IP_LOCAL_OUT：本机进程发出的包通过此检测点，OUTPUT包过滤在此点进行。

通过这些hook点，注册回调函数，可以进行包过滤, NAT, 以及细粒度的安全监控。

##3.编写netfilter hook函数

--------------
####1.编写hook函数

**编写规范（5个参数1个返回值）**

    unsigned int my_hook(unsigned int hooknum,
        struct sk_buff *skb,
        const struct net_device *in,
        const struct net_device *out,
        int (*okfn)(struct sk_buff *))  
    {
        printk("Hello packet! ");
        return NF_ACCEPT;
    }


每个注册的hook函数经过处理后都将返回下列值之一，告知Netfilter核心代码处理结果，以便对报文采取相应的动作：

- NF_ACCEPT：继续正常的报文处理；
- NF_DROP：将报文丢弃；
- NF_STOLEN：由钩子函数处理了该报文，不要再继续传送；
- NF_QUEUE：将报文入队，通常交由用户程序处理；
- NF_REPEAT：再次调用该钩子函数。

####2.注册hook函数

写完hook函数，就可以调用nf_register_hook()向netfilter进行注册挂接。  
关于注册函数nf_register_hook()的源码分析，参考[nf_hook_ops 钩子的注册](http://staff.ustc.edu.cn/~james/linux/netfilter-4.html)	

在调用nf_register_hook()之前，需要填写一个hook options结构

	struct nf_hook_ops {
         struct list_head list;		//挂接到nf_hooks上
         /* User fills in from here down. */
         nf_hookfn       *hook;		//hook函数
         struct module   *owner;	
         void            *priv;		
         u_int8_t        pf;		//协议族
         unsigned int    hooknum;	//hook点
         /* Hooks are ordered in ascending priority. */
         int             priority;	//优先级
	};

注释中说了，用户从第二项开始填写，list在注册时，由内核管理。

`协议族pf`

- PF_INET
- PF_INET6
- PF_ARP
- PF_BRIDGE
- ...

`hook点`

hooknum用于指定挂接的位置（即上面提到的5个hook点）
这5个宏定义在 linux/include/uapi/linux/netfilter_ipv4.h 中，只为了用户空间编程，内核空间不能使用，所以在内核编程中，要么直接使用数字，要么自己定义。
    
    /* IP Hooks */
    /* After promisc drops, checksum checks. */
    #define NF_IP_PRE_ROUTING       0
    /* If the packet is destined for this box. */
    #define NF_IP_LOCAL_IN          1
    /* If the packet is destined for another interface. */
    #define NF_IP_FORWARD           2
    /* Packets coming from a local process. */
    #define NF_IP_LOCAL_OUT         3
    /* Packets about to hit the wire. */
    #define NF_IP_POST_ROUTING      4
    #define NF_IP_NUMHOOKS          5

`优先级`

优先级是有符号的32位数，值越小优先级越高，netfilter预定义了 
   
  	 NF_IP_PRI_FIRST = INT_MIN;  
  	 NF_IP_PRI_CONNTRACK = -200;  
  	 NF_IP_PRI_MANGLE = -150;  
  	 NF_IP_PRI_NAT_DST = -100;   
  	 NF_IP_PRI_FILTER = 0;  
  	 NF_IP_PRI_NAT_SRC = 100;   
  	 NF_IP_PRI_LAST = INT_MAX;  



##4.监控数据包读写实例

----------


分别在NF_IP_LOCAL_IN和NF_IP_LOCAL_OUT这两个hook点注册一个函数，该函数取得报文的IP报头（如果它是IP包的话），输出源IP和目的IP信息。

####hello-packet.c

    #include <linux/module.h>    // included for all kernel modules
    #include <linux/kernel.h>    // included for KERN_INFO
    #include <linux/init.h>      // included for __init and __exit macros
    #include <linux/netfilter.h>
    #include <linux/netfilter_ipv4.h>
    #include <linux/netdevice.h>
    #include <linux/vmalloc.h>
    
    MODULE_LICENSE("GPL");
    MODULE_AUTHOR("geeksword");
    MODULE_DESCRIPTION("A Simple Hello Packet Module");
    
    enum {  NF_IP_PRE_ROUTING,
            NF_IP_LOCAL_IN,
            NF_IP_FORWARD,
            NF_IP_LOCAL_OUT,
            NF_IP_POST_ROUTING,
            NF_IP_NUMHOOKS  };
    
    static struct nf_hook_ops in_nfho;   //net filter hook option struct
    static struct nf_hook_ops out_nfho;   //net filter hook option struct
    
    static void dump_addr(unsigned char *iphdr)
    {
            int i;
            for(i=0; i<4; i++){
                    printk("%d.", *(iphdr+12+i));   
            }
            printk(" -> ");
            for(i=0; i<4; i++){
                    printk("%d.", *(iphdr+16+i));   
            }
            printk("\n");
    }
    
    unsigned int my_hook(unsigned int hooknum,
        struct sk_buff *skb,
        const struct net_device *in,
        const struct net_device *out,
        int (*okfn)(struct sk_buff *))  
    {
        printk("Hello packet! ");
        //printk("from %s to %s\n", in->name, out->name);
        unsigned char *iphdr = skb_network_header(skb);
        if(iphdr){
            dump_addr(iphdr);
        }
        return NF_ACCEPT;
    }
    
    static int init_filter_if(void)
    {
    //NF_IP_PRE_ROUTING hook
      in_nfho.hook = my_hook;
      in_nfho.hooknum = NF_IP_LOCAL_IN;
      in_nfho.pf = PF_INET;
      in_nfho.priority = NF_IP_PRI_FIRST;
    
      nf_register_hook(&in_nfho);
    
    //NF_IP_LOCAL_OUT hook
      out_nfho.hook = my_hook;
      out_nfho.hooknum = NF_IP_LOCAL_OUT;
      out_nfho.pf = PF_INET;
      out_nfho.priority = NF_IP_PRI_FIRST;
    
      nf_register_hook(&out_nfho);
      return 0;
    }
    
    static int hello_init(void)
    {
        printk(KERN_INFO "[+] Register Hello_Packet module!\n");
        init_filter_if();
        return 0;    // Non-zero return means that the module couldn't be loaded.
    }
    
    static void hello_exit(void)
    {
      nf_unregister_hook(&in_nfho);
      nf_unregister_hook(&out_nfho); 
      printk(KERN_INFO "Cleaning up Helllo_Packet module.\n");
    }
    
    module_init(hello_init);
    module_exit(hello_exit);

####download

- [源码下载](https://github.com/onestraw/kernel-module-fun/blob/master/hello-packet.c)
- [more kernel module example](https://github.com/onestraw/kernel-module-fun)
