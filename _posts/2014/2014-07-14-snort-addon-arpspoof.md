---
layout: single
author_profile: true
comments: true
title: Snort预处理插件arpspoof
categories: [Snort]
tags: [Snort]
---
<h1>引言</h1>
ARP协议为IP地址到硬件地址提供动态映射。
ARP欺骗&lt;==&gt;ARP spoofing&lt;==&gt;ARP病毒&lt;==&gt;ARP poisoning&lt;==&gt;ARP攻击。

<h1>ARP报文格式</h1>
|--14字节的以太网头部--|--28字节的ARP请求/应答--|

    0~1字节：硬件类型
    2~3字节：协议类型
    4字节：硬件地址长度
    5字节：协议地址长度
    6~7字节：操作字段op
    8~13字节：sender_MAC
    14~17字节：sender_IP
    18~23字节：target_MAC
    24~27字节：target_IP

op=1是arp 请求报文，target_MAC无效，为0，MAC层广播报文，MAC层的target_MAC 全ff:ff:ff:ff:ff:ff；  
op=2是arp 响应报文；

<h1>ARP欺骗原理</h1>

ARP缓存表用于存储其它主机或网关的IP地址与MAC地址的对应关系，每台主机、网关都有一个ARP缓存表，ARP缓存表里存储的每条记录实际上就是一个IP地址与MAC地址对，它可以是静态的，也可以是动态的。如果是静态的，那么该条记录不能被ARP应答包修改；如果是动态的，那么该条记录可以被ARP应答包修改。

主机在实现ARP缓存表的机制中存在一个缺陷，当主机收到一个ARP应答包后，它并不会去验证自己是否发送过这个ARP请求，而是直接将应答包里的MAC地址与IP对应的关系替换掉原有的ARP缓存表里的相应信息.

<h1>ARP欺骗防御</h1>

1. MAC地址绑定，使网络中每一台计算机的IP地址与硬件地址一一对应，不可更改。
2. 使用静态ARP缓存，用手工方法更新缓存中的记录，使ARP欺骗无法进行。

<h1>ARP欺骗检测</h1>

ARP欺骗的网络现象有：网络频繁掉线;网速突然变慢;甚至上不去网；检测ARP欺骗的方法有下面3种： 

1. 使用ARP –a命令发现网关的MAC地址与真实的网关MAC地址不相同;
2. 使用sniffer软件发现局域网内存在大量的ARP reply包；
3. 根据网络内MAC与IP地址的绑定关系，分析网络内的ARP流量， 判断arp reply中的sender_IP和sender_MAC是否匹配；该方法需要网络内主机的IP与MAC绑定关系，这种方法检测最精确；

<h1>Snort中arpspoof插件的实现</h1>
snort的arp欺骗检测插件使用的检测方法是方法3：根据IP与MAC地址的绑定关系，判断arp reply中的sender_IP和sender_MAC是否匹配；

1. void SetupARPspoof(void);是该插件对外提供的唯一接口，

    调用
    ARPspoofInit()
    ARPspoofHostInit()

2. static void ARPspoofInit(struct _SnortConfig *sc, char *args);   
在snort预处理插件链表上挂载 arpspoof检测函数DetectARPattacks(); 这样每一个数据包都会经过该函数处理一次。 
并调用 ParseARPspoofArgs()

3. static void ParseARPspoofArgs(ArpSpoofConfig *config, char *args);  
解析arpspoof 插件的参数，目前仅有一个有效参数 -unicast，是否监控分析单播的arp request包；

4. static void ARPspoofHostInit(struct _SnortConfig *sc, char *args);  
初始化被监控(保护)的主机列表，读取IP和MAC的绑定关系； 
调用 ParseARPspoofHostArgs()

5. static void ParseARPspoofHostArgs(IPMacEntryList *ipmel, char *args);  
从命令行终端以 如下格式读取IP和MAC的绑定关系  
arpspoof_detect_host: 10.10.10.10 29:a2:9a:29:a2:9a  

6. static void DetectARPattacks (Packet *p, void *context);   
这是检测ARP欺骗的核心函数，它对数据包p的处理如下： 

1）对于ARP REQUEST 单播包  
判断ethernet层 dst MAC 和 ARP层的 target MAC 是否匹配？  

2）对于ARP REQUEST 广播包
判断ethernet层src MAC 和 ARP 层的sender MAC是否匹配？  

3）对ARP REPLY包 
首先判断 ethernet层的src MAC和ARP层的sender MAC是否匹配？   
其次判断 ethernet层的dst MAC和ARP层的target MAC是否匹配？  

4）根据该数据包p的 sender IP在 IP-MAC绑定关系链表中 查询 right_sender_MAC  
判断right_sender_MAC 和数据包p的ethernet层的src MAC是否匹配？   
判断right_sender_MAC 和数据包p的ARP层的sender MAC是否匹配?    

上述判断匹配步骤，如有一步匹配失败，就报出相应错误，直接返回，不再向下执行。  

<h3>参考</h3>
1. TCP/IP详解卷1-chapter4
2. snort-2.9.5.5\src\preprocessors\spp_arpspoof.c
