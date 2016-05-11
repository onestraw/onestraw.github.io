---
layout: single
author_profile: true
comments: true
title: sfportscan插件检测端口扫描
categories: [Snort]
tags: [portscan, Snort]
---

<h1>0.引言</h1>
sfportscan插件将端口扫描分为四种类型，分析每一种扫描类型的特点，抽象出四个流量特征来描述每一种扫描类型，用简单的统计方法来检测扫描。首先有一个时间窗口，统计每一个时间窗口内4个特征值的大小，当这4个特征值满足一定条件（超出阈值）时，判定为某一扫描类型，其中的阈值是根据经验人为设定的。

sfportscan是一个内置插件，在snort-2.9.5.5中，该插件包含四个源文件portscan.h,portscan.c,spp_sfportscan.h, spp_sfportscan.c.
<h1>1.四种扫描类型</h1>
Portscan检测引擎的基本目标是捕获nmap和variant scanners. 该引擎跟踪TCP/UDP/ICMP/IP协议的连接尝试(请求，connection attempts)。 对一个连接，如果返回一个有效的响应，则标记该连接是有效的；如果没有响应，或者是无效响应（TCP RST），该引擎记录这些连接请求，得到两个数量统计：无效的响应数目和没有产生响应的连接请求数目，这两个数值能够区分普通的扫描(normal scan)和过滤的扫描(filtered scan).该引擎检测四种类型的扫描：
<ul>
	<li>- Portscan</li>
	<li>- Decoy Portscan</li>
	<li>- Distributed Portscan</li>
	<li>- Portsweep</li>
</ul>
<h2>1)Portscan</h2>
基本的端口扫描，扫描发起端（scanner_ip）与被扫描端(scanned_ip)是一对一的关系，scanned_ip上的许多端口被扫描。定义3个流量统计特征：
<ul>
	<li>diff_srcip_cnt：和目标主机通信的源主机数量；</li>
	<li>diff_prt_cnt：不同的端口数量；</li>
	<li>inv_res_con_cnt：无效的响应/连接请求数量；</li>
</ul>
<strong>判定标准：</strong>如果diff_srcip_cnt比较小，而diff_prt_cnt和inv_res_con_cnt比较大，则可说明发生 Portscan.
<h2>2)Distributed Portscan</h2>
分布式端口扫描，描发起端（scanner_ip）与被扫描端(scanned_ip)是多对一的关系，scanned_ip上的许多端口被扫描。

<strong>判定标准：</strong>如果diff_srcip_cnt、diff_prt_cnt和inv_res_con_cnt都比较大，则说明发生分布式端口扫描。

<h2>3)Decoy Portscan</h2>

诱骗端口扫描，它是分布式端口扫描的一个变体，scanner_ip和scanned_ip依然是多对一的关系，它和分布式端口扫描的不同点在于：Decoy Portscan 会尝试连接scanned_ip的一个端口多次，而Distributed Portscan对目标端口尝试连接一次。
Decoy的含义：scanner_ip多次连接scanned_ip上的一个端口，看上去像在通信一样，以此来欺骗。

<h2>4)Portsweep</h2>

scanned_ip和scanned_ip是一对多的关系，一个scanner_ip扫描多个scanned_ip，对每一个scanned_ip，仅扫描探测少数的端口，而不像前三类扫描那样，探测成千上万的端口。监测srcip，类似于特征diff_srcip_cnt，增加特征diff_dstip_cnt：和源主机通信的目标主机数量；

<strong>判定标准：</strong>如果diff_dstip_cnt和inv_res_con_cnt比较大，而diff_prt_cnt比较小，则可以判断是Portsweep.

<h1>2. PS_ALERT_CONF</h1>

	typedef struct s_PS_ALERT_CONF{
	short connection_count;
	short priority_count;
	short u_ip_count;
	short u_port_count;
	} PS_ALERT_CONF;

很遗憾，源码没有对该结构进行注释，不过我从snort用户手册2.2 Preprocessors中查到该结构体各成员的意义.

<h2>1）connection_count</h2>

Connection Count lists how many connections are active on the hosts (src or dst). This is accurate for connection-based protocols, and is more of an estimate for others. Whether or not a portscan was filtered is determined here. High connection count and low priority count would indicate filtered (no response received from target).  

<strong>connection_count</strong>指明了当前时间段内在主机(src or dst)上有多少活跃的连接。该字段对于基于连接的协议(TCP)很准确，对于其它协议（UDP等），它是一个估计值。portscan是否被过滤可以用该字段进行辨别，如果connection_count较大，而priority_count较小，则表明portscan被过滤了。

<h2>2）priority_count</h2>

Priority Count keeps track of bad responses (resets, unreachables). The higher the priority count, the more bad responses have been received.   

<strong>priority_count</strong>记录"bad responses"（无效响应，如TCP RST, ICMP unreachable）.priority_count越大，说明捕获的无效响应包越多. 从portscan.c文件中的alert函数来看，只所以叫priority_count，是因为在判断扫描时 priority_count 是先于 connection_count进行判断的，它们俩是并列的，但是priority_count优先和阈值比较.(参考ps_alert_one_to_one_decoy()函数)

