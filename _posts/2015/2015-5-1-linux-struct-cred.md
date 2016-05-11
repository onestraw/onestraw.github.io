---
layout: single
author_profile: true
comments: true
title: Linux 进程安全上下文 struct cred
tagline:  
category: Linux
tags : [Linux, 系统安全]
---

在学习LSM过程中，发现有的系统为实现特定功能，需要在进程上附加自定义的信息，其中一个系统laminar基于内核2.6，定义一个新的
`struct task_security_struct`，然后挂接到`task_struct`的`void *security`指针上，security指针是LSM框架的辅助信息。

内核3.x 在task_struct找不到security成员了，原来是将安全相关的信息剥离到一个叫做 cred 的结构体中，由cred负责保存进程安全上下文，
`struct cred`就是本文的主题，从这里领略一下Linux安全机制的概况：DAC, setuid, capabilities, namespace, LSM.

#struct cred

---------

       93 /*
       94  * The security context of a task
       95  *
       96  * The parts of the context break down into two categories:
       97  *
       98  *  (1) The objective context of a task.  These parts are used when some other
       99  *      task is attempting to affect this one.
      100  *
      101  *  (2) The subjective context.  These details are used when the task is acting
      102  *      upon another object, be that a file, a task, a key or whatever.
      103  *
      104  * Note that some members of this structure belong to both categories - the
      105  * LSM security pointer for instance.
      106  *
      107  * A task has two security pointers.  task->real_cred points to the objective
      108  * context that defines that task's actual details.  The objective part of this
      109  * context is used whenever that task is acted upon.
      110  *
      111  * task->cred points to the subjective context that defines the details of how
      112  * that task is going to act upon another object.  This may be overridden
      113  * temporarily to point to another security context, but normally points to the
      114  * same context as task->real_cred.
      115  */
      116 struct cred {
      117         atomic_t        usage;
      118 #ifdef CONFIG_DEBUG_CREDENTIALS
      119         atomic_t        subscribers;    /* number of processes subscribed */
      120         void            *put_addr;
      121         unsigned        magic;
      122 #define CRED_MAGIC      0x43736564
      123 #define CRED_MAGIC_DEAD 0x44656144
      124 #endif
      125         uid_t           uid;            /* real UID of the task */
      126         gid_t           gid;            /* real GID of the task */
      127         uid_t           suid;           /* saved UID of the task */
      128         gid_t           sgid;           /* saved GID of the task */
      129         uid_t           euid;           /* effective UID of the task */
      130         gid_t           egid;           /* effective GID of the task */
      131         uid_t           fsuid;          /* UID for VFS ops */
      132         gid_t           fsgid;          /* GID for VFS ops */
      133         unsigned        securebits;     /* SUID-less security management */
      134         kernel_cap_t    cap_inheritable; /* caps our children can inherit */
      135         kernel_cap_t    cap_permitted;  /* caps we're permitted */
      136         kernel_cap_t    cap_effective;  /* caps we can actually use */
      137         kernel_cap_t    cap_bset;       /* capability bounding set */
      138 #ifdef CONFIG_KEYS
      139         unsigned char   jit_keyring;    /* default keyring to attach requested
      140                                          * keys to */
      141         struct key      *thread_keyring; /* keyring private to this thread */
      142         struct key      *request_key_auth; /* assumed request_key authority */
      143         struct thread_group_cred *tgcred; /* thread-group shared credentials */
      144 #endif
      145 #ifdef CONFIG_SECURITY
      146         void            *security;      /* subjective LSM security */
      147 #endif
      148         struct user_struct *user;       /* real user ID subscription */
      149         struct user_namespace *user_ns; /* cached user->user_ns */
      150         struct group_info *group_info;  /* supplementary groups for euid/fsgid */
      151         struct rcu_head rcu;            /* RCU deletion hook */
      152 };

