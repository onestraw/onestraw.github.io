---
layout: single
author_profile: true
comments: true
title: 使用pyinstaller生成独立的可执行文件
category: Python
tags: [Python]
---
<h1><strong>一、ubuntu12.04下pyinstller安装及使用步骤</strong></h1>
1. 下载<a href="https://pypi.python.org/packages/source/P/PyInstaller/PyInstaller-2.1.tar.gz" target="_blank">PyInstaller</a>

最新版本的PyInstaller支持python2.4-2.7

用python –version查得本机python版本为 2.7.3

2. 安装

cd ~/下载
tar -xf PyInstaller-2.1.tar.gz
cd PyInstaller-2.1
sudo python setup.py install

出现错误

Traceback (most recent call last):
File “setup.py”, line 14, in
from setuptools import setup, find_packages
ImportError: No module named setuptools

网上找了下原因，是因为缺少python-setuptools

sudo apt-get install python-setuptools

然后再执行sudo python setup.py install  就安装成功了

3. 生成独立的可执行文件

cd ~/PyLLK
pyinstaller -F llk.py

在~/PyLLK/dist目录下生成独立的不信赖库的可执行文件llk，运行llk，需要将两个图片目录和llk放在同一目录下。
<h1><strong>二、windows7下pyinstller安装及使用步骤</strong></h1>
1、安装python setuptools

下载python setuptools <a href="http://pypi.python.org/packages/2.7/s/setuptools/setuptools-0.6c11.win32-py2.7.exe#md5=57e1e64f6b7c7f1d2eddfc9746bbaf20" target="_blank">setuptools-0.6c11.win32-py2.7.exe</a>

直接运行安装

2、下载<a href="https://pypi.python.org/packages/source/P/PyInstaller/PyInstaller-2.1.zip" target="_blank">PyInstaller</a>

也是支持2.4-2.7

win7下的python版本为2.7.6

进入PyInstaller-2.1目录，执行

python setup.py install

3、执行pyinstaller.py

E:\PyInstaller-2.1&gt;python pyInstaller.py -F D:\PyLLK\llk.py

提示以下错误
Error: PyInstaller for Python 2.6+ on Windows needs pywin32.
Please install from http://sourceforge.net/projects/pywin32/

4、安装pywin32

下载<a href="http://downloads.sourceforge.net/project/pywin32/pywin32/Build%20214/pywin32-214.win32-py2.7.exe?r=http%3A%2F%2Fzhidao.baidu.com%2Flink%3Furl%3D6vpwIza4rcsZPp3sFrkmUMh-SOvC7J1Wpy9NHMKEXszvcnP0XE7aYgMZsOZDARItabkbYVuTrCeL2XPj16Eyx_&amp;ts=1389506743&amp;use_mirror=jaist" target="_blank">pywin32</a>

直接运行安装即可，再次执行

E:\PyInstaller-2.1&gt;python pyInstaller.py -F D:\PyLLK\llk.py

(如果不想显示控制台命令行，加上 -w 选项)

会在当前目录下生成一个llk文件夹，llk/dist/llk.exe

最后将image图片和image_select图片文件夹拷贝到和llk.exe同一个文件夹下，这时就完成了发布，可以运行在任何win32平台下了。
