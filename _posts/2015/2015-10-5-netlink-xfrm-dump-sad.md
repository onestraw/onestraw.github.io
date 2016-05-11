---
layout: single
author_profile: true
comments: true
title: Netlink 查询 XFRM 状态消息
tagline: 
category: Linux
tags : [Linux, Netlink]
---

上一篇《[Netlink 监听 XFRM 状态消息](http://onestraw.net/linux/netlink-xfrm-state-listen)》学习了`抓包`和`解包`，本文来学习`构造包`和`发包`

监听是一种被动的方式，实现增删改查（also known as CRUD）需要主动与内核交互，也就是发包给内核“指示”。    

下面详细介绍了`struct nlmsghdr` 和 `sendmsg()`，在此基础上实现了SAD导出功能。

## 构造netlink header
{% highlight c %}
struct nlmsghdr {
    __u32 nlmsg_len;    /* Length of message including header. */
    __u16 nlmsg_type;   /* Type of message content. */
    __u16 nlmsg_flags;  /* Additional flags. */
    __u32 nlmsg_seq;    /* Sequence number. */
    __u32 nlmsg_pid;    /* Sender port ID. */
};
{% endhighlight %}
		   
- nlmsg_len  
	数据总长度，包含头部
	
- nlmsg_type  
	netlink message 类型，这个字段在构造和解析时都很常用，每一种netlink_family属于一大类。  
	NETLINK_ROUTE 对应的是 RTN_XXX的类型   
	NETLINK_XFRM 对应的是 XFRM_XXX的类型
	
	如果要获取内核中SAD信息，需要在nlmsghdr中指定type = XFRM_MSG_GETSA
	
- nlmsg_flags
	该字段比较生疏，调试时，发送的请求一直得不到正确的响应数据，就是因为nlmsg_flags设置错误。  
	请求导出SAD， `NLM_F_REQUEST` 是必须要设置的，但是还不够，还要设置 `NLM_F_DUMP`。   
	所有flags参考
	
	
{% highlight bash %}
  Standard flag bits in nlmsg_flags
  ----------------------------------------------------------
  NLM_F_REQUEST   Must be set on all request messages.
  NLM_F_MULTI     The  message  is part of a multipart mes-
       			   sage terminated by NLMSG_DONE.
  NLM_F_ACK       Request for an acknowledgment on success.
  NLM_F_ECHO      Echo this request.

  Additional flag bits for GET requests
  --------------------------------------------------------------------
  NLM_F_ROOT     Return the complete table instead of a single entry.
  NLM_F_MATCH    Return all entries matching criteria passed in  mes-
       			  sage content.  Not implemented yet.
  NLM_F_ATOMIC   Return an atomic snapshot of the table.
  NLM_F_DUMP     Convenience macro; equivalent to
       			  (NLM_F_ROOT|NLM_F_MATCH).

  Note  that  NLM_F_ATOMIC  requires  the  CAP_NET_ADMIN capability or an
  effective UID of 0.

  Additional flag bits for NEW requests
  ------------------------------------------------------------
  NLM_F_REPLACE   Replace existing matching object.
  NLM_F_EXCL      Don't replace if the object already exists.
  NLM_F_CREATE    Create object if it doesn't already exist.
  NLM_F_APPEND    Add to the end of the object list.
  
  from: http://stuff.onse.fi/man?program=netlink&section=7
{% endhighlight %}

- nlmsg_seq   
	类似于TCP中seq的作用
	 
- nlmsg_pid   
	标识消息的发送者，类似(IP, Port)的作用
		   
##调用sendmsg()

这里发包调用的是 sendmsg   

	ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags);

它和send, sendto的区别在于
		
> For send() and sendto(), the message is found in buf and has length len. For sendmsg(), the message is pointed to by the elements of the array msg.msg_iov. The sendmsg() call also allows sending ancillary data (also known as control information).  ——http://linux.die.net/man/2/sendmsg

一个关键的数据结构 msghdr

{% highlight c %}
/* Structure describing messages sent by
   `sendmsg' and received by `recvmsg'.  */
  struct msghdr
  {
	void *msg_name;             /* Address to send to/receive from.  */
	socklen_t msg_namelen;      /* Length of address data.  */

	struct iovec *msg_iov;      /* Vector of data to send/receive into.  */
	size_t msg_iovlen;          /* Number of elements in the vector.  */

	void *msg_control;          /* Ancillary data (eg BSD filedesc passing). */
	size_t msg_controllen;      /* Ancillary data buffer length.
					   !! The type should be socklen_t but the
					   definition of the kernel is incompatible
					   with this.  */

	int msg_flags;              /* Flags on received message.  */
  };  

struct iovec
{
	void *iov_base; 		/* BSD uses caddr_t (1003.1g requires void *) */
	__kernel_size_t iov_len; 	/* Must be size_t (1003.1g) */
};
{% endhighlight %}
		
iovec 作用？

> uses the iovec structure for scatter/gather I/O.  

什么是 scatter/gather I/O ?

> In computing, vectored I/O, also known as scatter/gather I/O, is a method of input and output by which a single procedure-call sequentially writes data from multiple buffers to a single data stream or reads data from a data stream to multiple buffers. (维基，含有C语言实例)——https://en.wikipedia.org/wiki/Vectored_I/O


## 示例程序

参考`ip xfrm state list`实现源码，实现了一个发包查询`xfrm state (SAD)`功能。 

