---
layout: single
author_profile: true
comments: true
title: 使用scapy快速统计pcap文件各类数据包个数
categories: [Python]
tags: [Python]
---
scapy中有一个rdpcap函数，它读取一个标准pcap文件，返回一个数据包列表（数组），可通过数组下标读取每个数据包。

对一个数据包pkt，首先根据以太网帧头的type字段判断数据包上层协议，ipv4, ipv6, arp。

	0×0800    ipv4
	0×0806    ARP
	0x86DD    ipv6

如果是ipv4协议，再进一步判断ip报头中的协议字段proto，分辨icmp, igmp, tcp, udp.

	1    ICMP
	2    IGMP
	6    TCP
	17    UDP
{% highlight python %}
#!/usr/bin/env python

import sys
from scapy.all import *
'''
0x0800	ipv4
	1	ICMP
	2	IGMP
	6	TCP
	17	UDP
0x0806	ARP
0x86DD	ipv6
'''

pkts_num={'ipv4':0,'icmp':0,'igmp':0,'tcp':0,'udp':0,'arp':0,'ipv6':0,'other':0}

def stats(pkt):
	if pkt.type==0x0800:
		pkts_num['ipv4'] += 1
		if pkt.proto==1:
			pkts_num['icmp'] += 1
		elif pkt.proto==2:
			pkts_num['igmp'] += 1
		elif pkt.proto==6:
			pkts_num['tcp'] += 1
		elif pkt.proto==17:
			pkts_num['udp'] += 1
	elif pkt.type==0x0806:
		pkts_num['arp'] += 1
	elif pkt.type==0x86dd:
		pkts_num['ipv6'] += 1
	else:
		pkts_num['other'] += 1

if __name__ == '__main__':
	if len(sys.argv)&lt;2:
		print("usage:%s pcap-file" %sys.argv[0])
		sys.exit(1)
	try:
		pkts = rdpcap(sys.argv[1])
		for pktno in range(len(pkts)):
			stats(pkts[pktno])
	except Scapy_Exception as e:
		print(e)
	for key in pkts_num:
		print("%s : %d"%(key, pkts_num[key]))
{% endhighlight %}

备注1：

	pkt = pkts[0]
	a = IP(str(pkt)) #pkt网络层以上的部分
	b = TCP(str(pkt)) #pkt应用层以上的部分

备注2：

	ipv6数据包没有进一步分析，用ipv6封装的也有icmpv6, tcp , udp等。
