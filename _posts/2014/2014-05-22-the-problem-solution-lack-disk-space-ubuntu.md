---
layout: single
author_profile: true
comments: true
title: ubuntu磁盘空间不足的现象及解决
categories: [Linux]
tags: [Linux]
---
一天早上打开电脑，ubuntu突然进不去了，屏幕提示：

<span style="color: #ff0000;"><strong>The system is running in low-graphics mode</strong></span>

一路OK下去，最终卡在：

<span style="color: #ff0000;"><strong>Checking Battery State.                    [OK]</strong></span>

这是怎么回事啊，昨天晚上还好好的，只是在睡觉前使用<strong>remastersys</strong>备份下系统，由于备份实在太慢，我开着电脑就睡了。

尝试各种方式无果，最后想到可能是硬盘空间不足，因为装的是双系统，给ubuntu12.04只分配了50G空间。

在启动ubuntu之后，<span style="color: #ff0000;">Ctrl + Alt +F1</span> 进行终端模式，登录成功以后，使用命令

<span style="color: #ff0000;">df -hl</span>

查看磁盘剩余空间，发现果然没有剩余空间了。

使用命令

rm -rf  备份文件的路径

删除备份文件，并且清空回收站

<span style="color: #ff0000;">rm -rf ~/.local/share/Trash/files/*</span>

然后reboot，可以在图形界面正常登录了。
