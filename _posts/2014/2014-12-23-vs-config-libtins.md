---
layout: single
author_profile: true
comments: true
title: VS2013配置libtins
tagline: windows构造数据包
category : Windows
tags : [网络安全]
---

Linux下使用C库libnet，跨平台的Python库scapy都是不错的选择，Windows平台一直没有找到理想的C/C++数据包构造库，
今天就来学习一下[libtins](http://libtins.github.io/),libtins是一个跨平台的C++数据包构造和嗅探库。

> libtins is a high-level, multiplatform C++ network packet sniffing and crafting library.

0. 下载编译好的libtins库  
[download here](http://libtins.github.io/download/binary/windows/libtins-latest-win32.zip)   
注意这是Release版本，相应的应用程序也应该是Relase，否则会出现_ITERATOR_DEBUG_LEVEL连接错误。  

1. 下载编译好的winpcap库  
[download here](http://www.winpcap.org/install/bin/WpdPack_4_1_2.zip)  

2. 配置include目录和lib目录  
分别是libtins和wpdPack的include和lib目录  

3. 加入连接依赖库   

    Ws2_32.lib  
    Iphlpapi.lib  
    wpcap.lib  
    tins.lib  
    
4. 测试程序   
[ARP Monitor](http://libtins.github.io/examples/arp-monitor/)
这个测试程序不能直接在windows下中跑，因为命令行参数是网卡名称，windows下面这个名字太长，它不像Linux下面eth0, wlan0等，windows下的网络设备名一般是下面这种东西：

    \Device\NPF_{2D0129D9-349C-4541-ADCB-C8C325E86AE3}

而且获取不方便！就是你把这个设备名给程序，也会出错。Ok，利用libtins的网络接口类[NetworkInterface](http://libtins.github.io/tutorial/#ifaces)
和IP地址和MAC地址类[IPv4Address]来获取网络设备名，将arp-monitor的命令行参数由设备名改成IP地址，让libtins自己根据IP地址找设备名吧，所以修改之后的测试程序如下：

{% highlight c %}
#include <tins/tins.h>
#include <map>
#include <iostream>
#include <functional>

using namespace Tins;

class arp_monitor {
public:
	void run(Sniffer &sniffer);
private:
	bool callback(const PDU &pdu);

	std::map<IPv4Address, HWAddress<6>> addresses;
};

void arp_monitor::run(Sniffer &sniffer)
{
	sniffer.sniff_loop(
		std::bind(
		&arp_monitor::callback,
		this,
		std::placeholders::_1
		)
		);
}

bool arp_monitor::callback(const PDU &pdu)
{
	// Retrieve the ARP layer
	const ARP &arp = pdu.rfind_pdu<ARP>();
	// Is it an ARP reply?
	if (arp.opcode() == ARP::REPLY) {
		// Let's check if there's already an entry for this address
		auto iter = addresses.find(arp.sender_ip_addr());
		if (iter == addresses.end()) {
			// We haven't seen this address. Save it.
			addresses.insert({ arp.sender_ip_addr(), arp.sender_hw_addr() });
			std::cout << "[INFO] " << arp.sender_ip_addr() << " is at "
				<< arp.sender_hw_addr() << std::endl;
		}
		else {
			// We've seen this address. If it's not the same HW address, inform it
			if (arp.sender_hw_addr() != iter->second) {
				std::cout << "[WARNING] " << arp.sender_ip_addr() << " is at "
					<< iter->second << " but also at " << arp.sender_hw_addr()
					<< std::endl;
			}
		}
	}
	return true;
}

int main(int argc, char *argv[])
{
	if (argc != 2) {
		std::cout << "Usage: " << *argv << " <interface'IP address>\n";
		return 1;
	}
	arp_monitor monitor;
	// Sniff on the provided interface in promiscuous mode
	NetworkInterface eth0(IPv4Address((const char*)argv[1])); 
	Sniffer sniffer(eth0.name(), Sniffer::PROMISC);

	// Only capture arp packets
	sniffer.set_filter("arp");
	monitor.run(sniffer);
}
{% endhighlight %}
注意是Release哦，因为tins.lib是release的

执行 

    arp-monitor 192.168.0.110

监控结果

    [INFO] 192.168.0.120 is at 90:b1:1c:0e:9e:c4
    [WARNING] 192.168.0.120 is at 90:b1:1c:0e:9e:c4 but also at 78:2b:cb:3d:61:56
