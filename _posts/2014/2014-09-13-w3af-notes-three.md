---
layout: single
author_profile: true
comments: true
title: w3af学习笔记（三）
categories: [WebSec]
tags: [Web安全]
---

本文以menu切换，插件加载与启动为主线，分析w3af源代码中函数调用和各类之间的继承与组合关系，阅读之前请先有一个DFS（深度优先遍历）的准备。
<h1>一、menu切换</h1>
<h2>1.menu类继承关系</h2>

- object
- <span style="color: #ff0000;">menu</span>
 + <span style="color: #ff0000;">rootMenu</span>
 + <span style="color: #ff0000;">pluginsMenu</span>
  * <span style="color: #ff0000;">pluginsTypeMenu</span>
 + ConfigMenu
 + profilesMenu
 + bug_report_menu
 + kbMenu
 + exploit
 
<h2>2.w3af>>plugins执行过程</h2>
输入plugins+回车之后，指令从ConsoleUI类的sh()方法依次执行：

self._onEnter()->self._execute()->self._context.execute()

为了分析清楚类之间的关系，下文统一将self替换成类的名字，如果实例对象对分析过程有影响，会特别标出对象名字。

sef._context，在sh()中初始化时是rootMenu类的实例，顾名思义，context是一个上下文，它会随着程序的执行而变化，具体的说它会随着w3af shell提示符的变化而变化，从回车之后，一步一步深入分析下去，过程如下。

ConsoleUI._onEnter()->ConsoleUI._execute()->rootMenu.execute()

<h3>2.1 rootMenu.execute()</h3>
ConsoleUI._context就是类rootMenu的对象，rootMenu是menu类的子类，rootMenu没有重新定义execute方法，所以最后执行的是menu.execute('plugins')
<p style="padding-left: 30px;">childrenmenu.get_children()#返回一个字典menu._children
child = children['plugins']#返回一个类pluginsMenu对象，它在rootMenu对象创建时完成的创建
reutn child.execute([])#即pluginsMenu.execute([])</p>
由于参数为空，转向执行父类的execute函数，menu.execute([])，在该函数中，由于参数为空，终止递归，return self，此处也就是pluginsMenu对象。

<h3>2.2 ConsoleUI._execute()</h3>

返回上一层 ConsoleUI._execute()，接着执行：

    menu = pluginsMenu  #这是2.1返回的一个对象，menu在此是一个临时变量
    ConsoleUI._trace.append(ConsoleUI._context) #保存路径
    if menu is not None:
    self._context = menu  #完成上下文切换

<h3>2.3 ConsoleUI._onEnter()</h3>

再返回上一层ConsoleUI._execute()，接着执行：

    ConsoleUI._initPrompt()
    ConsoleUI._showPrompt()#重新显示shell提示符：w3af/plugins>>>

<h2>3.w3af/plugins>>>back执行过程</h2>
输入back+回车之后，指令从ConsoleUI类的sh()方法依次执行：

ConsoleUI._onEnter()->ConsoleUI._execute()->pluginsMenu.execute()

<h3>3.1 pluginsMenu.execute()</h3>

目前ConsoleUI._context就是类pluginsMenu的对象，pluginsMenu是menu类的子类，由于back不是一个插件类型，所以最后执行的是menu.execute('back')，其执行过程如下：

    return pluginsMenu._cmd_back()
    return pluginsMenu._console.back()
    return ConsoleUI.back()
    return ConsoleUI._trace.pop()

从第2节能够知道_trace在尾部append的是rootMenu对象，所以此处弹出的也是rootMenu对象。

<h3>3.2 ConsoleUI._execute()</h3>

返回上一层 ConsoleUI._execute()，接着执行：

    menu = rootMenu  #这是2.1返回的一个对象，menu在此是一个临时变量
    if menu is not None:
        self._context = menu  #完成上下文切换

<h3>3.3 ConsoleUI._onEnter()</h3>

再返回上一层ConsoleUI._execute()，接着执行：

    ConsoleUI._initPrompt()
    ConsoleUI._showPrompt()#重新显示shell提示符：w3af>>>

<h1>二、加载插件</h1>

加载爬虫插件web spider的命令是：
w3af新版本命令：w3af/plugins>>>crawl web_spider
w3af旧版本命令：w3af/plugins>>>discovery webSpider

<h2>1.命令解析</h2>

