---
layout: single
author_profile: true
comments: true
title: Netlink 监听 XFRM 状态消息
tagline: 
category: Linux
tags : [Linux, Netlink, libnl]
---

> XFRM是 Linux 2.6 内核为安全处理引入的一个可扩展功能框架，用来在数据包经过路由路径的过程中对其进行修改，包含 3 种数据结构：策略(xfrm policy)，模板(template)和状态(xfrm state)。策略是通过模板和状态发生联系的。 ——http://goo.gl/ek9dJM

> xfrm is an IP framework for transforming packets (such as encrypting
       their payloads). This framework is used to implement the IPsec
       protocol suite (with the state object operating on the Security
       Association Database, and the policy object operating on the Security
       Policy Database). It is also used for the IP Payload Compression
       Protocol and features of Mobile IPv6. ——http://goo.gl/l30ZGn
       
解析XFRM SA/SP消息，可以根据Linux定义的SA/SP结构来解析相关字段
	
	struct xfrm_usersa_info *sa_info = nlmsg_data(nlmsghdr);
	struct xfrm_userpolicy_info *sp_info = nlmsg_data(nlmsghdr);
	
这种方式太过麻烦，下面利用libnl-xfrm-3.0 API做解析工作.

### 安装libnl

ubuntu下使用`apt-get install `安装的 libnl 没有 xfrm 库，所以下面手动安装了开发版libnl。

	root@ubuntu:~/libnl# git clone https://github.com/thom311/libnl
	root@ubuntu:~/libnl# cd libnl
	root@ubuntu:~/libnl# ./autogen.sh
	root@ubuntu:~/libnl# ./configure
	root@ubuntu:~/libnl# make
	root@ubuntu:~/libnl# make install 

检查是否安装成功

	root@ubuntu:~/libnl# pkg-config --cflags --libs libnl-xfrm-3.0 
	-I/usr/local/include/libnl3  -L/usr/local/lib -lnl-xfrm-3 -lnl-3  
	root@ubuntu:~/libnl# pkg-config --cflags --libs libnl-nf-3.0 
	-I/usr/local/include/libnl3  -L/usr/local/lib -lnl-nf-3 -lnl-route-3 -lnl-3  
	root@ubuntu:~/libnl# pkg-config --cflags --libs libnl-route-3.0 
	-I/usr/local/include/libnl3  -L/usr/local/lib -lnl-route-3 -lnl-3  
	root@ubuntu:~/libnl# pkg-config --cflags --libs libnl-genl-3.0 
	-I/usr/local/include/libnl3  -L/usr/local/lib -lnl-genl-3 -lnl-3

