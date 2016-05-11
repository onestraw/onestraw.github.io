---
layout: single
author_profile: true
comments: true
title: w3af学习笔记（二）
categories: [WebSec]
tags: [Web安全]
---

#0.引言

本文深入分析了w3af_console启动过程，详细说明了函数调用关系，有助于理解w3af框架。下面是w3af_console入口函数_main()
	
	def _main():
	    _configure_output_manager()
	    sys.exit(main())
    
#1._configure_output_manager深入分析

w3af_console执行的第一个函数_configure_output_manager()分析，执行的结果是创建的console对象保存在全局变量om.out中，om.out在main()函数中会用到，如console.sh()。下面<strong>用'->'表示调用或继承关系</strong>，一步一步的分析到源头。

	->(file:w3af_console)  
	_configure_output_manager()  
	
	->(file:w3af_console)  
	om.out.set_output_plugins( ['console'] )  
	
	ps: om就是out_manager module; out是类out_manager的一个全局对象,定义在\w3af\core\controllers\out_manager.py中  
	
	->(file:\w3af\core\controllers\out_manager.py)  
	set_output_plugins( ['console'] )  
	
	->(file:\w3af\core\controllers\out_manager.py)  
	_add_output_plugins( 'console')  
	
	->(file:\w3af\core\controllers\out_manager.py)  
	plugin = factory('w3af.plugins.output.' + OutputPluginName)  
	plugin.set_options(self._plugin_options[OutputPluginName])  
	self._output_plugin_instances.append(plugin)  
	
	->(file:\w3af\core\controllers\misc\factory.py)  
	<pre style="padding-left: 30px;" class="lang:python decode:true">factory('w3af.plugins.output.console')
	        #主要过程
	        __import__('w3af.plugins.output.console')
	        class_name = module_name.split('.')[-1]  #class_name='console'
	        module_inst = sys.modules['w3af.plugins.output.console']
	        a_class = getattr(module_inst, 'console')  # a_class 现在等价于console类
	        return a_class()</pre>
	<strong>(到此就返回一个console对象，下面是几个类的继承关系)</strong>
	->(file:\w3af\plugins\output\console.py)
	class console(OutputPlugin):
	...
	
	->(file:\w3af\core\controllers\plugins\output_plugin.py)
	class OutputPlugin(Plugin):
	"""
	This is the base class for data output, all output plugins should inherit
	from it and implement the following methods :
	1. debug( message, verbose )
	2. information( message, verbose )
	3. error( message, verbose )
	4. vulnerability( message, verbose )
	"""
	
	->(file:\w3af\core\controllers\plugins\plugins.py)
	class Plugin(Configurable):
	...
	
	->(file:\w3af\core\controllers\configurable.py)
	class Configurable(object):
	#This is mostly "an interface"

#2.main函数深入分析

假定启动w3af的命令为

	./w3af_console -s script_file

###2.1 ConsoleUI

main()函数中首先解析script_file,生成commands_to_run[]，根据commands,初始化一个ConsoleUI对象console，这是用户操作的UI. 有两个关键操作，self._handlers初始化和self._initRoot()
	
	#_handlers是一个字典，键盘输入字符是key，函数是value 
    self._handlers = {
            '\t': self._onTab,
            '\r': self._onEnter,
            term.KEY_BACKSPACE: self._onBackspace,
            term.KEY_LEFT: self._onLeft,
            term.KEY_RIGHT: self._onRight,
            term.KEY_UP: self._onUp,
            term.KEY_DOWN: self._onDown,
            '^C': self._backOrExit,
            '^D': self._backOrExit,
            '^L': self._clearScreen,
            '^W': self._delWord,
            '^H': self._onBackspace,
            '^A': self._toLineStart,
            '^E': self._toLineEnd
        }
        
    def __initRoot(self, do_upd):
        """
        Root menu init routine.
        """
        cons_upd = ConsoleUIUpdater(force=do_upd)
        cons_upd.update()
        # Core initialization
        self._w3af = w3afCore()
        self._w3af.plugins.set_plugins(['console'], 'output') #关键
        ...
        #一直递归下去，在类w3af_core_plugins（实例是上面的self._w3af.plugins）发现调用的是
        self._set_plugin_generic(self, 'output', ['console'])
            self._plugins_names_dict['output'] = ['console']</pre>

