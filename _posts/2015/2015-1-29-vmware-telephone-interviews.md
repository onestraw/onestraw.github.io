---
layout: single
author_profile: true
comments: true
title: VMware电话面试
tagline: to geeksword
category : essay
tags : [面试]
---

这次VMware电话面试聊了30多分钟，主要涉及四个方面`C语言`,`Python`,`Linux`,`TCP/IP`. 详细考查知识点如下：

##1.C语言
####1.1 static
主要有两个作用

- 函数内声明的static变量，在多次调用之间保持其值，不会重新初始化；
- 全局static变量/函数对其他源文件不可见，只能在声明它的源文件中使用；

[example](http://stackoverflow.com/questions/572547/what-does-static-mean-in-a-c-program)

####1.2 extern

跨源文件共享全局变量/函数，它仅仅是用来声明，其声明的变量一定要在其它源文件中定义，而且不能有static声明；

[example](http://stackoverflow.com/questions/1433204/how-do-i-use-extern-to-share-variables-between-source-files-in-c/1433387#1433387)


####1.3 #include<>与""区别

- 用<>导入的头文件会在系统库目录里去查找这个头文件；  
- 用""表示首先在当前目录下去找个头文件,如果没找到再去系统库里去查找这个头文件;

####1.4 头文件保护
`header/include guard`避免头文件重复包含

      #ifndef FOO_BAR_BAZ_H_
      #define FOO_BAR_BAZ_H_
      
      ...
      
      #endif  // FOO_BAR_BAZ_H_

####1.5 字节对齐
有一个struct定义如下

    struct T{
      char a;
      short b;
      int c;
    };   
     
问在一个32位系统上，sizeof(struct T)返回多少？  

正确答案是8，而不是7. 详细解答见[C语言字节对齐](http://blog.csdn.net/21aspnet/article/details/6729724)
     
####1.6 递归
一个递归函数，如果没有递归出口会出现什么情况？  

如果没有递归出口，那么会导致函数陷入无尽的自我调用之中，函数调用栈中栈帧急剧增加，最终导致栈溢出/栈击穿，程序崩溃。

####1.7 函数调用栈

被调用函数：callee  
调用函数：caller  

- callee的参数
- callee的返回地址
- caller栈空间的基址EBP
- callee的局部变量
- callee的buffer
- callee保存的寄存器状态
- ...

[参考](http://www.tenouk.com/Bufferoverflowc/Bufferoverflow2a.html)

##2.Python

####2.1 tuple和list的区别
  主要区别就是tuple是不可变对象，而list是可变的
  
  > 注：还有数字和字符串，对一个变量进行两次不同的赋值，表现上是可变的，实际上是新建的对象，可以用id()查看
  
####2.2 tuple和list可以做dict的key吗
  tuple可以，而list不可以  
  dict是通过对key进行hash，来找到value. 这里就要保证同一个key的hash值是确定的，不能变来变去，即要求key是不可变对象。  
  所以tuple可以作为字典的key, list却不行。

####2.3 正则表达式
python的regex模块re，search和match的区别?  
re.match只匹配字符串的开始，而re.search匹配整个字符串

    >>> import re
    >>> p = "abc"
    >>> re.match(p,"abcdef")
    <_sre.SRE_Match object at 0x0000000002D52578>
    >>> re.match(p,"babcdef")
    >>> re.search(p,"abcdef")
    <_sre.SRE_Match object at 0x0000000002D52510>
    >>> re.search(p,"babcdef")
    <_sre.SRE_Match object at 0x0000000002D52578>
    >>> 

`re.match("abc", "abcdef")`
相当于
`re.search("^abc", "abcdef")`

####2.4 调用工具pdb
这个我没用过，参考[Python代码调试技巧](http://www.ibm.com/developerworks/cn/linux/l-cn-pythondebugger/)  


有几个月没有用Python, 好多知识点都忘了，答的不好，所以Python这块问的不多。

##3.Linux

####3.1 进程资源回收

问：用fork()创建子进程, 父进程如何等子进程结束，然后回收子进程的资源？  
答：子进程中止会向父进程发送SIGCHLD信号，父进程可以用signal()捕获该信号，然后调用wait()/waitpid()来清理子进程。
问：父进程先于子进程终止，会导致什么结果？
答：子进程会变成孤儿进程/僵尸进程，然后由init进程收养。

参考 
- [Linux 避免僵尸进程](http://www.cnblogs.com/Robotke1/archive/2013/05/07/3065188.html)
- [孤儿进程](http://zh.wikipedia.org/wiki/%E5%AD%A4%E5%84%BF%E8%BF%9B%E7%A8%8B)

####3.2 from ring3 to ring0
    Linux进程空间分为用户态和内核态两种，如何实现用户态到内核的切换？
    
    - 系统调用(trap/system call)
    - 中断(interrupt)
    - 异常(如page fault)
    
####3.3 动态链接与静态链接的区别
  
  - 静态链接是在编译期间由链接器将不同的目标文件链接起来，生成的可执行文件体积较大；Linux下静态链接库一般是*.a文件; 
  - 动态链接是在运行时进行链接，生成的可执行文件体积较小；Linux下静态链接一般是*.so文件; 
  - 静态链接生成的可执行文件在运行时不会依赖其他库，配置简单；动态链接依赖其他库，需要在运行时进行符号重定位，配置不同的so相对复杂；
  
####3.4 如何查看程序所需要的动态链接
  `ldd` 查看ldd帮助文档  
  `print shared library dependencies`
  
####3.5 如何查看一个动态链接库的函数
  `objdump -T `和`nm -D`都可以查看动态符号表
  
  - man nm: `list symbols from object files`
  - nm -D: `Display the dynamic symbols ...`
  - man objdump: `display information from object file`
  - objdump -T: `print the dynamic sybmbol table entries of the file`
  
目标文件格式分析工具: `ar, nm, objdump, objcopy, readelf`
  
####3.6 coredump文件

> coredump文件是Linux程序由于各种异常或者bug导致在运行过程中异常退出或者中止时产生的。
  coredump文件会包含了程序运行时的内存，寄存器状态，堆栈指针，内存管理信息还有各种函数调用堆栈信息等. ——[coredump详解](http://blog.csdn.net/tenfyguo/article/details/8159176)
  
  个人使用coredump文件的经历是
  `gdb ./exefile core`  
  然后用bt查看函数调用栈信息。  
   
  core文件一般是默认关闭的，即core文件大小限制为0，至少在ubuntu下是这样，需要在root下执行  
  
  `ulimit -c unlimited`
  
  注意ubuntu下 `sudo ulimit -c unlimited`是不行的。

##4.TCP/IP

####4.1 IP报头

    uint8_t 	ip_hdr_len:4
     	The header length. 
    uint8_t 	ip_version:4
     	The IP version. 
    uint8_t 	ip_tos
     	Type of Service. 
    uint16_t 	ip_len
     	IP packet length (both data and header). 
    uint16_t 	ip_id
     	Identification. 
    uint16_t 	ip_off
     	Fragment offset. 
    uint8_t 	ip_ttl
     	Time To Live. 
    uint8_t 	ip_proto
     	The type of the upper-level protocol. 
    uint16_t 	ip_chk
     	IP header checksum. 
    uint32_t 	ip_src
     	IP source address (in network format). 
    uint32_t 	ip_dst
     	IP destination address (in network format). 


####4.2 TCP建立连接
  ![三次握手](http://alpha.tmit.bme.hu/meresek/twh_small.jpg)

####4.3 TCP关闭连接
  ![四次握手](http://upload.wikimedia.org/wikipedia/commons/thumb/5/55/TCP_CLOSE.svg/390px-TCP_CLOSE.svg.png)
  
这块比较熟悉，还把TCP报头的flags说到IP报头了，囧啊。

##5.其他

- 有没有编写过驱动
- 目前在做的项目
- 我询问了下工作内容
