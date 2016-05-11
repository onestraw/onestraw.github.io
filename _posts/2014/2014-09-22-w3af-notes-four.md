---
layout: single
author_profile: true
comments: true
title: w3af学习笔记（四）
categories: [WebSec]
tags: [Web安全]
---

本文分析了w3af的xss插件，旨在理解w3af检测xss漏洞的原理。

<h1>一、预备知识</h1>

<h3>1.XSS</h3>
当应用程序收到含有不可信的数据，在没有进行适当的验证和转义的情况下，就将它发送给一个网页浏览器，这就会产生跨站脚本攻击（简称XSS）。XSS允许攻击者在受害者的浏览器上执行脚本，从而劫持用户会话、危害网站、或者将用户转向至恶意网站。（参考：OWASP TOP 10 中文v1.2）
<h3>2.audit</h3>
对于审计（audit）的理解，审计就是找漏洞。代码审计是一种白盒测试方法，从源代码中发现漏洞；w3af中使用的audit，是一种黑盒测试方法，从外部寻找漏洞。
<h3>3.payload</h3>
有效载荷（payload）通常指的恶意代码执行破坏性操作的部分，在w3af中指的是包含特殊字符的测试字符串，用来检验这些特殊字符是否被过滤等，并非执行恶意操作。
<h3>4.fuzzing</h3>
模糊测试（fuzzing, fuzz testing）是一种基于缺陷注入的自动软件测试技术。通过编写fuzzer工具向目标程序提供某种形式的输入并观察其响应来发现问题，这种输入可以是完全随机的或精心构造的，为了寻找软件缺陷，主要是构造畸形数据。Fuzzer工具能很好的检测出可能导致程序崩溃的问题，如缓冲区溢出、跨站点脚本、拒绝服务攻击、格式漏洞和SQL注入。w3af中有一个FuzzableRequest类，该类灵活表示了一个http 请求包。
<h3>5.mutant</h3>
在w3af中mutant类是对FuzzableRequest类的包装，然后在mutant中添加设置不同的payload，用这些mutant扫描测试网站的漏洞。

<h1>二、XSS插件</h1>

以w3af\plugins\audit\xss.py的xss.audit函数为主线，深入分析了w3af扫描xss漏洞的原理。
<h3>1.xss.payloads</h3>

	PAYLOADS = ['RANDOMIZE&lt;/->',
	'RANDOMIZE/*',
	'RANDOMIZE"RANDOMIZE',
	"RANDOMIZE'RANDOMIZE",
	"RANDOMIZE`RANDOMIZE",
	"RANDOMIZE ="]

