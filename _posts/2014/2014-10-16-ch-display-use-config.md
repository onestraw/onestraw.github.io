---
layout: single
author_profile: true
comments: true
title: 通过配置文件修改屏幕分辨率
tagline: ubuntu9.10
category: Linux
tags: [Linux]
---

ubuntu 9.10下编译内核时，在执行make menuconfig时，由于屏幕太小，导致打不开配置菜单，屏幕有多小，小的连打开System->Preferences->Display看不到'Apply'按键，所以只能通过修改配置文件来完成了

###配置步骤
0. su
1. vi /etc/default/grub
  * 找到行： #GRUB_GFXMODE=640×480
  * 去掉注释符#
2. vi /etc/grub.d/00_header
  * 找到行：set gfxmode=${GRUB_GFXMODE}
  * 在下面添加新行：set gfxpayload=keep
3. update-grub
4. reboot
