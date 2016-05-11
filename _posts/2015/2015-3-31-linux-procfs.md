---
layout: single
author_profile: true
comments: true
title: 深入浅出 /proc
tagline: procfs
category: Linux
tags : [Linux]
---


Linux下的proc目录有什么用途，我们查看内核版本号、加载的模块、CPU和内存使用等信息，甚至[修改路由转发](http://onestraw.net/cybersecurity/arpspoof-and-dnsspoof/)都是通过/proc目录进行的，本文在揭示/proc本质的基础上，对比了/proc编程的新老接口，以及一个应用实例。  

`以下均在ubuntu 12.04 LTS上实验`  

##1. 体验/proc 

----

|目的|命令|
|:---|:---|
|查看内核符号表,用kprobes时会用到|     cat /proc/kallsyms  |        
|查看内核版本号 |    cat /proc/version|
|查看加载的模块 |    cat /proc/modules|
|查看可用设备的列表 |cat /proc/devices|
|查看CPU 的信息 (型号,家族,缓存)|cat /proc/cpuinfo|
|查看物理内存、交换空间等的信息|cat /proc/meminfo|
|查看已加载的文件系统的列表 |cat /proc/mounts|
|查看被支持的文件系统 |cat /proc/filesystems|
|查看系统启动时内核命令行参数（grub.cfg, menu.lst） |cat /proc/cmdline|
|查看socket状态 |    cat /proc/net/sockstat|
|查看arp表      |    cat /proc/net/arp|
|开启路由转发   |  echo 1 >/proc/sys/net/ipv4/ip_forward|


看一下/proc/filesystems和/proc/mounts的内容

`cat /proc/filesystems`

    nodev	sysfs
    nodev	rootfs
    nodev	ramfs
    nodev	bdev
    nodev	proc
    .....
        	ext3
        	ext2
        	ext4


`cat /proc/mounts`

    rootfs / rootfs rw 0 0
    sysfs /sys sysfs rw,nosuid,nodev,noexec,relatime 0 0
    proc /proc proc rw,nosuid,nodev,noexec,relatime 0 0
    ...

种种迹象表明proc是一个文件系统，YES(下一节)。此外，mounts文件中的nosuid, nodev, noexec是什么意思呢？下面深挖一下mount命令及选项

---------------------------

- 功能：加载指定的文件系统。 
- 语法：mount [-afFhnrvVw] [-L<标签>] [-o<选项>] [-t<文件系统类型>] [设备名] [加载点] 
- 举例

        mount -t sysfs -o nodev,noexec,nosuid none /sys
        mount -t proc -o nodev,noexec,nosuid none /proc 

- 选项    

        -t<文件系统类型> 指定设备的文件系统类型。sysfs
        
        -o<选项> 指定加载文件系统时的选项。有些选项也可在/etc/fstab中使用。这些选项包括： 
        ....
        nodev 不读文件系统上的字符或块设备。 
        noexec 无法执行二进制文件。 
        nosuid 关闭set-user-identifier(设置用户ID)与set-group-identifer(设置组ID)设置位。 
        nouser 使一位用户无法执行加载操作，默认设置。 
        ....
        none是设备名，/sys是挂载点



##2. /proc本质

------

`/proc`目录是一个文件系统，准确的说是一个伪文件系统（pseudo-filesystem
），该目录中的所有文件都不会消耗磁盘空间. 它提供了一个**用户空间和内核空间通信的接口**，可以用类似`cat /proc/meminfo`读取内核信息，也可以使用类似`echo 1 >>/proc/sys/net/ipv4/ip_forward`向内核写消息

最初procfs是为了提供有关系统中进程（`proc`ess）的信息，后来内核中的很多模块也开始使用它来报告信息，或启用动态运行时配置。

##3. demoproc模块

- 目标：实现一个模块，能够在用户空间（Terminal）通过/proc/demoproc和模块通信; 可以用echo 命令向模块发送数据，使用cat命令读取模块数据；  
- demoproc.c

        #include <linux/module.h>
        #include <linux/kernel.h>
        #include <linux/proc_fs.h>
        #include <linux/string.h>
        #include <linux/vmalloc.h>
        #include <asm/uaccess.h>
        
        MODULE_LICENSE("GPL");
        MODULE_DESCRIPTION("proc demo");
        MODULE_AUTHOR("geeksword");
        
        #define MAX_BUFF_SIZE       2048
        static char *sbuff;
        static char *entry_name = "demoproc";
        struct proc_dir_entry *proc_entry;
        
        static ssize_t demoproc_write(struct file *filp, const char __user * buff,
        			      size_t len, loff_t * offset)
        {
        	printk(KERN_INFO "[+demoproc] call demoproc_write()\n");
        	printk(KERN_INFO "[+demoproc] write:%s\n", buff);
        	len = len < MAX_BUFF_SIZE ? len : MAX_BUFF_SIZE;
        
        	if (copy_from_user(sbuff, buff, len)) {
        		printk(KERN_INFO "[+demoproc]: copy_from_user() error!\n");
        		return -EFAULT;
        	}
        	return len;
        }
        
        static ssize_t demoproc_read(struct file *filp, char __user * buff,
        			     size_t len, loff_t * offset)
        {
        	printk(KERN_INFO "[+demoproc] call demoproc_read()\n");
        	len = strlen(sbuff);
        	if (*offset >= len) {
        		return 0;
        	}
        	len -= *offset;
        	if (copy_to_user(buff, sbuff + *offset, len)) {
        		return -EFAULT;
        	}
        	*offset += len;
        	return len;
        }
        
        static const struct file_operations demoproc_fops = {
        	.owner = THIS_MODULE,
        	.write = demoproc_write,
        	.read = demoproc_read,
        };
        
        int init_demoproc_module(void)
        {
        	int ret = 0;
        	sbuff = (char *)vmalloc(MAX_BUFF_SIZE);
        	if (!sbuff) {
        		ret = -ENOMEM;
        	} else {
        		memset(sbuff, 0, MAX_BUFF_SIZE);
        		proc_entry =
        		    proc_create(entry_name, 0644, NULL, &demoproc_fops);
        		printk(KERN_INFO "[+demoproc] Module loaded.\n");
        	}
        	return ret;
        }
        
        void cleanup_demoproc_module(void)
        {
        	proc_remove(proc_entry);
        	vfree(sbuff);
        	printk(KERN_INFO "[+demoproc] Module removed.\n");
        }
        
        module_init(init_demoproc_module);
        module_exit(cleanup_demoproc_module);


##4. demoproc分析
关于基本的模块编程[移步](http://onestraw.net/linux/lkm-and-syscall-hook)  

1. 使用vmalloc申请一块内存，用于存放用户空间写入的数据
2. 用proc_create创建一个proc_dir_entry，即在/proc目录下创建一个文件
3. proc_dir_entry在创建时指定了名字`demoproc`，权限`0644`和file_operations`demoproc_fops`
4. 关于读写操作都通过demoproc_fops进行关联，即demoproc_read()和demoproc_write()
5. 在移除模块时，用proc_remove删除proc_dir_entry，并释放内存

**proc新老接口的对比**   

根据lxr.free-electrons.com的查询结果：
create_proc_entry()从linux3.10就没有了，ubuntu12.04 LTS的内核版本是3.13

|内核版本|<= v3.9| >= v3.10|
|:---|:---|:---|
|创建proc|create_proc_entry() |    proc_create()|
|移除proc|remove_proc_entry()  |   proc_remove()|
|关联文件操作|proc_dir_entry   |  file_operations|


在编写demoproc_read()的时候遇到一个有意思的问题

`struct file_operations`

      struct file_operations {
           struct module *owner;
           loff_t (*llseek) (struct file *, loff_t, int);
           ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
           ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
           ....
           }
  
  根据read函数指针编写demoproc_read, 一直不明白loff_t *指针(其实是long long *)的作用  。
  
  在函数[simple_read_from_buffer()](http://lxr.free-electrons.com/source/fs/libfs.c#L590)中loff_t*ppos标识buffer当前位置。
  我们通过实验来探索一下，将demoproc_read函数改成如下
  
        static ssize_t demoproc_read(struct file *filp, char __user * buff,
      			     size_t len, loff_t * offset)
        {
        	printk(KERN_INFO "[+demoproc] call demoproc_read()\n");
        	len = strlen(sbuff);
        	if (copy_to_user(buff, sbuff, len)) {
        		return -EFAULT;
        	}
        	return len;
        }
  
  编译安装后， 如果在终端输入
  
      echo "hello proc">/proc/demoproc
      cat /proc/demoproc
      
  结果将是终端无穷无尽的输出`hello proc`!   
  
  原因是demoproc_read()函数被反复调用，直到它返回0，也就是读取字节数为0（全部读完）。   
  
  如果将`return len`改成`return 0`，执行cat /proc/demoproc时将无任何输出。   
  
  所以我就使用这个参数loff_t*offset标识读取的数据在全局缓冲区sbuff的读取位置，
  仔细分析demoproc.c，你会判断出每次demoproc_read都会执行两次，第一次读取所有的数据，第二次读取0个字符，然后结束，通过`dmesg|tail`证实你的推断。   
  
  这和我们编写应用程序时，函数传地址参数是一样的，只不过这里的调用函数未知。
  
##5. 参考

----

- [proc内核文档](https://www.kernel.org/doc/Documentation/filesystems/proc.txt)
- [proc目录详细说明](http://man7.org/linux/man-pages/man5/proc.5.html)
- [mount命令](http://blog.chinaunix.net/uid-26683523-id-3074744.html)
- [使用 /proc 文件系统来访问 Linux 内核的内容](http://www.ibm.com/developerworks/cn/linux/l-proc.html)
- [使用procfs](http://x-slam.com/write_linux_kernel_module_use_procfs)
