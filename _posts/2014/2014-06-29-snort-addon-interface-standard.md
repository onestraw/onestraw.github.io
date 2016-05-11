---
layout: single
author_profile: true
comments: true
title: snort插件接口规范
categories: [Snort]
tags: [Snort]
---

<h1>0.引言</h1>
为什么要了解snort插件的接口呢？有3个出发点：

一、为snort编写插件；

二、从系统架构层面学习接口的设计；

三、在明白了snort的插件接口规范之后，可以将某一插件完全剥离出来，更好的研究它；

本文是参考snort-2.9.5.5\doc\README.PLUGINS，发现其中有几个错误之处。
<h1>1.插件组成</h1>
<div>snort插件分为两类：detection plugins和preprocessors plugins.</div>
<div>detections plugins需要有两个文件"sp_something.c", "sp_something.h";</div>
<div>preprocessors plugins需要有两个文件"spp_something.c", "spp_something.h";</div>
<h1>2.增加detection plugin</h1>
<h2>修改1</h2>
<div>编辑snort-2.9.5.5\src\plugbase.h文件，插入一个头文件</div>
<div>     #include "sp_something.h"</div>
<div style="color: #000000; font-family: Tahoma; font-style: normal; font-variant: normal; font-weight: normal; letter-spacing: normal; line-height: normal; orphans: 2; text-align: -webkit-auto; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-size-adjust: auto; -webkit-text-stroke-width: 0px; font-size: medium;"></div>
<div style="color: #000000; font-family: Tahoma; font-style: normal; font-variant: normal; font-weight: normal; letter-spacing: normal; line-height: normal; orphans: 2; text-align: -webkit-auto; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-size-adjust: auto; -webkit-text-stroke-width: 0px; font-size: medium;">
<h2>修改2</h2>
</div>
<div>编辑snort-2.9.5.5\src\plugbase.c文件，READ.PLUGINS中写到：在InitPlugins()函数中添加该插件的setup函数。实际上在snort-2.9.5.5版本中没有InitPlugins()函数，应该是 RegisterRuleOptions()函数。</div>
<div style="color: #000000; font-family: Tahoma; font-style: normal; font-variant: normal; font-weight: normal; letter-spacing: normal; line-height: normal; orphans: 2; text-align: -webkit-auto; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-size-adjust: auto; -webkit-text-stroke-width: 0px; font-size: medium;"></div>
<div style="color: #000000; font-family: Tahoma; font-style: normal; font-variant: normal; font-weight: normal; letter-spacing: normal; line-height: normal; orphans: 2; text-align: -webkit-auto; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-size-adjust: auto; -webkit-text-stroke-width: 0px; font-size: medium;">
<h2>修改3</h2>
</div>
<div>编辑snort-2.9.5.5\src\detection-plugins\Makefile.am 文件</div>
<div>READ.PLUGINS中写到：将"sp_something.c", "sp_something.h"两个文件添加到  "snort_SOURCES"一行中。实际上Makefile.am文件没有"snort_SOURCES"，应该是放在第24行"libspd_a_SOURCES"中，保存退出，运行automake.</div>
<h1>3.增加preprocessors plugin</h1>
<div style="color: #000000; font-family: Tahoma; font-style: normal; font-variant: normal; font-weight: normal; letter-spacing: normal; line-height: normal; orphans: 2; text-align: -webkit-auto; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-size-adjust: auto; -webkit-text-stroke-width: 0px; font-size: medium;"></div>
<div style="color: #000000; font-family: Tahoma; font-style: normal; font-variant: normal; font-weight: normal; letter-spacing: normal; line-height: normal; orphans: 2; text-align: -webkit-auto; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-size-adjust: auto; -webkit-text-stroke-width: 0px; font-size: medium;">
<h2>修改1</h2>
</div>
<div>编辑snort-2.9.5.5\src\plugbase.h文件，插入一个头文件</div>
<div>
<div>     #include "spp_something.h"</div>
<h2>修改2</h2>
<div>编辑snort-2.9.5.5\src\plugbase.c文件，在InitPreprocessors()函数中添加相应的setup()函数。也没有找到READ.PLUGINS中提到的InitPreprocessors()函数，实际应该是RegisterPreprocessors()函数。</div>
<h2>修改3</h2>
<div>编辑snort-2.9.5.5\src\preprocessors\Makefile.am文件，在第12行libspp_a_SOURCES=后面添加"spp_something.c", "spp_something.h"两个文件名。</div>
</div>
<h1>4.一个插件实例</h1>

