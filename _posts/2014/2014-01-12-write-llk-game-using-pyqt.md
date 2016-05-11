---
layout: single
author_profile: true
comments: true
title: 使用PyQt编写连连看小游戏
category: Python
tags: [pyqt, Python]
---
<img src='/assets/images/onestraw_llk.png'>

这个想法源于《编程之美》上的一个题目，而且以前也玩过这个小游戏，宿舍一兄弟是高手啊，手特别快，经常在QQ游戏中虐别人。

编写连连对我来说，主要挑战是界面这块，大概是9月份左右的时候在ubuntu下用C＋Gtk写过一次，主要功能有：随机生成图，刷新图，消去时显示最短路径连线，计时、胜利提示。主要问题有：用字母代替图片显示，连线消去问题，刷新出现白色字体。

实现一个功能or系统，选择工具很重要。

这次我用PyQt来实现，由于时间有限，写了大概160行代码，仅实现基本功能，包括：随机初始化界面、图片显示、按规则(小于3个弯)消去图片。「没有去做时间标签、难度级别、连线等，除了连线有点挑战，其它都很简单」

<strong>一、LLK界面实现步骤</strong>

1、首先我学习了一个代码，可以在窗口显示文字

<a href="http://zetcode.com/gui/pyqt4/drawing/" target="_blank">Draw Text</a>

并且在第一个代码中了解到paintEvent函数

paintEvent()函数自己重新定义，它在窗口初始化时，切换窗口时，以及点击鼠标时，会自动调用，也可以通过repaint()和update()函数进行调用。

2、如果把文字替换成图上，不就可以显示连连看的图片了吗？

然后在窗口中显示一个图片，参考

<a href="http://pyqt.sourceforge.net/Docs/PyQt4/qpixmap.html" target="_blank">显示一个位图   类</a>QPixmap

3、在窗口中捕获点击信号，及点击的坐标位置，参考

<a href="http://pyqt.sourceforge.net/Docs/PyQt4/qmouseevent.html" target="_blank">类QMouseEvent</a>

4、捕获鼠标点击事件mousePressEvent()函数

5、点击后相应的图片，实现重绘，这个过程其实是加载另一张图片。

<strong>二、LLK算法部分</strong>

1、随机分配图片

由于这个要保证成对出现，还要随机，所以放置一张图片时，要放在两个位置。

以前在C语言的版本实现思想是：放置一个图片时，先生成一个随机数，放在矩阵的未使用位置，然后再随机找一个没有占用的位置放置同一张图片。这个过程遍历矩阵次数比较多，效率比较低。

这次使用一个相对较好的方法，先顺序加载图片，加载完成之后，随机生成两个坐标位置，然后调换这两个位置的图片，以实现随机化。

2、用列表list()实现队列

用了list的append和remove函数

3、判断两个图片是否可以消去

根据「编程之美」上的思路，先判断转弯次数为0的可达节点，然后判断转弯次数为1的所有节点，最后是转弯次数为2的所有节点。注意，不必添加访问标志。

这里的一个关键问题是，怎么标识这个转弯次数，这时python的优势就发挥出来了，list可以动态添加元素，不用自己管理内存。我用的办法是在 点击位置preClick和curClick  （list 类型，已经从坐标转化为二维数组索引）后append一个转弯次数，初始为0，当读取preClick[2]&gt;2时说明没有找到可达路径，使用特别 方便。

<strong>三、图片存储</strong>

使用了20张图片头像和一个背景图，原始图片放在image目录下，选中效果图放在image_select目录下，背景图名bg.jpg，20张头像从0－19命名，扩展名均为.jpg，没有配置文件，硬编码到程序中的。

<strong>四、python源代码</strong>

<a href="https://github.com/onestraw/code/tree/master/pyllk" target="_blank">onestraw</a>
