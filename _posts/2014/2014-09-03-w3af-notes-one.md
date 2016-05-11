---
layout: single
author_profile: true
comments: true
title: w3af学习笔记（一）
categories: [WebSec]
tags: [Web安全]
---

#w3af简介

w3af is a Web Application Attack and Audit Framework.即Web应用攻击和审计框架。w3af用python编写，依赖的库主要有2类，分别如下：

+ Core requirements:
	* Python 2.6
	* fpconst-0.7.2：用于处理IEEE 754浮点数;
	* nltk：自然语言处理工具包;
	* SOAPpy：SOAP是简单对象访问协议，是一种交换数据的协议规范，基于XML;
	* pyPdf：处理PDF文档，提取信息，分割/合并页面等；
	* Python bindings for the libxml2 
	* library：libxml2是C库，这里是个python中间件
	* Python OpenSSL：实现SSL与TLS的套件，https;
	* json.py：json是一种轻量级的数据交换格式；
	* scapy：可以用来发送、嗅探、解析和伪造网络数据包;
	* pysvn：支持subversion操作；
	* python sqlite3：精简的嵌入式开源数据库，使用一个文件存储整个数据库，没有独立的维护进程，全部由应用程序进行维护，使用特别方便；
	* yappi：支持配置每个线程的CPU时间（https://code.google.com/p/yappi/）

+ Graphical user interface requirements:
	- graphviz：可视化图表graph；
	- pygtk 2.0：生成GUI；
	- gtk 2.12：跨平台的图形工具包；

#w3af 架构

主要分3部分  

1. 内核  
The core, which coordinates the whole process and provides libraries for using in plugins.

2. UI（console和GUI）  
The user interfaces, which allow the user to configure and start scans

3. 插件  
The plugins, which find links and vulnerabilities

#黑盒测试–web应用扫描过程

* 识别所有的链接，表单，查询字条串参数；  
Identify all links, forms, query string parameters.

* 对于每个输入（表单），发送构造的畸形字符串，然后分析输出；  
Send specially crafted strings to each input and analyze the output

* 生成报告  
Generate a report with the findings

#w3af工作过程

1. 调用crawl plugins（如web spider）寻找所有的Links,forms,query string parameters. 通过这一步骤，将创建一个form和Links映射。
2. 调用audit plugins（比如sqli）发送畸形数据，来尽可能的触发漏洞。
2. 通过output plugins将发现的漏洞、调试和错误信息反馈给用户。

#源码初窥

文件：w3af/core/controllers/w3afCore.py
类w3afCore：这是整个框架的核心，它调用所有的插件，处理异常，协调所有的工作，创建线程……

	scan_start_hook(self)
	功能：创建目录、线程和“消费者”，用来执行一次w3af扫描。
	调用：在core初始化，重新启动扫描（清除前一个扫描的所有结果和状态）

	start(self)
	功能：UI调用该方法来启动整个扫描过程

	_safe_message_print(self,msg)
	功能：当磁盘空间耗尽，程序不能再向磁盘写入日志信息时，会引发异常，该函数就是来处理这种异常。像backtrack这种LiveCD经常出现这种异常。

	worker_pool(self)
	功能：返回一个类Pool的实例

	cleanup(self)
	功能：GUI 调用该方法，当一个扫描被终止，并且用户想启动一个新的扫描时，kb所有的数据被删除。

	stop(self)
	功能：当用户停止扫描时，UI层调用。

	quit(self)
	功能：退出s3af

	pause(self)
	功能：对于一个扫描，暂停或者取消暂停

	verify_environment(self)
	功能：检查用户配置的所有参数是否正确。

	scan_end_hook(self)
	功能：对应scan_start_hook()

	exploit_phase_prerequisites()
	pass

	_home_directory(self)
	功能：处理和创建/管理主目录相关的工作

	_tmp_directory(self)
	功能：创建tmp目录，存储大量资料，/tmp/w3af/<pid>/

#进阶计划

读了几个源码文件，感觉具体到某一个文件源码，看明白没什么困难，但是看完之后基本上没有收获，所以下一步准备修改源码再编译，希望能有所收获。
