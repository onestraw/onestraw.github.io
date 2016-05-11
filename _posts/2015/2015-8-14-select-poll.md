---
layout: single
author_profile: true
comments: true
title: 异步I/O select & poll
tagline: 
category: essay
tags : [面试]
---


看问题要抓本质，前戏都是浮云，高潮才是关键！  

学东西先抓核心，提纲契领。

-----------------------


### Blocking I/O

注意下面两个fd: `listenfd` & `connfd`

>	listenfd = socket(): create an endpoint for communication    
	bind(listenfd,...):  bind a name to a socket  
	listen(listenfd,...): listen for connections on a socket  
	connfd = accept(listenfd,...): accept a connection on a socket  
	read/write(connfd,...)   

上述程序会阻塞在 accept 上，直接有一个连接到来。
	
> The  accept()  system call is used with connection-based socket types (SOCK_STREAM, SOCK_SEQPACKET).  It extracts the first connection request on the queue of pending connections for the listening socket, sockfd, creates a new connected socket, and returns a new file descriptor referring to that socket.  The newly created socket is not in the listening state.  The original socket sockfd is unaffected by this call.

- [simple socket server](http://www.thegeekstuff.com/2011/12/c-socket-programming/)

### select

select提供异步I/O 多路复用（synchronous I/O multiplexing），人如其名，它可以用从多个文件描述符中select准备好数据的文件描述符进行操作，而不是阻塞在一个文件描述符上。详细的介绍参考APUE。

> - FD_ZERO - Clear an fd_set  
	- FD_ISSET - Check if a descriptor is in an fd_set
	- FD_SET - Add a descriptor to an fd_set
	- FD_CLR - Remove a descriptor from an fd_set

使用异步I/O的服务端编写思路：  

1. 先将负责监听的socket (listenfd)加入read_fd_set，因为服务端接收连接请求，是读listenfd。
2. 每accept一个连接，就返回一个新的连接socket(connfd)，connfd可以读写数据，将connfd加入read_fd_set。

>	listenfd = socket()  
	bind(listenfd,...)    
	listen(listenfd,...)  
	fd_set read_fd_set;  
	FD_SET(listenfd, &read_fd_set)  
	select (FD_SETSIZE, &read_fd_set, NULL, NULL, NULL)   
	FD_ISSET (i, &read_fd_set) && i==listenfd   
	connfd = accept(listenfd,...)   
	FD_SET(connfd, &read_fd_set)  
	read/write(connfd,...)  
	
- [server code using select](http://www.gnu.org/software/libc/manual/html_node/Server-Example.html)


### poll


> poll wait for some event on a file descriptor, it waits for one of a set of file descriptors to become ready to perform I/O.    

>	int poll(struct pollfd *fds, nfds_t nfds, int timeout); 

		struct pollfd {
               int   fd;         /* file descriptor */
               short events;     /* requested events */
               short revents;    /* returned events */
         };

| event | 意义 |
|:------- |:------- |
|POLLIN 	| There is data to read.|
|POLLPRI	| There is urgent data to read (e.g., out-of-band data on TCP socket; pseudoterminal master in packet mode has seen state change in slave).|
|POLLOUT	| Writing now will not block.|
|POLLRDHUP|	Stream socket peer closed connection, or shut down writing half of connection.  The _GNU_SOURCE feature test macro must be defined (before including any header files) in order to obtain this definition.|
|POLLERR	|	Error condition (output only).	|
|POLLHUP	|	Hang up (output only). |
|POLLNVAL	| Invalid request: fd not open (output only).|  

下面是一个用poll写的服务端例子  
[server example using poll](http://cboard.cprogramming.com/c-programming/158125-sockets-using-poll.html)


其实poll和select很类似，都需要不断轮询fd集合，select 通过参数设置read_fd_set和write_fd_set，而poll用pollfd.event的POLLIN或者POLLOUT来判断可读可写。
