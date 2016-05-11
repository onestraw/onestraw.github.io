---
layout: single
author_profile: true
comments: true
title: Netlink分层模型及消息格式
tagline: 
category: Linux
tags : [Linux, Netlink, libnl]
---

通过`libnl`能够很快的编写一个netlink程序框架，隐藏了socket，bind，send/recv等复杂调用，
但是数据的构造和解析还是很头疼的，尤其对于我这样的初学者来说，下面按TCP/IP分层模型来学学Netlink消息格式。

## Netlink "三层" 模型

下面以netlink**第一、二、三层**类比TCP/IP模型中的**网络层，传输层和应用层**

#### 1. Netlink消息第一层`netlink protocol`，有下面这20种

{% highlight c %}
    #define NETLINK_ROUTE           0       /* Routing/device hook                          */
    #define NETLINK_USERSOCK        2       /* Reserved for user mode socket protocols      */
    #define NETLINK_FIREWALL        3       /* Unused number, formerly ip_queue             */
    #define NETLINK_SOCK_DIAG       4       /* socket monitoring                            */
    #define NETLINK_NFLOG           5       /* netfilter/iptables ULOG */
    #define NETLINK_XFRM            6       /* ipsec */
    #define NETLINK_SELINUX         7       /* SELinux event notifications */
    #define NETLINK_ISCSI           8       /* Open-iSCSI */
    #define NETLINK_AUDIT           9       /* auditing */
    #define NETLINK_FIB_LOOKUP      10      
    #define NETLINK_CONNECTOR       11
    #define NETLINK_NETFILTER       12      /* netfilter subsystem */
    #define NETLINK_IP6_FW          13
    #define NETLINK_DNRTMSG         14      /* DECnet routing messages */
    #define NETLINK_KOBJECT_UEVENT  15      /* Kernel messages to userspace */
    #define NETLINK_GENERIC         16
    #define NETLINK_SCSITRANSPORT   18      /* SCSI Transports */
    #define NETLINK_ECRYPTFS        19
    #define NETLINK_RDMA            20
    #define NETLINK_CRYPTO          21      /* Crypto layer */
{% endhighlight %}

#### 2. Netlink消息第二层，以`NETLINK_ROUTE`为例

 +  LINKS
 +  ADDRESSES
 +  ROUTES
 +  NEIGHBORS
 +  RULES
 +  DISCIPLINES
 +  CLASSES
 +  FILTERS

#### 3. Netlink消息第三层，以`ROUTES` 为例
	
	  封装的ROUTE的属性头部及数据
	

netlink 数据结构可形象的表示为

    <----- NLMSG_HDRLEN ----->                      <-------- RTM_PAYOAD(rtm) ---> 
                                                                  <RTA_PAYLOAD(r)>
    +------------------+- - -+---------------+- - -+--------------+--------+ - -+
    | Netlink Header   | Pad | Family Header | Pad | Attributes   | rtattr | Pad|
    | struct nlmsghdr  |     | struct rtmsg  |     | stuct rtattr |  data  |    |
    +------------------+- - -+---------------+- - -+--------------+--------+ - -+
    ^                        ^                     ^              ^             ^
    nlh                      |                     |              |             |
    NLMSG_DATA(nlh)  --------^                     |              |             |
    RTM_RTA(rtm)-----------------------------------^              |             |
    RTA_DATA(rta)-------------------------------------------------^ RTA_NEXT(rta)
	

`Pad`表示为字节对齐所填充的数据。
	
	
## nlmsghdr

{% highlight c %}
    struct nlmsghdr {
            __u32           nlmsg_len;      /* Length of message including header */
            __u16           nlmsg_type;     /* Message content */
            __u16           nlmsg_flags;    /* Additional flags */
            __u32           nlmsg_seq;      /* Sequence number */
            __u32           nlmsg_pid;      /* Sending process port ID */
    };

    #define NLMSG_ALIGNTO   4U
    #define NLMSG_ALIGN(len) ( ((len)+NLMSG_ALIGNTO-1) & ~(NLMSG_ALIGNTO-1) )
    #define NLMSG_HDRLEN     ((int) NLMSG_ALIGN(sizeof(struct nlmsghdr)))
    #define NLMSG_LENGTH(len) ((len) + NLMSG_HDRLEN)
    #define NLMSG_SPACE(len) NLMSG_ALIGN(NLMSG_LENGTH(len))
    #define NLMSG_DATA(nlh)  ((void*)(((char*)nlh) + NLMSG_LENGTH(0)))
    #define NLMSG_NEXT(nlh,len)      ((len) -= NLMSG_ALIGN((nlh)->nlmsg_len), \
                                      (struct nlmsghdr*)(((char*)(nlh)) + NLMSG_ALIGN((nlh)->nlmsg_len)))
    #define NLMSG_OK(nlh,len) ((len) >= (int)sizeof(struct nlmsghdr) && \
                               (nlh)->nlmsg_len >= sizeof(struct nlmsghdr) && \
                               (nlh)->nlmsg_len <= (len))
    #define NLMSG_PAYLOAD(nlh,len) ((nlh)->nlmsg_len - NLMSG_SPACE((len)))
{% endhighlight %}

- NLMSG_ALIGNTO  
	字节对齐的值，这里按4字节对齐，4U的意思就是 (unsigned int)4。

- NLMSG_ALIGN(len)  
	按4字节对齐的长度，返回字节对齐后的值align_len，len=< align_len <=len+4。