PAYLOADS中起作用的只是里面特殊字符:&lt;/->,/*,",',`,=;
RANDOMIZE只是为了在http响应报文中定位到特殊字符的位置，在些它只是一个占位用的标识字符串，在使用时会随机生成一个字符串，替换RANDOMIZE;

<h3>2.xss.audit(freq,orig_response)</h3>
orig_response在函数中没有使用，可能是预留的一个参数；
freq是FuzzableRequest类对象，表示一个模糊http请求包；
功能：检测一个URL的XSS漏洞，被检测的URL在freq中；  
执行过程如下：
<ol>
	<li>创建一组freq的变体fake_mutants;</li>
	<li>对于一个fake_mutant，扫描（持久型）XSS漏洞；</li>
	<li>首先扫描简单的XSS漏洞（所有字符不经过编码和过滤就直接反射回来）,详细过程见_identify_trival_xss()，如果找到XSS漏洞，直接返回；</li>
	<li>然后分析fake_mutant，扫描检测反射型XSS漏洞，详细过程见_search_xss()；</li>
</ol>
<h3>3.xss._identify_trival_xss(mutant)</h3>
功能：识别简单的XSS漏洞：所有字符不经过编码和过滤就直接反射回来;
<ol>
	<li>将所有的特殊字符(&lt;/->,/*,",',`,=)放在一块，组成一个payload；</li>
	<li>将payload添加到mutant中;</li>
	<li>将mutant发到服务器，返回一个HTTPResponse;</li>
	<li>如果payload在reponse.body中（也就是说所有特殊字符没有被过滤或者编码就直接反射回来），那就是存在xss漏洞；</li>
	<li>_report_vuln()创建一个Vuln，并且mutant等存储到KB中;</li>
</ol>
<h3>4.xss._search_xss(mutant)</h3>
<p style="color: #000000;">_search_xss()执行流程</p>

<ul style="color: #000000;">
	<li>0.用xss.PAYLOADS构造http请求包变体列表mutant_list</li>
	<li>1.发送mutant，得到一个HttpResponse</li>
	<li>2.调用body=HttpResponse.get_body()
<ul>
	<li>调用HttpResponse._charset_handling()，返回unicode(decoded) body 和used charset</li>
	<li>http body可能就是&lt;body>标签内的正文，或者是整个网页？</li>
</ul>
</li>
	<li>3.根据mutant得到发送的payload</li>
	<li>4.调用get_context_iter(body, payload)函数
<ul>
	<li>1). chunks = body.split(payload)</li>
	<li>2). data=''</li>
	<li>3). 遍历data从chunks[0]迭加到chunks[-1](倒数第2个)</li>
	<li>4). byte_chunk = ByteChunk(data)
<ul>
	<li>ByteChunk数据成员如下</li>
	<li>self.attributes=dict()</li>
	<li>self.data = data</li>
</ul>
</li>
	<li>5). 遍历get_contexts()
<ul>
	<li>[HtmlAttrSingleQuote(), HtmlAttrDoubleQuote(), HtmlAttrBackticks(), HtmlAttr(), HtmlTag(), HtmlText(), HtmlComment(), ScriptMultiComment(), ScriptLineComment(),ScriptSingleQuote(), ScriptDoubleQuote(), ScriptText(),StyleText(), StyleComment(), StyleSingleQuote(),StyleDoubleQuote()]，具体定义见下一节Context</li>
</ul>
</li>
	<li>6). 判断context.match(byte_chunk)
<ul>
	<li>如果匹配，1.context.save(data);2.迭代返回context</li>
</ul>
</li>
</ul>
</li>
	<li>5. 如果context.is_executable()或者context.can_break()为True，则判定为XSS漏洞</li>
	<li>6._report_vuln()创建一个Vuln，并且存储到KB中</li>
	<li>7.注释：Context类族，match(), is_executable(), can_break()分析见下一节Context</li>
</ul>
<h3>5.Context</h3>
<ul style="color: #000000;">
	<li>数据成员name, data</li>
	<li>下面是类继承关系，大写字母开头的</li>
	<li>HtmlContext
<ul>
	<li>HtmlTag
<ul>
	<li>can_break
<ul>
	<li>如果' '，'>'在payload中，就判定存在xss漏洞</li>
</ul>
</li>
</ul>
</li>
	<li>HtmlText
<ul>
	<li>can_break
<ul>
	<li>如果'&lt;'在payload中，就判定存在xss漏洞</li>
</ul>
</li>
</ul>
</li>
	<li>HtmlComment
<ul>
	<li>can_break
<ul>
	<li>如果'-','>','&lt;'在payload中，就判定存在xss漏洞</li>
</ul>
</li>
</ul>
</li>
	<li>HtmlAttr
<ul>
	<li>HtmlAttrQuote
<ul>
	<li>HtmlAttrSingleQuote</li>
	<li><span id="show1_1" class="foldclosed" style="color: #666666;"></span><span id="hide1_1" class="foldopened" style="color: white;">-</span> HtmlAttrDoubleQuote
<ul id="fold1_1">
	<li>HtmlAttrDoubleQuote2Script
<ul>
	<li>HtmlAttrDoubleQuote2ScriptText，它的父类还有一个ScriptText</li>
</ul>
</li>
</ul>
</li>
	<li>HtmlAttrBackticks</li>
	<li>上面是HtmlAttrQuote的子类，下面是它的成员</li>
	<li>self.quote_character
<ul>
	<li>有三个',",`(backtick,可能显示不出来), （但是没找到赋值的地方）</li>
</ul>
</li>
	<li><span style="color: #ff0000;"><strong>can_break()</strong></span>
<ul>
	<li>如果quote_character在payload中，就判定为xss漏洞</li>
</ul>
</li>
	<li><strong><span style="color: #ff0000;">is_executable()</span></strong>
<ul>
	<li>该函数先将self.data变成小写，删除空格判断self.data是否以下面的字符串结尾，如果是，就返回True['href', 'src','onclick', 'ondblclick', 'onmousedown', 'onmousemove','onmouseout', 'onmouseover', 'onmouseup', 'onchange', 'onfocus', 'onblur', 'onscroll', 'onselect', 'onsubmit', 'onkeydown', 'onkeypress', 'onkeyup', 'onload', 'onunload'] +'=' +self.quote_character. 比如data的尾部是onclick=" ，则返回True</li>
</ul>
</li>
	<li><span style="color: #ff0000;"><strong>match(byte_chunk)</strong></span>
<ul>
	<li>有两个装饰器，其实是返回inside_html(not_html_comment(_match(byte_chunk)))</li>
	<li>_match(byte_chunk)
<ul>
	<li>data= byte_chunk.nhtml
<ul>
	<li><span id="show1_2" class="foldclosed" style="color: #666666;"></span><span id="hide1_2" class="foldopened" style="color: white;">-</span> normalize_html(data)
<ul id="fold1_2">
	<li>(data.replace("\\'",'')).replace('\\"','')</li>
	<li>data = StringIO(data)</li>
	<li>然后一个字符一个字符的处理data</li>
	<li>1.删除闭合的',",`</li>
	<li>2.处理标签&lt;>，分别替换为&amp;lt, &amp;gt。注意不处理注释&lt;!--</li>
</ul>
</li>
</ul>
</li>
	<li>如果data中含有&lt;...>，则返回False</li>
	<li>接下来会判断',",`是否闭合</li>
	<li>函数的功能好像是判断'.",`,尖括号是否闭合</li>
</ul>
</li>
	<li>not_html_comment()功能：如果出现&lt;!-->，即在注释中，返回False</li>
	<li>inside_html()
<ul>
	<li>判断inside_js
<ul>
	<li>搜索&lt;script></li>
</ul>
</li>
	<li>判断inside_style
<ul>
	<li>搜索&lt;style></li>
</ul>
</li>
	<li>当inside_js和inside_style同时False，inside_html返回True</li>
</ul>
</li>
</ul>
</li>
</ul>
</li>
	<li>can_break
<ul>
	<li>如果' '，'='在payload中，就判定存在xss漏洞</li>
</ul>
</li>
</ul>
</li>
</ul>
</li>
	<li>ScriptContext
<ul>
	<li>ScriptMultiComment
<ul>
	<li>can_break
<ul>
	<li>如果'/','*'在payload中，判定存在xss漏洞</li>
</ul>
</li>
</ul>
</li>
	<li>ScriptLineComment\</li>
	<li>ScriptQuote</li>
	<li>ScriptSingleQuote</li>
	<li>ScriptDoubleQuote</li>
	<li>ScriptText</li>
</ul>
</li>
	<li>StyleContext
<ul>
	<li>StyleText
<ul>
	<li>can_break
<ul>
	<li>如果'&lt;','/'在payload中，判定存在xss漏洞</li>
</ul>
</li>
</ul>
</li>
	<li>StyleComment</li>
	<li>StyleQuote</li>
	<li>StyleSingleQuote</li>
	<li>StyleDoubleQuote</li>
</ul>
</li>
</ul>

<h1>三、W3AF扫描XSS漏洞实例</h1>

<h3>1.搭建一个有xss漏洞的网站</h3>

安装Appserv，快速配置网站，在www目录下创建一个xss文件夹，在xss目录下新建两个文件如下：  

test.html

	<html>
	<head>
	<title> XSS 测试  </title>
	</head>
	<body>
	<form action="XSS.php" method="POST">
	请输入名字：<br>
	<input type="text" name="name" value=""></input>
	<input type="submit" value="提交"></input>
	</body>
	</html>

xss.php  

	<?php
	echo $_REQUEST[name];
	?>
	
<h3> 2.在w3af_console下扫描xss漏洞</h3>
配置扫描目标URL、配置爬虫webSpider、加载xss插件，扫描过程及结果如下：

	w3af>>> target set target http://localhost/xss/test.html
	w3af>>> plugins
	w3af/plugins>>> discovery webSpider
	w3af/plugins>>> discovery config webSpider
	w3af/plugins/discovery/config:webSpider>>> set onlyForward True
	w3af/plugins/discovery/config:webSpider>>> back
	w3af/plugins>>> audit xss
	w3af/plugins>>> back
	w3af>>> start
	Auto-enabling plugin: grep.httpAuthDetect
	New URL found by webSpider plugin: http://localhost/xss/XSS.php
	New URL found by webSpider plugin: http://localhost/xss/
	Found 3 URLs and 9 different points of injection.
	The list of URLs is:
	- http://localhost/xss/test.html
	- http://localhost/xss/XSS.php
	- http://localhost/xss/
	The list of fuzzable requests is:
	- http://localhost/xss/ | Method: GET
	- http://localhost/xss/ | Method: GET | Parameters: (C="D", O="A")
	- http://localhost/xss/ | Method: GET | Parameters: (C="M", O="A")
	- http://localhost/xss/ | Method: GET | Parameters: (C="N", O="A")
	- http://localhost/xss/ | Method: GET | Parameters: (C="N", O="D")
	- http://localhost/xss/ | Method: GET | Parameters: (C="S", O="A")
	- http://localhost/xss/XSS.php | Method: GET
	- http://localhost/xss/XSS.php | Method: POST | Parameters: (name="")
	- http://localhost/xss/test.html | Method: GET
	Cross Site Scripting was found at: "http://localhost/xss/XSS.php", using HTTP me
	thod POST. The sent post-data was: "name=<ScRIPT>a=/dmr6/%0Aalert(a.source)</SCR
	iPT>". This vulnerability affects ALL browsers. This vulnerability was found in
	the request with id 51.
	Scan finished in 3 seconds.
	w3af>>>

<h3>3.验证XSS漏洞</h3>

在IE浏览器下关闭XSS过滤，输入

	http://localhost/xss/XSS.php?name=<ScRIPT>a=/dmr6/%0Aalert(a.source)</SCRiPT>

可以看到弹窗。

<h1>四、小结</h1>

本文总结了漏洞扫描相关的几个基本概念，梳理了w3af对xss漏洞检测扫描的基本过程，深入源码进行分析，明白了扫描原理，最后构造一个xss漏洞，使用w3af_console进行XSS扫描，了解了其运行步骤。
