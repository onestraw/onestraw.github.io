---
layout: single
author_profile: true
comments: true
title: libnl从内核获取taskstats信息
tagline: 
category: Linux
tags : [Linux, Netlink, libnl]
---

> Taskstats is a netlink-based interface for sending per-task and per-process statistics from the kernel to userspace. --https://goo.gl/aTdgpp

Taskstats接口提供两种方式

- 获取指定进程的统计数据。（用户态程序提供pid号）
- 当进程终止时，获取其统计数据。（用户态程序提供一个cpumask）

详细的接口说明本文不再赘述，请参考[kernel doc](https://goo.gl/aTdgpp)。本文目的在于通过Taskstats接口学习libnl编程方法。

## 基于libnl的用户态程序

- 创建一个nl_sock

		struct nl_sock *sock = nl_socket_alloc()
		
	其中并没有调用socket系统调用，主要是分配一个nl_sock结构体，填充一些字段，如`nl_family = AF_NETLINK`
	
- 创建socket并绑定

		genl_connect(sock)
	
	该函数才调用了socket(), bind()
	
		sk->s_fd = socket(AF_NETLINK, SOCK_RAW | flags, protocol)
		
	从socket类型可以看出这是无连接的，内核怎么知道哪个进程需要监听taskstats信息呢？   
	**发包告诉内核**
	
- 构造nl_msg

		struct nl_msg
		{
				int                     nm_protocol;
				int                     nm_flags;
				struct sockaddr_nl      nm_src;
				struct sockaddr_nl      nm_dst;
				struct ucred            nm_creds;
				struct nlmsghdr *       nm_nlh;
				size_t                  nm_size;
				int                     nm_refcnt;
		};
		struct nlmsghdr {
			__u32           nlmsg_len;      /* Length of message including header */
			__u16           nlmsg_type;     /* Message content */
			__u16           nlmsg_flags;    /* Additional flags */
			__u32           nlmsg_seq;      /* Sequence number */
			__u32           nlmsg_pid;      /* Sending process port ID */
		};

		struct nlattr {
				__u16           nla_len;
				__u16           nla_type;
		};

	nlmsg_alloc分配nl_msg内存和及default_msg_size内存
default_msg_size = getpagesize();
4KB

在`nlmsg_alloc`中为nl_msg->nm_nlh分配了4KB大小 的内存，nlmsghdr在4KB头部，nlmsghdr, nlattr, data的关系可以参考下图  

<pre>
      <------- NLMSG_ALIGN(hlen) ------> <---- NLMSG_ALIGN(len) --->
     +----------------------------+- - -+- - - - - - - - - - -+- - -+
     |           Header           | Pad |       Payload       | Pad |
     |      struct nlmsghdr       |     |                     |     |
     +----------------------------+- - -+- - - - - - - - - - -+- - -+

      <-------- GENL_HDRLEN -------> <--- hdrlen -->
                                     <------- genlmsg_len(ghdr) ------>
     +------------------------+- - -+---------------+- - -+------------+
     | Generic Netlink Header | Pad | Family Header | Pad | Attributes |
     |    struct genlmsghdr   |     |               |     |            |
     +------------------------+- - -+---------------+- - -+------------+
     genlmsg_data(ghdr)--------------^                     ^
     genlmsg_attrdata(ghdr, hdrlen)-------------------------

</pre>


[link](http://libnl.sourcearchive.com/documentation/1.1/group__genl.html)   


- 发送nl_msg

		nl_send_auto_complete(sock, msg);
		
	实际上是调用的nl_sock的callback函数  `cb_send_ow`，这一步就让内核知道你这个进程想要获取进程数据，下面就等着内核给你数据就OK了。
	
	
- 注册nl_msg处理函数

		nl_socket_modify_cb(sock, NL_CB_MSG_IN, NL_CB_CUSTOM, msg_recv_cb, NULL);
	
	用pcap抓过包的同学都知道pcap_loop注册数据包处理函数然后进入循环抓包分析过程。
	
- 接收nl_msg

		nl_recvmsgs_default(sock);
		
	该函数只是接收一个包，对于持续监听终止进程的统计数据，要自己写个死循环。


**下面是完整的代码   **

{% highlight c %}
/*
 gcc read_taskstats.c `pkg-config --cflags --libs libnl-3.0 libnl-genl-3.0` -o read_taskstats
*/
#include <stdlib.h>
#include <linux/taskstats.h>
#include <netlink/netlink.h>
#include <netlink/genl/genl.h>
#include <netlink/genl/ctrl.h>

void usage(int argc, char *argv[]);
int msg_recv_cb(struct nl_msg *, void *);

int main(int argc, char *argv[])
{
	struct nl_sock *sock;
	struct nl_msg *msg;
	int family;
	int pid = -1;
	char *cpumask;

	if (argc > 2 && strcmp(argv[1], "--pid") == 0) {
		pid = atoi(argv[2]);
	} else if (argc > 2 && strcmp(argv[1], "--cpumask") == 0) {
		cpumask = argv[2];
	} else {
		usage(argc, argv);
		exit(1);
	}

	sock = nl_socket_alloc();

	// Connect to generic netlink socket on kernel side
	genl_connect(sock);

	// get the id for the TASKSTATS generic family
	family = genl_ctrl_resolve(sock, "TASKSTATS");

	// register for task exit events
	msg = nlmsg_alloc();

	genlmsg_put(msg, NL_AUTO_PID, NL_AUTO_SEQ, family, 0, NLM_F_REQUEST,
		    TASKSTATS_CMD_GET, TASKSTATS_VERSION);
	if (pid > 0) {
		nla_put_u32(msg, TASKSTATS_CMD_ATTR_PID, pid);
	} else {
		nla_put_string(msg, TASKSTATS_CMD_ATTR_REGISTER_CPUMASK,
			       cpumask);
	}
	nl_send_auto_complete(sock, msg);
	nlmsg_free(msg);

	// specify a callback for inbound messages
	nl_socket_modify_cb(sock, NL_CB_MSG_IN, NL_CB_CUSTOM, msg_recv_cb,
			    NULL);
	if (pid > 0) {
		nl_recvmsgs_default(sock);
	} else {
		while (1)
			nl_recvmsgs_default(sock);
	}
	return 0;
}

void usage(int argc, char *argv[])
{
	printf("USAGE: %s option\nOptions:\n"
	       "\t--pid pid : get statistics during a task's lifetime.\n"
	       "\t--cpumask mask : obtain statistics for tasks which are exiting. \n"
	       "\t\tThe cpumask is specified as an ascii string of comma-separated \n"
	       "\t\tcpu ranges e.g. to listen to exit data from cpus 1,2,3,5,7,8\n"
	       "\t\tthe cpumask would be '1-3,5,7-8'.\n", argv[0]);
}

#define printllu(str, value)    printf("%s: %llu\n", str, value)

int msg_recv_cb(struct nl_msg *nlmsg, void *arg)
{
	struct nlmsghdr *nlhdr;
	struct nlattr *nlattrs[TASKSTATS_TYPE_MAX + 1];
	struct nlattr *nlattr;
	struct taskstats *stats;
	int rem;

	nlhdr = nlmsg_hdr(nlmsg);

	// validate message and parse attributes
	genlmsg_parse(nlhdr, 0, nlattrs, TASKSTATS_TYPE_MAX, 0);

	if (nlattr = nlattrs[TASKSTATS_TYPE_AGGR_PID]) {
		stats = nla_data(nla_next(nla_data(nlattr), &rem));

		printf("---\n");
		printf("pid: %u\n", stats->ac_pid);
		printf("command: %s\n", stats->ac_comm);
		printf("status: %u\n", stats->ac_exitcode);
		printf("time:\n");
		printf("  start: %u\n", stats->ac_btime);

		printllu("  elapsed", stats->ac_etime);
		printllu("  user", stats->ac_utime);
		printllu("  system", stats->ac_stime);
		printf("memory:\n");
		printf("  bytetime:\n");
		printllu("    rss", stats->coremem);
		printllu("    vsz", stats->virtmem);
		printf("  peak:\n");
		printllu("    rss", stats->hiwater_rss);
		printllu("    vsz", stats->hiwater_vm);
		printf("io:\n");
		printf("  bytes:\n");
		printllu("    read", stats->read_char);
		printllu("    write", stats->write_char);
		printf("  syscalls:\n");
		printllu("    read", stats->read_syscalls);
		printllu("    write", stats->write_syscalls);
	}
	return 0;
}
{% endhighlight %}

-----------

Tip 1.   

> 发现一个带有函数调用图的源码阅读网站http://sourcecodebrowser.com，体验一下，还是cscope+ctags方便。  

