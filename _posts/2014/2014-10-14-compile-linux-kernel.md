---
layout: single
author_profile: true
comments: true
title: 编译Linux内核实战
tagline: from 3.2.0 to 2.6.22.6
category: Linux
tags: [Linux]
---

#任务

- 给Linux-2.6.22.6内核打上一个定制的补丁laminar-os.path，编译内核。  

#思路

- 首先对Linux-2.6.22版内核打两个补丁patch-2.6.22.6和laminar-os.patch，生成内核linux-2.6.22.6;  
- 然后编译linux-2.6.22.6;  

#打补丁

##1.下载内核

从Linus的Github上下载base kernel 2.6.22版本：[linux-2.6.22.tar.gz](https://github.com/torvalds/linux/releases/tag/v2.6.22)

##2.下载补丁
	
下载patch-2.6.22.6，在kernel.org网站上没有找到，使用google找到的一个下载地址[patch-2.6.22.6.bz2](http://pkgs.fedoraproject.org/repo/pkgs/kernel/patch-2.6.22.6.bz2/f2948e364ab3e4736b9e34f02173472f/)

##3.打补丁
	
		cd ~
		bunzip2  patch-2.6.22.6.bz2
		tar xvf linux-2.6.22.tar.gz

		cd linux-2.6.22
		patch -p1 < ../patch-2.6.22.6
		patch -p1 < ../laminar-os.patch
		cd ..
		mv  linux-2.6.22  linux-2.6.22.6
		tar -czf linux-2.6.22.6.tar.gz linux-2.6.22.6/

#编译内核
环境：32位的ubuntu12.04-desktop虚拟机  

##1.配置内核

	tar -zvf linux-2.6.22.6.tar.gz
	cd linux-2.6.22.6
	make menuconfig
  
##2.make

错误：gcc: error: elf_i386: No such file or directory   
详细信息如下：  

	arch/i386/kernel/sysenter.c: In function ‘arch_setup_additional_pages’:
	arch/i386/kernel/sysenter.c:312:11: warning: taking address of expression of type ‘void’ [enabled by default]
	  LDS     arch/i386/kernel/vsyscall.lds
	  AS      arch/i386/kernel/vsyscall-int80.o
	  AS      arch/i386/kernel/vsyscall-note.o
	  SYSCALL arch/i386/kernel/vsyscall-int80.so
	gcc: error: elf_i386: No such file or directory
	make[1]: *** [arch/i386/kernel/vsyscall-int80.so] Error 1
	make: *** [arch/i386/kernel] Error 2

网上的一个答案：

>	这个问题是由于 gcc 4.6 不再支持 linker-style 架构。将arch/x86/vdso/Makefile中以
>	VDSO_LDFLAGS_vdso.lds开头的语句的 "-m elf_x86_64" 替换为 "-m64"。
>	VDSO_LDFLAGS_vdso32.lds 开头所在行的 "-m elf_x86" 替换为 "-m32"。
>	然后在执行make bzImage便可通过。

	尝试解决
	vim arch/i386/kernel/Makefile
	在第58行将"-m elf_i386"改为"-m32"
	然后再执行make命令，顺利完成。
  
##3.make modules_install
输出  

	if [ -r System.map -a -x /sbin/depmod ]; then /sbin/depmod -ae -F System.map  2.6.22.6geeksword; fi

就执行完了，相比make就太快了。  

##4.make install
执行也很快，打印如下信息：  

	sh /home/sword/linux-2.6.22.6/arch/i386/boot/install.sh 2.6.22.6geeksword arch/i386/boot/bzImage System.map "/boot"
	run-parts: executing /etc/kernel/postinst.d/apt-auto-removal 2.6.22.6geeksword /boot/vmlinuz-2.6.22.6geeksword
	run-parts: executing /etc/kernel/postinst.d/initramfs-tools 2.6.22.6geeksword /boot/vmlinuz-2.6.22.6geeksword
	update-initramfs: Generating /boot/initrd.img-2.6.22.6geeksword
	run-parts: executing /etc/kernel/postinst.d/pm-utils 2.6.22.6geeksword /boot/vmlinuz-2.6.22.6geeksword
	run-parts: executing /etc/kernel/postinst.d/update-notifier 2.6.22.6geeksword /boot/vmlinuz-2.6.22.6geeksword
	run-parts: executing /etc/kernel/postinst.d/zz-update-grub 2.6.22.6geeksword /boot/vmlinuz-2.6.22.6geeksword
	Generating grub.cfg ...
	Found linux image: /boot/vmlinuz-3.2.0-29-generic-pae
	Found initrd image: /boot/initrd.img-3.2.0-29-generic-pae
	Found linux image: /boot/vmlinuz-2.6.22.6geeksword
	Found initrd image: /boot/initrd.img-2.6.22.6geeksword
	Found memtest86+ image: /boot/memtest86+.bin
	done

##5.生成内核文件

	root@ubuntu:/home/sword/linux-2.6.22.6# mkinitrd -o initrd.img-2.6.22.6 2.6.22.6
	mkinitrd: command not found

网上一个解决方案

>	ubuntu 找不到命令mkinitrd，取而代之的是mkinitramfs
	ubunut下没有这个命令，使用mkinitramfs命令就可以，方式：
	mkinitramfs -o /boot/initrd-img-2.6.30-uk 2.6.30-uk
	2.6.30-uk是/lib/modules/下的名字。

	root@ubuntu:/home/sword/linux-2.6.22.6# ls /lib/modules/
	2.6.22.6geeksword  3.2.0-29-generic-pae

然后再执行

	mkinitramfs -o initrd.img-2.6.22.6geeksword 2.6.22.6geeksword
	
执行完成后，没有输出。  

查看新生成的内核文件

	root@ubuntu:/home/sword/linux-2.6.22.6# ll /boot/
	total 26848
	drwxr-xr-x  3 root root     4096 Oct 14 00:02 ./
	drwxr-xr-x 23 root root     4096 Aug 11  2013 ../
	-rw-r--r--  1 root root   800453 Jul 27  2012 abi-3.2.0-29-generic-pae
	-rw-r--r--  1 root root    24755 Oct 14 00:02 config-2.6.22.6geeksword
	-rw-r--r--  1 root root   147379 Jul 27  2012 config-3.2.0-29-generic-pae
	drwxr-xr-x  3 root root    12288 Oct 14 00:02 grub/
	-rw-r--r--  1 root root  2169463 Oct 14 00:02 initrd.img-2.6.22.6geeksword
	-rw-r--r--  1 root root 14180463 Aug 11  2013 initrd.img-3.2.0-29-generic-pae
	-rw-r--r--  1 root root   176764 Nov 27  2011 memtest86+.bin
	-rw-r--r--  1 root root   178944 Nov 27  2011 memtest86+_multiboot.bin
	-rw-r--r--  1 root root   787219 Oct 14 00:02 System.map-2.6.22.6geeksword
	-rw-------  1 root root  2311324 Jul 27  2012 System.map-3.2.0-29-generic-pae
	-rw-r--r--  1 root root  1656956 Oct 14 00:02 vmlinuz-2.6.22.6geeksword
	-rw-r--r--  1 root root  5010176 Aug 17  2012 vmlinuz-3.2.0-29-generic-pae

##6.配置bootloader
修改/boot/grub/grub.cfg文件，相当于以前的menu.lst，在ubuntu12.04上修改十分方便，只需要一条命令。

	root@ubuntu:/home/sword# update-grub2
	Generating grub.cfg ...
	Found linux image: /boot/vmlinuz-3.2.0-29-generic-pae
	Found initrd image: /boot/initrd.img-3.2.0-29-generic-pae
	Found linux image: /boot/vmlinuz-2.6.22.6geeksword
	Found initrd image: /boot/initrd.img-2.6.22.6geeksword
	Found memtest86+ image: /boot/memtest86+.bin
	done

##最后重启
重启时并没有出现期待的启动菜单，原来默认状态下启动菜单并不显示，启动菜单超时时间为0。
按照http://blog.csdn.net/wjh_bh/article/details/9020083的解决方法，修改了/etc/grub.d/30_os-prober文件中3个set timeout值，然后重启出现了启动工菜单，由于我编译的内核版本比当前系统ubuntu12.04使用的内核版本低，我编译的内核启动项出现在Previous Linux versions。很遗憾，出现了Kernel panic。

