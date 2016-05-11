---
layout: single
author_profile: true
comments: true
title: 无法挂载root fs
tagline: 深入分析
category: Linux
tags: [Linux]
---
前些日子编译内核碰到一个根文件系统加载失败的问题，困扰了我好久，终于在昨天完美解决了:)

##无法挂载根文件系统的现象

1. Kernel panic - not syncing: Attempted to kill init!

2. Kernel panic - not syncing: VFS: unable to mount root fs on unknown-block(0,0)

3. ALERT! /dev/disk/by-uuid/XXXXXXXXXXXXXXXXXXXXXXXX does not exist. Dropping to a shell

上述问题都是因为Linux启动过程中无法挂载根文件系统root fs！
要想明白为什么无法挂载root fs，先要理解挂载root fs发生在启动过程中的哪一阶段，以及在挂载root fs之前要做哪些准备工作？

##挂载root fs的过程

推荐阅读鸟哥的Linux私房菜和阮老师的博客  

* [1.2 BIOS, boot loader 与 kernel 加载](http://vbird.dic.ksu.edu.tw/linux_basic/0510osloader.php#startup_loader)（有点难度）
* [计算机是如何启动的？](http://www.ruanyifeng.com/blog/2013/02/booting.html)（浅显）
* [Linux 的启动流程](http://www.ruanyifeng.com/blog/2013/08/linux_boot_process.html)（浅显）

**内核启动时挂载root fs的条件如下**

> 1. 挂载根文件系统需要读取硬盘
> 2. 读取硬盘需要设备驱动
> 3. 而设备驱动存放在根文件系统的/lib/modules/2.6.22.version下面
> 4. 读取驱动程序要先“挂载根文件系统”？？？

**问题来了，你根本没有挂载根文件系统呢，怎么去读取根文件系统内的驱动程序？**  
所以说计算机的boot过程是(boot的由来):

> "pull oneself up by one's bootstraps"  
  "拽着鞋带把自己拉起来"

这是一个死循环啊，打破这个循环的就是initramfs/initrd，initrd是一个临时文件系统，它在Linux启动过程中使用，
注意initramfs中的ram哦，这就是内存啊，内存文件系统？不错，initrd就在内核加载过程中直接加载进内存的，
可以查看menu.lst/grub.lst启动项。文件系统在内存中，这样访问文件系统就不需要硬盘驱动了，
initrd这个文件系统内有硬盘驱动(解压initrd可以发现/lib/modules/目录和文件系统的/lib/modules/目录一样)，
这样就可以运行驱动程序加载真正的根文件系统了。
 
##挂载root fs失败原因

**如果initrd中没有我们需要的硬盘驱动，或者硬盘驱动不对号，那么还是无法加载真正的根文件系统，就会出现第一节所报的各种错误。**

配置内核时，注意以下3个选项（SCSI和ScsiHost是模块哦）

    Genernal setup
    	[*] Initial RAM filesystem and RAM disk (initramfs/initrd) support
    Device Drivers
    	SCSI device support
    		<M> SCSI device support
    		[*] legacy /proc/scsi/ support
    		<M> SCSI disk support
    	Fusion MPT device Support 
    		<M> Fusion MPT ScsiHost drivers for SPI	
		    < > Fusion MPT ScsiHost drivers for FC	
		    < > Fusion MPT ScsiHost drivers for SAS	
		    
1. initramfs/initrd 一定要支持了，否则在scsi硬盘上没办法加载root fs;
2. 因为我在VMWare虚拟的机器是SCSI类型的硬盘，所以也要选择SCSI device 驱动了;
3. Fusion MPT是什么东西  

**SCSI**
在VMWare中安装ubuntu时，在“选择I/O控制器类型”步骤SCSI控制器有3种类型：
	
	* BusLogic(U)
	* LSI Logic(L)
	* LSI Logic SAS(S)
	
**默认选择的是LSI Logic(L)**, Buslogic和LSI logic都是虚拟硬盘SCSI设备的类型，旧版本的OS 默认的是Bus Logic，LSI Logic类型的硬盘改进了性能，
对于小文件的读取速度有提高,支持非SCSI硬盘比较好。   
SCSI硬盘是采用SCSI接口的硬盘，SCSI是Small Computer System Interface（小型计算机系统接口）
的缩写[硬盘分类wiki](http://zh.wikipedia.org/wiki/%E7%A1%AC%E7%9B%98)  
	
**FUSION MPT**

* LSI Logic是一家公司  
* Fusion MPT是一种架构，MPT指Message Passing Technology  

> LSI Logic公司的LSI53C1030T基于Fusion-MPT架构，是一种为高端目标模式应用的PCI-X双通道Ultra320 SCSI控制器，适用于缓存同步群集(cluster)、桥控制器和RAID子系统前端。Fusion-MPT是由LSI Logic公司开发的，目的是为了使客户能更为容易的实现SCSI和Fibre Channel的解决方案。这种开放式的Fusion-MPT架构具有最高的I/O性能，同时它还能降低产品验证的时间和推向市场的时间   
  Fusion MPT架构支持Ultra320 SCSI、光纤通道和SAS 

<a href="http://bbs.chinaunix.net/thread-1109749-1-1.html" target="_blank">参考</a>	
	
**ScsiHost drivers**

Scsi host driver位于SCSI中间层之下，属于SCSI总线控制器的驱动。如果有人做了一个SCSI适配器（host bus adapter），那么需要为该适配器写一个驱动，这个驱动就是scsi host driver。如果SCSI适配器是一个真实的硬件设备，绝大多数情况是采用PCI接口，那么scsi host driver也是一个PCI设备驱动。假设给一个真实的SCSI适配器写了一个驱动，那么内核驱动的体系结构如下图所示：  
<img src="/assets/images/scsi-host-driver.jpg" />  
SCSI device driver是SCSI设备驱动，也可以称之为功能驱动（Function driver）。SCSI middle level driver为SCSI中间层驱动，抽象了SCSI的总线逻辑。Scsihost driver控制SCSI总线控制器，实现SCSI数据的物理层传输。
<a href="http://bbs.ednchina.com/BLOG_ARTICLE_129907.HTM" target="_blank">参考</a>	  

 

###Solution
在VMware安装Ubuntu时，我们选择的SCSI硬盘，而且scsi控制类型是LSI Logic，所以将scsi驱动scsihost驱动编译为模块。  

**为什么不选择ScsiHost drivers for FC或者SAS**

> 1. SPI是串行外设接口（Serial Peripheral Interface）的缩写，它是一种高速的，全双工，同步的通信总线.
> 2. SAS（Serial Attached SCSI）是新一代的SCSI技术，和SATA硬盘相同，都是采取序列式技术以获得更高的传输速度，可达到6Gb/s。此外也透过缩小连接线改善系统内部空间等。
> 3. FC（Fibre Channel，光纤通道接口），拥有此接口的硬盘在使用光纤联接时具有热插拔性、高速带宽(4Gb/s或10Gb/s)、远程连接等特点；内部传输速率也比普通硬盘更高。限制于其高昂的售价, 通常用于高端服务器领域。

安装虚拟机时SCSI控制器类型有LSI Logic SAS(S)，我使用的是LSI Logic(L)，所以不选择Fusion MPT ScsiHost drivers for SAS。	  
维基百科里说了FC价格高昂, 通常用于高端服务器领域，我这一个小小虚拟机用的硬盘应该不是FC接口，所以选择第一个。 
