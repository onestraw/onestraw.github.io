---
layout: single
author_profile: true
comments: true
title: windows下构造链路层数据包
tagline: based on WpdPack
category : Windows
tags: [网络安全]
---
###不能解析硬件地址
昨天配置好libtins之后，又尝试了另外一个Example[Arp Spoofing](http://libtins.github.io/examples/arp-spoof/)，
一些跨平台引起的编译错误排除后，运行时出现MAC地址解析错误。
执行
  
    arpspoof.exe gateway_ip, victim_ip
    

> Runtime error: Could not resolve hardware address

调试发现出错位置do_arp_spoofing函数

    // Resolves gateway's hardware address.
    gw_hw = Utils::resolve_hwaddr(iface, gw, sender);
  	// Resolves victim's hardware address.
    victim_hw = Utils::resolve_hwaddr(iface, victim, sender);

###resolve_hwaddr原理
简单看一下[libtins源码](https://github.com/mfontanini/libtins)

Runtime error: Could not resolve hardware address是发生在下面函数中  

    Utils::resolve_hwaddr(iface, gw, sender);

libtins/src/utils.cpp中

{% highlight c++ %}
 
HWAddress<6> resolve_hwaddr(const NetworkInterface &iface, IPv4Address ip, PacketSender &sender) 
{
    IPv4Address my_ip;
    NetworkInterface::Info info(iface.addresses());
    EthernetII packet = ARP::make_arp_request(ip, info.ip_addr, info.hw_addr);
    Internals::smart_ptr<PDU>::type response(sender.send_recv(packet, iface));
    if(response.get()) {
        const ARP *arp_resp = response->find_pdu<ARP>();
        if(arp_resp)
            return arp_resp->sender_hw_addr();
    }
    throw std::runtime_error("Could not resolve hardware address");
}
{% endhighlight %}

它是通过发送一个arp请求包，来获得ip对应的mac地址，核心是response(sender.send_recv(packet, iface));
也就是sender.send_recv(packet, iface)，接着向下看libtins是如何处理windows平台发送arp数据包的. 

sender  
PacketSender  

libtins/src/packet_sender.cpp

{% highlight c++ %}

PDU *PacketSender::send_recv(PDU &pdu) {
    return send_recv(pdu, default_iface);
}

PDU *PacketSender::send_recv(PDU &pdu, const NetworkInterface &iface) {
    try {
        pdu.send(*this, iface);
    }
    catch(std::runtime_error&) {
        return 0;
    }
    return pdu.recv_response(*this, iface);
}
 
{% endhighlight %}

pdu.send  
pdu.recv_response  
pdu  
PDU  
在函数中resolve_hwaddr发现EthernetII应该是PDU的子类，事实上在libtins/include/pdu.h发现 
send和recv_response都是虚函数

{% highlight c++ %}
virtual void send(PacketSender &sender, const NetworkInterface &iface);  

virtual PDU *recv_response(PacketSender &sender, const NetworkInterface &iface);


//libtins/src/pdu.cpp

void PDU::send(PacketSender &, const NetworkInterface &) { 
    
}

PDU *PDU::recv_response(PacketSender &, const NetworkInterface &) { 
    return 0; 
}
{% endhighlight %}

libtins/include/tins/ethernetII.h

{% highlight c++ %}
class EthernetII : public PDU {
// Windows does not support sending L2 PDUs.
#ifndef WIN32
void send(PacketSender &sender, const NetworkInterface &iface);
#endif // WIN32

#ifndef WIN32
PDU *recv_response(PacketSender &sender, const NetworkInterface &iface);
#endif // WIN32
}
{% endhighlight %}

注意源码中的注释和#ifndef WIN32

> Windows does not support sending L2 PDUs.

windows不支持发送L2(TCP/IP第二层, 即数据链路层)数据包(PDU, 协议数据单元)，
再向下看，会发现PacketSender类成员函数send_l2和recv_l2前面都会有一句#ifndef WIN32

###windows xp sp2及后续OS对Raw Socket的限制
* [解释一](https://groups.google.com/forum/#!msg/libtins/qj_NHJdZM6k/w9IIjWJYaVcJ)：  
  目前windows平台支持构造的数据包是网络层及以上;
* [Limitations on Raw Sockets](http://msdn.microsoft.com/en-us/library/ms740548.aspx): 

>  On Windows 7, Windows Vista, Windows XP with Service Pack 2 (SP2), and Windows XP with Service Pack 3 (SP3), the ability to send traffic over raw sockets has been restricted in several ways:
  * TCP data cannot be sent over raw sockets.
  * UDP datagrams with an invalid source address cannot be sent over raw sockets. The IP source address for any outgoing UDP datagram must exist on a network interface or the datagram is dropped. This change was made to limit the ability of malicious code to create distributed denial-of-service attacks and limits the ability to send spoofed packets (TCP/IP packets with a forged source IP address).
  * A call to the bind function with a raw socket for the IPPROTO_TCP protocol is not allowed.

###突破限制
windows不允许发送底层数据包，我们就束手无策了吗，No，scapy，cain & abel都能够很容易的进行arp spoof！一个简单的解决方法就是winpcap,
参考[winpcap 发送arp包](http://blog.csdn.net/wegatron/article/details/7636929)  
不过这个发送一个arp包太复杂了，需要选择网卡，还需要输入源IP, 源MAC, 目的IP, 目的MAC，发送个数。 
而且MAC地址还必需是AB CD EF..空格分割，不能使用AB-CD-EF..格式的.   

我的目标是做成kali linux下面那个arpspoof那样的接口[arpspoof-and-dnsspoof](http://onestraw.net/cybersecurity/arpspoof-and-dnsspoof/)

> arpspoof -i eth0 -t 192.168.0.1 192.168.0.254

192.168.0.1是欺骗主机，192.168.0.254是网关  

完成arpspoof的arp reply：

    sender MAC: 192.168.0.1' MAC address
    send IP:    192.168.0.1
    target MAC: 我的MAC
    target IP:  192.168.0.254

如果能够完成mac地址的自动解析，像第一节提到的resolve_hw_addr, 就方便了，难不成要自已构造一个arp request, 然后再过滤arp reply包，
进而获取对应的MAC，太麻烦了！

####SendARP
微软对RawSocket做那些限制只是为了安全考虑，如防止伪造源IP的DoS，但也不能影响正常的服务啊，所以微软提供了一个函数[SendARP](http://msdn.microsoft.com/en-us/library/windows/desktop/aa366358%28v=vs.85%29.aspx).
它的功能是发送ARP Request查询一个IP的MAC地址。

到此，思路已经很清晰了，用SendARP函数解析IP对应的MAC，通过本地IP选择网卡（[VS2013配置libtins](http://onestraw.net/windows/vs-config-libtins)）,这样整个过程只需要输入简单的IP地址就可以了。