<div>针对版本snort-2.9.5.5的一个预处理插件sfportscan(检测端口扫描)说明snort插件接口的使用方法。</div>
<h2>1）添加头文件包含</h2>
<div>头文件"spp_sfportscan.h"没有被包含在snort-2.9.5.5\src\plugbase.h中，而在snort-2.9.5.5\src\plugbase.c头部.</div>
<div></div>

    /* built-in preprocessors */ 
    #include "preprocessors/spp_rpc_decode.h" 
    #include "preprocessors/spp_bo.h"
    #include "preprocessors/spp_stream5.h"
    #include "preprocessors/spp_arpspoof.h"
    #include "preprocessors/spp_perfmonitor.h"
    #include "preprocessors/spp_httpinspect.h"
    #include "preprocessors/spp_sfportscan.h"
    #include "preprocessors/spp_frag3.h"
    #include "preprocessors/spp_normalize.h"

<h2>2）挂载sfportscan插件</h2>
<div>位置：snort-2.9.5.5\src\plugbase.c</div>

    void RegisterPreprocessors(void)
    {
      LogMessage("Initializing Preprocessors!\n");
      SetupARPspoof();
      #ifdef NORMALIZER
      SetupNormalizer();
      #endif
      SetupFrag3();
      SetupStream5();
      SetupRpcDecode();
      SetupBo();
      SetupHttpInspect();
      SetupPerfMonitor();
      SetupSfPortscan();
    }
    
<div>注意SetupSfPortscan();是该插件提供的唯一的一个接口</div>
<div>
<div>(snort-2.9.5.5\src\preprocessors\spp_sfportscan.h)</div>
</div>

    #ifndef __SPP_SFPORTSCAN_H__
    #define __SPP_SFPORTSCAN_H__
    void SetupSfPortscan(void);
    #endif</div>
  
<h2>3) 加入 makefile编译</h2>
<div>
<div>位置：snort-2.9.5.5\src\preprocessors\Makefile.am</div>
</div>
sfportscan插件包含2个.h文件和2个.c文件，其中portscan.*属于插件内部使用文件，spp_sfportscan.*对外提供一个接口SetupSfPortscan().

    libspp_a_SOURCES = spp_arpspoof.c spp_arpspoof.h spp_bo.c spp_bo.h \
    spp_rpc_decode.c spp_rpc_decode.h  \
    stream_expect.c stream_expect.h \
    spp_perfmonitor.c spp_perfmonitor.h \
    perf.c perf.h \
    perf-base.c perf-base.h \
    perf-flow.c perf-flow.h \
    perf-event.c perf-event.h \
    $(PROCPIDSTATS_SOURCE) \
    spp_httpinspect.c spp_httpinspect.h \
    snort_httpinspect.c snort_httpinspect.h \
    portscan.c portscan.h \
    spp_sfportscan.c spp_sfportscan.h \
    spp_frag3.c spp_frag3.h \
    str_search.c str_search.h \
    spp_stream5.c spp_stream5.h \
    stream_api.c stream_api.h \
    spp_normalize.c spp_normalize.h \
    normalize.c normalize.h


<h1>5.结束语</h1>
本文介绍了snort插件接口规范，并拿sfportscan预处理插件作为实例进行了分析，下一步要学习sfportscan插件的源码。