<h2>3）u_ip_count</h2>

IP Count keeps track of the last IP to contact a host, and increments the count if the next IP is different. For one-to-one scans, this is a low number. For active hosts this number will be high regardless, and one-to-one scans may appear as a distributed scan.  

<strong>u_ip_count</strong>记录着和主机最后进行通信的IP地址(last_ip)，如果新来一个数据包，其源IP地址src_ip，如果src_ip 不等于last_ip，就对u_ip_count字段加1。对于Portscan类型扫描，该值比较小；对于活跃的主机（和外界通信频繁），这个值会比较大，这样有可能导致portscan被检测成Distributed scan.

个人理解：u_ip_count 应该是 unique ip count，记录着一个时间段内和某一主机通信的所有不同的IP地址个数。

<h2>4）u_port_count</h2>
Port Count keeps track of the last port contacted and increments this number when that changes. We use this count (along with IP Count) to determine the difference between one-to-one portscans and one-to-one decoys.  
<strong>u_port_count</strong>记录着和主机最后进行通信的端口（last_port），当新来的数据包的目的端口(dst_port)不等于last_port，那么对u_port_count加1.

该字段和u_ip_count一样，我有同样的疑惑。

联系第1节中提到的diff_ip_cnt, diff_prt_cnt, inv_res_con_cnt; 这三个值和PS_ALERT_CONF结构体对应关系如下：
<span style="color: #ff0000;">connection_count, priority_count <==>inv_res_con_cnt</span>
<span style="color: #ff0000;">u_ip_count <==> diff_ip_cnt</span>
<span style="color: #ff0000;">u_port_count <==> diff_prt_cnt</span>

<h1>3. 时间窗口</h1>
snort-2.9.5.5\src\portscan.c: 667行  
ps_proto_update_window()函数中将时间窗口按敏感级别low, medium, high分别设置为60s, 90s,600s.

<h1>4. PS_ALERT_CONF阈值</h1>
sfportscan插件根据扫描类型（ps, dist_ps, decy_ps, sweep）、协议（TCP, UDP, IP, ICMP）和敏感级别(low, medium, high)分别设置了阈值.
例如

	static PS_ALERT_CONF g_tcp_med_ps = {200,10,60,15};
	static PS_ALERT_CONF g_tcp_med_decoy_ps = {200,30,120,60};
	static PS_ALERT_CONF g_tcp_med_sweep = {30,7,7,10};
	static PS_ALERT_CONF g_tcp_med_dist_ps = {200,30,120,30};

<h1>5. 判定端口扫描类型</h1>
三个参数的含义：
PS_PROTO *scanner ：扫描者
PS_PROTO *scanned ：被扫描者
PS_ALERT_CONF *conf ：阈值

下面分别介绍四种端口扫描类型的判定标准，分别对应四个alert函数，对函数进行了简化，只保留阈值比较部分。
<h2>1）Portscan</h2>
ps_alert_one_to_one()函数.

	if(scanned->priority_count >= conf->priority_count
	&& scanned->u_ip_count < conf->u_ip_count
	&& scanned->u_port_count >= conf->u_port_count)
	{
	scanned->alerts = PS_ALERT_ONE_TO_ONE;
	return 0;
	}
	if(scanned->priority_count < conf->priority_count /*隐含条件，体现了优先*/
	&&scanned->connection_count >= conf->connection_count
	&& conf->connection_count > 0
	&& scanned->u_ip_count < conf->u_ip_count
	&& scanned->u_port_count >= conf->u_port_count)
	{
	scanned->alerts = PS_ALERT_ONE_TO_ONE_FILTERED;
	return 0;
	}

<h2>2）Decoy Portscan</h2>
ps_alert_one_to_one_decoy()函数.

	if(scanned->priority_count >= conf->priority_count
	&& scanned->u_ip_count > conf->u_ip_count
	&& scanned->u_port_count >= conf->u_port_count)
	{
	scanned->alerts = PS_ALERT_ONE_TO_ONE_DECOY;
	return 0;
	}
	if(scanned->priority_count < conf->priority_count /*隐含条件，体现了优先*/
	&&scanned->connection_count >= conf->connection_count
	&& conf->connection_count > 0
	&& scanned->u_ip_count > conf->u_ip_count
	&& scanned->u_port_count >= conf->u_port_count)
	{
	scanned->alerts = PS_ALERT_ONE_TO_ONE_DECOY;
	return 0;
	}
	
