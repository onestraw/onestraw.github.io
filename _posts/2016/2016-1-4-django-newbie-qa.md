---
layout: single
author_profile: true
comments: true
title: Django Newbie Q&A
tagline: 
category : web
tags : [django, Web]
---
### 1.建议
建议新手先看完一本django开发入门书籍，熟悉`manage.py -h`常用命令！

### 2.安装特定版本的django，如1.8.7

	pip install django==1.8.7

### 3.查看当前django的版本
		
	python -c "import django; print django.VERSION"
		
### 4.render()渲染函数的context参数是什么？

	简单来说context是一个传送给template的key-value字典参数;   
	http://stackoverflow.com/a/20958082 
	
### 5.django 富文本编辑器
	tinymce: 配置简单，功能也简单，无图片上传；  
	ckeditor 功能强大，配置也复杂；  
	
### 6.ckeditor实现图片上传功能
	http://goo.gl/R1VlUz 

### 7.django 查询model时 filter和get的区别？ 

	如果查询结果存在，filter得到的是queryset，get得到的是一个对象；  
	http://stackoverflow.com/q/1018886   

### 8.django foreignkey 可以直接索引到对象
	这是ORM技术。传统的SQL关系设计一般用id连接，这里存储的外键，相当于对象；（具体细节不知）

### 9.django admin如何在一个model中动态添加多个对象？
	使用ManyToManyField 和 filter_horizontal，自动创建关系表；
	
### 10.django model 汉化
	设置verbose_name   
	https://goo.gl/F7wVdm 
	
### 11.在 admin 数据显示列表List_display 添加额外的链接  
	 http://stackoverflow.com/q/1949248   
	 
### 12.怎么在admin界面添加按钮，弹出对话框？  

	html modal技术+list_display的 url技术。  
	- 弹出modal	http://goo.gl/FTezq9 
	- 显示多个modal	http://stackoverflow.com/a/16494302 
	
### 13.怎样根据登录用户的属性显示不同的网站标题？
	不同的登录用户显示不同的用户名。类比：扩展user 表+修改 admin模板；  
	- 扩展User的官方文档 https://goo.gl/jb4X3p 
	- 修改模板：查找使用主题的admin/base.html
	
### 14.在多用户环境下，如何让每个用户只能读取自己的数据？（权限隔离）
	在admin.py中重写get_queryset()，根据request.user过滤数据；
	
### 15.导出数据
	https://github.com/django-import-export/django-import-export 
	
### 16.怎样在admin界面的 “增加XXXX”左边添加一个按钮？
	参考import-export应用，修改相应app/model的change_list.html模板
	
### 17.django-suit主题将一个app的model分成多个菜单显示
	参考https://github.com/darklow/django-suit/issues/96   
	注意同一个model不要显示在多个菜单，目前只显示model第一次出现的菜单中； 
	
### 18.如何设置django-suit 配置文件中的图标？
	bootstrap icon去这里找 http://marcoceppi.github.io/bootstrap-glyphicons/ 


