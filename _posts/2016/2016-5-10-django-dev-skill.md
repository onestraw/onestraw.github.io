---
layout: single
author_profile: true
title: Django开发技巧
excerpt: "针对常见问题，介绍一些实用的Django开发技巧"
tags: [Django, Python]
category: web
comments: true
---
{% include toc icon="gears" title="目录" %}

## 如何将新创建的User自动加入Group

> 这个问题引申为：如何劫持创建User动作，对新用户自定义一些操作 ?  

Django 包含一个信息分发器`signal dispatcher`，在松耦合的框架中用信号的方式通知另一处代码执行操作。下面是连接信号发送者和接收者的connect函数：

{% highlight python %}
Signal.connect(receiver, sender=None, weak=True, dispatch_uid=None)

#receiver函数的格式为
def recv_callback(sender, **kwargs):
	pass
{% endhighlight %}

Django model包含一些内置信号函数。

	django.db.models.signals.pre_save
	django.db.models.signals.post_save
	django.db.models.signals.pre_delete
	django.db.models.signals.post_delete

post_save在Model 的 save()执行结束前给receiver函数发送信号，执行特定操作。pre_save 在 进入save()时就发送信号。下面就使用post_save()完成新user加入group的操作。


{% highlight python %}
from django.contrib.auth.models import User, Group
from django.db.models.signal import post_save

def create_user_callback(sender, **kwargs):
    user = kwargs['instance']
    if kwargs['created']:
        user.is_staff = True
        user.save()
        try:
            g = Group.objects.get(name='group2')
        except Group.DoesNotExist:
	    #创建用户组
            g = create_group2()
        g.user_set.add(user)

#创建User, save()完成前向create_user_callback函数发送信号
post_save.connect(create_user_callback, sender=User)
{% endhighlight %}

此外，信号还有一些用途，当一具Model有FileField时，

- 上传文件时可利用pre_save信号，执行文件扫描任务；
- 删除文件时利用post_delete或者pre_delete，从磁盘中删除文件；


## 如何创建一个Group，并设置权限？

假设现在有一个Model是Product，需要将Product的增加和修改权限加入group2;
每一个Model都有3个权限add, change, delete；存储在Django的Permission表中。


{% highlight python %}
from django.contrib.auth.models import User, Group, Permission

def create_group2():
    permlist = [
        'add_product',
        'change_product',
        #'delete_product',
    ]
    g,created = Group.objects.get_or_create(name='group2')
    for name in permlist:
        perm = Permission.objects.get(codename=name)
        g.permissions.add(perm)
    g.save()
    return g
{% endhighlight %}


## 在多用户场景下，Django admin如何对各个用户的数据进行隔离？

需求：
	
1. superuser可以看到所有数据；
2. 非superuser只能看到自己的数据； 前两个用get_queryset 即可实现；
3. Model中有一个ManyToManyField，在创建新对象时，要控制可选数据列表；实现formfield_for_manytomany函数 ；

{% highlight python %}
from django.contrib import admin
class ProductAdmin(admin.ModelAdmin):
    def get_queryset(self, request):
        qs = super(ProductAdmin, self).get_queryset(request)
        if request.user.is_superuser:
            return qs
        return qs.filter(user=request.user)

class ActivityAdmin(admin.ModelAdmin):
    def formfield_for_manytomany(self, db_field, request, **kwargs):
        if db_field.name == 'product_list':
            kwargs['queryset'] = models.Product.objects.filter(
                user=request.user)
        return super(ActivityAdmin, self).formfield_for_manytomany(
            db_field, request, **kwargs)
{% endhighlight %}

## Django admin 搜索外键
如果B有一个外键A，A有一个外键User，在B的admin中想通过User.username进行搜索过滤，下面是一个可行方法

{% highlight python %}
# model.py
class A(models.Model):
	user = models.OneToOneField(User)
	...

class B(models.Model):
	a = models.ForeignKey(A)
	...

# admin.py
class BAdmin(admin.ModelAdmin):
	search_fields = ('a__user__username', )
	...
{% endhighlight %}

## 禁止通过admin界面删除对象

对于一些管理系统，可能有些对象只准程序操作，不准用户添加、删除，只许查看，防止用户滥用职权。


{% highlight python %}
class BAdmin(admin.ModelAdmin):
    ...
    def get_actions(self, request):
        actions = super(BAdmin, self).get_actions(request)
        if 'delete_selected' in actions:
            del actions['delete_selected']
        return actions
    # or   
    def has_delete_permission(self, request):
        return False

    def has_add_permission(self, request):
	pass

{% endhighlight %}


## views.py 文件过大，如何组织代码结构？

假定myapp下的views.py文件超过1000行，可行的组织方法为：

- 创建`myapp/views`目录
- 按功能分隔成a.py, b.py, ...，放在views目录下
- 创建`myapp/views/\_\_init\_\_.py，内容如下

{% highlight python %}
	from a import *
	from b import *
{% endhighlight %}
