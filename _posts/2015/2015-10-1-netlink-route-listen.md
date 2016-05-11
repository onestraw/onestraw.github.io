---
layout: single
author_profile: true
comments: true
title: Netlink 监听路由变化消息
tagline: 
category: Linux
tags : [Linux, Netlink, libnl]
---

绑定`NETLINK_ROUTE`协议，加入内核提供的`RTMGRP_IPV4_ROUTE `广播组，接收监听路由变化消息。

使用 libnl 编写主程序，对于`libnl-route`是否提供对rtmsg进行解析的API？尚未研究。本文根据上一篇文章《[Netlink分层模型及消息格式 ](http://onestraw.net/linux/netlink-message/)》一步一步解析netlink message。

- 创建 nl_sock  

	sock = nl_socket_alloc();
	
- 加入广播组 RTMGRP_IPV4_ROUTE  

	nl_join_groups(sock, RTMGRP_IPV4_ROUTE);
	
- 创建并绑定真实的socket

	nl_connect(sock, NETLINK_ROUTE);
		
nl_join_groups()必须在nl_connect()之前调用，因为nl_connect()封装了socket()和bind()，而真正的加入广播组是在bind()本地地址时完成的。  
	
- 设置回调函数callback （nlmsg处理函数）

	nl_socket_modify_cb(sock, NL_CB_MSG_IN, NL_CB_CUSTOM, parse_nlmsg, NULL);

该函数会将 callback函数parse_nlmsg注册到nl_sock->s_cb->cb_set[]上 .   
`NL_CB_MSG_IN`  表示每收到一个数据包，都会调用callback函数   
`NL_CB_CUSTOM`  表示收到数据包后，调用用户指定的回函调用（parse_nlmsg()即这里定义的回调函数），如果使用`NL_CB_DEBUG`，回调函数会设置成 `nl_msg_dump(nlmsg, stdout);` （实际上是nl_msg_in_handler_debug() ）   输出数据包基本信息及原始数据。  
	

函数原型如下
{% highlight c %}
/**
 * Modify the callback handler associated with the socket
 * @arg sk              Netlink socket.
 * @arg type            which type callback to set
 * @arg kind            kind of callback
 * @arg func            callback function
 * @arg arg             argument to be passed to callback function
 *
 * @see nl_cb_set
 */
int nl_socket_modify_cb(struct nl_sock *sk, enum nl_cb_type type,
						enum nl_cb_kind kind, nl_recvmsg_msg_cb_t func,
						void *arg)
{
		return nl_cb_set(sk->s_cb, type, kind, func, arg);
}
{% endhighlight %}
		
		
回调函数类型及含义如下


{% highlight c %}
/**
 * Callback types
 * @ingroup cb
 */
enum nl_cb_type {
		/** Message is valid */
		NL_CB_VALID,
		/** Last message in a series of multi part messages received */
		NL_CB_FINISH,
		/** Report received that data was lost */
		NL_CB_OVERRUN,
		/** Message wants to be skipped */
		NL_CB_SKIPPED,
		/** Message is an acknowledge */
		NL_CB_ACK,
		/** Called for every message received */
		NL_CB_MSG_IN,
		/** Called for every message sent out except for nl_sendto() */
		NL_CB_MSG_OUT,
		/** Message is malformed and invalid */
		NL_CB_INVALID,
		/** Called instead of internal sequence number checking */
		NL_CB_SEQ_CHECK,
		/** Sending of an acknowledge message has been requested */
		NL_CB_SEND_ACK,
		/** Flag NLM_F_DUMP_INTR is set in message */
		NL_CB_DUMP_INTR,
		__NL_CB_TYPE_MAX,
};
{% endhighlight %}

回调函数属性

{% highlight c %}
/**
 * Callback kinds
 * @ingroup cb
 */
enum nl_cb_kind {
		/** Default handlers (quiet) */
		NL_CB_DEFAULT,
		/** Verbose default handlers (error messages printed) */
		NL_CB_VERBOSE,
		/** Debug handlers for debugging */
		NL_CB_DEBUG,
		/** Customized handler specified by the user */
		NL_CB_CUSTOM,
		__NL_CB_KIND_MAX,
}; 
{% endhighlight %}

nl_socket_modify_cb 最终在调用过程中的nl_cb_set函数中，有如下设置
	
{% highlight c %}
if (kind == NL_CB_CUSTOM) {
        cb->cb_set[type] = func;
        cb->cb_args[type] = arg;
} else {
        cb->cb_set[type] = cb_def[type][kind];
        cb->cb_args[type] = arg;
} 		
{% endhighlight %}

- 接收消息

>	nl_recvmsgs_default(sock);

	阻塞，每到来一个消息数据包，就调用nl_sock->s_cb中注册好的callback函数。
	
- 解析nl_msg，分如下三个层次

	int parse_nlmsg(struct nl_msg *, void *); 
	void parse_rtmsg(struct nlmsghdr *nlhdr);
	void parse_rtm_rtattr(struct rtattr *rta, int len);
		
- 代码实例 

{% highlight c %}
/*
    gcc rt_listen.c `pkg-config --cflags --libs libnl-3.0` 
*/
#include <arpa/inet.h>
#include <linux/netlink.h>
#include <linux/rtnetlink.h>
#include <netlink/genl/genl.h>
#include <netlink/genl/ctrl.h>
#include <stdio.h>
#include <string.h>

int parse_nlmsg(struct nl_msg *, void *);
void parse_rtmsg(struct nlmsghdr *nlhdr);
void parse_rtm_type(struct rtmsg *rtm);
void parse_rtm_scope(struct rtmsg *rtm);
void parse_rtm_table(struct rtmsg *rtm);
void parse_rtm_rtattr(struct rtattr *rta, int len);

int main()
{
	struct nl_sock *sock;
	sock = nl_socket_alloc();

	nl_join_groups(sock, RTMGRP_IPV4_ROUTE);

	nl_connect(sock, NETLINK_ROUTE);

	nl_socket_modify_cb(sock, NL_CB_MSG_IN, 
                            NL_CB_CUSTOM, 
                            //NL_CB_DEBUG, 
                            //NL_CB_DEFAULT, 
                            //NL_CB_VERBOSE, 
                            parse_nlmsg, NULL);

	while (1)
		nl_recvmsgs_default(sock);
	return 0;
}

int parse_nlmsg(struct nl_msg *nlmsg, void *arg)
{
	printf("[+%s]\n", __FUNCTION__);
//	nl_msg_dump(nlmsg, stdout);

	struct nlmsghdr *nlhdr;
	struct rtmsg *rtm;
	int len;
	nlhdr = nlmsg_hdr(nlmsg);
	len = nlhdr->nlmsg_len;	// + NLMSG_HDRLEN;

	for (nlhdr; NLMSG_OK(nlhdr, len); nlhdr = NLMSG_NEXT(nlhdr, len)) {
		switch (nlhdr->nlmsg_type) {
		case RTM_NEWROUTE:
			printf("RTM_NEWROUTE\n");
			parse_rtmsg(nlhdr);
			break;
		case RTM_DELROUTE:
			printf("RTM_DELROUTE\n");
			parse_rtmsg(nlhdr);
			break;
		default:
			printf("nlmsg_type:%d\n\n", nlhdr->nlmsg_type);
			break;
		}
	}
	return 0;
}

void parse_rtmsg(struct nlmsghdr *nlhdr)
{
	struct rtmsg *rtm = (struct rtmsg *)NLMSG_DATA(nlhdr);
	parse_rtm_type(rtm);
	parse_rtm_scope(rtm);
	parse_rtm_table(rtm);
	parse_rtm_rtattr(RTM_RTA(rtm), RTM_PAYLOAD(nlhdr));
}


#define PTYPE(type)    \
	printf("\ttype : %s\n", type)

void parse_rtm_type(struct rtmsg *rtm)
{
	switch (rtm->rtm_type) {
	case RTN_UNICAST:
		PTYPE("unicast");
                break;
	case RTN_UNSPEC:
		PTYPE("specified");
		break;
	case RTN_BROADCAST:
		PTYPE("broadcast");
		break;
	case RTN_LOCAL:
		PTYPE("local");
		break;
	case RTN_NAT:
		PTYPE("NAT");
		break;
	}
}

#define PSCOPE(scope)  \
        printf("\tscope : %s\n", scope)

void parse_rtm_scope(struct rtmsg *rtm)
{
	switch (rtm->rtm_scope) {
	case RT_SCOPE_UNIVERSE:
		PSCOPE("global");
		break;
	case RT_SCOPE_SITE:
		PSCOPE("AS local");
		break;
	case RT_SCOPE_LINK:
		PSCOPE("link local");
		break;
	case RT_SCOPE_HOST:
		PSCOPE("host local");
		break;
	case RT_SCOPE_NOWHERE:
		PSCOPE("none (no destination)");
		break;
	}
}

#define PTABLE(table)  \
        printf("\trouting table : %s\n", table)

void parse_rtm_table(struct rtmsg *rtm)
{
	switch (rtm->rtm_table) {
	case RT_TABLE_UNSPEC:
                PTABLE("unspecified");
		break;
	case RT_TABLE_DEFAULT:
                PTABLE("default");
		break;
	case RT_TABLE_MAIN:
                PTABLE("main");
		break;
	case RT_TABLE_LOCAL:
                PTABLE("local");
		break;
	}
}

#define PADDR(str, addr)   \
        printf("\t %s : %s", str, \
        inet_ntoa((*(struct in_addr *)RTA_DATA(rta))));

#define PIF(str, rta)    \
        printf("\t %s : %u", str, \
        (*(uint32_t *)RTA_DATA(rta)));

void parse_rtm_rtattr(struct rtattr *rta, int len)
{
	for (rta; len > 0 && RTA_OK(rta, len); rta = RTA_NEXT(rta, len)) {
		switch (rta->rta_type) {
		case RTA_GATEWAY:
			PADDR("gateway", rta);
			break;
		case RTA_DST:
			PADDR("destination", rta);
			break;
		case RTA_SRC:
			PADDR("source", rta);
			break;
		case RTA_IIF:
			PIF("input interface", rta);
			break;
		case RTA_OIF:
			PIF("output interface", rta);
			break;
		}
	}
	printf("\n");
}

{% endhighlight %}


- 测试   

编译上述代码，执行。用类似下面的命令增加删除路由（必须保证执行成功，即路由表变化，才能收到netlink广播消息）
	
	route add -net 30.30.30.0 netmask 255.255.255.0 gw 20.20.20.21
	route del -net 30.30.30.0 netmask 255.255.255.0 gw 20.20.20.21

- 参考  

> https://github.com/rcatolino/nlroute-state-watch
