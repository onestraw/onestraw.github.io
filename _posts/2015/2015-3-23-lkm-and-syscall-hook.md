---
layout: single
author_profile: true
comments: true
title: Linux 模块编程和syscall hook技术
tagline: ubuntu 12.04 LTS
category: Linux
tags : [Linux]
---
{% include toc icon="gears" title="目录" %}

# 编写Linux Module基础知识
Modules是可以自由装载进内核(从内核卸载)的一块代码，不用重启系统就能扩展内核功能。设备驱动是一种module
本文所说的Linux Module是
External Modules, out-of-tree kernel module, Loadable kernel module(LKM).

#### 应用编程和内核编程对比

--------------

|   Item   |    应用编程 | 内核编程  |
| :-------- | :--------| :-- |
| 使用函数  | glibc(如printf) |  内核函数(如printk)   |
| 头文件 |  /usr/include/ | /usr/src/linux-headers-`uname -r`/include/|
| 编译 | gcc | Makefile[实例](#makefile) |
| 连接 | gcc | insmod |
| 运行| execve | insmod |
| 调试| gdb | kdb |
| 运行权限      |  普通用户 | root  |
| 运行空间     |   用户空间 |  内核空间  |
| 入口函数 | main() | module_init()/init_module() |
| 退出函数 | exit() | module_exit()/cleanup_module()|

#### Modules' Makefile

---------------

<a name="makefile"></a>

hello_kernel's makefile example

	obj-m += hello_kernel.o  

	all:  
		make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules  
	 
	clean:
		make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean

注意这个makefile中没有使用gcc编译器，用的是make，make不是编译器，它通过-C选项找到内核源码根目录下的Makefile，编译过程主要由它负责。 M选项指定外部模块的绝对路径，modules/clean	用来告诉内核Makefile是构建还是清除。具体选项说明如下

	make -C $KDIR M=$PWD [target]
	-C $KDIR
		The directory where the kernel source is located.
		"make" will actually change to the specified directory
		when executing and will change back when finished.
	M=$PWD
		Informs kbuild that an external module is being built.
		The value given to "M" is the absolute path of the
		directory where the external module (kbuild file) is
		located.	
	target
		modules
			The default target for external modules. It has the
			same functionality as if no target was specified. See
			description above. 默认是modules，可不写。
		modules_install
			Install the external module(s). The default location is
			/lib/modules/<kernel_release>/extra/, but a prefix may
			be added with INSTALL_MOD_PATH.
		clean
			Remove all generated files in the module directory only.
		help
			List the available targets for external modules.
			
[参考kernel module doc](https://www.kernel.org/doc/Documentation/kbuild/modules.txt)	



#### 几个相关命令

----------

1. insmod     装载
2. lsmod     查看
3. rmmod     卸载
4. dmesg     查看输出信息     cat /var/log/syslog
5. strace     跟踪系统调用


# 使用IDT进行syscall hooking

参考
https://github.com/ebradbury/linux-syscall-hooker


#### asmlinkage修饰符

-------------------
  
	#define asmlinkage     __attribute__((regparm(0)))  

告诉gcc编译器该函数不需要通过任何寄存器来传递参数，参数只是通过堆栈来传递。

	#define fastcall    __attribute__((regparm(3)))  

告诉gcc编译器这个函数可以通过寄存器传递多达3个的参数，这3个寄存器依次为EAX、EDX 和 ECX。



#### 系统调用的过程

---------------

当加载了系统的 C 库调用索引和参数时，就会调用一个软件中断（0x80 中断），它将执行 system_call 函数（通过中断处理程序），这个函数会按照 eax 内容中的标识处理所有的系统调用。在经过几个简单测试之后，使用 system_call_table 和 eax 中包含的索引来执行真正的系统调用了。从系统调用中返回后，最终执行 syscall_exit，并调用 resume_userspace 返回用户空间。然后继续在 C 库中执行，它将返回到用户应用程序中。
http://www.ibm.com/developerworks/cn/linux/l-system-calls/


#### Hook一个syscall的步骤

---------------------

  1. Locates the Interrupt Descriptor Table using the sidt instruction.
  2. Locates the syscall handler routine through the IDT.
  3. Locates the system call table (sys_call_table) by scanning for a known code pattern in memory in the syscall handler.
  4. Saves the state of the sys_call_table.
  5. Disables memory protection on the sys_call_table.
  6. Overwrites entries in the sys_call_table with pointers to the hooked functions.

对上面每一步进行解释

1. 第1步:   
SIDT是取IDTR寄存器的内容。

2. 第2步:  
进入系统调用时，汇编指令是int 0x80，0x80就是system_call中断服务程序在中断描述符表中的序号。
中断描述符表idt每一项8字节，两头的4个字节（0~1字节和6~7字节）保存中断服务程序的入口地址偏移，2~3两个字节是段选择符

3. 第3步:  
"call {sys_call_table address}(,%eax,4)" in memory is:  
"0xff 0x14 0x85 0x{sys_call_table address}"   
参考论文《Linux kernel rootkits: protecting the system’s “Ring-Zero”  
找到int 0x80的中断服务程序之后，内核根据eax（上层执行的系统调用函数在系统调用表syscall table里的偏移量），找到相应的系统调用，"call {sys_call_table address}(,%eax,4)"就是执行对应的系统调用。而且system_call的起始地址和该call指令的地址相距并不远，从system_call开始查找`0xff1485`特征码就能找到system_call table的内存地址。


4. 第4步:  
使用全局变量__NR_uname，即uname在system_call_table中的偏移。

5. 第5步:   
PTE: 即page table entry, 它有32bit, 前20bit表示页面的基地，后12bit是一些读写保护等标志位。  
参考 http://duartes.org/gustavo/blog/post/how-the-kernel-manages-your-memory/

# 实例: hook_mkdir

-----------

参考ebradbury的文章，我写了一个hook sys_mkdir的模块.

#### hook_mkdir.c

----
{% highlight c %}
/*
* This kernel module locates the sys_call_table by scanning
* the system_call interrupt handler (int 0x80)
*/

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/unistd.h>
#include <linux/utsname.h>
#include <asm/pgtable.h>

/*
** module macros
*/
MODULE_LICENSE("GPL");
MODULE_AUTHOR("geeksword");
MODULE_DESCRIPTION("hook sys_mkdir");

/*
** module constructor/destructor
*/
typedef void (*sys_call_ptr_t)(void);
sys_call_ptr_t *_sys_call_table = NULL;

typedef asmlinkage long (*old_mkdir_t)(const char __user *pathname, umode_t mode);
old_mkdir_t old_mkdir = NULL;

// hooked mkdir function
asmlinkage long hooked_mkdir(const char __user *pathname, umode_t mode) {
        static char* msg = "hooked sys_mkdir(), mkdir name: ";
        printk("%s%s\n", msg, pathname);
        old_mkdir(pathname, mode);
}

// memory protection shinanigans
unsigned int level;
pte_t *pte;

// initialize the module
static int hooked_init(void) {
    printk("+ Loading hook_mkdir module\n");
    
    // struct for IDT register contents
    struct desc_ptr idtr;

    // pointer to IDT table of desc structs
    gate_desc *idt_table;

    // gate struct for int 0x80
    gate_desc *system_call_gate;

    // system_call (int 0x80) offset and pointer
    unsigned int _system_call_off;
    unsigned char *_system_call_ptr;

    // temp variables for scan
    unsigned int i;
    unsigned char *off;

    // store IDT register contents directly into memory
    asm ("sidt %0" : "=m" (idtr));

    // print out location
    printk("+ IDT is at %08x\n", idtr.address);

    // set table pointer
    idt_table = (gate_desc *) idtr.address;

    // set gate_desc for int 0x80
    system_call_gate = &idt_table[0x80];

    // get int 0x80 handler offset
    _system_call_off = (system_call_gate->a & 0xffff) | (system_call_gate->b & 0xffff0000);
    _system_call_ptr = (unsigned char *) _system_call_off;

    // print out int 0x80 handler
    printk("+ system_call is at %08x\n", _system_call_off);

    // scan for known pattern(0xff1485xx) in system_call (int 0x80) handler
    // pattern is just before sys_call_table address
    for(i = 0; i < 128; i++) {
        off = _system_call_ptr + i;
        if(*(off) == 0xff && *(off+1) == 0x14 && *(off+2) == 0x85) {
            _sys_call_table = *(sys_call_ptr_t **)(off+3);
            break;
        }
    }

    // bail out if the scan came up empty
    if(_sys_call_table == NULL) {
        printk("- unable to locate sys_call_table\n");
        return 0;
    }

    // print out sys_call_table address
    printk("+ found sys_call_table at %08x!\n", _sys_call_table);

    // now we can hook syscalls ...such as uname
    // first, save the old gate (fptr)
    old_mkdir = (old_mkdir_t) _sys_call_table[__NR_mkdir];

    // unprotect sys_call_table memory page
    pte = lookup_address((unsigned long) _sys_call_table, &level);

    // change PTE to allow writing
    set_pte_atomic(pte, pte_mkwrite(*pte));

    printk("+ unprotected kernel memory page containing sys_call_table\n");

    // now overwrite the __NR_uname entry with address to our uname
    _sys_call_table[__NR_mkdir] = (sys_call_ptr_t) hooked_mkdir;

    printk("+ sys_mkdir hooked!\n");

    return 0;
}

static void hooked_exit(void) {
    if(old_mkdir != NULL) {
        // restore sys_call_table to original state
        _sys_call_table[__NR_mkdir] = (sys_call_ptr_t) old_mkdir;

        // reprotect page
        set_pte_atomic(pte, pte_clear_flags(*pte, _PAGE_RW));
    }
    
    printk("+ Unloading hook_mkdir module\n");
}

/*
** entry/exit macros
*/
module_init(hooked_init);
module_exit(hooked_exit);
{% endhighlight %}

#### Makefile

	obj-m += hook_mkdir.o
	 
	all:
		make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
	 
	clean:
		make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean

#### 测试

$make   
$insmod hook_mkdir.ko   
$mkdir testdir  
$rmmod hook_mkdir.ko  
$dmesg | tail  
	
	[32759.059318] + Loading hook_mkdir module
	[32759.059321] + IDT is at fffba000
	[32759.059322] + system_call is at c168b328
	[32759.059322] + found sys_call_table at c1697140!
	[32759.059323] + unprotected kernel memory page containing sys_call_table
	[32759.059327] + sys_mkdir hooked!
	[32823.023391] hooked sys_mkdir(), mkdir name: testdir
	[32853.324998] + Unloading hook_mkdir module
	
从倒数第二行可以看到，成功劫持了sys_mkdir().
