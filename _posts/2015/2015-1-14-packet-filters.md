---
layout: single
author_profile: true
comments: true
title: 数据包过滤规则总结
tagline: packet filters
category : cybersecurity
tags : [网络安全]
---

###问题

在分析一个基于libnids(它基于libpcap抓包)的程序时，遇到一个问题，于是用一个input.pcap文件作为输入进行调试，
由于应用程序内部使用`nids_params.filter`进行了过滤（来提高性能），结果导致`应用程序没有从input.pcap读入一个包，全部被屏蔽掉了`。
（这里绝不是过滤规则的作用）  
假定input.pcap有10.10.10.10主机的数据包！   
使用

> tcpdump -r input.pcap host 10.10.10.10

并没有得到期望的数据包，感觉这是tcpdump的局限，用如下方法

> tshark -2 -r input.pcap -R "host 10.10.10.10" -w output.pcap

能够得到过滤数据包.

最后注释掉`nids_params.filter`，才能通过pcap文件正常调试。

###过滤规则汇总

1. **libpcap**  
  
  -[BPF](http://biot.com/capstats/bpf.html)

2. **wireshark***  
  
  有两种过滤规则 `Capture Filters`和`Display Filters`  

  - [Capture Filters](http://wiki.wireshark.org/DisplayFilters)  捕获前过滤，一般在`Capture`菜单下设置
  - [Display Filters](http://wiki.wireshark.org/CaptureFilters)   捕获后过滤，便于分析，一般位于工具栏下方

3. **tcpdump**  
  
  - [tcpdump](http://www.tcpdump.org/manpages/pcap-filter.7.html)
  - [15个tcpdump例子](http://www.thegeekstuff.com/2010/08/tcpdump-command-examples/)

4. **tshark**  
  
  - [tshark实例](http://www.packetlevel.ch/html/tshark/tsharkfilt.html)

