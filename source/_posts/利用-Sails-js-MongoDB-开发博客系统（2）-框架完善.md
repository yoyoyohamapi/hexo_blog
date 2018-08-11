title: 利用 Sails.js + MongoDB 开发博客系统（2）-- 框架完善
date: 2015-05-02 09:50:44
tags:
- Sails
- Node
- MongoDB

categories: 利用 Sails.js + MongoDB 开发博客系统

---



章节概述
---------

在本章中，你将能学到如下知识：

- 如何将 Sails 默认模板引擎替换为 swig，并且设置和扩展 swig。
- 集成 bower 来管理我们的前端库。
- 集成 compass 来更优雅的撰写 css。
- 通过 grunt 来监听 scss 文件变动，并自动编译。
- 前端模块化开发思路即实现。
- 集成 semantic-ui 来撰写 UI。
- 利用 grunt 来对产品环境下的访问进行优化。

<!--more-->

更换模板引擎
------------

Sails 默认的模板引擎为 ejs，仅就我个人而言，对这个框架不是很喜欢。尤其 ejs 不支持继承，令我大为恼火。所以我会考虑用 [**jinja**](http://jinja.pocoo.org/) 风格的 swig 来做模板引擎。

### 安装 swig

```
npm install swig --save
```

标准的 swig 提供的 filter、tag 等可能不够用，比如字符串分割（split）等就未提供，为此，我们也需要安装其扩展：

```
npm install swig-extras --save
```

### 替换默认引擎

在 config/views.js 中，修改 egine 为 swig，并对 swig 做一些方便我们开发的配置，比如我们设定一些常用资源路径，避免了在页面中每次都要书写冗长的路径前缀。同时，为了在开发环境下修改 swig 而不用重启服务器，我们需要设置 swig 默认不缓存:

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

在 views 下创建一些 swig 页面,其中 partial 文件夹下为一些公用视图部分或者视图模板：

<div style="text-align:center">
<img src="http://7pulhb.com1.z0.glb.clouddn.com/sails-views_dir.png" width="300"></img>
</div>

并初始化一些内容

**layout.swig**:

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

**index.swig**:

```twig
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

### 设置路由

为我们新添加的 index.twig 添加路由

```js
'/' : {
    view :'index'
}
```

OK，现在我们执行

```
sails lift
```

访问 [http://localhost:1337](http://localhost:1337)，就能看到如下页面：

<div style="text-align:center">
<img src="http://7pulhb.com1.z0.glb.clouddn.com/sails-index_show.png" width="300"></img>
</div>

利用 bower 来管理我们的前端库
----------

### 安装 bower

```
npm install bower -g
```

### 配置 bower

在项目根目录下创建 .bowerrc 文件,并在文件中设置 bower 库目录

```json
{
  "directory":"assets/bower_components",
}
```

之所以需要将 bower 下载下的组件放置在 assets 目录下，是方便我们在不用撰写任何新的 grunt 任务的情况下就能在页面中引用 bower 下载的库资源。

### 使用bower

我们尝试利用 bower 来下载 jquery

```
bower install jquery --save
```

可以看到，jquery 已经成功地安装在了在 assets/bower_components 目录下

<div style="text-align:center">
<img src="http://7pulhb.com1.z0.glb.clouddn.com/sails-bower_jquery.png" width="300"></img>
</div>

利用 compass 来撰写 css
-------------

### 安装compass

类似地，我们选择将 compass 放置在 assets 目录下：

```
gem install compass
cd assets
compass init
```

### 配置compass

修改 assets/config.rb 的文件内容如下：

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

### 设置 compass 任务

接下来我们需要创建 grunt task 来设置 compass 的编译任务，使得每次 assets/sass 文件夹下的 scss 文件变动时，不用手动输入命令就能完成 scss 到 css 的编译工作:

### 安装 grunt-contrib-compass

```
npm install grunt-contrib-compass
```

### 创建 compass 任务

在 tasks/config 下新建 compass.js 并添加如下内容：

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

### 监视 scss 文件变动

修改 tasks/config 下的 watch.js 任务，使得当 scss 文件变动时，compass 能够自动编译 scss 至 css。

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

修改 register/compileAssets.js 任务流，将 compass 添加到编译流程当中：

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

前端模块化开发
-------

### 前言

现在我们的前端模块化开发有如下思路：

<div style="text-align:center">
<img src="http://7pulhb.com1.z0.glb.clouddn.com/sails-%E5%89%8D%E7%AB%AF%E6%A8%A1%E5%9D%97%E5%8C%96%E5%BC%80%E5%8F%91.png" width="500"></img>
</div>

即每个页面的前端逻辑受一个**模块（module）**控制，且该模块暴露一个 `run()` 方法，当该页面加载成功时，`run()` 方法会被执行。该模块也可引用其他一些模块，只是这些模块对页面透明，这样就做到页面和其对应逻辑的一一对应，方便前端开发者的分工协作。

### 利用 Requirejs 进行模块化开发

#### 安装

```
bower install requirejs --save
```

#### 声明

在 views/partial/scripts.swig 下声明 Requirejs：

```twig
<script data-main="{{ path.script }}/common/main" src="{{ path.bower }}/requirejs/require.js"></script>
```

其中，path 是我们定义好的 swig 扩展。

#### 配置

在 assets/js/common 下新建 main.js ，并添加如下内容，注意，`baseURL` 写绝对路径:

```js
// 第三方模块声明
require.config({
    baseUrl: '/bower_components/',
    paths: {

    }});
```

#### 优化

然而每次需要我们手动在 main.js 中维护 bower 下载的库并不是一种优雅的做法，我们现在需要借助[ bower-requirejs ](https://github.com/yeoman/bower-requirejs)这个插件来帮助我们在 main.js 中自动维护 bower 下载的相关库。

```
npm install bower-requirejs --save-dev
```

通过 bower 的 hook 机制，我们将 bower-requejs 任务绑定到每次 `bower install` 之后，亦即每次通过 bower 安装了插件之后，我们的库会被自动添加到 main.js 中。在 .bowerrc 中添加如下内容：

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

* [semantic-ui](http://www.semantic-ui.cn/)： 将来我们用来完善页面显示的前端样式框架，
* [Backbone.js](http://backbonejs.org/)：前端 MVC 框架。

```
bower install semantic-ui --save
bower install backbone --save
```

> backbone 的依赖项 undersocre，bower 会为我们解决

可以看到，main.js 中已经自动生成了相关配置。

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

接下来，我们新建 assets/js/default/index.js，随便写入点内容：

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

views/scripts.swig:

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

我们新建 assets/js/common/app.js 来调用我们的模块方法，当 app 模块初始化时，会调用 module 的 `run()` 方法:

```js
define(['jquery','underscore','backbone','/js/'+app.action+'.js'],function($,_,Backbone,module){
    return {
        init: function(){
            module.run();
        }
    }
});
```

修改 assets/js/common/main.js，让其加载 app:

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

在 views/index.swig 中通过 swig 的 `set` 标签来制定该页面对应的前端控制逻辑：

```twig
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

```
sails lift
```

访问[ http://localhost:1337 ](http://localhost:1337)，如果页面背景为黑色就代表控制逻辑成功被调用

<div style="text-align:center">
<img src="http://7pulhb.com1.z0.glb.clouddn.com/sails-black.png" width="500"></img>
</div>

利用 semantic-ui 来做界面展示
--------------

关于 bootstrap 和 semantic-ui 选择纯粹是出于个人爱好了，我更喜欢 semantic-ui 的语义化 css 控制以及控件风格，最暖心的是，semantic-ui 的模态框不会抖动有木有！！

之前我们已经在 assets/js/common/main.js 中声明了对 semantic-ui 相应 JavaScript 控制逻辑的依赖，接下来在 views/partial/stylesheets.swig 中添加 semantic-ui 的 css 调用：

```twig
{# semantic #}
<link href="{{ path.bower }}/semantic-ui/dist/semantic.min.css" rel="stylesheet" type="text/css"/>
```

产品（Production）环境下的访问优化
--------------

sails通过：

```
sails lift --prod
```

来将应用部署到产品环境。我们希望产品环境的访问速度能够得到更大的优化，因而就要考虑对我们自定义的 js、css 模块进行压缩，这里就会修改 uglify.js 及 cssmin.js 两个 grunt 压缩任务，并相应地修改 prod.js 任务流：

- tasks/config/uglify.js：

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

- tasks/config/cssmin.js：

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

- tasks/register/prod.js

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

现在执行

```
sails lift --prod
```

在 .tmp/public 目录下，我们可以看到我们之前撰写的 css 及 js 文件都被压缩。

章节预告
------------

下一章中，我们将正式进入博客系统的开发，我们先会实现我们的账户系统。
