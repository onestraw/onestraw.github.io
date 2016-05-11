---
layout: single
author_profile: true
comments: true
title: 15步编译Linux内核
tagline: 翻译《How to Compile the Linux Kernel》
category: Linux
tags: [Linux, 翻译]
---

<pre>
Linux内核是Linux系统的核心。它管理用户输入/输出，硬件，并且控制计算机的电源。通常Linux发行版使用的内核功能是丰富全面的（sufficient，可理解为面向大众的，功能丰富全面，有很多功能对你都没有用处），这允许你定制自己的专业内核！
</pre>

#步骤  
<pre>

<pre>
1.下载最新的Linux内核版本http://www.kernel.org(http://www.kernel.org "下载Linux内核").
</pre>

<pre>
2.确保下载的源码是完整的，点击'Latest stable vesion is..'，否则你下载到的可能只是补丁，当你的当前内核版本的补丁号较低时，可以打补丁，例如3.4.4.1>>3.4.4.2。
	<pre>
	Linux内核版本号含义		
	Linux的版本号分为两部分，即内核版本与发行版本。内核版本号由3个数字组成：r.x.y.z。
	r：目前发布的内核主版本。
	x：偶数表示稳定版本；奇数表示开发中版本。
	y：错误修补的次数。
	z：当前版本微调patch的次数。
	一般来说，x位为偶数的版本是一个可以使用的稳定版本，如2.4.4；x位为奇数的版本一般加入了一些新的内容，不一定很稳定，是测试版本，如2.1.111。
	</pre>
</pre>

<pre>
3.确保你下载的是完整的源码，而不是补丁或者变更日志。
</pre>

<pre>
4.下载完成后，打开一个终端terminal.
</pre>

<pre>
5.解压内核压缩包，使用如下命令：
```
tar xjvf kernel(-j 选项用于bz2压缩包)
```
</pre>

<pre>
6.解压完成后，用cd命令进入刚创建的目录。
</pre>

<pre>
7.配置内核，这里有4种常见方法：

	-Make oldconfig, 系统在终端会向你询问内核应该支持什么，有很多问题，整个过程比较慢。
	-Make menuconfig, 创建一个菜单，在菜单中你能浏览内核支持的选项。执行这个命令需要ncurses库的支持。
	-Make qconfig/xconfig/gconfig, 除了配置菜单是基于图形界面的，其余都类似于menuconfig, 'qconfig'需要 QT库的支持。
	-使用当前内核的配置。在内核源码目录下运行命令'cp /boot/config-`uname -r` .config'. 这会节省许多时间，但是你可能会想改变即将编译的内核的版本号，以避免替换当前系统内核。常规设置是将“本地版本叼”附加“内核版本号”后面，例如，如果内核版本号为3.13.0，你可以将编译的内核版本号改为3.13.0.RC1。
</pre>

<pre>
8.一旦配置窗口打开，你将会看到配置参数的我写类型已经选中，如支持基本的驱动器、支持Broadcom无线、支持EXT4文件系统等。此外，你可以自定义选项，如增加支持特定设备/控制器/驱动器类型、添加对NTFS文件系统的支持，从“文件系统>> DOS/FAT/NT/>>选择NTFS文件系统的支持"，从而充分自定义内核。
</pre>

<pre>
9.注：在配置内核时，你会看到一个称为kernel hacking的部分（hacking在此代表深入摸索研究），其中有不同类型的选项用来hacking内核和学习它。如果你想使用它，那么你可以添加更多的选项，否则你可能会禁用选项“内核调试”，因为它使内核更加庞大，并且可能不当在生产环境中使用。
</pre>

<pre>
10.配置完成后，就可以编译和安装内核了。你可以通过“&&”分隔要运行的多个命令，做到单行加载多个命令。编译和安装内核要运行很长时间。编译和安装命令如下。
<code>
make && make modules_install && make install
</code>
使用 -j选项执行make，可以创建子进程来编译内核，例如"make -j 3"表示创建3个进程来编译内核。
</pre>

<pre>
11.如果顺利的话，内核已经被安装好了，最后你需要将内核设置成可引导启动的(bootable)。
</pre>

<pre>
12.Go to boot.（去设置引导新安装的内核）
</pre>

<pre>
13.运行命令"mkinitrd -o initrd.img-[kernelversion] [kernelversion]" (对于RedHat发行版，你不需要创建initrd，因为它是默认创建的)，记住要将[kernelversion]替换为你编译的新内核版本。
</pre>

<pre>
14.将引导装载程序(boot loader)指向新内核，这样新内核才能启动。使用你的Linux发行版自带的boot loader工具(如grub)来配置你的bootloader，为新内核添加一个新的入口点(entry)。
</pre>

<pre>
15.重新启动来体验你的自定义内核！
</pre>
</pre>

#原文链接
http://www.wikihow.com/Compile-the-Linux-Kernel
