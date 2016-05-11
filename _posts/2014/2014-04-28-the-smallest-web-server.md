---
layout: single
author_profile: true
comments: true
title: 最小的web服务器
category: Python
tags: [Python, 网络编程]
---


写过网络程序的人都知道，网络上两个主体进行通信，需要遵守一个“约定”，这个“约定”由双方协商确定，根据这个“约定”，通信双方才能准确知道对方所发送信息的含义。我们写一些简单的网络工具（最明显的就是C/S程序），可能需要自己制定一些简单的通信“约定”，比如发送“USER”，表明后面跟着的数据是用户名，发送“PASSWD”，表示后面跟着的数据是口令。网络上公共的“约定”叫做协议。

B/S结构其实是C/S结构的一种，只不过客户端Client统一简化了，都是浏览器Browser，服务器端Server提供各种各样的功能，来自不同的单位，但是它们都遵守一个共同的“约定”——HTTP协议，只有这样，我们在PC机上通过一个浏览器访问不同的服务器才不会出现问题。

<span style="color: #ff0000;"><strong>Web服务器可以理解为解析HTTP协议，执行相应“指令”，完成客户端功能的一个程序。</strong></span>

为简单模拟一下web服务器的工作过程，用python实现了一个简单的不能再简单的服务器，服务器端使用select I/O模型编写，功能简单，仅支持：
<ul>
	<li>支持HTTP-GET 请求；</li>
	<li>支持后缀为html的标准网页，返回相应页面(可嵌入图片、js脚本)或者”404 page not found”；</li>
	<li>可配置监听IP、端口、默认页面。</li>
</ul>
	#!/usr/bin/python
	
	import sys
	import os
	import getopt
	import socket
	import select
	import Queue
	import datetime
	import threading
	
	class Server(threading.Thread):
	        def __init__(self, ip, port, default_page):
	                threading.Thread.__init__(self)
	                self.ip = ip
	                self.port = port
	                self.default_page = default_page
	                self.listenNum = 50
	                self.sock = self.newSocket()
	                self.runFlag = 1
	                self.printServerInfo()
	        def printServerInfo(self):
	                print u"IP:%s\nPort:%d\nDefault Page:%s\n" %(self.ip,self.port,self.default_page)
	        def run(self):
	                inputs = [self.sock]
	                outputs = []
	                msgQueues = {}
	                while self.runFlag:
	                        readable,writable,exceptional=select.select(inputs,outputs,inputs)
	                        for s in readable:
	                                if s is self.sock:
	                                        conn,cli_addr = s.accept()
	                                        print "connection from",cli_addr
	                                        inputs.append(conn)
	                                        msgQueues[conn]=Queue.Queue()
	                                else:#connection is created
	                                        data = s.recv(1024)
	                                        data = self.parse_data(data)
	                                        #assume no errors
	                                        if data:
	                                                print "received ",data," from ", s.getpeername()
	
	                                                #send page content
	                                                data = self.pack_http_packet(data[1:])
	                                                msgQueues[s].put(data)
	                                                if s not in outputs:
	                                                        outputs.append(s)
	                                        else:
	                                                print "closing ", cli_addr
	                                                if s in outputs:
	                                                        outputs.remove(s)
	                                                        
	                                                inputs.remove(s)
	                                                s.close()
	                                                del msgQueues[s]
	                        for s in writable:
	                                try:
	                                        if s in msgQueues.keys():
	                                            nextMsg = msgQueues[s].get_nowait()
	                                except Queue.Empty:
	                                        print s.getpeername()," message queue is empty"
	                                        outputs.remove(s)
	                                else:
	                                        #print "sending ", nextMsg, " to ",s.getpeername()
	                                        s.send(nextMsg)
	                                        #s.close()
	                                        #inputs.remove(s)
	                                        #outputs.remove(s)
	                        for s in exceptional:
	                                print "exceptional condition on ",s.getpeername()
	                                inputs.remove(s)
	                                if s in outputs:
	                                        outputs.remove(s)
	                                s.close()
	                                del msgQueues[s]
	        def stop(self):
	                self.runFlag = 0
	                if self.sock:
	                        self.sock.close()
	        def newSocket(self):
	                sock = socket.socket(socket.AF_INET,socket.SOCK_STREAM)
	                sock.setsockopt(socket.SOL_SOCKET,socket.SO_REUSEADDR,1)
	                sock.bind((self.ip,self.port))
	                sock.listen(self.listenNum)
	                return sock
	
	        def parse_data(self,data):
	                data = data.split(' ')
	                if len(data)&lt;2:
	                        return self.default_page
	                return data[1]
	        def pack_http_packet(self,html_file):
	                GMT_FORMAT = '%a, %d %b %Y %H:%M:%S GMT'
	                now = datetime.datetime.utcnow()
	                expires = datetime.timedelta(days=1)
	                sname="onestraw.net/0.1"
	                if not os.path.exists(html_file):
	                        content="&lt;html&gt;&lt;h1&gt;404 Page Not Found&lt;/h1&gt;&lt;/html&gt;"
	                        http_code="404"
	                else:   
	                        fr = open(html_file,"rb")
	                        content = fr.read()
	                        fr.close()
	                        http_code="200 OK"
	                responses="HTTP/1.1 %s\r\nHost:%s\r\nDate: %s\r\nExpires:%s\r\nContent-Type:text/html; charset=utf-8\r\nContent-Length:%d\r\nConnection:keep-alive\r\n\r\n%s" %(http_code,sname,now.strftime(GMT_FORMAT),(now+expires).strftime(GMT_FORMAT),len(content),content)       
	                return responses
	
	if __name__=='__main__':
	        opt,left = getopt.gnu_getopt(sys.argv[1:],"",["ip=","port=","default_page="])
	        ip = "127.0.0.1"
	        port = 6677
	        defaultPage = "/index.html"
	        for i in range(len(opt)):
	                if opt[i][0]=='--ip':
	                        ip = opt[i][1]
	                elif opt[i][0]=='--port':
	                        port = int(opt[i][1])
	                elif opt[i][0]=='--default_page':
	                        defaultPage=opt[i][1]
	        try:
	                s = Server(ip,port,defaultPage)
	                s.start()
	        except KeyboardInterrupt:
	                s.stop()
	                print "CTRL+Z/D ,server end "
