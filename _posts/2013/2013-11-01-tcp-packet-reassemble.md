---
layout: single
author_profile: true
comments: true
title: TCP报文重组方式
excerpt: 研究TCP流重组方式
categories: [cprogram]
tags: [网络编程, Linux, C/C++]
---

## 1.问题描述
-----------

TCP 流重组是 TCP 协议中的必需功能。 IP 包在传输过程中可能被分片，并可能以乱序方式到达接收方。 TCP 数据流是通过 IP 协议进行传输的，数据流同样可能乱序。接收方协议栈必须对数据包重排序.重组.这种接受有序号的，乱序到达的数据包，并重建原数据流的过程叫做重组 (reassembly)。TCP/IP 协议栈的重组采用缓存法。缓存法是的重组方法，它将分片缓存并拼接到适当位置，在全部分片到达之后，重建重组的数据包供下一步处理。网络收到的 TCP 分段还可能出现重叠现象。即两个分段中包含相同的序号区间。对于重叠分段 TCP 协议并没有规定具体实现方法，而不同 TCP/IP 协议栈的重叠分段重组方法也有所不同。对于两个包含重叠数据的分段，根据重叠数据在分段中的位置分为：  

1. 后缀分段（重叠数据是该分段的后缀）；
2. 前缀分段（重叠数据是该分段的前缀）。

## 2.思路
-------------

向echo server发送两个有重叠的数据包，检查echo server返回的数据，以确定echo server所在系统的报文重组方式。
构造两个数据包内容如下：

      01234
      …….56789

1. 如果从echo server收到01234789，则重组方式是先到优先，保留后缀。
2. 如果从echo server收到01256789，则重组方式是后到优先，保留前缀。

## 3.实验环境
-----------

- 操作系统：ubuntu12.04 desktop
- 编程工具：C、socket、libpcap、libnet

## 4.程序框架
---------------

![packet-reassemble](/assets/images/packet.jpg)

## 5.几个问题
-------

1. 使用libpcap编译好过滤规则pcap_setfilter之后，就可对流经网卡的数据包进行过滤。然后进行三次握手，之后调用pcap_next()捕获第二次握手包。不能在建立连接之前调用pcap_next()，否则程序会一直等待，直到捕获到所需数据包之后才进行建立连接。
2. 注意seq, ack捕获与发送时的关系。
3. 由于本地建立连接时，所选的端口是随机的，所以最后发送两个数据包时的源端口要用第二次握手包中获取，也可以使用getsockname()来获取本地IP和PORT。

## 6.结论
-----

在ubuntu12.04上，从echo server收到的报文是01234789，丢弃了第二个报文的重叠前缀，数据报的重组方式是后缀优先。(ps:不同的操作系统重组方式可能不同。)

## 附源码
---------

[tcp_packet_reassemble.c](https://github.com/onestraw/code/blob/master/tcp_packet_reassemble.c)
