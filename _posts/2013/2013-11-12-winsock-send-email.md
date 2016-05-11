---
layout: single
author_profile: true
comments: true
title: 使用winsock发送email
categories: [cprogram]
tags: [C/C++, 网络编程]
---

使用socket编写发送邮件的程序非常简单，只需要按照smtp协议的过程，一步步来即可。

1. 本文注册了一个126邮箱，用户名和密码用BASE64编码，向163邮箱和QQ邮箱发送邮件已经测试过，不会被当作垃圾邮件过滤。
2. 具体的smtp步骤可参看源代码的switch部分。
3. 根据自己的邮箱修改源代码的第8-11行

{% highlight C %}

#include<stdio.h>
#include<stdlib.h>
#include<winsock.h>
#pragma comment(lib, "Ws2_32.lib") 

#define DATA "DATA\r\n"
#define QUIT "QUIT\r\n"
#define smtp_serv "smtp.126.com"
#define from_addr "username@126.com"  //注册一个测试邮箱
#define USER "email addr"  //上面注册的邮件地址BASE64编码
#define PASS "password"    //邮箱密码的BASE64编码

char subject[]="主题onestraw.net";
char content[100]="邮件正文。你好";
char to_addr[]="1093965800@qq.com";

char buf[BUFSIZ+1];
int len;

void send_socket(SOCKET sock, char *data)
{
	send(sock, data, strlen(data),0);
    printf("Client:%s\n",data);
}
void read_socket(SOCKET sock)
{
    len = recv(sock, buf, BUFSIZ, 0);
	printf("Server:%s\n",buf);
}

/* start from here*/ 
int main(int argc, char *argv[])
{
	WSADATA wsa;
	SOCKET sockfd;
	struct sockaddr_in smtpAddr;
	struct hostent *host;
	unsigned long smtp_ip;
	short smtp_port;
	int i;
	/*
	 * 与Linux不同的地方，程序中调用任何一个Winsock API函数
	 * 第一件事情就是必须通过WSAStartup函数完成对Winsock服务的初始化
	*/
    if(WSAStartup(MAKEWORD(2,2), &wsa) != 0)
	{
        printf("socket initial failed\n");
        exit(1);
    }
	/*
	* 解析域名，获得IP
	*/
	if((host= gethostbyname(smtp_serv)))
	{
		char ip_addr[20];
		strcpy(ip_addr, inet_ntoa (*(struct in_addr *)host->h_addr_list[0]));
		smtp_ip = inet_addr(ip_addr);
	}
	else
	{
		printf("%s域名无法解析!\n",smtp_serv);
		exit(1);
	}
	if((sockfd=socket(AF_INET, SOCK_STREAM, IPPROTO_TCP))==INVALID_SOCKET)
	{
		printf("创建socket失败!\n");
		exit(1);
	}
	smtp_port = 25;
	memset(&smtpAddr, 0, sizeof(struct sockaddr_in));
	smtpAddr.sin_family = AF_INET;
	smtpAddr.sin_addr.S_un.S_addr = smtp_ip;
	smtpAddr.sin_port = htons(smtp_port);
	/*
	* 建立与smtp服务器的连接
	*/
	if(connect(sockfd, (struct sockaddr *)&smtpAddr, sizeof(smtpAddr))==SOCKET_ERROR)
	{
		printf("连接%s失败!\n", smtp_serv);
		exit(1);
	}
    read_socket(sockfd);
	/*
	* 按SMTP步骤填充buf, 并发送
	*/
	for(i=0; i<10; i++)
	{	
		switch(i)
		{
			case 0:
				sprintf(buf, "HELO %s\r\n", from_addr);	break;
			case 1:
				sprintf(buf, "AUTH LOGIN\r\n");		break;
			case 2:
				sprintf(buf, "%s\r\n", USER);	break;
			case 3:
				sprintf(buf, "%s\r\n", PASS);	break;
			case 4:
				sprintf(buf, "MAIL FROM <%s>\r\n", from_addr);	break;
			case 5:
				sprintf(buf, "VEFY  %s\r\n", from_addr);	break;
			case 6:
				sprintf(buf, "RCPT TO <%s>\r\n", to_addr);	break;
			case 7:
				sprintf(buf,"%s", DATA);	break;
			case 8:
				sprintf(buf, "Subject: %s\r\n",subject);
				send_socket(sockfd, buf);
				sprintf(buf, "FROM: %s\r\n", from_addr);
				send_socket(sockfd, buf);
				sprintf(buf, "TO: %s\r\n\r\n", to_addr);
				send_socket(sockfd, buf);
				
				sprintf(buf, "%s\r\n.\r\n", content);	break;
			case 9:
				sprintf(buf, "%s", QUIT);	break;
		}
		send_socket(sockfd, buf);
		read_socket(sockfd);
	}

	closesocket(sockfd);
	return 0;
}

{% endhighlight %}

[@github](https://github.com/onestraw/code/blob/master/winsock_send_email.c)
