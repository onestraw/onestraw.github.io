---
layout: single
author_profile: true
comments: true
title: 推导PE文件与虚拟内存之间的映射关系
categories: [Windows]
tags: [系统安全]
---
如何计算一条指令在PE文件中的偏移地址？

0 day 安全 一书中给出一个公式：

<strong>文件偏移地址 = 虚拟地址 – 装载基址 – 节偏移 = RVA – 节偏移</strong>

节偏移  是指指令在代码段中(.text section)的偏移. 下面结合实例推导一下。

1. 在反汇编工具（如OllyDbg）得到某一条指令的地址，该地址是虚拟内存地址（参考现代操作系统的虚拟内存管理），记为VA；

2. 通过LoadPE查看ImageBase
    ImageBase 是装载基址，Visual C++建立的EXE文件的基地址为00400000h，DLL文件基地址是10000000h；

3. 通过LoadPE查看 ROffset, VOffset
      
    1) LoadPE打开exe文件，选中右键，在菜单中选择”load into PE editor”
      
    2) 然后点击右侧的”Sections”按钮，可查看代码段（.text section）的相关信息。
      
      ROffset: section（.text等）的虚拟内存地址偏移，相对ImageBase而言；
      VOffset：section（.text等）的文件偏移地址，.text section在exe文件中的偏移量；

4. 计算一条汇编指令在exe文件中的偏移地址

    1）指令属于代码段.text section的，首先计算出虚拟地址空间中，该指令相对于.text段的偏移offset
    offset = VA – ImageBase – ROffset
    为什么减一次ImageBase呢？
    
    虚拟地址空间中
    —————A———————B———————C
    
    A表示ImageBase
    B表示.text段基址，AB表示Roffset
    C表示该指令的虚拟地址VA
    现在已知AB，A的横坐标，C的横坐标，求BC长度？
    AB = ROffset
    AC = vA – ImageBase
    BC = AC – AB = VA – ImageBase – ROffset
    
    2）然后用该指令偏移offset 加上 .text段在exe文件的偏移，就可以得出该指令相对于exe文件的偏移。
    
    exe文件中
    A————B————C
    
    A相当于exe文件开始处，偏移为0
    B相当于.text段，偏移为VOffset
    C相对B的偏移为BC（通过虚拟地址空间求得BC = VA – ImageBase – ROffset），那么C相对于A的距离AC = BC+AB = BC+VOffset
    所以根据1）中的BC，得到
    指令在文件中的偏移AC = VA – ImageBase – ROffset + VOffset

5. PLC

    可在LoadPE中进行虚拟地址与文件偏移的自动转换，方法是
    
    在 PE  editor 窗口点击 PLC 打开“File Location Calculator”