- NLMSG_HDRLEN  
	struct nlmsghdr 所占内存大小的对齐后值
	
- NLMSG_LENGTH(len)  
	nlmsghdr长度加上len
	
- NLMSG_SPACE(len)   
	定义如上，作用参见下面的RTM_PAYLOAD(n)
	
- NLMSG_DATA(nlh)  
	从nlh首地址向后移动到data起始位置
	
- NLMSG_NEXT(nlh, len)  
	使用了逗号表达式完成两件事，先调整剩余长度len，减去当前nlmsg的总长度；再定位到下一个nlmsg的起始位置。
	
- NLMSG_OK(nlh, len)  
	检查nlmsg是否合法，剩余数据长度len要大于nlmsg头部长度，nlh->nlmsg_len 要大于nlmsg头部长度并且小于剩余长度len。
	
- NLMSG_PAYLOAD(nlh, len)  
	nlmsg去掉头部的数据长度。
	
## rtmsg

    struct rtmsg {
            unsigned char           rtm_family;
            unsigned char           rtm_dst_len;
            unsigned char           rtm_src_len;
            unsigned char           rtm_tos;
    
            unsigned char           rtm_table;      /* Routing table id */
            unsigned char           rtm_protocol;   /* Routing protocol; see below  */
            unsigned char           rtm_scope;      /* See below */
            unsigned char           rtm_type;       /* See below    */
    
            unsigned                rtm_flags;
    };
    
    #define RTM_RTA(r)  ((struct rtattr*)(((char*)(r)) + NLMSG_ALIGN(sizeof(struct rtmsg))))
    #define RTM_PAYLOAD(n) NLMSG_PAYLOAD(n,sizeof(struct rtmsg))

- RTM_RTA(r)  
	输入route message指针 struct rtmsg* r，返回route第一个属性首地址

- RTM_PAYLOAD(n)  
	
    	- RTM_PAYLOAD(n) 
    	- NLMSG_PAYLOAD(n,sizeof(struct rtmsg))
    	- ((n)->nlmsg_len - NLMSG_SPACE(sizeof(struct rtmsg)))
    	- ((n)->nlmsg_len - NLMSG_ALIGN(NLMSG_LENGTH(sizeof(struct rtmsg))))
    	- ((n)->nlmsg_len - NLMSG_ALIGN((sizeof(struct rtmsg) + NLMSG_HDRLEN)))
    	- ((n)->nlmsg_len - NLMSG_ALIGN((sizeof(struct rtmsg) + ((int) NLMSG_ALIGN(sizeof(struct nlmsghdr))))))
	
去除字节对齐，，将n替换成 `nlmsghdr*nlh` 可以简化成如下
	
> ((nlh)->nlmsg_len - sizeof(struct rtmsg) - sizeof(struct nlmsghdr))
	
即rtmsg层封装的数据长度，相当于TCP数据包去掉IP报头和TCP报头长度得到TCP数据部分长度。
	
## rtattr

{% highlight c %}
    struct rtattr {
            unsigned short  rta_len;		//整个属性部分的长度，包括头部和数据
            unsigned short  rta_type;	//
    };
  
    /* Macros to handle rtattributes */
    
    #define RTA_ALIGNTO     4
    #define RTA_ALIGN(len) ( ((len)+RTA_ALIGNTO-1) & ~(RTA_ALIGNTO-1) )
    #define RTA_OK(rta,len) ((len) >= (int)sizeof(struct rtattr) && \
                             (rta)->rta_len >= sizeof(struct rtattr) && \
                             (rta)->rta_len <= (len))
    #define RTA_NEXT(rta,attrlen)   ((attrlen) -= RTA_ALIGN((rta)->rta_len), \
                                     (struct rtattr*)(((char*)(rta)) + RTA_ALIGN((rta)->rta_len)))
    #define RTA_LENGTH(len) (RTA_ALIGN(sizeof(struct rtattr)) + (len))
    #define RTA_SPACE(len)  RTA_ALIGN(RTA_LENGTH(len))
    #define RTA_DATA(rta)   ((void*)(((char*)(rta)) + RTA_LENGTH(0)))
    #define RTA_PAYLOAD(rta) ((int)((rta)->rta_len) - RTA_LENGTH(0))
{% endhighlight %}


- RTA_ALIGNTO  
	字节对齐的值，这里按4字节对齐。
	
- RTA_ALIGN(len)  
	输入一个长度len，返回字节对齐后的值align_len，len=< align_len <=len+RTA_ALIGNTO。
	
- RTA_OK(rta,len)  
	输入一个struct rtattr* rta，和整个rtmsg剩余长度len，返回一个bool值，判断一个属性rta是否正确，条件有3个。
	
- RTA_NEXT(rta,attrlen)  
	用了逗号表达式，先对attrlen减去rta属性内容的全部长度，然后返回下一个rtattr的首地址。

- RTA_LENGTH(len)  
	struct rtattr 对齐后内存大小 加上 len。
	
- RTA_SPACE  
	定义如下，目前在代码中没找到用处。
	
- RTA_DATA(rta)  
	返回rta数据的起始位置，即rta位置向后移动一个 对齐的struct rtattr大小。 
	
- RTA_PAYLOAD(rta)  
	返回有效数据的长度。
	

## 参考

-------------

- libnl source code
- http://people.redhat.com/nhorman/papers/netlink.pdf