(ps: file:\w3af\core\ui\console\console_ui.py  

ConsoleUI class represents the console.It handles the keys pressed and delegate the completion and execution tasks to the current menu.)

###2.2 accept_disclaimer

创建完console实例后，console调用accept_disclaimer方法，它输出w3af的免责条款，如果用户接受，就可以继续使用，否则退出。

###2.3 sh

最后调用console.sh()，进行主循环，创建一个w3af shell，执行w3af的命令。

->(file:\w3af\core\ui\console\console_ui.py)
删除异常处理语句等，sh()完成的主要工作如下

	def sh(self,name="w3af",callback=None)
	    self._context = rootMenu(name, self, self._w3af)
	    self._showPrompt()
	    self._active = True
	    term.setRawInputMode(True)
			
	    self._executePending()
			
	    while self._active:#主循环，等待用户输入，解析命令及参数
	        c = term.getch()
	        self._handleKey(c) #根据字典self._handlers调用相关方法
	        
###2.4 rootMenu

对于sh()的第一条语句：

	self._context = rootMenu(name, self, self._w3af)

初始化一个rootMenu类的实例_context，顾名思义，包括plugins,target,exploit等根菜单；rootMenu初始化的关键代码如下
->(file:\w3af\core\ui\console\rootMenu.py)
	
	class rootMenu(menu):
		def __init__(self, name, console, core, parent=None):
			menu.__init__(self, name, console, core, parent)
			self._load_help('root')

			#   At first, there is no scan thread
			self._scan_thread = None

			mapDict(self.addChild, {
				'plugins': pluginsMenu,
				'target': (ConfigMenu, self._w3af.target),
				'misc-settings': (ConfigMenu, MiscSettings()),
				'http-settings': (ConfigMenu, self._w3af.uri_opener.settings),
				'profiles': profilesMenu,
				'bug-report': bug_report_menu,
				'exploit': exploit,
				'kb': kbMenu
			})
		def mapDict(fun, dct):
			for p in dct:
				fun(p, dct[p])
				
		def addChild(self, name, constructor):
			if type(constructor) in (tuple, list):
				constructor, params = constructor[0], constructor[1:]
			else:
				params = []

			self._children[name] = constructor(
				name, self._console, self._w3af, self, *params)</pre>
				
对于字典的第一项'plugins': pluginsMenu，mapDict()执行过程如下
self.addchild('plugins',pluginsMenu)
-->
self._children['plugins']=pluginsMenu('plugins', self._console, self._w3af, self,[])

-->(file:\w3af\core\ui\console\plugins.py)
类pluginsMenu
__init__中首先得到plugins目录下的插件类型types（除去attack,tests,.gits的所有目录名）

	types=['audit','auth','bruteforce','crawl','evasion','grep','infrastructure','mangle','output']
	for t in types:
	self.addChild(t, pluginsTypeMenu)
	self._children['audit']=pluginsTypeMenu('audit',...)

-->(file:\w3af\core\ui\console\plugins.py)
类pluginsTypeMenu
__init__中将plugins/plugin_types/目录下所有插件读入到plugins[]中
最后保存到一个插件字典中self._plugins={}
self._plugins[plugin_name]=plugin_option_num

###2.5 _showPrompt

功能输出提示符[plugins...]>>>

	->(file:\w3af\core\ui\console\console_ui.py)
	term.write(self._context.get_path() + ">>> ")
	
	->(file:\w3af\core\ui\console\menu.py)
	get_path()

###2.6 _executePending

调用_executePending(),它顺序读取console初始化时的输入参数commands指令curCmd  
_paste(curCmd)将指令在终端stdout显示出来，同时保存至self._line[]  
_onEnter()依次：执行指令，初始化当前行_position=0和_line=[]，在新一行输出">>>"；  
(ps:在当前类中找不到方法，去父类中找，如rootMenu中没有execute，在其父类menu中找)

