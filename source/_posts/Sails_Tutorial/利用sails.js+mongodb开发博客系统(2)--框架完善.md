title: 利用Sails.js+MongoDB开发博客系统(2)--框架完善
tags: sails,mongodb,nodejs
categories: 利用Sails.js+MongoDB开发博客系统
---
## 章节概述
在本章中，你将能学到如下知识：

* 如何将sails的模板引擎替换为swig，并且设置和扩展swig
* 集成bower来管理我们的前端库
* 集成compass来更优雅的撰写css
* 通过grunt来监听scss文件变动，并自动编译
* 前端模块化开发思路即实现
* 集成semantic-ui来撰写UI
* 利用grunt来对产品环境下的访问进行优化

## 更换模板引擎
Sails默认的模板引擎为ejs，仅就我个人而言，纯主观上来说，对这个框架不是很喜欢，尤其ejs不支持继承，令我大为恼火。所以我会考虑用[__jinja__](http://jinja.pocoo.org/)风格的swig来做模板引擎。

### 安装swig
> 标准的swig提供的filter，tag等可能不够用，比如字符串分割（split）等就未提供，为此，我们也需要安装其扩展

```
npm install swig --save
npm install swig-extras --save
```


### 替换默认引擎
在__config/views.js__中，修改egine为swig,并对swig做一些方便我们开发的配置，比如在这里，我们设定一些常用资源路径，避免了在页面中每次都要书写冗长的路径前缀。同时，为了在开发环境下修改swig而不用重启服务器，我们需要设置swig默认不缓存:

```js
var extras = require('swig-extras');
module.exports.views = {
    engine: {
        /* Template File Extension */
        ext: 'swig',

        /* Function to handle render request */
        fn: function (path, data, cb) {
            /* Swig Renderer */
            var swig = require('swig');
            // 保证我们在开发环境下每次更改swig不用重启sails
            if (data.settings.env === 'development') {
                swig.setDefaults({cache: false});
            }
            /*
             * 绑定一些常用路径
             * Thanks to: https://github.com/mahdaen/sails-views-swig
             * */
            var paths = {
                script: '/js',
                style: '/styles/default',
                image: '/images',
                font: '/fonts',
                icon: '/icons',
                bower: '/bower_components'
            };

            if (!data.path) {
                data.path = paths;
            }
            else {
                for (var key in paths) {
                    if (!key in data.path) {
                        data.path[key] = paths[key];
                    }
                }
            }
            // 补充extra
            extras.useFilter(swig, 'split');
            /* Render Templates */
            return swig.renderFile(path, data, cb);
        }
    },

    layout: 'layout',

    partials: false

};

};
```

### 创建一些页面
在__views/__下创建一些swig页面,其中__partial__文件夹下为一些公用视图部分或者视图模板：

![views_dir](http://7pulhb.com1.z0.glb.clouddn.com/sails-views_dir.png)

并初始化一些内容
__layout.swig__:

```twig
<!DOCTYPE html>
<html>
<head>
    <title>{% block title -%}{%- endblock %}</title>
    {# 引入常用css文件 #}
    {% include "stylesheets.swig" -%}
    {% block stylesheets -%}{%- endblock %}
</head>
<body>
    {# 引入导航栏 #}
    {% include "header.swig" -%}
    {# 主内容显示部分，提供给集成页面重写 #}
    <div class="mainContainer">
        {% block content -%}
        {%- endblock %}
    </div>
    {# 引入常用脚本 #}

    {% include "scripts.swig" %}
</body>
</html>
```

__index.swig__:

```js
{% extends 'partial/layout.swig' %}
{% block title -%}Woo的博客{%- endblock %}
{% block content -%}
    <div>
        <p>
            这是Woo的博客?
        </p>
    </div>
{%- endblock %}
```

### 设置路由：
为我们新添加的index.twig添加路由

```js
 '/' : {
    view :'index'
  }
```

OK,现在我们执行

```
sails lift
```

并输入[http://localhost:1337](http://localhost:1337),就能看到如下页面：

![index_show](http://7pulhb.com1.z0.glb.clouddn.com/sails-index_show.png)

## 利用bower来管理我们的前端库
### 安装bower

```
npm install bower -g
```

### 在项目中配置bower
在项目根目录下创建.bowerrc文件,并在文件中设置bower库目录

```json
{
  "directory":"assets/bower_components",
}
```

之所以需要将bower下载下的组件放置在assets目录下，是方便我们在不用撰写任何grunt任务的情况下就能在页面中引用bower下载的库资源。

### 使用bower
我们尝试利用bower来下载jquery

```
bower install jquery --save
```

可以看到，jquery的响应资源已经成功地安装在了在assets/bower_components目录下

![bower_jquery](http://7pulhb.com1.z0.glb.clouddn.com/sails-bower_jquery.png)

## 利用compass来撰写CSS
### 安装compass
基于和bower的目录配置同样的理由，我们选择将compass放置在assets目录下：

```
gem install compass
cd assets
compass init
```

### 配置compass
修改assets/config.rb的文件内容如下：

```ruby
require 'compass/import-once/activate'
# Require any additional compass plugins here.

# Set this to the root of your project when deployed:
http_path = "/"
css_dir = "styles"
sass_dir = "sass"
images_dir = "images"
javascripts_dir = "js"


# You can select your preferred output style here (can be overridden via the command line):
# output_style = :expanded or :nested or :compact or :compressed
output_style = :expanded

# To enable relative paths to assets via compass helper functions. Uncomment:
# relative_assets = true

# To disable debugging comments that display the original location of your selectors. Uncomment:
# line_comments = false


# If you prefer the indented syntax, you might want to regenerate this
# project again passing --syntax sass, or you can uncomment this:
# preferred_syntax = :sass
# and then run:
# sass-convert -R --from scss --to sass sass scss && rm -rf sass && mv scss sass
```

### 设置compass任务
OK，接下来我们需要创建响应的grunt task来设置compass的编译任务,使得每次__assets/sass__文件夹下的scss文件变动时，不用手动输入命令就能完成scss到css的编译工作:

### 安装grunt-contrib-compass

```
npm install grunt-contrib-compass 
```

### 创建compass任务
在__tasks/config__下新建__compass.js__并添加如下内容：

```js
/**
 * 指定Compass配置文件，完成sass编译
 */
module.exports = function(grunt){
    grunt.config.set('compass',{
        dist: {
            options: {
                config: 'assets/config.rb',
                // 重要，如果不声明assets，compass无法找到待编译的scss文件
                basePath : 'assets'
            }
        }
    });

    grunt.loadNpmTasks('grunt-contrib-compass');

}
```

### 监视scss文件变动
修改__tasks/config__下的__watch.js__任务，使得当scss文件变动时，compass能够自动编译scss至css。

```js
module.exports = function(grunt) {

	grunt.config.set('watch', {
		options :{
			livereload: true
		},
		api: {

			// API files to watch:
			files: ['api/**/*', '!**/node_modules/**']
		},
		assets: {

			// Assets to watch:
			files: ['assets/**/*', 'tasks/pipeline.js', '!**/node_modules/**'],

			// When assets are changed:
			tasks: ['syncAssets' , 'linkAssets']
		},
		compass: {
			files: ['assets/sass/{,*/}*.scss'],
			tasks: ['compass','sync:dev']
		}
	});

	grunt.loadNpmTasks('grunt-contrib-watch');
};
```

### 更新任务流
修改__register/compileAssets.js任务流，将compass添加到编译流程当中。

```js
module.exports = function (grunt) {
	grunt.registerTask('compileAssets', [
		'clean:dev',
		'jst:dev',
		'compass',
		'copy:dev',
		'coffee:dev'
	]);
};
```

## 前端模块化开发
### 前言
现在我们的前端模块化开发有如下思路：

![前端模块化开发思路](http://7pulhb.com1.z0.glb.clouddn.com/sails-前端模块化开发.png)

即每个页面的前端逻辑受一个__模块(module)__控制，且该模块暴露一个__run()__方法，当该页面加载成功时，__模块__的__run()__方法会被执行。该模块也可引用其他一些模块，只是这些模块对页面透明，这样就做到页面和其对应逻辑的一一对应，方便前端开发者的分工协作。

### 利用requirejs进行模块化开发

#### 安装

```
bower install requirejs --save
```


#### 声明
在__views/partial/scripts.swig__下声明requirejs,注意相应路径写绝对路径：

```twig
<script data-main="{{ path.script }}/common/main" src="{{ path.bower }}/requirejs/require.js"></script>
```
其中，path是我们定义好的swig扩展。

#### 配置
在__assets/js/common__下新建__main.js__,并添加如下内容，注意，baseURL写绝对路径:

```js
// 第三方模块声明
require.config({
    baseUrl: '/bower_components/',
    paths: {
        
    }});
```


#### 优化
然而每次需要我们手动在__main.js__中维护bower下载的库并不是一种优雅的做法，我们现在需要借助[bower-requirejs](https://github.com/yeoman/bower-requirejs)这个插件来帮助我们在main.js中自动维护bower下载下的相关库。

```
npm install bower-requirejs --save-dev
```

通过bower的hook机制，我们将bower-requejs任务绑定到每次__bower install__之后，亦即每次通过bower安装了插件之后，我们的库会被自动添加到main.js中，在__.bowerrc__中添加如下内容：

```json
{
  "directory":"assets/bower_components",
  "scripts":{
    "postinstall": ".node_modules/.bin/bower-requirejs/bin/bower-requirejs -c assets/js/common/main.js -b assets/bower_components/"
  }
}
```

### 测试
现在我们安装

* [semantic-ui](http://www.semantic-ui.cn/)--将来我们用来完善页面显示的前端样式框架，
* [Backbone.js](http://backbonejs.org/)--前端MVC框架。
* 
观察bower-requirejs是否配置成功。

```
bower install semantic-ui --save
```

```
bower install backbone --save
```

> backbone的依赖项__undersocre__，bower会为我们解决

可以看到，main.js中已经自动生成了相关配置。

```js
require.config({
    baseUrl: '/bower_components/',
    paths: {
        jquery: 'jquery/dist/jquery',
        requirejs: 'requirejs/require',
        'semantic-ui': 'semantic-ui/dist/semantic',
        underscore: 'underscore/underscore',
        backbone: 'backbone/backbone'
    },
    packages: [

    ]
});
```

接下来，我们新建__assets/js/default/index.js__,随便写入点内容：

```js
// 每个模块暴露一个run()方法执行
define(function(){
    return {
        run: function() {
            $('body').css('background-color','black');
        }
    }
});
```

#### 配置页面和模块的映射关系：

___views/scripts.swig__:

```twig
<script>
    var app = {};
    //    如果指明了预调用模块，则为app设置动作
    {% if module -%}
    app.action = '{{ module }}';
    {%- endif  %}
</script>
<script data-main="{{ path.script }}/common/main" src="{{ path.bower }}/requirejs/require.js"></script>
```

我们新建__assets/js/common/app.js__来调用我们的模块方法,当app模块初始化时，会调用module的run()方法:

```js
define(['jquery','underscore','backbone','/js/'+app.action+'.js'],function($,_,Backbone,module){
    return {
        init: function(){
            module.run();
        }
    }
});
```

修改__assets/js/common/main.js__，让其加载app:

```js
require.config({
    baseUrl: '/bower_components/',
    paths: {
        jquery: 'jquery/dist/jquery',
        requirejs: 'requirejs/require',
        'semantic-ui': 'semantic-ui/dist/semantic',
        underscore: 'underscore/underscore',
        backbone: 'backbone/backbone'
    },
    packages: [

    ]
});
// 加载app，并运行
require(['/js/common/app.js'],function(app){
    app.init();
});
```

在__views/index.swig__中通过swig的__set__标签来制定该页面对应的前端控制逻辑：

```js
{% extends 'partial/layout.swig' %}
{# 声明本页面调用的模块 #}
{% set module = 'default/index' -%}

{% block title -%}Woo的博客{%- endblock %}
{% block content -%}
    <div>
        <p>
            这是Woo的博客?
        </p>
    </div>
{%- endblock %}
```

OK,接下来我们执行

```sails lift```

并访问[http://localhost:1337](http://localhost:1337),如果页面背景为黑色就代表控制逻辑成功被调用

![sails lift](http://7pulhb.com1.z0.glb.clouddn.com/sails-black.png)

## 利用semantic-ui来做界面展示

关于bootstrap和semantic-ui选择纯粹是出于个人爱好了，我更喜欢semantic-ui的语义化css控制以及其自带一些控件风格，最暖心的是，semantic-ui的模态框不会抖动有木有！！

之前我们已经在__assets/js/common/main.js__中声明了对semantic-ui相应js控制逻辑的依赖，接下来再__views/partial/stylesheets.swig__中添加semantic-ui的css调用：

```twig
{# semantic #}
<link href="{{ path.bower }}/semantic-ui/dist/semantic.min.css" rel="stylesheet" type="text/css"/>
```

##产品（Production）环境下的访问优化:
sails通过：

```
sails lift --prod
```

来将应用部署到产品环境，在其基础上，我们希望产品环境的访问速度能够得到更大的优化，因而就要考虑对我们自定义的js，css模块进行压缩，这里就会修改__uglify.js__及__cssmin.js__两个grunt压缩任务，并相应地修改__prod.js__任务流，如下所示:

__tasks/config/uglify.js__:

```js
module.exports = function(grunt) {

	grunt.config.set('uglify', {

		dist: {

			src: ['.tmp/public/concat/production.js'],
			dest: '.tmp/public/min/production.min.js'
		},
		// 压缩各个自定义模块js
		modules: {
			files:[{
				expand: true,
				cwd: '.tmp/public/js',
				src: '**/*.js',
				dest: '.tmp/public/js'
			}]
		}
	});

	grunt.loadNpmTasks('grunt-contrib-uglify');
};
```

__tasks/config/cssmin.js__:

```js
module.exports = function(grunt) {

	grunt.config.set('cssmin', {
		dist: {
			src: ['.tmp/public/concat/production.css'],
			dest: '.tmp/public/min/production.min.css'
		},
		modules: {
			files:[{
				expand: true,
				cwd: '.tmp/public/styles',
				src: '**/*.css',
				dest: '.tmp/public/styles'
			}]
		}
	});

	grunt.loadNpmTasks('grunt-contrib-cssmin');
};
```

__tasks/register/prod.js__:

```js
module.exports = function (grunt) {
	grunt.registerTask('prod', [
		'compileAssets',
		'concat',
		'uglify:dist',
		'uglify:modules', //压缩自定义模块
		'cssmin:dist',
		'cssmin:modules', // 压缩自定义css
		'sails-linker:prodJs',
		'sails-linker:prodStyles',
		'sails-linker:devTpl',
		'sails-linker:prodJsJade',
		'sails-linker:prodStylesJade',
		'sails-linker:devTplJade'
	]);
};
```

OK,现在执行

```
sails lift --prod
```

我们看到在__.tmp/public__目录下，我们之前撰写的css及js文件都被压缩:

![css_compressed](http://7pulhb.com1.z0.glb.clouddn.com/sails-css_compressed.png)

![js_compressed](http://7pulhb.com1.z0.glb.clouddn.com/sails-js_compressed.png)


-----------
## 章节预告
下一章中，我们将正式进入博客系统的开发，我们先会实现我们的账户系统。
