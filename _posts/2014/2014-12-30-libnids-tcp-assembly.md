---
layout: single
author_profile: true
comments: true
title: Libnids TCP会话重组分析
tagline: 源码分析
category : cybersecurity
tags : [Linux, 网络安全]
---

Vim源码分析环境搭建好了[here](http://onestraw.net/linux/vim-notes)，今天拿libnids练练手，虽然cscope命令才掌握几个常用的，但是阅读速度已经很快了，相对于source insight，至少不用担心成为鼠标手了。

> Libnids performs:  
  a) assembly of TCP segments into TCP streams  
  b) IP defragmentation   
  c) TCP port scan detection   

即Libnids的功能有

1. 将TCP分段重组TCP流会话
2. IP分片重组
3. 检测TCP端口扫描

下面围绕第一点[重组TCP流会话]来分析一下源码，我主要有两个问题

1. **libnids工作过程，如何进行TCP会话重组？**
2. **我试验了sample/printall.c程序，为什么不能输出会话重组后的所有数据？**

##问题1

main函数一般是下面这个样子


{% highlight c %}
 
int main ()
{
  if (!nids_init ())
  {
  	fprintf(stderr,"%s\n",nids_errbuf);
  	exit(1);
  }
  nids_register_tcp (tcp_callback);
  nids_run ();
  return 0;
}
{% endhighlight %}

- nids_init()
- nids_register_tcp(tcp_callback)
- nids_run()

`nids_init`主要进行

0. pcap抓包前的配置工作，如pcap_compile, pcap_setfilter
1. init_procs()初始化全局回调函数链表，包括ip_frag_procs, ip_procs, tcp_procs, udp_procs; 
ip_frag_procs链表默认有一个处理函数gen_ip_frag_proc;
ip_procs链表默认有一个处理函数gen_ip_proc;
tcp_procs和udp_procs为空;

2. tcp_init()
	- 初始化全局tcp_stream_table, 它是一个二维tcp_stream指针;
	- 初始化streams_pool，它是一个一维的tcp_stream指针;
	- init_hash()初始化哈希表;
3. ip_frag_init()
4. scan_init()
5. g_thread_init()


`nids_register_tcp(tcp_callback)`将回调函数插入到链表tcp_procs头部;

`nids_run`主要的工作就是调用pcap_loop循环抓包，将数据交付nids_pcap_handler回调函数处理。
该回调函数首先处理一下数据包头部，然后交于`ip_frag_procs`链表中的每个函数来处理，这时链表中只一个处理函数gen_ip_frag_proc

gen_ip_frag_proc判断该IP包是不是分片包(IPF_NOTF, IPF_NOTF)，是不是第一个到达的分片包(IPF_NEW)，然后调用ip_procs链表上的全部函数，此时只有一个gen_ip_proc

gen_ip_proc根据应用层协议类型调用相应的process_tcp/udp/icmp函数

process_tcp(data, skblen)

find_stream()返回当前数据包匹配的tcp_stream
	nids_find_tcp_stream根据tuple4的哈希值查找tcp_stream_table
	
{% highlight c %}

struct tcp_stream *
nids_find_tcp_stream(struct tuple4 *addr)
{
  int hash_index;
  struct tcp_stream *a_tcp;

  hash_index = mk_hash_index(*addr);
  for (a_tcp = tcp_stream_table[hash_index];
       a_tcp && memcmp(&a_tcp->addr, addr, sizeof (struct tuple4));
       a_tcp = a_tcp->next_node);
  return a_tcp ? a_tcp : 0;
}
 
{% endhighlight %}

从这个函数可以看出，它是使用单独链表法解决哈希冲突/碰撞，tcp_stream_table第一维存储的是直接哈希值，第二维是相同哈希值的不同tuple4的单链表，用于处理碰撞。   

建立时NIDS_JUST_EST会将tcp_procs链表上的所有函数变成tcp_stream的listensers，然后在a_tcp->nids_state==NIDS_DATA时
调用tcp_procs链表上的所有函数（此时有一个我们自己挂载的函数tcp_callback）

最后处理完成数据，调用notify函数通知tcp_procs链表上的函数来处理，函数调用路线如下：

    process_tcp
    tcp_queue
    add_from_skb
    notify
    ride_lurkers
    tcp_callback
    
##问题2

src/tcp.c中的函数add_from_skb()中有三处  

`rcv->offset = rcv->count; /* clear the buffer */`   

我看了n遍还是很奇怪，`为什么这一句能够清除缓冲区?`  

终于在看第n+1遍时，终于发现玄机就在add2buf函数中！
{% highlight c %}
static void
add2buf(struct half_stream * rcv, char *data, int datalen)
{
  int toalloc;
  
  if (datalen + rcv->count - rcv->offset > rcv->bufsize) {
    if (!rcv->data) {
      if (datalen < 2048)
	toalloc = 4096;
      else
	toalloc = datalen * 2;
      rcv->data = malloc(toalloc);
      rcv->bufsize = toalloc;
    }
    else {
      if (datalen < rcv->bufsize)
      	toalloc = 2 * rcv->bufsize;
      else	
      	toalloc = rcv->bufsize + 2*datalen;
      rcv->data = realloc(rcv->data, toalloc);
      rcv->bufsize = toalloc;
    }
    if (!rcv->data)
      nids_params.no_mem("add2buf");
  }
  memcpy(rcv->data + rcv->count - rcv->offset, data, datalen);
  rcv->count_new = datalen;
  rcv->count += datalen;
}

{% endhighlight %}

虽然前面写了申请内存，或者扩展内存，容易误导我们，但是注意  
 
`memcpy(rcv->data + rcv->count - rcv->offset, data, datalen);`  

如果rev->count == recv->offset，那么这条语句就造价于

`memcpy(rcv->data, data, datalen);`  

也就是覆盖了同一会话中前面到达的数据！所以在同一TCP会话后面就无法找到前续数据，好像nids_discard保留一些TCP包的数据，但是也有数量限制，
最安全的办法就是在自定义的tcp_callback函数中存储会话数据，然后再通过tuple4, offset, count, count_new加以组合.

###参考
[libnids](https://github.com/MITRECND/libnids)
[process_tcp源码注释](http://blog.csdn.net/aaa6695798/article/details/5221867)
[libnids笔记](http://www.cnblogs.com/chenyadong/archive/2011/12/17/2291157.html)
