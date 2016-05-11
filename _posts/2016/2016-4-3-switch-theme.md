---
layout: single
author_profile: true
title: 升级Jekyll主题
excerpt: "从Jekyll-bootstrap切换到Minimal Mistakes"
tags: [jekyll, theme]
category: web
comments: true
---
{% include toc icon="gears" title="目录" %}

使用Minimal Mistakes 3.1.5 版本，详细配置文档见[quick start guide](https://mmistakes.github.io/minimal-mistakes/docs/quick-start-guide/)

## 添加文章目录
在post/page 正文头部添加如下代码
{% raw %}
	{% include toc icon="gears" title="目录" %}
{% endraw %}


Kramdown解析 `{:toc}`， 生成目录。


## 评论插件disqus

使用Minimal Mistakes主题，需要在每一篇post头部指定是否允许评论。

{% highlight bash %}
cd _posts
grep "tags" -r .
# 发现写过2014-12-28-vim-notes.md，包含很多tags
grep "layout:" -r .
# 正文内容没有layout
find . -name "*.md" |xargs sed -i '/layout:/a\comments: true'
{% endhighlight %}

## 代码高亮

{% highlight html %}{% raw %}
{% highlight bash %}
	touch hello
	rm hello
{% endhighlight %}
{% endraw %}{% endhighlight %}
使用标签 `highlight`和`endhighlight` 高亮代码，在highlight后用lang指定语言名，可用语言列表见[language names and aliases](http://highlightjs.readthedocs.org/en/latest/css-classes-reference.html)

## 修改layout

{% highlight bash %}
cd _posts
find . -name "*.md" |xargs sed -i 's/layout: post/layout: single/g'
{% endhighlight %}

## 添加author_profile

{% highlight bash %}
cd _posts
find . -name "*.md" |xargs sed -i '/layout: /a\author_profile: true'
{% endhighlight %}

## ruby 术语

连ruby菜鸟也算不上的生手搭建jekyll本地环境要了解一些ruby术语: [rvm, bundle, gem等](http://www.cnblogs.com/ziyouchutuwenwu/p/4099690.html)  

启动本地服务的命令： `bundle exec jekyll serve --host 192.168.6.136`
