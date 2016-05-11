---
layout: single
author_profile: true
comments: true
title: VMware手动配置虚拟网卡NAT
categories: [Windows]
tags: []
---

不知什么原因，我在win7下安装好VMware Workstation 9.0.2后，网络连接中并没有多出VMnet1,VMnet8两个虚拟网卡，我在虚拟机中安装的BackTrack5，需要更新一些软件，网上查了很多 资料，大多复制过来，粘贴过去的，都讲VMware网络连接的三种模式原理，不知所云。  

下面进行的是如何手动配置NAT模式的虚拟网卡  

1、新建VMware8  

打开VMware Workstation ，在“编辑-&gt;虚拟网络编辑器”中添加VMware8  

选择“NAT”  

选择“主机虚拟适配器连接到此网络”  

选择“使用本地DHCP服务分发到虚拟机的IP地址”   

查看NAT设置，发现网段192.168.241.0/24，网关是192.168.241.2，注意这个IP  


2、在win7的网络连接中右键“本地连接”，在“属性”中选择“共享”标签  

勾选“允许其他网络用户通过此计算机的Internet的连接来连接”  

并在家庭网络连接下拉框中选择“VMware Network Adapter VMnet8”  

点击确定会弹出一个对话框，提示LAN适配器被分配一个IP

192.168.137.1

3、此时在虚拟机backtrack中还没有连上Internet  

可以通过  

  vi /etc/network/interfaces   

来查看eth0网卡的状态，是否是dhcp    

然后重启网卡  

  /etc/init.d/networking restart  

然后使用ifconfig命令可看到eth0被分配一个192.168.137.0/24网段的IP  

  192.168.137.163

4、ping不一定可靠，backtrack5也有浏览器，打开浏览器，输入onestraw.net  

访问成功，网络配置完成。  


5、认识虚拟网卡NAT的本质  

注意上面被我标红的3个IP地址，从中可以学到虚拟网卡NAT的本质原理  

1）“网络连接”中的“本地连接”和“VMnet8”是两块网卡，合在一块，可以起到一个路由器网络地址转换的作用，它俩就相当于一个NAT路由器，VMware中使用VMnet8的虚拟机相当于内网机器。  

2）一个问题   

backtrack的eth0并没有使用第一步中的网段IP，网关也不是192.168.241.2，而是我们在共享“本地连接”时分配的ip ： 192.168.137.1  

在backtrack中使用  

  traceroute onestraw.net  

发现第一跳确实是192.168.137.1，说明网关就是192.168.137.1

