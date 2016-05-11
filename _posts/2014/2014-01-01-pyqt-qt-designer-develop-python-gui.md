---
layout: single
author_profile: true
comments: true
title: 利用QT Designer 快速开发Python GUI
category: Python
tags: [pyqt, Python]
---
pyqt入门任务：用pyqt显示一个新年快乐的对话框

<h1><strong>1、使用qt designer简单设计一个Dialog</strong></h1>
终端输入designer &amp;

后台启动qt designer,如果还没有安装，会提示你安装qt3 or qt4

可能是我区域位置设置的问题，默认安装的是中文版的。

<img src='/assets/images/2014-ui.png'>

生成的ui文件其实是一个xml文件（GTK+使用的glade文件也是xml文件）

<h1><strong>2、生成ui文件对应的python代码</strong></h1>

“窗体”-&gt;”查看代码”，发现源代码是C++代码,如果是C++程序的话，可以直接粘贴过去。

使用pyuic4命令根据2014.ui文件生成python代码

	pyuic4 2014.ui -o ui_2014.py
	or
	pyuic4 2014.ui &gt; ui_2014.py

（程序“pyuic4”尚未安装。  您可以使用以下命令安装：sudo apt-get install pyqt4-dev-tools）

<h1><strong>3、编写HappyNewYear类及测试程序</strong></h1>

happynewyear.py

<pre class="lang:python decode:true">#!/usr/bin/python
import sys
from ui_2014 import *
from PyQt4.QtGui import *

class HappyNewYear(QDialog, Ui_HappyNewYear):
	def __init__(self,parent=None):
		super(HappyNewYear,self).__init__(parent)
		self.setupUi(self)


if __name__=="__main__":
	app = QApplication(sys.argv)
	dlg = HappyNewYear()
	dlg.show()
	app.exec_()</pre>

1)导入模块说明

import sys是因为要用sys.argv

from ui_2014 import *  
ui_2014.py是我们自己生成GUI模块，要使用类Ui_HappyNewYear  

from PyQt4.QtGui import * 
类QDialog和QApplication在模块PyQt4.QtGui中

2)HappyNewYear类

定义一个HappyNewYear类，它继承了QDialog和Ui_HappyNewYear

super(HappyNewYear,self).__init__(parent)   调用两个父类的初始化函数__init()

self.setupUi(self)  调用Ui_HappyNewYear类的setupUi函数，负责初始化图形界面

3)测试程序

main函数中
dlg = HappyNewYear()  
dlg.show()  
这两句好理解，新建一个HappyNewYear对象并调用show来显示  

app = QApplication(sys.argv)  
app.exec_()  
这两句貌似和我们要显示对话框没什么关系  

QApplication类管理图形用户界面应用程序的控制流和主要设置。

它包含主事件循环，在其中来自窗口系统和其它资源的所有事件被处理和调度。它也处理应用程序的初始化和结束，并且提供对话管理。它也处理绝大多数系统范围和应用程序范围的设置。对于任何一个使用Qt的图形用户界面应用程序，都存在一个QApplication对象

app.exec_()  有什么作用呢，我把最后一句注释了，运行一下，程序不会显示对话框，直接就结束了。从字面理解，它是执行app，QApplication的作用前面已经说了。
<h1><strong>4、运行</strong></h1>
现在有2014.ui, ui_2014.py, happynewyear.py三个文件

	chmod +x happynewyear.py
	./happynewyear.py

<img src='/assets/images/happynewyear.png'>

运行之后，happynewyear.py所在目录下生成一个ui_2014.pyc文件，它是一个ui_2014.py编译后的二进制文件。

