---
layout: single
author_profile: true
comments: true
title: IPSec VPN 简介及实战
tagline: 
category : cybersecurity
tags : [网络安全, Linux]
--- 

----

##目录

---------

      1.IPSec
        1.1. 简介&功能
      	1.2. 组成&原理
      2.IPSec VPN
      	2.1 setkey
      	2.2 openswan
      3.推荐阅读
	

##1. IPSec

-----------------

####1. 简介 & 功能
Internet Protocol Security (IPsec) is a protocol suite for securing
Internet Protocol (IP) communications by authenticating and encrypting
each IP packet of a data stream.   

这是wikipedia对IPsec的定义中，有几个关键点：

- protocol suite: IPsec不是一个协议，它是一个套件，一个框架，包含多个协议
- IP: 它是用来保护IP通信的，在网络层对IP数据进行加密
- authenticate: 认证，保护IP通信的途径之一
- encrypt: 加密，保护IP通信的途径之二

IPSec是IETF（Internet Engineering Task Force，Internet工程任务组）的IPSec小组建立的一组IP安全协议集。   


IPSec定义了在网际层使用的安全服务，其功能包括

- 数据加密
- 对网络单元的访问控制
- 数据源地址验证
- 数据完整性检查
- 防止重放攻击

IPSec的安全服务要求支持共享密钥完成认证和/或保密，并且手工输入密钥的方式是必须要支持的，
其目的是要保证IPSec协议的互操作性。   
当然，手工输入密钥方式的扩展能力很差，因此在IPSec协议中引入了一个密钥管理协议，称Internet密钥交换协议——IKE，
该协议可以动态认证IPSec对等体，协商安全服务，并自动生成共享密钥。

####3. 组成 & 原理

- AH(Authentication Header) 协议。
- ESP(Encapsulated Security Payload) 协议。
- IKE(Internet Key Exchange Protocol)协议。Diffie-Hellman是一种建立密钥的方法，而不是加密方法。协商确定对称密钥。
- SP(Security Policy, 安全策略)
- SA(Security Association, 安全关联)一套专门将安全服务/密钥和需要保护的通信数据联系起来的方案。它保证了IPSec数据报封装及提取的正确性，同时将远程通信实体和要求交换密钥的IPSec数据传输联系起来。即SA解决的是如何保护通信数据、保护什么样的通信数据以及由谁来实行保护的问题。
- ISAKMP(Internet Security Association and Key Management Protocol) 协议
定义了协商、建立、修改和删除SA的过程和包格式。ISAKMP只是为SA的属性和协商、修改、删除SA的方法提供了一个通用的框架，并没有定义具体的SA格式。

    
`IPSec原理简述`   

1. 传输模式在AH、ESP处理前后IP头部保持不变，主要用于End-to-End的应用场景。
2. 隧道模式则在AH、ESP处理之后再封装了一个外网IP头，主要用于Site-to-Site的应用场景。



##2. IPSec VPN

----------

下面IPSec 配置实例，分别介绍手动配置密钥和自动协商密钥。  

网络拓扑1

  			PC-A   <======>		PC-B
  	20.20.20.21/24			20.20.20.22/24
	
网络拓扑2

    subnet-A  <------>  Gateway-A  <======>  Gateway-B  <------>  subnet-B
    10.10.10.0/24	20.20.20.21/24		20.20.20.22/24		30.30.30.0/24

下面的实例在网络拓扑1下配置传输模式(transport mode)，在网络拓扑2下配置隧道模式(tunnel mode)

####setkey

      加载SA和SP的命令：		setkey -f setkey.conf
      查看SA的命令：			setkey -D
      查看SP的命令：			setkey -DP
      清除SA的命令：			setkey -F
      清除SP的命令：			setkey -PF

下面是在主机20.20.20.21上的配置文件，在20.20.20.22上，只需将'-P out'和'-P in'交换即可。

transport mode configure file

		#!/usr/sbin/setkey -f
		flush;
		spdflush;

		#ESP
		add 20.20.20.21 20.20.20.22 esp 0x201 -E aes-cbc 0xfd64273b58d4e10af257ee5f7518a5e88a9ae77cf6f5a741 auth hmac-md5 0xca2cef3e4e00a0a111d3aa0048ec1ce1;
		add 20.20.20.22 20.20.20.21 esp 0x301 -E aes-cbc 0xfd64273b58d4e10af257ee5f7518a5e88a9ae77cf6f5a742 auth hmac-md5 0xca2cef3e4e00a0a111d3aa0048ec1ce2;

		#Security policies
		spdadd 20.20.20.21 20.20.20.22 any -P out ipsec
				esp/transport//require;

		spdadd 20.20.20.22 20.20.20.21 any -P in ipsec
				esp/transport//require;
	
tunnel mode configure file
	
		#!/usr/sbin/setkey -f
		flush;
		spdflush;
		#ESP
		add 20.20.20.21 20.20.20.22 esp 0x201 -m tunnel -E 3des-cbc 0xfd64273b58d4e10af257ee5f7518a5e88a9ae77cf6f5a74e -A hmac-md5 0xca2cef3e4e00a0a111d3aa0048ec1ce3;
		add 20.20.20.22 20.20.20.21 esp 0x301 -m tunnel -E 3des-cbc 0x435f03cdea95e04661282253b073b4d9502d96fd153fe5b -A hmac-md5 0xab9a40aa45320f50d584eae1bb762058;

		#Security policies
		spdadd 10.10.10.0/24 30.30.30.0/24 any -P out ipsec
				esp/tunnel/20.20.20.21-20.20.20.22/require;

		spdadd 30.30.30.0/24 10.10.10.0/24 any -P in ipsec
				esp/tunnel/20.20.20.22-20.20.20.21/require;
	
	
####openswan
在ubuntu下`apt-get install openswan`进行安装。   

配置好`/etc/ipsec.secret`和`/etc/ipsec.conf`之后，重启`ipsec`服务： `service ipsec restart`    

tansport mode

主机A和主机B配置完全一样

`/etc/ipsec.secrets`

      20.20.20.21 20.20.20.22: PSK "hello,world!"

`/etc/ipsec.conf`

    conn host-host
               	type=transport
                authby=secret
                left=20.20.20.21
                right=20.20.20.22
                pfs=yes
                auto=start


tunnel mode


主机A和主机B配置完全一样

`/etc/ipsec.secrets`

    20.20.20.21 20.20.20.22: PSK "hello,world!"

`/etc/ipsec.conf`

      conn net-net
                type=tunnel
                authby=secret
                left=20.20.20.21
                leftsubnet=10.10.10.0/24
                right=20.20.20.22
                rightsubnet=30.30.30.0/24
                pfs=yes
                auto=start


####验证VPN

			#on host 20.20.20.21, eth2's IP is 20.20.20.21
			ping 20.20.20.22
			tcpdump -i eth2	-e

				
##3.推荐阅读

--------

- [An Illustrated Guide to IPsec ](http://unixwiz.net/techtips/iguide-ipsec.html) 
- [IPSec源码分析](http://wenku.baidu.com/view/8ab1aa31cfc789eb172dc8c9.html)
- [IPSec VPN基本原理](www.h3c.com.cn/service/channel_service/operational_service/icg_technology/201005/675214_30005_0.htm)
- [Linux Kernel 2.6 using KAME-tools](http://www.ipsec-howto.org/x304.html)
