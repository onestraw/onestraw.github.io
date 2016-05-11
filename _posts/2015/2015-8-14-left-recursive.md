---
layout: single
author_profile: true
comments: true
title: 坑我两次的左递归
tagline: 
category: essay
tags : [面试]
---


人不能在同一个地方跌倒两次  
人不能踏进同一条河流  

-----------------------

呵呵，惨痛的教训！
由于我在简历上提到了flex/bison，两次被面试官问到左递归的概念，第一次是在阿里春招实习生的时候，第二次是前些天阿里内推HR面（审查实习生面试记录）。  

##左递归  left-recursive

> A grammar is left-recursive if and only if there exists a nonterminal symbol A that can derive to a sentential form with itself as the leftmost symbol.  
  在上下文无关文法G中，一个非终结符P经过一次或多次推导得到Pa（即能推导出以P开头的式子）， 则称G是左递归的。

即 

>	P => Pa 


左递归根据是直接推导还是间接推导，又分为直接左递归和间接左递归。详细见https://en.wikipedia.org/wiki/Left_recursion


##实例

最近简单看了下setkey的源码，它在解析配置文件(如/etc/ipsec.conf)时用的就是flex & bison，下面是一个左递归的例子。

		add_command
				:       ADD ipaddropts ipandport ipandport protocol_spec spi extension_spec algorithm_spec EOT

		extension_spec
				:       /*NOTHING*/
				|       extension_spec extension
				;

		extension
				:       F_EXT EXTENSION { p_ext |= $2; }
				|       F_EXT NOCYCLICSEQ { p_ext &= ~SADB_X_EXT_CYCSEQ; }
				|       F_MODE MODE { p_mode = $2; }
				|       F_MODE ANY { p_mode = IPSEC_MODE_ANY; }
				|       F_REQID DECSTRING { p_reqid = $2; }
				|       F_REPLAY DECSTRING
				
即 

>	extension_spec => NULL  
	extension_spec => extension_spec extension  
	
反应在setkey中的add命令上，它的作用是什么呢？ 

**extension可以设置多个参数选项**


> add 10.10.10.1 10.10.10.2 ah 0x200 -m any -u 1234 -A hmac-md5 0xca2cef3e4e00a0a111d3aa0048ec1ce3; 

上面的两个参数`-m any -u 1234 `均属于extension。
