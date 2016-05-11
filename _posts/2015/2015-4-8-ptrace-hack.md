---
layout: single
author_profile: true
comments: true
title: ptrace攻防对抗
tagline: Yama安全模块
category: Linux
tags : [Linux]
---
本文内容包括

1. ptrace简介
2. ptrace漏洞
3. ptrace保护——Yama
4. Yama实战

##1. ptrace简介

------

强大的调试工具`gdb`，系统调用和信号跟踪工具`strace`都是用ptrace实现的！  

ptrace是一个系统调用，它提供了一种方法来让‘父’进程可以观察和控制其它进程的执行，检查和改变其核心映像以及寄存器。 主要用来实现断点调试和系统调用跟踪。  
  
ptrace会在什么时候出现呢？在执行系统调用之前，内核会先检查当前进程是否处于被“跟踪”(traced)的状态。如果是的话，内核暂停当前进程并将控制权交给跟踪进程，使跟踪进程得以察看或者修改被跟踪进程的寄存器。

##2. ptrace漏洞

------

利用ptrace可以

1. 劫持另一个进程的系统调用，修改传入参数，返回值
2. 向另一个进程注入代码，获得一个进程的eip,esp后，建立自己的栈空间，并使eip指向自己的代码

黑客主要利用ptrace进行提权，在exploit-db中搜索`ptrace`找到19枚EXP/POC，例如：  

|漏洞编号|漏洞名字|exploit下载|
|:---|:---|:---|
|CVE-2003-0127 |    Linux Kernel 2.2.x - 2.4.x ptrace/kmod Local Root Vulnerability   |http://www.exploit-db.com/exploits/3/|
|CVE-2007-4573  |   Linux Kernel 2.6.x - Ptrace Local Privilege Escalation Vulnerability| http://www.exploit-db.com/exploits/30604/|
|CVE-2013-2171   |  FreeBSD 9.0-9.1 mmap/ptrace - Privilege Esclation Vulnerability   |http://www.exploit-db.com/exploits/26368/ |    
|CVE-2014-4699    | Linux Kernel - ptrace/sysret - Local Privilege Escalation   |http://www.exploit-db.com/exploits/34134/  |


##3. ptrace保护——Yama

------

Yama is a Linux Security Module that collects a number of system-wide DAC security protections that are not handled by the core kernel itself.

Yama源码位于security/yama/，它基于LSM实现，管理员可以通过/proc/sys/kernel/yama/ptrace_scope来配置ptrace保护级别，配置方法有3种：