{% highlight c %}
/*
	gcc xfrm_list.c `pkg-config --cflags --libs libnl-3.0 libnl-xfrm-3.0` -w -g
*/
#include <arpa/inet.h>
#include <linux/netlink.h>
#include <linux/xfrm.h>
#include <netlink/genl/genl.h>
#include <netlink/genl/ctrl.h>
#include <netlink/types.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>

#define RTA_BUF_SIZE 2048
#define PFUNC()  printf("[+%s]\n", __FUNCTION__)
#define NLMSG_TYPE(type)    printf(#type " : %d\n", type)
#define zero(mem)   memset(&mem, 0, sizeof(mem))

int parse_nlmsg(struct nlmsghdr *, int msglen);
void parse_sa(struct nlmsghdr *nlh);
void nlmsg_type_map();
void dump_hex(char *buf, int len);
void dump_nlmsg(char *buf, int len);

int main()
{
	int sock;
	int err;
	int msglen;
	char buf[16384];

	struct sockaddr_nl local;
	struct sockaddr_nl server;
	struct nlmsghdr *nlh;
/*
		request packet
		|   nlmsghdr    |   xfrm_usersa_id  |
*/
	struct {
		struct nlmsghdr n;
		struct xfrm_usersa_id xsid;
		char buf[RTA_BUF_SIZE];
	} req;
	req.n.nlmsg_len = NLMSG_LENGTH(sizeof(req.xsid));
	req.n.nlmsg_flags = NLM_F_ROOT | NLM_F_MATCH | NLM_F_REQUEST;
	req.n.nlmsg_type = XFRM_MSG_GETSA;
	req.n.nlmsg_seq = time(NULL);
	req.n.nlmsg_pid = getpid();

	zero(req.xsid);
	req.xsid.family = AF_INET;
	req.xsid.proto = 0;

	struct iovec iov = {
		.iov_base = (void *)&req.n,
		.iov_len = req.n.nlmsg_len
	};

	struct msghdr msg = {
		.msg_name = &server,
		.msg_namelen = sizeof(server),
		.msg_iov = &iov,
		.msg_iovlen = 1,
	};

	zero(server);
	server.nl_family = AF_NETLINK;

	zero(local);
	local.nl_family = AF_NETLINK;
	local.nl_pid = getpid();

	sock = socket(AF_NETLINK, SOCK_RAW, NETLINK_XFRM);
	if (sock < 0) {
		perror("socket fail.\n");
		return -1;
	}

	err = bind(sock, (struct sockaddr *)&local, sizeof(local));
	if (err < 0) {
		perror("bind fail.\n");
		return -1;
	}

	err = sendmsg(sock, &msg, 0);
	if (err < 0) {
		perror("sendmsg fail.\n");
		return -1;
	}
	perror("sendmsg()");

	zero(buf);
	iov.iov_base = buf;
	iov.iov_len = sizeof(buf);

	while (1) {
		msglen = recvmsg(sock, &msg, 0);
		//dump_nlmsg(buf, msglen);
		parse_nlmsg((struct nlmsghdr *)buf, msglen);
	}
	return 0;
}

void nlmsg_type_map()
{
	/*
	   xfrm message type list
	   libnl/include/linux-private/linux/xfrm.h :152
	 */
	printf("+----------<XFRM_MSG_TYPE : ID>--------+\n");
	NLMSG_TYPE(XFRM_MSG_NEWSA);
	NLMSG_TYPE(XFRM_MSG_DELSA);
	NLMSG_TYPE(XFRM_MSG_GETSA);
	NLMSG_TYPE(XFRM_MSG_NEWPOLICY);
	NLMSG_TYPE(XFRM_MSG_DELPOLICY);
	NLMSG_TYPE(XFRM_MSG_GETPOLICY);
	NLMSG_TYPE(XFRM_MSG_FLUSHSA);
	NLMSG_TYPE(XFRM_MSG_FLUSHPOLICY);
	printf("+--------------------------------------+\n");
}

int parse_nlmsg(struct nlmsghdr *nlh, int msglen)
{
	PFUNC();
	//nlmsg_type_map();
	for (nlh; NLMSG_OK(nlh, msglen); nlh = NLMSG_NEXT(nlh, msglen)) {
		switch (nlh->nlmsg_type) {
		case XFRM_MSG_NEWSA:
		case XFRM_MSG_GETSA:
			parse_sa(nlh);
			break;
		default:
			printf("other nlmsg_type(%d). \nexit\n",
				   nlh->nlmsg_type);
			exit(1);
		}
	}
	return 0;
}

void parse_sa(struct nlmsghdr *nlh)
{
		参考http://onestraw.net/linux/netlink-xfrm-state-listen
}

void dump_nlmsg(char *buf, int len)
{
	int i;
	for (i = 0; i < len; i++) {
		if (i % 16 == 0) {
			printf("\n");
		}
		printf("%02x ", buf[i] & 0xff);
	}
	printf("\n");
}

void dump_hex(char *buf, int len)
{
	int i;
	printf("0x");
	for (i = 0; i < len; i++) {
		printf("%02x", buf[i] & 0xff);
	}
	printf("\n");
}
{% endhighlight %}


现在掌握了 SAD 导出和监听技术，这些都是setkey，ip xfrm 实现的基础，有时间再试验下 `XFRM_MSG_NEWSA`, `XFRM_MSG_DELSA`, `XFRM_MSG_UPDSA` . 

## 参考

-------------

- iproute2:  https://github.com/shemminger/iproute2/blob/master/ip/xfrm_state.c
- 《xfrm interface experiences..》第4页 http://vger.kernel.org/netconf2005_jamal.pdf
