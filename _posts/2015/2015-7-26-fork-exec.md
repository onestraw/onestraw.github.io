---
layout: single
author_profile: true
comments: true
title: 谈谈进程创建与程序加载
tagline: 
category: Linux
tags : [Linux, 系统安全]
---

#fork

在Linux中，创建新进程使用fork/vfork/clone/kernel_thread方法，除了第一个进程pid=0，该进程是手工构造启动的，
然后由进程0创建进程1(init process)，init进程是所有进程的祖先。 也就是说后来所有的进程均有上述四个方法创建，
而这四个函数中，均调用`do_fork()->copy_process()完成进程的创建，子进程获取父进程的数据空间，堆和栈，暂时共享正文段。


#execve

通常创建新进程后，会调用一个exec函数来替换新进程的正文、数据、堆栈段，来执行新的功能。exec函数族包含6个

  - execlp
  - execvp
  - execl
  - execv 
  - execle
  - execve (system call)

所有另外5个exec函数都是库函数，只有`execve`是系统调用，是所有exec函数族的入口，这6个函数只是参数(argv, env)有所区别。


#cred
  之前写过一篇[linux进程安全上下文struct cred](http://onestraw.net/linux/linux-struct-cred/)，但是并没有清楚如何设置cred，注意
  
  
     struct task_struct{
       ...
       /* process credentials */
       const struct cred __rcu *real_cred; /* objective and real subjective task credentials (COW) */
       const struct cred __rcu *cred;  /* effective (overridable) subjective task credentials (COW) */
       ...
     }
  
  task_struct中是const，不能通过task-struct->cred直接修改。通过源码分析，好像只有在可执行程序加载的过程中通过linu_binprm来设置新进程的cred;  

  从系统调用execve 到 commit_creds 的调用过程如下
  
      
      SYSCALL_DEFINE3(execve, filename, argv, envp)
      
    	int do_execve(...)
    	
    	static int do_execveat_common(...)
    	
    	static int exec_binprm(struct linux_binprm *bprm)
    	
    	int search_binary_handler(struct linux_binprm *bprm)
    	{
    	 retval = fmt->load_binary(bprm);
    	}
    	
    	static int load_elf_binary(struct linux_binprm *bprm)
    	
    	//install the new credentials for this executable
    	void install_exec_creds(struct linux_binprm *bprm)
    	
    	//commit_creds - Install new credentials upon the current task
    	int commit_creds(struct cred *new)
    	
  
  关于cred，新创建的子进程cred是继承父进程的(do_fork)，只有当子进程加载可执行程序时(execve)，通过linux_binprm，才能获取新的cred;  
  
      linux_binprm: This structure is used to hold the arguments that are used when loading binaries.
