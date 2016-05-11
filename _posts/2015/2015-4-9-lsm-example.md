---
layout: single
author_profile: true
comments: true
title: LSM(Linux Security Module)编程实例
tagline: more secure linux
category: Linux
tags : [Linux]
---

##1. LSM 学习资料

1. 论文：Linux Security Module Framework 
2. 基于LSM的模块：SELinux, Smack, Tomoyo, Apparmor, Yama
2. [Linux 安全模块（LSM）简介](https://www.ibm.com/developerworks/cn/linux/l-lsm/part1/)
3. [Linux Security](http://elinux.org/Security)


##2. LSM 简介

LSM 是Linux内核的一个轻量级通用访问控制框架。用户可以根据其需求选择适合的安全模块加载到Linux内核中，从而大大提高了Linux安全访问控制机制的灵活性和易用性。   
LSM 增强Linux安全的方式如下：  
<img src="/assets/images/LSM-Hook-Architecture.PNG" />  

LSM不干扰原先的安全策略，它通过hook技术执行增加一层（访问）控制策略，也就是在通往内核的大道上加了一个关卡，这个关卡如果没有哨兵就是空架子（对应于LSM没有加载任何安全机制）。  

##3. LSM 编程框架

编写LSM安全模块主要有两步：  
**1. 填写security_operations; 2. 注册security_operations**


LSM的核心是Hook技术，那就先看一个Hook的实例——task_create钩子。  

1. 用户创建进程需要调用系统调用 `sys_fork()/sys_vfork()/sys_clone()`
2. sys_fork()等函数中调用 `do_fork()`
3. do_fork()调用`copy_process()`
3. copy_process()调用`security_task_create()`，**从这里开始向下就是进行额外的安全检查了**
4. security_task_create()的主体就是`security_ops->task_create()`
5. securite_ops是一个全局的`struct security_operations`变量

**Tip 1**: 查找hook点，可以用[内核源码交叉引用网站](http://lxr.free-electrons.com/)快速定位。  

LSM的核心是Hook技术，struct security_operations是实现Hook技术的核心，即LSM核心中的核心！[security_operations](http://lxr.free-electrons.com/source/include/linux/security.h#L192)结构体中除了一个名字，成员全是函数指针（hook）, 分为16类钩子:  

    struct security_operations{
        Security hooks for program execution operations
        Security hooks for filesystem operations
        Security hooks for inode operations
        Security hooks for file operations
        Security hooks for task operations
        Security hooks for Netlink messaging
        Security hooks for Unix domain networking
        Security hooks for socket operations
        Security hooks for XFRM operations
        Security hooks affecting all Key Management operations
        Security hooks affecting all System V IPC operations
        Security hooks for individual messages held in System V IPC message queues
        Security hooks for System V IPC Message Queues
        Security hooks for System V Shared Memory Segments
        Security hooks for System V Semaphores
        Security hooks for Audit
    }

**填写security_operations**就是根据其成员函数指针的形式（参数，返回值），编写相应的权限检查函数hook_func，然后将hook_func赋值于自定义security_operations: geek_ops.  
**注册security_operations**就是将填好的geek_ops通过register_operations注册进内核，即**将geek_ops赋值于内核全局变量security_ops**。  
  
##4. LSM 编程实例

LSM从内核v2.6.24开始，就不能以LKM的形式动态加载进内核，只能在编译的时候就编译进内核.    

下面编写一个统计创建进程个数的模块geek，使用的是task_create()钩子。  

- 内核版本：3.13.0
- 模仿：Yama

需要在内核源码根目录中的security目录下创建geek，它包含三个文件

- geek
  - geek_lsm.c
  - Makefile
  - Kconfig


并且修改security目录下的Makefile和Kconfig文件。  

`geek_lsm.c`

      #include<linux/security.h>
      #include<linux/sysctl.h>
      static unsigned long long count = 0;
      int task_create_hook(unsigned long clone_flags)
      {
      	printk("[+geek] call task_create(). count=%llu\n", ++count);	
      	return 0;
      }
      
      static struct security_operations geek_ops = {
      	.name = "geek",
      	.task_create = task_create_hook,
      };
      
      static __init int geek_init(void)
      {
      	printk("[+geek] loading...\n");	
      	if(register_security(&geek_ops)){
      		printk("[+geek] register faild\n");	
      	}
      	return 0;
      }
      
      security_initcall(geek_init);


`Makefile`

    obj-$(CONFIG_SECURITY_GEEK) := geek.o
    geek-y := geek_lsm.o

`Kconfig`
    
    config     SECURITY_GEEK
       bool     "geek support"
       depends on SECURITY
       select     SECURITYFS
       select     SECURITY_PATH
       default     n
       help
            introduction of geek modules

修改在security目录下的Makefile，在Yama相应的条目下添加对应的项

    subdir-$(CONFIG_SECURITY_GEEK)     += geek
    obj-$(CONFIG_SECURITY_GEEK)        += geek/built-in.o

修改在security目录下的Kconifg，在Yama相应的条目下添加对应的项

    source security/geek/Kconfig
    default DEFAULT_SECURITY_GEEK if SECURITY_GEEK
    config DEFAULT_SECURITY_GEEK
         bool "Geek" if SECURITY_GEEK=y
    default "geek" if DEFAULT_SECURITY_GEEK

最后[编译内核](http://onestraw.net/linux/compile-linux-kernel)，make menuconfig时添加上GEEK模块，编译后重启就成功加载了geek模块。

      onestraw@ubuntu:~$ dmesg |tail -10
      [   52.407228] [+geek] call task_create(). count=2799
      [   52.412253] [+geek] call task_create(). count=2800
      [   52.415083] [+geek] call task_create(). count=2801
      [   52.418165] [+geek] call task_create(). count=2802
      [   54.407308] [+geek] call task_create(). count=2803
      [   54.424693] [+geek] call task_create(). count=2804
      [   54.449111] [+geek] call task_create(). count=2805
      [   54.666831] [+geek] call task_create(). count=2806
      [   55.386112] [+geek] call task_create(). count=2807
      [   55.386661] [+geek] call task_create(). count=2808


**Tip 2**: 写的钩子函数一定要有返回值，因为调用函数要判断LSM hooks是否允许该操作。
  像task_create这种创建进程的hook如果不允许，那么系统根本无法启动，因为连第一个进程都无法创建。
  
  
##5. 内核Makefile和Kconfig相关资料

在变量$(CONFIG_SECURITY_GEEK)定义在内核源码顶层目录下的.config中，通过make *config配置完内核后，会生成.config，对于每个模块可以选择y直接编译进内核，选择n不编译，选择编译成LKM，减小vmlinux大小。

上面的$(CONFIG_SECURITY_GEEK)也就是subdir-y, obj-y。

    Kbuild compiles all the $(obj-y) files.  It then calls
    "$(LD) -r" to merge these files into one built-in.o file.
    built-in.o is later linked into vmlinux by the parent Makefile.

1. [Makefile编写规范](https://www.kernel.org/doc/Documentation/kbuild/makefiles.txt)
2. [Kconfig编写规范](https://www.kernel.org/doc/Documentation/kbuild/kconfig.txt)
3. [Kbuild: the Linux Kernel Build System](http://www.linuxjournal.com/content/kbuild-linux-kernel-build-system)
