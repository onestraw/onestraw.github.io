---
layout: single
author_profile: true
comments: true
title: VC++6.0编译使用开源日志系统log4cplus
categories: [cprogram]
tags: [C/C++]
---

#一、生成dll和lib文件

1. 下载log4cplus-1.0.2版本http://sourceforge.net/projects/log4cplus/files/log4cplus-stable/1.0.2/
2. 解压到D:\log4cplus-1.0.2
3. 使用VC++6.0打开工作空间D:\log4cplus-1.0.2\msvc6\log4cplus.dsw
4. 直接按F7组建[连接]，在Debug中生成log4cplusd.dll文件和log4cplusd.lib

#二、使用log4cplus
1. 创建一个新工程logTest，将log4cplus.lib和log4cplus.dll拷贝到logTest目录下。
2. 在源文件头部加入一条语句#pragma comment(lib, “log4cplusd.lib”)。
3. 在VC++6.0的“工具”-&gt;“选项”-&gt;“目录”中添加对log4cplus头文件的搜索路径“D:\log4cplus-1.0.2\include”.
4. 在“工程“-&gt;”设置”-&gt;”C/C++”-&gt;“预处理定义”中添加LOG4CPLUS_STATIC。或者在源文件头部加入

	#ifndef LOG4CPLUS_STATIC  
	#define LOG4CPLUS_STATIC  
	#endif  

5. 写了一个log4cplus.cfg配置文件和exe执行程序放在同一目录下。

#三、日志记录接口
分为六个级别：

	* LOG4CPLUS_TRACE(Logger::getRoot(), “printMessages()”);  
	* LOG4CPLUS_DEBUG(Logger::getRoot(), “This is a DEBUG message”);  
	* LOG4CPLUS_INFO(Logger::getRoot(), “This is a INFO message”);  
	* LOG4CPLUS_WARN(Logger::getRoot(), “This is a WARN message”);   
	* LOG4CPLUS_ERROR(Logger::getRoot(), “This is a ERROR message”);  
	* LOG4CPLUS_FATAL(Logger::getRoot(), “This is a FATAL message”);   

#四、测试程序

1. 由于本程序用到了shlwapi.h，所以要在文件头部加入#pragma comment(lib, “shlwapi.lib”). 

2. 测试程序使用了MFC类库，按文章《__endthreadex和__beginthreadex错误》解决。  

3. [源码](https://github.com/onestraw/blogcode/blob/master/log4cplus_test.cpp)