- echo "1">/proc/sys/kernel/yama/ptrace_scope ([深入浅出/proc](http://onestraw.net/linux/linux-procfs/)介绍过的)
- sysctl -w kernel.yama.ptrace_scope="1"  && sysctl -p
- 修改/etc/sysctl.conf

根据http://lxr.free-electrons.com/的查询结果
Yama在v3.4进入内核，v3.4中kernel.yama.ptrace_scope只有两个值0和1，到v3.5时，kernel.yama.ptrace_scope扩展到四个值0-3

      #define YAMA_SCOPE_DISABLED     0     
      #define YAMA_SCOPE_RELATIONAL   1
      #define YAMA_SCOPE_CAPABILITY   2
      #define YAMA_SCOPE_NO_ATTACH    3

- 0: classic ptrace permissions，正如宏定义中DISABLED，即Yama不起作用。
- 1: restricted ptrace, 正如宏定义使用的名字RELATIONAL，进程可以跟踪有血缘关系（后代）的进程。
- 2: admin-only, 拥有CAP_SYS_PTRACE能力的进程才可以使用ptrace.
- 3: no attach, 不允许任何进程执行ptrace，一旦升级为3，就不能再降级，即使root也不行。

！！注意，0-2对于root没有任何限制


##4. Yama实战

------
两个测试程序：被跟踪程序tracee.c和跟踪程序tracer.c   

      onestraw@ubuntu:~/code/apue$ cat tracee.c 
      #include <stdio.h>
      //#include <unistd.h>
      
      int main()
      {
      	while(1)
      	{
      		sleep(20);
      		static int i = 0;
      //		printf("i=%d\n", i++);	
      	}
      	return 0;
      }
      onestraw@ubuntu:~/code/apue$ cat tracer.c 
      #include <sys/ptrace.h>
      #include <sys/types.h>
      #include <sys/wait.h>
      #include <sys/reg.h>
      #include <sys/user.h>		   
      #include <unistd.h>
      
      int main(int argc, char *argv[])
      {
      	int ret = 0;
      	long data = 0; 
      	pid_t apid = atoi(argv[1]);	//attached process 
      	struct user_regs_struct regs;
      	
      	printf("tracer pid: %ld\ntracee pid: %ld \n", getpid(), apid); 
      	if((ret = ptrace(PTRACE_ATTACH, apid, NULL, NULL)) < 0){
      		printf("attach %s error\n", argv[1]);
      		exit(-1);
      	}
      	wait(NULL);
      	ptrace(PTRACE_GETREGS, apid, NULL, &regs);
      	data = ptrace(PTRACE_PEEKTEXT, apid, regs.eip, NULL);
      
      	printf("eip=0x%lx, $eip=0x%lx\n", regs.eip, data);
      	
      	ptrace(PTRACE_DETACH, apid, NULL, NULL);
      	return 0;
      }
      onestraw@ubuntu:~/code/apue$ 


------

实验中除了用tracer跟踪外，还使用strace工具跟踪，标准输出和dmesg会打印Yama防护信息.
如果违反Yama保护规则，tracer会在标准输出打印

    attach pid error
    
strace则会在标准输出打印

    attach: ptrace(PTRACE_ATTACH, ...): Operation not permitted
    Could not attach to process.  If your uid matches the uid of the target
    process, check the setting of /proc/sys/kernel/yama/ptrace_scope, or try
    again as the root user.  For more details, see /etc/sysctl.d/10-ptrace.conf

Yama会在/var/log/syslog中记录tracer和strace的失败操作，格式如下

      [ 1940.823365] ptrace of pid 2995 was attempted by: strace (pid 3282)

下面分别对Yama对ptrace的保护级别（0-3）进行测试

      ##/proc/sys/kernel/yama/ptrace_scope设置为1
      onestraw@ubuntu:~/code/apue$ cat /proc/sys/kernel/yama/ptrace_scope 
      0
      onestraw@ubuntu:~/code/apue$ ./tracee &
      [1] 2946
      onestraw@ubuntu:~/code/apue$ ./tracer 2946
      tracer pid: 2948
      tracee pid: 2946 
      eip=0xb7715424, $eip=0xc3595a5d
      onestraw@ubuntu:~/code/apue$ strace -p 2946
      Process 2946 attached - interrupt to quit
      restart_syscall(<... resuming interrupted call ...>) = 0
      rt_sigprocmask(SIG_BLOCK, [CHLD], [], 8) = 0
      rt_sigaction(SIGCHLD, NULL, {SIG_DFL, [], 0}, 8) = 0
      rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
      nanosleep({20, 0}, 
      ^C <unfinished ...>
      Process 2946 detached
      onestraw@ubuntu:~/code/apue$ kill 2946
      [1]+  Terminated              ./tracee
      
      
      
      ##/proc/sys/kernel/yama/ptrace_scope设置为1
      onestraw@ubuntu:~/code/apue# cat /proc/sys/kernel/yama/ptrace_scope 
      1
      onestraw@ubuntu:~/code/apue$ ./tracee &
      [1] 2954
      onestraw@ubuntu:~/code/apue$ ./tracer 2954
      tracer pid: 2955
      tracee pid: 2954 
      attach 2954 error
      onestraw@ubuntu:~/code/apue$ strace -p 2954
      attach: ptrace(PTRACE_ATTACH, ...): Operation not permitted
      Could not attach to process.  If your uid matches the uid of the target
      process, check the setting of /proc/sys/kernel/yama/ptrace_scope, or try
      again as the root user.  For more details, see /etc/sysctl.d/10-ptrace.conf
      onestraw@ubuntu:~/code/apue$ dmesg |tail -2
      [  353.395278] ptrace of pid 2954 was attempted by: tracer (pid 2955)
      [  359.061381] ptrace of pid 2954 was attempted by: strace (pid 2956)
      onestraw@ubuntu:~/code/apue$ su
      Password: 
      root@ubuntu:/home/onestraw/code/apue# strace -p 2954
      Process 2954 attached - interrupt to quit
      restart_syscall(<... resuming interrupted call ...>
      ^C <unfinished ...>
      Process 2954 detached
      root@ubuntu:/home/onestraw/code/apue# ./tracer 2954
      tracer pid: 2982
      tracee pid: 2954 
      eip=0xb777e424, $eip=0xc3595a5d
      
      
      
      ##/proc/sys/kernel/yama/ptrace_scope设置为2
      onestraw@ubuntu:~/code/apue# cat /proc/sys/kernel/yama/ptrace_scope 
      2
      onestraw@ubuntu:~/code/apue$ ./tracee &
      [1] 2995
      onestraw@ubuntu:~/code/apue$ su
      Password: 
      root@ubuntu:/home/onestraw/code/apue# ./tracer 2995
      tracer pid: 3020
      tracee pid: 2995 
      eip=0xb775a424, $eip=0xc3595a5d
      root@ubuntu:/home/onestraw/code/apue# strace -p 2995
      Process 2995 attached - interrupt to quit
      restart_syscall(<... resuming interrupted call ...>
      ^C <unfinished ...>
      Process 2995 detached
      root@ubuntu:/home/onestraw/code/apue# exit					##切换回onestraw用户
      exit
      onestraw@ubuntu:~/code/apue$ strace -p 2995
      attach: ptrace(PTRACE_ATTACH, ...): Operation not permitted
      Could not attach to process.  If your uid matches the uid of the target
      process, check the setting of /proc/sys/kernel/yama/ptrace_scope, or try
      again as the root user.  For more details, see /etc/sysctl.d/10-ptrace.conf
      onestraw@ubuntu:~/code/apue$ dmesg|tail -1
      [  564.859302] ptrace of pid 2995 was attempted by: strace (pid 3029)
      onestraw@ubuntu:~/code/apue$ 
      onestraw@ubuntu:~/code/apue$ sudo setcap CAP_SYS_PTRACE=ep /usr/bin/strace 			##给strace添加CAP_SYS_PTRACE 能力
      onestraw@ubuntu:~/code/apue$ strace -p 2995
      Process 2995 attached - interrupt to quit
      restart_syscall(<... resuming interrupted call ...>^C <unfinished ...>
      Process 2995 detached
      onestraw@ubuntu:~/code/apue$ sudo setcap CAP_SYS_PTRACE=ep tracer			##给./tracer添加CAP_SYS_PTRACE 能力
      onestraw@ubuntu:~/code/apue$ ./tracer 2995
      tracer pid: 3247
      tracee pid: 2995 
      eip=0xb775a424, $eip=0xc3595a5d
      onestraw@ubuntu:~/code/apue$ 
      
      
      
      ##/proc/sys/kernel/yama/ptrace_scope设置为3
      onestraw@ubuntu:~/code/apue$ cat /proc/sys/kernel/yama/ptrace_scope 
      3
      onestraw@ubuntu:~/code/apue$ ./tracer 2995
      tracer pid: 3281
      tracee pid: 2995 
      attach 2995 error
      onestraw@ubuntu:~/code/apue$ strace -p 2995
      attach: ptrace(PTRACE_ATTACH, ...): Operation not permitted
      Could not attach to process.  If your uid matches the uid of the target
      process, check the setting of /proc/sys/kernel/yama/ptrace_scope, or try
      again as the root user.  For more details, see /etc/sysctl.d/10-ptrace.conf
      onestraw@ubuntu:~/code/apue$ dmesg|tail -2
      [ 1936.268513] ptrace of pid 2995 was attempted by: tracer (pid 3281)
      [ 1940.823365] ptrace of pid 2995 was attempted by: strace (pid 3282)
      root@ubuntu:/home/onestraw/code/apue# cat /proc/sys/kernel/yama/ptrace_scope 
      3
      root@ubuntu:/home/onestraw/code/apue# echo "0" > /proc/sys/kernel/yama/ptrace_scope 	##尝试降低ptrace的保护能力
      bash: echo: write error: Invalid argument
      root@ubuntu:/home/onestraw/code/apue# echo "1" > /proc/sys/kernel/yama/ptrace_scope 
      bash: echo: write error: Invalid argument
      root@ubuntu:/home/onestraw/code/apue# echo "2" > /proc/sys/kernel/yama/ptrace_scope 
      bash: echo: write error: Invalid argument
      root@ubuntu:/home/onestraw/code/apue#


1. 当ptrace_scope为1时，onestraw用户跟踪失败是因为tracer和tracee进程没有父子关系, root用户不理会此条件；
2. 当ptrace_scope为2时，onestraw用户跟踪失败是因为tracer和tracee进程没有CAP_SYS_PTRACE能力，加上该能力之后就可以跟踪，root用户可以获取任何能力，所以不理会该限制；
3. 当ptrace_scope为3时，onestraw和root用户trace失败是因为Yama不允许attach，对于所有用户均禁止attach；

##相关资料

---

1. [ptrace运行原理及使用详解](http://blog.csdn.net/hmsiwtv/article/details/11022241)
2. [ptrace函数详解](http://www.cnblogs.com/eddy-he/archive/2012/03/08/linux_ptrace.html)
3. [ptrace(): A Linux Virus](http://www.exploit-db.com/papers/13061/)
4. [Playing with ptrace, Part I](http://www.linuxjournal.com/article/6100)
4. [Playing with ptrace, Part II](http://www.linuxjournal.com/article/6210)
5. [Yama.txt](https://www.kernel.org/doc/Documentation/security/Yama.txt)
6. [Linux的capability深入分析](http://www.cnblogs.com/iamfy/archive/2012/09/20/2694977.html)
