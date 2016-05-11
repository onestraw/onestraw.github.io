---
layout: single
author_profile: true
comments: true
title: 基于snort的域名监控工具DnsEye
categories: [Snort]
tags: [Snort, DNS]
---

<h1>0.源码学习</h1>
snort 作为世界上最流行的开源入侵检测系统，至今已有15的历史了，最新release2.9.6.1的源码有近5MB大小，使用命令

>wc snort-2.9.6.1/src/*

统计共有<span style="color: #ff0000;">66873</span>行代码，已经是很大工程了。看linux源码，大多从linux0.11开始，依照这个 方法，找到了snort最原始的版本0.96， 发现该版本太简单了，两个源文件（snort.c, snort.h），总共约<span style="color: #ff0000;">1600行</span>代码。

其实snort0.96的功能也简单，基于libpcap进行抓包分析，仅仅是一个简单的嗅探器，还没有图形界面 （回想去年一个大作业：编写网络嗅探器，要是使用这个源码，自己加个界面，就事半功倍了）。


学习源码的一个重要步骤就是修改，重新编译。我在<strong><span style="color: #ff0000;">snort0.96</span></strong>基础上添加一个功能：

<strong>解析所有的DNS请求包，监控局域网所有用户访问域名的情况。</strong>

监控域名这有什么用途呢？
<ul>
	<li>发现用户的网站浏览癖好；</li>
	<li>谁使用了翻墙工具，谁经常访问成人网站；</li>
	<li>从安全角度讲，有些僵尸网络botnet使用domain-flux技术进行通信，可以发现那些随机域名；</li>
	<li>...</li>
</ul>
<h1>2. DnsEye实现</h1>

使用四层链表结构记录域名访问信息：  

第一层链表记录发送DNS请求的source ip；  

后面三层分别记录各级域名，分解到第三级；  

第二层链表记录顶级域名，如com, net, org;  

第三层链表记录二级域名，如onestraw;  

最后一层链表记录域名第三级和更高级的所有域名信息，如www，a.b.c.d等；  

此外，各层链表均记录请求次数。

1）在snort.h 中添加如下代码
<pre>//son of second level ,namely, third level damin
struct SSLD{
    char name[64];
    u_long cnt;
    struct SSLD *next;
};
//second level domain, its length &amp;lt; 64
struct SLD{
    char name[64];
    u_long cnt;
    struct SSLD *ssld;
    struct SLD *next;
};
//top level domain
struct TLD{
    char name[5];
    u_long cnt;
    struct SLD *sld;
    struct TLD *next;
};
//requester, namely,src addr
struct DNSRequest{
    u_long saddr;
    u_long cnt;
    struct TLD *tld;
    struct DNSRequest *next;
};
//struct TLD *g_dnlist=NULL;
struct DNSRequest *g_dnslist=NULL;
static u_int pcnt;

void PrintDNlist(int level);
void ReleaseDNlist();
void RecordDomainName(u_long addr, char *dname);
void DecodeDNS(u_char *pkt, int len);
</pre>
2）snort.c中修改一下DecodeUDP函数
<pre>   if(pip.dport==53)
   {
       pktidx = pktidx +8;
       DecodeDNS(pktidx, len-8);
   }
</pre>
3）CleanExit函数中添加
<pre>   if(g_dnslist)
   {
      PrintDNlist(3);
      ReleaseDNlist();
   }</pre>
   
4）定义四个函数

	void PrintDNlist(int level);
	void ReleaseDNlist();
	void RecordDomainName(u_long addr, char *dname);
	void DecodeDNS(u_char *pkt, int len);


<a href="https://github.com/onestraw/DnsEye" target="_blank">DnsEye源码</a>

#附录

1.snort所有版本下载: http://sourceforge.jp/projects/sfnet_snort/releases/

