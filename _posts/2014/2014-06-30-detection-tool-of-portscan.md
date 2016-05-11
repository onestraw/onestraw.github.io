---
layout: single
author_profile: true
comments: true
title: Portscan检测工具
categories: [CyberSecurity]
tags: [portscan]
---

<h1>0. 引言</h1>
<div>snort的sfportscan插件将端口扫描分为Portscan, Decoy Portscan, Distributed Portscan, Portsweep四种类型，其中Portscan最简单，也最容易被检测，本文介绍一种检测portscan的思路，并实现一个检测工具。</div>
<h1>1. portscan检测思路</h1>
portscan的扫描方式是在短时间内从一台主机scanner扫描另一台主机scanned的大量端口，以便快速获得scanned主机的端口信息(closed, filtered, open).
四元组<span style="color: #ff0000;">（src_ip, dst_ip, src_port, dst_port）
</span>能唯一的标识一个会话session，正常情况下，两个主机之间的会话数量session_count在短时间内是比较小的，但是当发现portscan类型（一对一）的端口扫描时，这个scanner和scanned的会话数量会增加，而增加的根本原因，是四元组中目的端口dst_port的增加。设定一个时间窗口和阈值，将两两主机之间不同的目的端口的数目(diff_port_cnt)作为统计特征，在这个时间窗口内统计diff_port_cnt，当diff_port_cnt超过阈值时，则判定为portscan.
<h1>2. 实现</h1>
<div>为了区分tcp portscan 和udp portscan, 将四元组扩展成五元组<span style="color: #ff0000;">（protocol, src_ip, dst_ip, src_port, dst_port）</span>.</div>
<div>设定时间窗口interval = 5s. tcp_portscan_limit=10, udp_portscan_limit=10.</div>
<div>为统计两两主机之间的TCP/UDP会话信息，设计一个三层链表数据结构，各层的链表结点如下所示：</div>
<pre style="color: #000000; font-style: normal; font-variant: normal; font-weight: normal; letter-spacing: normal; line-height: normal; orphans: 2; text-align: -webkit-auto; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-size-adjust: auto; -webkit-text-stroke-width: 0px; font-size: medium;">//third level
struct dportNode{
    u_short dport;
    u_short sport;
    struct dportNode *next;
};
//second level
struct saddrNode {
    u_long saddr;
    u_long diff_dport_cnt;//different dst port count
    u_long high_freq_sport_cnt;//high frequency port, such as 80
    struct dportNode *dport;
    struct saddrNode *next;
};
//first level
struct daddrNode{
    u_long daddr;    //monitored ip addr
    struct saddrNode *tcp;    //for tcp scan
    struct saddrNode *udp;    //for udp scan
    struct daddrNode *next;
};
</pre>
<div>在对捕获的数据包进行处理时，对每一个数据包解析出一个会话session(protocol,dst_ip, src_ip, dst_port, src_port). 然后去查找链表，判断链表是否包含当前会话，如果链表中没有当前会话，则创建适当结点插入链表，并增加diff_dport_cnt值（实际上session中的src_port没有用到）。考虑到主机和web服务器通信时，可能用到一些临时端口，这里将源端口为80的会话排除，所以增加了high_freq_sport_cnt字段。每5s（时间窗口大小）遍历一次第一层链表和第二层链表，查找超过阈值的节点.</div>
<div style="color: #000000; font-family: Tahoma; font-style: normal; font-variant: normal; font-weight: normal; letter-spacing: normal; line-height: normal; orphans: 2; text-align: -webkit-auto; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-size-adjust: auto; -webkit-text-stroke-width: 0px; font-size: medium;">
<div>
<pre class="lang:c decode:true ">if( pda-&gt;tcp-&gt;diff_dport_cnt - high_freq_sport_cnt &gt; tcp_portscan_limit )
	alert("tcp portscan from pda-&gt;tcp-&gt;saddr to pda-&gt;daddr")
if(pda-&gt;udp-&gt;diff_dport_cnt - high_freq_sport_cnt &gt; udp_portscan_limit )
	alert("udp portscan from pda-&gt;udp-&gt;saddr to pda-&gt;daddr")
</pre>
</div>
</div>
<div>遍历结束之后，报告检测出的端口扫描（包括scanner,scanned,tcp/udp portscan），最后释放所有链表节点，在下一个时间窗口统计时，又会动态创建节点.</div>
<h1>3. 总结</h1>
<div>本文没有使用sfportscan插件的检测思想，而是针对portscan这种一对一的简单扫描类型，使用一个流量统计特征进行检测，在实现过程中利用一个三级链表结构保存两两主机之间的会话五元组信息。使用nmap实际测试结果表明，该工具能够有效检测不同的tcp/udp portscan.</div>
<h1>附录</h1>
<div>1. 完整代码见https://github.com/onestraw/PortScanDetect</div>
<div style="color: #000000; font-family: Tahoma; font-style: normal; font-variant: normal; font-weight: normal; letter-spacing: normal; line-height: normal; orphans: 2; text-align: -webkit-auto; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-size-adjust: auto; -webkit-text-stroke-width: 0px; font-size: medium;">2. 参考入侵检测工具watcher  http://www.xfocus.net/articles/200007/29.html</div>
&nbsp;
