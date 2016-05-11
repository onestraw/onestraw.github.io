---
layout: single
author_profile: true
comments: true
title: pyrcc4处理qrc资源文件
categories: [Python]
tags: [Python, pyqt]
---

qrc文件是xml格式的资源配置文件，扩展名为qrc，应用程序通过qrc文件来关联使用硬盘上的图片等资源。

<h3>1、在Qt Designer中新建qrc文件并添加图片</h3>

1)在designer界面的右下角的资源浏览器中，点击上方的“编辑资源”；  

2)打开界面如下，首先在窗口左侧新建一个llk.qrc 文件，然后添加图片资源；  

3)在llk.ui文件目录下查看llk.qrc文件内容，它是一个类似xml格式文件  

<pre class="lang:default decode:true">&lt;RCC&gt;
&lt;qresource&gt;
&lt;file&gt;bg.jpg&lt;/file&gt;
&lt;file&gt;bg-2.jpg&lt;/file&gt;
&lt;/qresource&gt;
&lt;/RCC&gt;</pre>

从QRC文件内容发现，作用&lt;RCC&gt;和&lt;qresource&gt;标签，我们可以很简单的手动建立qrc文件

并且通过&lt;file&gt;标签手动添加图片能资源

<h3>2、在python脚本中使用qrc文件</h3>

通过pyuic4 -o ui_llk.py llk.ui生成ui文件的python脚本之后，还不能正常运行，错误提示如下

    Traceback (most recent call last):
    File “./main_llk.py”, line 6, in &lt;module&gt;
    from ui_llk import *
    File “/home/dell/python/llk/ui_llk.py”, line 67, in &lt;module&gt;
    import llk_rc
    ImportError: No module named llk_rc

问题的原因是，没有生成qrc文件对应 的python脚本，使用pyrcc4命令(注意不同于pyuic4命令)

    pyrcc4 -o llk_rc.py llk.qrc 

运行之后，也会生成一个编译后的二进制文件llk_rc.pyc，再次运行就不需要llk_rc.py了(可以删除试试)