crawl/discovery代表插件类型，w3af的插件类型就是w3af\plugins目录下的目录名（除去attack，tests），包括audit,crawl,auth,output等。它对应于类pluginsTypeMenu.

web_spider/webSpider代表具体的插件名字，位于w3af\plugins\插件类型\目录下。

<h2>2.获取所有的可用插件</h2>

从w3af_console启动，到获取每一种类型的所有插件的过程如下

    1）ConsoleUI的_init_()中 self._w3af = w3afCore()
    2）ConsoleUI的sh()中调用rootMenu(...,self._w3af,...)
    3）rootMenu的_init_()中生成pluginsMenu对象
    4）pluginsMenu的_init_()中调用w3af.plugins.get_plugin_types()，返回所有的插件类型types
    5）对于每一个插件类型type，生成一个pluginsTypeMenu对象
    6）pluginsTypeMenu的_init_()中调用w3af.plugins.get_plugin_list(name)，返回一种插件的所有插件名字
    7）对于一种插件的所有具体插件，pluginsTypeMenu的_init_()中调用self._w3af.plugins.get_plugin_inst(self._name, p).get_options()返回一个具体插件的使用选项说明

ps: w3af.plugins是类w3af_core_plugins对象

<h2>3.w3af/plugins>>>crawl web_spider</h2>

此时的上下文是pluginsMenu（参考一.3）

ConsoleUI._onEnter()->ConsoleUI._execute()->pluginsMenu.execute(['crawl','web_spider'])

<h3>3.1 pluginsMenu.execute(['crawl','web_spider'])</h3>

1)pluginsMenu.execute(['crawl','web_spider'])

2)menu.execute(['crawl','web_spider'])

    commands='crawl'</p>
    params=['web_spider']</p>
    #由上一节可知，每一个插件类型已经生成pluginsTypeMenu对象，此外查找crawl对应的pluginsTypeMenu

3)pluginsTypeMenu.execute(['web_spider'])

4)pluingsTypeMenu._enablePlugins('web_spider')

5)w3afCore.w3af_core_plugins.set_plugins([...,'web_spider'],'crawl')

6)w3af_core_plugins._set_plugins_generic('crawl','web_spider') #加入字典_plugins_names_dict

至此，web_spider插件加载成功。
<h3>3.2 返回过程省略</h3>

<h1>三、启动插件</h1>

w3af>>start执行后，如何创建web_spider插件类实例，并启动插件呢？

<h2>1. 创建web_spider实例</h2>
首先，当前的上下文_context是rootMenu。函数调用过程如下

1)rootMenu._cmd_start()

2)rootMenu._real_start()

3)rootMenu._w3af.plugins.init_plugins()

即w3af_core_plugins.init_plugins()

4)w3af_core_plugins.plugin_factory()

5)w3af_core_plugins.create_instances()

6)w3af_core_plugins.get_plugin_inst()
    
    def get_plugin_inst(self, plugin_type, plugin_name):
        """
        :return: An instance of a plugin.
        """
        plugin_inst = factory('w3af.plugins.%s.%s' % (plugin_type, plugin_name))
        plugin_inst.set_url_opener(self._w3af_core.uri_opener)
        plugin_inst.set_worker_pool(self._w3af_core.worker_pool)
        
        if plugin_name in self._plugins_options[plugin_type].keys():
            custom_options = self._plugins_options[plugin_type][plugin_name]
            plugin_inst.set_options(custom_options)

        # This will init some plugins like mangle and output
        if plugin_type == 'attack' and not self.initialized:
            self.init_plugins()
            
        return plugin_inst
        
7）factory类，就是import web_spider 插件，同时保证插件中的类名字和文件名字完全相同；web_spider类的继承关系为 

    object-->Configurable-->plugin-->CrawlPlugin-->web_spider

<h2>2. 运行插件</h2>
web_spider插件的启动过程如下：

1) rootMenu._cmd_start()

2) rootMenu._real_start()

3) rootMenu._w3af.start()

4) w3af_core_strategy.start()

5) w3af_core_strategy._setup_crawl_infrastructure()

6) crawl_infrastructure.start()

<h1>四、小结</h1>

本文用DFS的方式，跟踪分析了w3af_console的menu切换，插件加载与启动的过程，理清了函数调用关系，初步弄懂了各类之间的关联。