->(file:\w3af\core\ui\console\console_ui.py)

	def _executePending(self):
        while (self._commands):
            curCmd, self._commands = self._commands[0], self._commands[1:]
            self._paste(curCmd)
            self._onEnter()
    def _paste(self, text):
        tail = self._line[self._position:]
        for c in text:
            self._line.insert(self._position, c)
            self._position += 1

        term.write(text)
        term.write(''.join(tail))
        term.moveBack(len(tail))
        
    def _onEnter(self):
        self._execute()  #关键
        self._initPrompt()
        self._showPrompt()

    def _execute(self):
        line = self._getLineStr()
        term.setRawInputMode(False)
        om.out.console('')
        if len(line) and not line.isspace():
            self._get_history().remember(self._line)

            params = self.in_raw_line_mode() and line or self._parseLine(line)
            menu = self._context.execute(params)

            if menu:
                if callable(menu):
                    menu = menu()
                elif menu != self._context:
                    # Remember this for the back command
                    self._trace.append(self._context)
                if menu is not None:
                    self._context = menu
        term.setRawInputMode(True)</pre>

从上述代码发现函数调用路线如下  

	self._executePending()->self._onEnter()->self._execute()->self._context.execute()   

self._context是类rootMenu的对象，但是rootMenu类中没有execute()方法，那么去其父类menu中查找。  

###2.7 menu.execute

(file:\w3af\core\ui\console\menu.py)

execute方法完整注释如下

	#class menu
    def execute(self, tokens):
        if len(tokens) == 0:
            return self  

        command, params = tokens[0], tokens[1:]
        handler = self.get_handler(command)
        # hander 可能是:
		# _cmd_back(), _cmd_exit(), _cmd_keys(), _cmd_help(), _cmd_print()
		# 子类中按需要增加了各自特殊_cmd__XXX，如ConfigMenu类中增加了_cmd_set(),_cmd_view()..
        if handler:   # 递归出口
            return handler(params)

        children = self.get_children() 
		'''
		return self._children, it's  a dict
        在menu的子类rootMenu中对self._children进行了初始化，代码见2.4 rootMenu
		
		'''
		if command in children:  # in children.keys()
            child = children[command]
			#child可能是一个pluginsMenu, ConfigMenu
			
            child.set_child_call(True)
			#set_child_call函数处理 set命令，如w3af>>> target set target http://w3af.org/
			
            try:
                return child.execute(params) 
				# 按上例，params = set target http://w3af.org/
				# 由于child也是继承自menu类，所以execute是同一个方法
				# 下次调用execute时在handler中找到了_cmd_set()方法
				# 直接执行_cmd_set(' target http://w3af.org/')，不再向下递归
            finally:
                child.set_child_call(False)

        raise BaseFrameworkException("Unknown command '%s'" % command)</pre>
        
对于上面的最终执行函数handler()，仅分析一个set命令:类ConfigMenu的_cmd_set()方法。

###2.8 _cmd_set

(file: \w3af\core\ui\console\config.py)

_cmd_set()的主干如下

	#去掉的异常处理等
    def _cmd_set(self, params):
        name = params[0]
        value = ' '.join(params[1:])
        
        self._options[name].set_value(value)
        self._unsaved_options[name] = value

        if value not in self._memory[name]:
            self._memory[name].append(value)
  
        if self._child_call:
            self._cmd_save([])</pre>
            
比如命令 w3af>>> target set target http://w3af.org/  

从execute()方法执行到_cmd_set()方法时  

params=['target','http://w3af.org/']  

即name='target', value='http://w3af.org/'  

下面关键的就是self._options  

###2.9 w3af_core_target

(file:\w3af\core\ui\console\config.py)

	class ConfigMenu(menu)
	def __init__(self, name, console, w3af, parent, configurable):
	self._configurable = configurable
	self._options = self._configurable.get_options()

通过回滚，找到'target'插件对象创建的时刻就是在rootMenu对象创建时，去前文找mapDict()。核对参数发现当时传入的configurable是self._w3af.target（file:\w3af\core\ui\console\rootMenu.py）

->(file:\w3af\core\controllers\w3afCore.py)
class w3afCore的__init__中有target的创建
self.target = w3af_core_target()
->(file:\w3af\core\controllers\core_helpers\target.py)

get_options()涉及到的细节过多，跟w3af_console主干没有太多关系，因此不再深挖下去，以后再做分析。

#3. 总结

w3af启动时，首先添加console插件，配置一个output_manager全局对象om.out，这是创建ConsoleUI，与用户交互的基础。然后在main函数中创建ConsoleUI对象，执行初始化，进入最重要的环节，生成一个w3af shell，创建rootMenu，读取命令行参数script，进行相关设置，最后进行死循环while self._active，等待用户输入，解析输入，如果是正确的命令，就（有些需要调用插件）执行相应操作，至此用户才能跟w3af进行正常交互，w3af启动成功。