[FROM HERE](http://lxr.free-electrons.com/source/include/linux/cred.h?v=3.4#L93)

注意两点

1. security的编译条件CONFIG_SECURITY
2. 正如uid,euid的关系一样，task_struct也有两种身份cred

        struct task_struct{
        ...
        /* process credentials */
        const struct cred __rcu *real_cred; /* objective and real subjective task credentials (COW) */
        const struct cred __rcu *cred;  /* effective (overridable) subjective task credentials (COW) */
        ...
        }

struct inode也有类似的安全域`i_security`

#各种id

---------

uid 和 gid 都是成双成对的，这里就拿uid说明其作用：  

- uid 是创建进程的用户的id，不是创建可执行程序的用户id
- euid 是进程运行过程中实时的ID（或者说动态获取的ID）
- suid 是保存的euid切换之前的id，用于euid切换回来 

为什么需要动态的改变id，因为普通用户需要执行特权程序，你可能会说执行特权程序加sudo 啊，
但是sudo本身就是特权程序，执行sudo跟谁要特权啊，这样就陷入一个无限递归之中。打破这个递归的就是setuid。   

详细的学习资料见参考[UID, EUID, SUID, FSUID](http://elsila.blog.163.com/blog/static/173197158201241104049660/), 这里仅简单的介绍一下。

`setuid`

- 对于可执行文件，执行时拥有其创建者（属主）的权限（uid）
- 对于目录无效。

`setgid`

- 对于可执行文件，执行时拥有其创建者（属组）的组权限（gid）
- 对于目录dir，任何用户在设置了setgid后，在该目录dir中创建的文件的gid都和dir的gid相同。因为目录的可执行权限x的含义是是否能进入该目录，所以如果目录设置了s位，那么用户进入该目录就拥有了其创建者的身份。

`举例`

      user-1@ubuntu:~$ ll /usr/bin/sudo
      -rwsr-xr-x 2 root root 69708 Mar 12 09:35 /usr/bin/sudo*
      
      user-1@ubuntu:~$ id
      uid=1002(user-1) gid=1002(user-1) groups=1002(user-1)
      user-1@ubuntu:~$ cat outid.c 
      #include<sys/types.h>
      #include<unistd.h>
      #include<stdio.h>
      
      int main()
      {
      	printf("current euid:%d, uid:%d, egid:%d, gid:%d\n",
      		geteuid(), getuid(), getegid(), getgid());
      	return (0);	
      }
      
      user-1@ubuntu:~$ ll outid
      -rwxr-xr-x 1 root root 7315 May  1 03:42 outid*
      user-1@ubuntu:~$ ./outid 
      current euid:1002, uid:1002, egid:1002, gid:1002
      
      user-1@ubuntu:~$ sudo chmod u+s outid
      user-1@ubuntu:~$ ll outid
      -rwsr-xr-x 1 root root 7315 May  1 03:42 outid*
      user-1@ubuntu:~$ ./outid 
      current euid:0, uid:1002, egid:1002, gid:1002
      
      user-1@ubuntu:~$ sudo chmod g+s outid
      user-1@ubuntu:~$ ll outid
      -rwsr-sr-x 1 root root 7315 May  1 03:42 outid*
      user-1@ubuntu:~$ ./outid 
      current euid:0, uid:1002, egid:0, gid:1002

说到这里，另外一个不得不提的就是粘滞位`sticky bit`，对于sticky bit只需记住两点：  

- sticky只对目录有效
- 设置了sticky的目录，该目录下的文件只有3个人（文件属主、目录属主、root）可以重命令或者删除它

      When a directory's sticky bit is set, the filesystem treats the files in such directories in a special way so only the file's owner, the directory's owner, or root user can rename or delete the file. 
      --[wikipedia](http://en.wikipedia.org/wiki/Sticky_bit)


`fsuid`


      The intent of fsuid is to permit programs (e.g., the NFS server) to limit themselves to the file system rights of some given uid without giving that uid permission to send them signals. 
      --[wikipedia](http://en.wikipedia.org/wiki/User_identifier#File_system_user_ID)


感觉像是，给你uid身份，但有不给你一些权限。


#各种cap

-------------

上面说了setuid可以让普通用户执行一些程序时获得root特权，这里明显有一个很大的安全隐患，
像ping这种程序，只需要有一个创建原始socket的能力，而root身份是可以做任何事情的。  
capabilities机制将权限进行了细化，遵守`最小特权原则`。 


      user-1@ubuntu:~$ ll /bin/ping
      -rwsr-xr-x 1 root root 34740 Nov  8  2011 /bin/ping*

[能力列表](http://man7.org/linux/man-pages/man7/capabilities.7.html)   

实战  

- `setcap` set file capabilities
- `getcap` examine file capabilities


        user-1@ubuntu:~$ sudo setcap CAP_SYS_PTRACE=ep test 
        user-1@ubuntu:~$ getcap test 
        test = cap_sys_ptrace+ep


#security指针

----------

内核3.x之前放在task_struct中，之后放在task_struct->cred中。  
security指针是无类型的，所以开发者可以定义什么形式的数据，然后挂接到`task_struct->cred->security`上.  

#其它

-----------

有待进一步学习，下面是一些学习资料

###1. user_namespace

进程隔离，LXC技术

- [Docker基础技术：Linux Namespace（上）](http://coolshell.cn/articles/17010.html)
- [Docker基础技术：Linux Namespace（下）](http://coolshell.cn/articles/17029.html)

####2.keyring

- [Linux 密钥保留服务入门](http://www.ibm.com/developerworks/cn/linux/l-key-retention.html)

####3.seccomp
支持一种简单的沙箱机制

- [seccomp](https://www.kernel.org/doc/Documentation/prctl/seccomp_filter.txt)

end by geeksword   
from onestraw.net