<h2>3）Distributed Portscan</h2>
ps_alert_many_to_one()函数.

	if(scanned->priority_count >= conf->priority_count
	&& scanned->u_ip_count <= conf->u_ip_count
	&& scanned->u_port_count >= conf->u_port_count)
	{
	scanned->alerts = PS_ALERT_DISTRIBUTED;
	return 0;
	}
	if(scanned->priority_count < conf->priority_count /*(隐含的条件, 体现了优先)*/
	&& scanned->connection_count >= conf->connection_count
	&& conf->connection_count > 0
	&& scanned->u_ip_count <= conf->u_ip_count
	&& scanned->u_port_count >= conf->u_port_count)
	{
	scanned->alerts = PS_ALERT_DISTRIBUTED_FILTERED;
	return 0;
	}
	
<h2>4）Portsweep</h2>
ps_alert_one_to_many()函数.

	if(scanner->priority_count >= conf->priority_count
	&& scanner->u_ip_count >= conf->u_ip_count
	&& scanner->u_port_count <= conf->u_port_count)
	{
	scanner->alerts = PS_ALERT_PORTSWEEP;
	return 0;
	}
	if(scanner->priority_count < conf->priority_count /*(隐含的条件, 体现了优先)*/
	&& scanner->connection_count >= conf->connection_count
	&& conf->connection_count > 0
	&& scanner->u_ip_count >= conf->u_ip_count
	&& scanner->u_port_count <= conf->u_port_count)
	{
	scanned->alerts = PS_ALERT_PORTSWEEP_FILTERED;
	return 0;
	}

<span style="color: #ff00ff;"><strong>注意点1：</strong></span>区分Portscan和Distributed portscan，因为if判定基本完全相同，区别仅在于u_ip_count比较时一个小于，另一个是小于等于.关键点在于阈值，观察下面红色字体：<span style="color: #000000; font-family: Tahoma; font-style: normal; font-variant: normal; font-weight: normal; letter-spacing: normal; line-height: normal; orphans: 2; text-align: -webkit-auto; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-size-adjust: auto; -webkit-text-stroke-width: 0px; font-size: medium; display: inline !important; float: none;">
</span>

static PS_ALERT_CONF g_udp_low_ps = {0,5,<strong><span style="color: #ff0000;">25</span></strong>,5};  
static PS_ALERT_CONF g_udp_low_dist_ps = {0,15,<span style="color: #ff0000;"><strong>50</strong></span>,15};  

static PS_ALERT_CONF g_udp_med_ps = {200,10,<span style="color: #ff0000;"><strong>60</strong></span>,15};  
static PS_ALERT_CONF g_udp_med_dist_ps = {200,30,<span style="color: #ff0000;"><strong>120</strong></span>,30};  

static PS_ALERT_CONF g_udp_hi_ps = {200,3,<span style="color: #ff0000;"><strong>100</strong></span>,10};  
static PS_ALERT_CONF g_udp_hi_dist_ps = {200,3,<span style="color: #ff0000;"><strong>200</strong></span>,10};  

发现distributed portscan的u_ip_count阈值是Portscan的两倍，而检测时是按<span style="color: #000000; font-family: Tahoma; font-style: normal; font-variant: normal; font-weight: normal; letter-spacing: normal; line-height: normal; orphans: 2; text-align: -webkit-auto; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-size-adjust: auto; -webkit-text-stroke-width: 0px; font-size: medium; display: inline !important; float: none;">Portscan,</span><span style="color: #000000; font-family: Tahoma; font-style: normal; font-variant: normal; font-weight: normal; letter-spacing: normal; line-height: normal; orphans: 2; text-align: -webkit-auto; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-size-adjust: auto; -webkit-text-stroke-width: 0px; font-size: medium; display: inline !important; float: none;"> Decoy Portscan,</span><span style="color: #000000; font-family: Tahoma; font-style: normal; font-variant: normal; font-weight: normal; letter-spacing: normal; line-height: normal; orphans: 2; text-align: -webkit-auto; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-size-adjust: auto; -webkit-text-stroke-width: 0px; font-size: medium; display: inline !important; float: none;"> Distributed Portscan,</span><span style="color: #000000; font-family: Tahoma; font-style: normal; font-variant: normal; font-weight: normal; letter-spacing: normal; line-height: normal; orphans: 2; text-align: -webkit-auto; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-size-adjust: auto; -webkit-text-stroke-width: 0px; font-size: medium; display: inline !important; float: none;"> Portsweep的顺序进行判定检测的，如果发现前面的扫描类型，直接返回，不再向下执行，所以该插件能准确区分不同的扫描类型。</span>

<strong><span style="color: #ff00ff;">注意点2：</span></strong>Portsweep的判定中用的是scanner，而另外三个是scanned.原因在第1节中已经提到，Portsweep的检测思路是监测源IP，统计diff_dstip_cnt.

<h1>6. 结束语</h1>
本文介绍了sfportscan插件检测端口扫描的思路，以及区分4种端口扫描类型的判定条件。但是有一点没有提到，就是如何统计提取这四个特征，这也是下一步要学习的，它用到了stream5预处理器（维持TCP流状态，进行会话重组）。

<em>（参考：snort-2.9.5.5\src\preprocessors\portscan.c和snort用户手册）</em>
