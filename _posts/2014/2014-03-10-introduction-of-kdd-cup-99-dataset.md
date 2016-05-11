---
layout: single
author_profile: true
comments: true
title: KDD CUP 99数据集简介
categories: [cybersecurity]
tags: [DDoS]
---
KDD CUP 99数据集是什么，为什么很多入侵检测领域的科研论文都用该数据集进行实验？

<strong>一、KDD CUP 99 数据集概述</strong>

<strong>KDD CUP 99数据集</strong>是1999年KDD CUP竞赛使用的数据。

KDD CUP是ACM每年举办的<strong>知识发现和数据挖掘竞赛</strong>
『注：KDD 是指Knowledge Discovery and Data Mining，CUP是奖杯』

1999年KDD CUP的题目是Computer network intrusion detection

KDD CUP 99 数据集使用的是DARPA 1998 DataSet的原始数据，在DARPA 98数据集的基础上进行了预处理，提取出了以“连接”为单位的一条条记录。

『注：DARPA  98 数据集是 MIT Lincoln实验室搭建一个模拟US空军局域网的环境，捕获了9周的原始数据包，包含多种攻击』

<strong>二、KDD CUP 99数据集内容</strong>

<strong>1. KDD CUP 99数据集包括训练数据和测试数据</strong>

训练数据：7周的tcpdump数据有4G大小，经处理约有5百万条“连接”记录；
测试数据：2周的tcpdump数据是，约2百万条“连接”记录；

“A connection is a sequence of TCP packets starting and ending at some well defined times, between which data flows to and from a source IP address to a target IP address under some well defined protocol. ”

<strong>“连接”是在一个固定的时间间隔内，从一个源IP到一个目的IP的一系列TCP数据包。</strong>

我的理解是在时间间隔t（如2秒）内，(srcIP, dstIP)标识一个连接，后面的特征就是基于这个连接的。
每个连接被标记为正常或攻击。

<strong>2. 该数据集包括4类主要攻击</strong>

* DOS: denial-of-service, e.g. syn flood;
* R2L: unauthorized access from a remote machine, e.g. guessing password;
* U2R:  unauthorized access to local superuser (root) privileges, e.g., various “buffer overflow” attacks;
* probing: surveillance and other probing, e.g., port scanning.

注意：测试数据中可能包括训练中未出现过的攻击类型，很多入侵检测专家认为绝大多数新型攻击是已知攻击的变体，可能根据已知攻击的特征来检测新型攻击。

<strong>3. 特征</strong>

设定统计的时间窗口（间隔）为2s

<strong>“same host”</strong>定义：当前连接的dstIP，过去2s内和当前连接有相同dstIP的连接。same service定义类似。

3.1.独立的TCP连接(不同于通常所说的TCP连接)的基本特征

    duration:连接的持续时间，以秒为单位
    protocol_type:网络层协议类型，如tcp,udp等
    service:应用层的服务(协议)类型，如http,telnet等
    src_bytes:从srcIP发送到dstIP的数据包字节数，单位B
    dst_bytes:从dstIP发送到srcIP的数据包字节数，单位B
    flag:连接的正常或者错误状态标志
    land:如果一个连接的srcIP/dstIP等于”same host”, 或者srcPort/dstPort等于”same port”, land=1; 否则land=0
    wrong_fragment:错误的分片数
    urgent:urgent数据包个数

3.2.连接内部的内容特征

    hot:number of “hot” indicators
    num_failed_logins:登录失败的次数
    logged_in:如果成功登录，该标志置1，否则置0
    num_compromised:number of “compromised” conditions
    root_shell:如果root shell被获取，置1，否则置0
    su_attempted:如果尝试了su root命令，置1，否则置0
    num_root:以root身份访问的次数
    num_file_creations:执行创建文件操作的次数
    num_shells:shell的开启个数
    num_access_files:访问控制文件上的操作次数
    num_outbound_cmds:ftp会话中”向外传输命令“的使用次数
    is_hot_login:1 if the login belongs to the “hot” list; 0 otherwise
    is_guest_login:1 if the login is a “guest”login; 0 otherwise

3.3.使用2s的时间窗口计算的流量特征

<strong>条件SH:</strong> 过去2s内和当前连接有相同dstIP

    count:满足条件SH的连接个数
    serror_rate:满足条件SH且有SYN错误的连接个数/count
    rerror_rate:满足条件SH且有REJ错误的连接个数/count
    same_srv_rate:满足条件SH且和当前连接有相同service的连接个数/count
    diff_srv_rate:满足条件SH且和当前连接的service不同的连接个数/count

<strong>条件SR:</strong> 过去2s内和当前连接有相同service

    srv_count:满足条件SR的连接个数
    srv_serror_rate:满足条件SR且有SYN错误的连接个数/srv_count
    srv_rerror_rate:满足条件SR且有REJ错误的连接个数/srv_count
    srv_diff_host_rate:满足条件SR且和当前连接的dstIP不同的连接个数/srv_count

<strong><em>参考：</em></strong>
KDD CUP:  http://www.sigkdd.org/kddcup/index.php
DARPA98数据集: http://www.ll.mit.edu/mission/communications/cyber/CSTcorpora/ideval/data/1998data.html
KDD CUP 99数据集:  https://kdd.ics.uci.edu/databases/kddcup99/task.html
