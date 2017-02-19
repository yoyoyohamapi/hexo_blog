title: Symfony学习笔记（2）-- View视图
tags: symfony
categories: Symfony学习笔记
-----
PHP的操纵前端的主要手段就是模板引擎，如Smarty等，Symfony的前端引擎用的是[Twig](http://twig.sensiolabs.org/)。

Twig通过如下三种形式同html代码区分开，并包裹所需要的动态内容，信息甚至是业务逻辑：

包裹变量或者表达式的值:

```twig
{{ ... }} 
```

包裹控制逻辑或者函数语句:

```twig
{% ... %}
```

包裹注释:

```twig
{# ... #}
```

如下代码片所示:

```twig
<!DOCTYPE html>
<html>
 <head>
 <title>{{ page_title }}</title>
 </head>
 <body>
 <h1>{{ page_title }}</h1>
<ul id="navigation">
 {% for item in navigation %}
 <li><a href="{{ item.url }}">{{ item.label }}</a></li>
 {% endfor %}
 </ul>
 </body>
</html>
```

###对象属性的获取:

```twig
{# 1. Simple variables #}
{# array('name' => 'Fabien') #}
{{ name }}
{# 2. Arrays #}
{# array('user' => array('name' => 'Fabien')) #}
{{ user.name }}
{# alternative syntax for arrays #}
{{ user['name'] }}
{# 3. Objects #}
{# array('user' => new User('Fabien')) #}
{{ user.name }}
{{ user.getName }}
{# alternative syntax for objects #}
{{ user.name() }}
{{ user.getName() }}
```

###模板继承（即可以继承已有的布局（layout）文件，重新装填布局内元素）：

```twig
{# src/Acme/DemoBundle/Resources/views/Demo/hello.html.twig #}
{% extends "AcmeDemoBundle::layout.html.twig" %}//该::说明不需要通过控制器
{% block title "Hello " ~ name %}
{% block content %}
 <h1>Hello {{ name }}!</h1>
{% endblock %}
```

###使用标签、过滤和函数：

```twig
<h1>{{ article.title|trim|capitalize }}</h1>
<p>{{ article.content|striptags|slice(0, 1024) }}</p>
<p>Tags: {{ article.tags|sort|join(", ") }}</p>n
<p>Next article will be published on {{ 'next Monday'|date('M j, Y')}}</p>
```

###引入其他模板：

```twig
{# src/Acme/DemoBundle/Resources/views/Demo/hello.html.twig #}
{% extends "AcmeDemoBundle::layout.html.twig" %}
{# override the body block from embedded.html.twig #}
{% block content %}
{{ include("AcmeDemoBundle:Demo:embedded.html.twig") }}
{% endblock %}
``` 

###超链接（Path函数产相对路径，URL函数产生绝对路径）：

```twig
<a href="{{ path('_demo_hello', { 'name': 'Thomas' }) }}">Greet Thomas!</a>
```

###引入资源文件：

```twig
<link href="{{ asset('css/blog.css') }}" rel="stylesheet" type="text/css" />
<img src="{{ asset('images/logo.png') }}" />
```