### xfrm_listen.c
  
  程序结构类似上篇《[Netlink 监听路由变化消息 ](http://onestraw.net/linux/netlink-route-listen/)》，变化的是netlink协议，广播组，
  而且 xfrm 消息不像 rtmsg那样包含很多属性，XFRM 数据都从头部获取。   
  
- 如果 nlmsghdr->nlmsg_type == XFRM_MSG_NEWSA, nlmsg_data(nlmsghdr)就是  struct xfrm_usersa_info
- 如果 nlmsghdr->nlmsg_type == XFRM_MSG_NEWPOLICY, nlmsg_data(nlmsghdr)就是  struct xfrm_userpolicy_info
  
  此外，程序只对SA进行了解析，依葫芦画瓢，解析SP也很简单，对读者来说也是一个不错的练习。

{% highlight c %}
/*
    gcc xfrm_listen.c `pkg-config --cflags --libs libnl-3.0 libnl-xfrm-3.0`
*/
#include <arpa/inet.h>
#include <linux/netlink.h>
#include <linux/xfrm.h>
#include <netlink/types.h>
#include <netlink/genl/genl.h>
#include <netlink/genl/ctrl.h>
#include <stdio.h>
#include <string.h>

#define PFUNC()  printf("[+%s]\n", __FUNCTION__)
#define NLMSG_TYPE(type)    printf(#type " : %d\n", type)

int parse_nlmsg(struct nl_msg *, void *);
void nlmsg_type_map();
void parse_sa(struct nlmsghdr *nlh);
void parse_sp(struct nlmsghdr *nlh);

int main()
{
	struct nl_sock *sock;
	sock = nl_socket_alloc();

/* broadcast group
#define XFRMGRP_ACQUIRE         1
#define XFRMGRP_EXPIRE          2
#define XFRMGRP_SA              4
#define XFRMGRP_POLICY          8
#define XFRMGRP_REPORT          0x20
*/
	nl_join_groups(sock, XFRMGRP_SA | XFRMGRP_POLICY);

	nl_connect(sock, NETLINK_XFRM);

	nl_socket_modify_cb(sock,
			    NL_CB_MSG_IN, NL_CB_CUSTOM, parse_nlmsg, NULL);

	while (1)
		nl_recvmsgs_default(sock);
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

int parse_nlmsg(struct nl_msg *nlmsg, void *arg)
{
	PFUNC();
	//nlmsg_type_map();
	//nl_msg_dump(nlmsg, stdout);

	struct nlmsghdr *nlhdr;
	int len;
	nlhdr = nlmsg_hdr(nlmsg);
	len = nlhdr->nlmsg_len;

	for (nlhdr; NLMSG_OK(nlhdr, len); nlhdr = NLMSG_NEXT(nlhdr, len)) {
		switch (nlhdr->nlmsg_type) {
		case XFRM_MSG_NEWSA:
		case XFRM_MSG_DELSA:
		case XFRM_MSG_GETSA:
			parse_sa(nlhdr);
			break;
		case XFRM_MSG_NEWPOLICY:
		case XFRM_MSG_DELPOLICY:
		case XFRM_MSG_GETPOLICY:
			parse_sp(nlhdr);
			break;
		}
	}
	return 0;
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

void parse_sa(struct nlmsghdr *nlh)
{
	PFUNC();
	/*
	   libnl/include/netlink/xfrm/sa.h
	 */
	struct xfrmnl_sa *sa = xfrmnl_sa_alloc();
	xfrmnl_sa_parse(nlh, &sa);

	struct xfrmnl_sel *sel = xfrmnl_sa_get_sel(sa);
	struct nl_addr *sel_src = xfrmnl_sel_get_saddr(sel);
	struct nl_addr *sel_dst = xfrmnl_sel_get_daddr(sel);
	char src[16];
	char dst[16];
	nl_addr2str(sel_src, src, 16);
	nl_addr2str(sel_dst, dst, 16);

	struct nl_addr *nlsaddr = xfrmnl_sa_get_saddr(sa);
	struct nl_addr *nldaddr = xfrmnl_sa_get_daddr(sa);
	char saddr[16];
	char daddr[16];
	nl_addr2str(nlsaddr, saddr, 16);
	nl_addr2str(nldaddr, daddr, 16);

/*
    root@ubuntu:~/libnl# cat /etc/protocols |grep -i ipsec
    esp 50  IPSEC-ESP   # Encap Security Payload [RFC2406]
    ah  51  IPSEC-AH    # Authentication Header [RFC2402]
*/
	uint8_t proto = (uint8_t) xfrmnl_sa_get_proto(sa);
	uint32_t spi = (uint32_t) xfrmnl_sa_get_spi(sa);
	uint32_t reqid = xfrmnl_sa_get_reqid(sa);
	int mode = xfrmnl_sa_get_mode(sa);
	char s_mode[32];
	xfrmnl_sa_mode2str(mode, s_mode, 32);
	uint8_t replay_win = xfrmnl_sa_get_replay_window(sa);

	char enc_alg[64];
	char enc_key[1024];
	unsigned int enc_key_len;
	xfrmnl_sa_get_crypto_params(sa, enc_alg, &enc_key_len, enc_key);

	char auth_alg[64];
	char auth_key[1024];
	unsigned int auth_key_len;
	unsigned int auth_trunc_len;
	xfrmnl_sa_get_auth_params(sa, auth_alg, &auth_key_len, &auth_trunc_len,
				  auth_key);
/*
    dump to stdout
*/
	printf(" src : %s\t\t dst : %s\n", saddr, daddr);
	printf(" proto : %d(esp:50 ah:51)\t\tspi : 0x%x \n", proto, spi);
	printf(" repid : %u \t\tmode : %s\n", reqid, s_mode);
	printf(" replay window : %d\n", replay_win);
	printf(" %s \t", auth_alg);
	dump_hex(auth_key, auth_key_len / 8);
	printf(" %s \t", enc_alg);
	dump_hex(enc_key, enc_key_len / 8);
	printf(" sel src : %s\t dst : %s\n", src, dst);

	xfrmnl_sa_put(sa);
}

void parse_sp(struct nlmsghdr *nlh)
{
	PFUNC();
	/*
	   libnl/include/netlink/xfrm/sp.h
	 */
}
{% endhighlight %}



### 测试

我在《[IPSec VPN 简介及实战 ](http://onestraw.net/cybersecurity/ipsec-vpn/)》讲过IPsec VPN基本概念及配置方法，
下面就用一个transport模式的配置实例来测试`xfrm_listen`.     

先在一个shell中执行 `xfrm_listen`，在另一个shell中配置`setkey -f transport_setkey.cfg `，成功的话xfrm_listen会输出如下结果：

    root@ubuntu:~/apac/netlink# ./xfrm_listen 
    [+parse_nlmsg]
    [+parse_nlmsg]
    [+parse_nlmsg]
    [+parse_sa]
     src : 20.20.20.21		 dst : 20.20.20.22
     proto : 50(esp:50 ah:51)		spi : 0x1000 
     repid : 0 		mode : transport
     replay window : 0
     hmac(md5) 	0xca2cef3e4e00a0a111d3aa0048ec1ce1
     cbc(aes) 	0xfd64273b58d4e10af257ee5f7518a5e88a9ae77cf6f5a741
     sel src : 0.0.0.0/0	 dst : 0.0.0.0/0
    [+parse_nlmsg]
    [+parse_sa]
     src : 20.20.20.22		 dst : 20.20.20.21
     proto : 50(esp:50 ah:51)		spi : 0x1100 
     repid : 0 		mode : transport
     replay window : 0
     hmac(md5) 	0xca2cef3e4e00a0a111d3aa0048ec1ce2
     cbc(aes) 	0xfd64273b58d4e10af257ee5f7518a5e88a9ae77cf6f5a742
     sel src : 0.0.0.0/0	 dst : 0.0.0.0/0
    [+parse_nlmsg]
    [+parse_sp]
    [+parse_nlmsg]
    [+parse_sp]
    [+parse_nlmsg]
    [+parse_sp]


`ip xfrm`查询security association 如下所示

<pre>
root@ubuntu:~# ip xfrm state list
src 20.20.20.22 dst 20.20.20.21
	proto esp spi 0x00001100 reqid 0 mode transport
	replay-window 0 
	auth-trunc hmac(md5) 0xca2cef3e4e00a0a111d3aa0048ec1ce2 96
	enc cbc(aes) 0xfd64273b58d4e10af257ee5f7518a5e88a9ae77cf6f5a742
	sel src 0.0.0.0/0 dst 0.0.0.0/0 
src 20.20.20.21 dst 20.20.20.22
	proto esp spi 0x00001000 reqid 0 mode transport
	replay-window 0 
	auth-trunc hmac(md5) 0xca2cef3e4e00a0a111d3aa0048ec1ce1 96
	enc cbc(aes) 0xfd64273b58d4e10af257ee5f7518a5e88a9ae77cf6f5a741
	sel src 0.0.0.0/0 dst 0.0.0.0/0 
</pre>

security policy 如下所示	

<pre>
root@ubuntu:~# ip xfrm policy list
src 20.20.20.22/32 dst 20.20.20.21/32 
	dir fwd priority 2147481648 
	tmpl src 0.0.0.0 dst 0.0.0.0
		proto esp reqid 0 mode transport
src 20.20.20.21/32 dst 20.20.20.22/32 
	dir out priority 2147481648 
	tmpl src 0.0.0.0 dst 0.0.0.0
		proto esp reqid 0 mode transport
</pre>		
		

### libnl-xfrm问题

libnl-xfrm API 设计的还很粗糙，如

- proto 是 uint8_t类型的，而下面却返回 Int

	int xfrmnl_sa_get_proto();
		
- spi 是 uint32_t类型的， 而下面却返回 int

	xfrmnl_sa_get_spi();
		
- 分配 sa struct 用 `xfrmnl_sa_alloc()`，释放却用

	 xfrmnl_sa_put()
		 
