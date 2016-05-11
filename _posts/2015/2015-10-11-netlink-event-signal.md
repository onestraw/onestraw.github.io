---
layout: single
author_profile: true
comments: true
title: Netlink 基于事件的信号机制
tagline: 
category: Linux
tags : [Linux, Netlink]
---


> Event-based signaling mechanisms: they allow to deliver events so that user-space processes do not have to poll for data to make sure that have up-to-date information on some kernel-space aspect. 

link: [Communicating between the kernel and user-space in Linux using Netlink sockets](http://1984.lsi.us.es/~pablo/docs/spae.pdf)  


之前写的《[Netlink 监听 XFRM 状态消息](http://onestraw.net/linux/netlink-xfrm-state-listen/)》 就用到了事件信号通知机制。 

- 用户态进程阻塞在recvmsgs()上，等待内核消息；
- 当`NEWSA`事件发生时会通知所有监听该事件的用户态进程；
- 用户态进程的recvmsgs() 收到数据返回；

下面以`NEWSA` 事件为例分析下内核中Netlink处理的流程。以消息类型`XFRM_MSG_NEWSA`为线索在内核源码中穿梭。  

## xfrm_add_sa

-----
通过查找`XFRM_MSG_NEWSA`，快速发现了xfrm_add_sa()函数和km_event.

{% highlight c %}
//	linux/net/xfrm/xfrm_user.c
static int xfrm_add_sa(struct sk_buff *skb, struct nlmsghdr *nlh,
			struct nlattr **attrs)
{
	struct net *net = sock_net(skb->sk);
	struct xfrm_usersa_info *p = nlmsg_data(nlh);
	struct xfrm_state *x;
	struct km_event c;

	err = verify_newsa_info(p, attrs);

	x = xfrm_state_construct(net, p, attrs, &err);

	xfrm_state_hold(x);
	if (nlh->nlmsg_type == XFRM_MSG_NEWSA)
			err = xfrm_state_add(x);

	c.seq = nlh->nlmsg_seq;
	c.portid = nlh->nlmsg_pid;
	c.event = nlh->nlmsg_type;

	km_state_notify(x, &c); 
}
{% endhighlight %}

从函数名来看，xfrm_add_sa 是将XFRM_MSG_NEWSA类型的netlink message解析并添加到内核的SAD中，最后将该事件广播出去。  

- 先看是谁调用的xfrm_add_sa()
- 再分析km_state_notify()

## 从netlink_socket创建到调用xfrm_add_sa

------------------
我是逆向查找的xfrm_add_sa()调用过程，为了便于查看，我把结果给“正”了过来。

- 创建netlink socket

{% highlight c %}
    static int __net_init xfrm_user_net_init(struct net *net)
    {
            struct sock *nlsk;
            struct netlink_kernel_cfg cfg = {
                    .groups = XFRMNLGRP_MAX,
                    .input  = xfrm_netlink_rcv,
            };   
    
            nlsk = netlink_kernel_create(net, NETLINK_XFRM, &cfg);
            if (nlsk == NULL)
                    return -ENOMEM;
            net->xfrm.nlsk_stash = nlsk; /* Don't set to NULL */
            rcu_assign_pointer(net->xfrm.nlsk, nlsk);
            return 0;
    }
{% endhighlight %}

注意 socket注册的Input处理函数，忽略groups

- xfrm_netlink_rcv

消息类型与处理函数通过 `xfrm_link` 建立了映射

{% highlight c %}
    static void xfrm_netlink_rcv(struct sk_buff *skb)
    {
            netlink_rcv_skb(skb, &xfrm_user_rcv_msg);
    }
    
    struct xfrm_link xfrm_dispatch[] = {
    	[XFRM_MSG_NEWSA       - XFRM_MSG_BASE] = { .doit = xfrm_add_sa        },
    	...
    };
    
    static int xfrm_user_rcv_msg(struct sk_buff *skb, struct nlmsghdr *nlh)
    {
            struct net *net = sock_net(skb->sk);
            struct nlattr *attrs[XFRMA_MAX+1];
            const struct xfrm_link *link;
    
            type = nlh->nlmsg_type;
    
            type -= XFRM_MSG_BASE;
            link = &xfrm_dispatch[type];
    
            err = nlmsg_parse(nlh, xfrm_msg_min[type], attrs, XFRMA_MAX,
                              xfrma_policy);
    
            return link->doit(skb, nlh, attrs);
    }
{% endhighlight %}
    

## km_state_notify()分析

-------------

参数包含了xfrm_state 的所有信息，保证了监听程序能得到所有的SA信息

{% highlight c %}
    void km_state_notify(struct xfrm_state *x, const struct km_event *c)
    {
            struct xfrm_mgr *km; 
            rcu_read_lock();
            list_for_each_entry_rcu(km, &xfrm_km_list, list)
                    if (km->notify)
                            km->notify(x, c);
            rcu_read_unlock();
    }
{% endhighlight %}

- xfrm_km_list 的构造

事件类型和信号函数通过 `xfrm_mgr` 关联起来

{% highlight c %}
    static struct xfrm_mgr netlink_mgr = {
          .id             = "netlink",
          .notify         = xfrm_send_state_notify,
          .acquire        = xfrm_send_acquire,
          .compile_policy = xfrm_compile_policy,
          .notify_policy  = xfrm_send_policy_notify,
          .report         = xfrm_send_report,
          .migrate        = xfrm_send_migrate,
          .new_mapping    = xfrm_send_mapping,
    };

    static int __init xfrm_user_init(void)
    {
          rv = xfrm_register_km(&netlink_mgr);
    }
    
    static LIST_HEAD(xfrm_km_list);

    int xfrm_register_km(struct xfrm_mgr *km)
    {
            list_add_tail_rcu(&km->list, &xfrm_km_list);
    }
{% endhighlight %}


- xfrm_send_state_notify()


{% highlight c %}
    static int xfrm_send_state_notify(struct xfrm_state *x, const struct km_event *c)
    {
          switch (c->event) {
              case XFRM_MSG_NEWSA:
                  return xfrm_notify_sa(x, c);
    	  	}
    }

    static int xfrm_notify_sa(struct xfrm_state *x, const struct km_event *c)
    {
        struct net *net = xs_net(x);
        struct xfrm_usersa_info *p;
        struct xfrm_usersa_id *id; 
        struct nlmsghdr *nlh;
        struct sk_buff *skb;

        skb = nlmsg_new(len, GFP_ATOMIC);
   
        err = copy_to_user_state_extra(x, p, skb);

        return nlmsg_multicast(net->xfrm.nlsk, skb, 0, XFRMNLGRP_SA, GFP_ATOMIC);
    }
{% endhighlight %}


- nlmsg_multicast

    - nlmsg_multicast
    - netlink_broadcast
    - netlink_broadcast_filtered
    - do_one_broadcast

如何判断一个socket是不是加入了广播组呢？

{% highlight c %}
      static int do_one_broadcast(struct sock *sk, 
                                  struct netlink_broadcast_data *p)
      {
          struct netlink_sock *nlk = nlk_sk(sk);
  
          if (nlk->portid == p->portid || p->group - 1 >= nlk->ngroups ||
              !test_bit(p->group - 1, nlk->groups))
                  goto out; 
        out:
            return 0;
      }
{% endhighlight %}


## test_bit()

-------------

{% highlight c %}
      static inline int test_bit(int nr, const volatile void * addr)
      {
          return (1UL & (((const int *) addr)[nr >> 5] >> (nr & 31))) != 0UL;
      }
{% endhighlight %}


addr指向一个32位整数数组

> nr>>5 即  nr/32

所以`((const int *) addr)[nr >> 5]`

表示定位数组addr中第 nr/32 个元素.   

> nr & 31 即 nr%32

所以 `(((const int *) addr)[nr >> 5] >> (nr & 31))`  

表示数组addr中第 nr/32 个元素的 第 nr%32 位 .   

所以test_bit(nr, addr)的作用是  

**判断数组addr中第 nr/32 个元素的第 nr%32 位是否为1**  

test_bit()搞明白了，其中第一个参数`p->group`也知道，它此处就是`XFRMGRP_SA`;   

但是 `nlk->groups` 是什么呢？ 还得回到 `netlink_broadcast_filtered`中



## netlink广播组

---------

{% highlight c %}
    struct netlink_table {
        struct nl_portid_hash   hash;
        struct hlist_head       mc_list;
        struct listeners __rcu  *listeners;
        unsigned int            flags;
        unsigned int            groups;
        struct mutex            *cb_mutex;
        struct module           *module;
        void                    (*bind)(int group);
        bool                    (*compare)(struct net *net, struct sock *sock);
        int                     registered;
    };
    
    struct netlink_table *nl_table;
    EXPORT_SYMBOL_GPL(nl_table);

    int netlink_broadcast_filtered(struct sock *ssk, struct sk_buff *skb, u32 portid,
            u32 group, gfp_t allocation,
            int (*filter)(struct sock *dsk, struct sk_buff *skb, void *data),
            void *filter_data)
    {
        struct net *net = sock_net(ssk);
        struct netlink_broadcast_data info;
        struct sock *sk; 

        skb = netlink_trim(skb, allocation);

        //construct info
        info.group = group;

        sk_for_each_bound(sk, &nl_table[ssk->sk_protocol].mc_list)
                do_one_broadcast(sk, &info);
    }
{% endhighlight %}

遍历 `nl_table` ,对每一个sock判断其是否包含info.group，如果是就发送数据info.  

重点就是 nl_table，如果还记得` nl_join_groups()`，当时是否还好奇join到哪里去了，现在看来，
join group 就是将socket和广播组的映射保存到内核的nl_table中。   


一个socket加入一个group的流程是这样的

    struct sockaddr_nl addr;
    int nl_sock = socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE);
    addr.nl_family = AF_NETLINK;
    addr.nl_groups = /*RTMGRP_LINK | */RTMGRP_IPV4_IFADDR | RTMGRP_IPV4_ROUTE;
    bind(nl_sock, (struct sockaddr *)&addr, sizeof(addr));

`join group` 应该发生在 bind系统调用过程中。


## bind()

--------------

{% highlight c %}
    SYSCALL_DEFINE3(bind, int, fd, struct sockaddr __user *, umyaddr, int, addrlen)
    {
        struct socket *sock;
        struct sockaddr_storage address;
        int err, fput_needed;

        sock = sockfd_lookup_light(fd, &err, &fput_needed);
        if (sock) {
                err = move_addr_to_kernel(umyaddr, addrlen, &address);
                if (err >= 0) { 
                        err = security_socket_bind(sock,
                                                   (struct sockaddr *)&address,
                                                   addrlen);
                        if (!err)
                                err = sock->ops->bind(sock,
                                                      (struct sockaddr *)
                                                      &address, addrlen);
                }    
                fput_light(sock->file, fput_needed);
        }    
        return err; 
    }

    static const struct proto_ops netlink_ops = {
        .family =       PF_NETLINK,
        .owner =        THIS_MODULE,
        .release =      netlink_release,
        .bind =         netlink_bind,
        .connect =      netlink_connect,
        .socketpair =   sock_no_socketpair,
        .accept =       sock_no_accept,
        .getname =      netlink_getname,
        .poll =         netlink_poll,
        .ioctl =        sock_no_ioctl,
        .listen =       sock_no_listen,
        .shutdown =     sock_no_shutdown,
        .setsockopt =   netlink_setsockopt,
        .getsockopt =   netlink_getsockopt,
        .sendmsg =      netlink_sendmsg,
        .recvmsg =      netlink_recvmsg,
        .mmap =         netlink_mmap,
        .sendpage =     sock_no_sendpage,
    };
    
    static int netlink_create(struct net *net, struct socket *sock, int protocol,
                              int kern)
    {
      err = __netlink_create(net, sock, cb_mutex, protocol);
    }

    static int __netlink_create(struct net *net, struct socket *sock,
                            struct mutex *cb_mutex, int protocol)
    {
        sock->ops = &netlink_ops;
    }
{% endhighlight %}
    
bind 最终调用了`netlink_bind` 

{% highlight c %}
    static int netlink_bind(struct socket *sock, struct sockaddr *addr,
                          int addr_len)
    {
        struct sock *sk = sock->sk;
        struct net *net = sock_net(sk);
        struct netlink_sock *nlk = nlk_sk(sk);
        struct sockaddr_nl *nladdr = (struct sockaddr_nl *)addr;

        /* Only superuser is allowed to listen multicasts */
        if (nladdr->nl_groups) {
                if (!netlink_capable(sock, NL_CFG_F_NONROOT_RECV))
                        return -EPERM;
                err = netlink_realloc_groups(sk);
                if (err)
                        return err; 
        } 
        ...
        netlink_update_subscriptions(sk, nlk->subscriptions +
                                         hweight32(nladdr->nl_groups) -
                                         hweight32(nlk->groups[0]));
      }
{% endhighlight %}

分配groups   
  
{% highlight c %}
      static int netlink_realloc_groups(struct sock *sk)
      {
          struct netlink_sock *nlk = nlk_sk(sk);
          netlink_table_grab();
  
          groups = nl_table[sk->sk_protocol].groups;
  
          new_groups = krealloc(nlk->groups, NLGRPSZ(groups), GFP_ATOMIC);

          nlk->groups = new_groups;
          nlk->ngroups = groups;
      }
{% endhighlight %}

更新 `nl_table`

{% highlight c %}
    static void
    netlink_update_subscriptions(struct sock *sk, unsigned int subscriptions)
    {
        struct netlink_sock *nlk = nlk_sk(sk);
        sk_add_bind_node(sk, &nl_table[sk->sk_protocol].mc_list);
        nlk->subscriptions = subscriptions;
    }

{% endhighlight %}

## End

----------

已经写的太多了，Linux这颗大树枝繁叶茂，飞进去容易迷路，还好有Ctags地图和Cscope指南针带我突围。 本文以 XFRMGRP_NEWSA 为起点，
分析了Netlink 发送NEWSA通知的过程，以及bind系统调用 Join Group的过程。
