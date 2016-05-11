---
layout: single
author_profile: true
comments: true
title: 关闭TCP连接
tagline: shutdown与close的区别
category: essay
tags : [面试]
---

关闭TCP连接的面试问题三境界  

1. 描述下TCP连接关闭的过程？
2. 描述下TCP连接关闭的步骤，以及每一步的状态？  
3. shutdwon与close的区别？  

| step | packet | client state | server state | remark |
|:-------|:---------|:----------|:-------------|:------ |
|0 |	| ESTABLISHED | ESTABLISHED | |
|1  | client 向 server 发送FIN包(seq=x) server收到之前| FIN_WAIT_1 | ESTABLISHED| client调用close|
|2 | server收到FIN包，server向client发送ACK包(ack=x+1)，client收到之后 | FIN_WAIT_2 |  CLOSE_WAIT | |
|3 | server向client发送 FIN包(seq=y)，client收到之前  | FIN_WAIT_2 |  LAST_ACK | server 调用close|
|4 | client向server发送 ACK包(ack=y+1)，server收到确认之后  | TIME_WAIT |  CLOSED | |

![terminate tcp connection](http://www.masterraghu.com/subjects/np/introduction/unix_network_programming_v1.3/files/02fig05.gif)


###shutdwon & close

`man 2 shutdown`

    NAME
           shutdown - shut down part of a full-duplex connection
    
    SYNOPSIS
           #include <sys/socket.h>
    
           int shutdown(int sockfd, int how);
    
    DESCRIPTION
           The  shutdown()  call  causes  all  or part of a full-duplex connection on the socket associated with sockfd to be shut down.  If how is SHUT_RD, further receptions will be disallowed.  If how is SHUT_WR, further
           transmissions will be disallowed.  If how is SHUT_RDWR, further receptions and transmissions will be disallowed.
           

shutdown 可以通过how参数选择关闭sockfd读端、写端或者读写端。   
close 会关闭sockfd的读写两端，不能选择性关闭。  


