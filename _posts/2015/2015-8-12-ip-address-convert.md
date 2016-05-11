---
layout: single
author_profile: true
comments: true
title: IP地址转换总结对比
tagline: 
category: essay
tags : [网络编程]
---

##函数对比

|	 函数	|	功能	|	头文件(Linux)	|	原型	|	源码实现	|
|:----|:----|:----|:----|:----|
|	**inet_ntop**	|	convert IPv4 and IPv6 addresses from binary to text form	|	arpa/inet.h		|	const char *inet_ntop(int af, const void *src, char *dst, socklen_t size); | [here](http://www.opensource.apple.com/source/xnu/xnu-1456.1.26/bsd/libkern/inet_ntop.c)	|
|	inet_ntoa	|	converts the Internet host address in, given in network byte order, to a string in IPv4 dotted-decimal notation. The string is returned in a statically allocated buffer, which subsequent calls will overwrite. |	arpa/inet.h		|	char *inet_ntoa(struct in_addr in);	|	[here](http://www.opensource.apple.com/source/Libc/Libc-167/net.subproj/inet_ntoa.c) |
|	inet_addr	|	converts the Internet host address cp from IPv4 numbers-and-dots notation into binary data in network byte order.	|	arpa/inet.h		|	in_addr_t inet_addr(const char *cp);	|	封装了inet_ntoa	|
|	**inet_pton** 	|	convert IPv4 and IPv6 addresses from text to binary form	|	arpa/inet.h		|	int inet_pton(int af, const char *src, void *dst); |	[here](http://www.opensource.apple.com/source/curl/curl-57/curl/lib/inet_pton.c)	|
|	inet_aton	|	converts the Internet host address cp from the IPv4 numbers-and-dots notation into binary form (in network byte order) and stores it in the structure that inp points to. 	|	arpa/inet.h		|	 int inet_aton(const char *cp, struct in_addr *inp);		|	|


inet_ntoa()	在内部使用了static 数组存储IP，有两个问题：

1.在一个语句中两次以上调用 inet_ntoa ，会输出同样的IP地址，如：
	printf("src=%s, dst=%s\n", inet_ntoa(src), inet_ntoa(dst));
	
2.经常出现[段错误](http://stackoverflow.com/questions/15635803/segmentation-fault-for-inet-ntoa) 

**推荐使用inet_pton/inet_ntop**

##网络地址sockaddr_in

		#include <netinet/in.h>
		   
		struct sockaddr_in {
			short            sin_family;   	
			unsigned short   sin_port; 	
			struct in_addr   sin_addr;     	
			char             sin_zero[8];  // 填充对齐
		};
		
		typedef uint32_t in_addr_t;

        struct in_addr {
            in_addr_t s_addr;
        };

		struct sockaddr {
               sa_family_t sa_family;
               char        sa_data[14];
           };

`sockaddr` 的作用？

>  int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);  
	The only purpose of sockaddr is to cast the structure pointer passed in addr in order to avoid compiler warnings. 

bind()作为一个标准接口，不仅接受Network socket address参数，还有下面两种，传参时避免编译警告。
		   
- 用于进程间通信的 UNIX socket address 结构是


  		struct sockaddr_un {
        sa_family_t sun_family;               /* AF_UNIX */
        char        sun_path[108];            /* pathname */
      };
		
		
示例： https://goo.gl/XpBgoh
		
- 用于内核与用户空间通信的Netlink socket address结构是

      struct sockaddr_nl {
        sa_family_t     nl_family;  /* AF_NETLINK */
        unsigned short  nl_pad;     /* Zero. */
        pid_t           nl_pid;     /* Port ID. */
        __u32           nl_groups;  /* Multicast groups mask. */
      };

##示例程序

- inet_aton

<pre>
    struct sockaddr_in myaddr;
    int s;
    
    myaddr.sin_family = AF_INET;
    myaddr.sin_port = htons(3490);
    inet_aton("63.161.169.137", &myaddr.sin_addr.s_addr);
    
    s = socket(PF_INET, SOCK_STREAM, 0);
    bind(s, (struct sockaddr*)myaddr, sizeof(myaddr));
</pre>

- inet_ntoa


       #define _BSD_SOURCE
       #include <arpa/inet.h>
       #include <stdio.h>
       #include <stdlib.h>

       int
       main(int argc, char *argv[])
       {
           struct in_addr addr;

           if (argc != 2) {
               fprintf(stderr, "%s <dotted-address>\n", argv[0]);
               exit(EXIT_FAILURE);
           }

           if (inet_aton(argv[1], &addr) == 0) {
               fprintf(stderr, "Invalid address\n");
               exit(EXIT_FAILURE);
           }

           printf("%s\n", inet_ntoa(addr));
           exit(EXIT_SUCCESS);
       }



- inet_ntop & inet_pton


       #include <arpa/inet.h>
       #include <stdio.h>
       #include <stdlib.h>
       #include <string.h>

       int
       main(int argc, char *argv[])
       {
           unsigned char buf[sizeof(struct in6_addr)];
           int domain, s;
           char str[INET6_ADDRSTRLEN];

           if (argc != 3) {
               fprintf(stderr, "Usage: %s {i4|i6|<num>} string\n", argv[0]);
               exit(EXIT_FAILURE);
           }

           domain = (strcmp(argv[1], "i4") == 0) ? AF_INET :
                    (strcmp(argv[1], "i6") == 0) ? AF_INET6 : atoi(argv[1]);

           s = inet_pton(domain, argv[2], buf);
           if (s <= 0) {
               if (s == 0)
                   fprintf(stderr, "Not in presentation format");
               else
                   perror("inet_pton");
               exit(EXIT_FAILURE);
           }

           if (inet_ntop(domain, buf, str, INET6_ADDRSTRLEN) == NULL) {
               perror("inet_ntop");
               exit(EXIT_FAILURE);
           }

           printf("%s\n", str);

           exit(EXIT_SUCCESS);
       }
	   

##参考

- man7.org
