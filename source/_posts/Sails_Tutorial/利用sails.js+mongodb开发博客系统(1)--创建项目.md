title: 利用Sails.js+MongoDB开发博客系统(1)--创建项目
tags: sails,mongodb,nodejs
categories: 利用Sails.js+MongoDB开发博客系统
---

## Sails.js 简介
Sails是构建于[Express](http://expressjs.com/)之上的一个实时Node MVC框架，其整个风格来源于[Ruby on Rails](http://rubyonrails.org/),包括提供了类似于Rails的脚手架功能，同时又吸纳了不少现代web app工具和功能，比如grunt和websocket等。

显然，Sails的最佳应用场景会是一些实时性较强的场景，比如聊天室，游戏等，但是官方也笃定的认为sails适用于任何web app的开发。对于web之前我已经使用过了的php的symfony2和ruby的rails，但在学习了Nodejs之后，我需要一个node的框架进行项目实战，因此，我充满感性的因为那只小章鱼和官网健全的文档选择了sails，这一点都不机智。

本教程（也可以说是开发日志）将帮助各位开发一个基本的个人技术博客站点，旨在让大家熟悉sails的开发流程，并且在好好的串联一下有关js，有关前端，有关mongo的相关知识，真正要搭建一个健壮的博客系统，教程上的内容还远远不够，需要各位自己努力。

[项目demo的源码](https://github.com/yoyoyohamapi/blog)已经托管到了github，方便各位在遇到困惑的时候进行查阅。

![sails](http://www.sailsjs.org/images/bkgd_squiddy.png)
## 前驱知识
在完成博客系统搭建前，你需要去认识或者学习以下罗列出的几门知识，个人认为，每次新技能get是会带来如同网游中技能树成长的快感的。

* [__Node.js__](https://nodejs.org/):

	要注意，Node并非javascript一个框架，相比框架，Node更加底层。其实Node.js归为前驱知识也并不必要，因为大部分想要利用Sails来构建博客系统的读者在之前已经对Node.js有所接触了。
	
	教程：任何工程技术最好的教学资料一定是官方文档(理由不是官方文档写的多么精彩，而是足够时效)，除了官方文档外，我的Node启蒙还有[《Node.js实战》](http://item.jd.com/11457487.html)。
	
	

* [__Grunt（智慧野猪）__](http://gruntjs.com/):

	一个前端自动化构建工具，从现在开始，应当把以前你那混乱不堪，毫无优化的前端代码扔到历史的垃圾桶了，我们需要__Grunt__让我们的前端更加智慧。Grunt本意是"咕噜声",官方也是贴切的采用了野猪作为其Logo，猪是一种大智若愚的动物，而__Grunt__“愚“在一定学习成本，配置撰写，“智”在撰写完成后行云流水的自动化构建体验。

* [__Bower（美丽园丁鸟）__](http://bower.io/):
	
	__Bower__是一个前端插件管理器，类似于PHP的[__composer__](),__bower__能够帮助开发者脱离手动下载包，手动管理包的依赖关系等等繁重业务劳动。BTW,似乎前端这个技术的Logo（看见小鸟嘴上衔着的叶纸了吗，是不是就是那片璀璨的jquery哇）总是设计精良，名字也是诗意盎然（勤劳如园丁鸟，衔取树枝木条，造就景观大厦），相比之下，__composer__那残念的指挥家简直不能看。
	
对于__Grunt__和__Bower__的学习，除了官方资料外，[materliu](https://github.com/materliu)的[视频教程](http://www.imooc.com/learn/30)十分推荐，PPT制作精良，讲解也充满文艺气息，风格非常对我胃口。

* [__Sass__](http://sass-lang.com/) & [__Compass__](http://compass-style.org/):
	
	__Sass__能够程序化我们的css，提高了我们css代码的重用性，而__Compass__则又让Sass变得无比强大。借助于__Compass__，我们现在甚至都只用一句话就能写出一个健壮的box shadow，而不用在充满疲惫地去为该属性添加一个个内核前缀。

* [__RequireJS__](http://requirejs.org/): 其实我是国内[Seajs]()的忠实拥趸，但是考虑到更好的兼容性，前端模块化开发这次选择了__RequireJS__，如果你曾经接触过__Seajs__，那么学习__RequireJS__并不是难事。




## 环境搭建

### 基础设施
1. node环境搭建：[戳我](https://github.com/joyent/node/wiki/Installing-Node.js-via-package-manager)
2. ruby环境搭建：[戳我](https://www.ruby-lang.org/en/documentation/)
3. mongodb安装：[戳我](https://www.mongodb.org/downloads)

------------

### 编辑器 or IDE
如果是想用编辑器开发，[sublime](http://www.sublimetext.com/)依旧会是首选，git官方出品的[Atom](https://atom.io/)目前也发布了1.0稳定版，大家可以尝鲜。如果是IDE，作为jetbrain家的铁粉，会强烈推荐[webstorm](https://www.jetbrains.com/webstorm/)，各种愉悦，爽快加轻松不一一列举辣。

--------------

## 项目创建
### 安装sails

```
sudo npm -g install sails
```

### 创建项目

```
sails new blog
```

如此这般，我们就在当前目录下创建了一个sails项目（如果你知道[Yeoman](http://yeoman.io/),那也不建议用__Yeaman__来创建项目，上面sails的相关构建器（generator）已经许久没有更新了）。

## 目录结构：

![sails目录结构](http://7pulhb.com1.z0.glb.clouddn.com/sails-目录结构.png)

__blog

..__api__:存放业务逻辑，数据模型，安全策略等

..__assets__: 用过web开发框架应该对此很熟悉，就是存放图像，js，css等静态资源的目录，值得一提的是，sails项目运行起来后，静态资源会被grunt中配置的响应任务压缩并转移到.tmp文件下，即网站上对这些资源的寻址实际也是定位到.tmp下，这样做有两个好处：
	
	1. 压缩的资源能够提高网页的加载和访问效率。
	2. 保留了源文件，便于开发者调试和修改。
	
..__config__: 配置文件存放目录

..__node_modules__: node包存放目录

..__tasks__: grunt任务存放目录,我们可以将自己撰写的grunt任务放到__tasks/config__目录下，然后在__tasks/register__注册任务，就能够通过grunt执行我们的构建过程了。

..__views__: 视图存放目录

-------------

## Grunt Tasks说明

![grunt task](http://7pulhb.com1.z0.glb.clouddn.com/sails-grunt_tasks.png)

###tasks/config/下配置任务说明

* __clean.js__: 

__dev__模式下，清除__.tmp__目录下文件，亦即给项目一个干净的启动状态。__build__模式下，清除__www__目录。

* __coffee.js__: 

将__assets/js__下的js文件转化成CoffeScript并输出至__.tmp/public__目录。

* __concat.js__: 

文件合并，将已经注入到页面的各个js及各个css文件合并，并输出至__.tmp/public__目录。

* __copy.js__: 

文件复制，__dev__模式下，将__assets__下的资源复制到__.tmp/public目录__。__build__模式下，将__.tmp/public__目录下文件复制到__www__目录下。

* __cssmin.js__:

将__concat.js__合并好的css文件压缩。

* __jst.js__: 

预编译模板（模板引擎为jst）。

* __less.js__: 

因为我们用了sass，所以不再考虑用less。

* __sails-linker.js__: 

sails链接器，当你在html文件中插入

![sails_linker](http://7pulhb.com1.z0.glb.clouddn.com/sails-sails_linkers.png)

等代码包裹块后，该任务能够自动注入在__pipeline.js__中设置好的js文件，css文件的注入也类似。注入后的文件输出至__.tmp/public__目录及__views__目录。

* __sync.js__: 

同步两个目录，这里同步的是__assets__目录和.tmp/public，该任务和__copy.js__任务非常类似，不同的是，同步仅发生在文件改变时。

* __uglify.js__: 

这个大家不会陌生，用来压缩js文件的，这里的压缩对象是__concat.js__合并好的js文件。

* __watch.js__: 

文件监听，当文件改变时触发相应地任务，如在sails中，__aseests__等目录下的文件变化时，会触发[__syncAssets__ , __linkAssets__]这一任务流，该任务流中的这两个任务会在之后介绍。

### tasks/register/下注册的任务流说明

* __compileAssets.js__:

顾名思义,该任务旨在编译__assets__目录下文件，该任务流的任务执行过程为:

clean:dev --》 jst:dev --》  less:dev --》 copy:dev --》 coffee:dev
 
亦即：缓存清除 --》编译jst至__.tmp/public__ --》编译less至__.tmp/publc__ --》剩余文件(字体，图片等)拷贝至__.tmp/public__ --》js转换成coffeescript2至__.tmp/public__

* __linkAssetsXXXX.js__:

利用__sails-linker__,完成一系列的注入操作

* __syncAssets.js__:

同步__assets/__目录下文件的任务流，其过程为：

jst:dev --》 less:dev --》 sync:dev --》coffee:dev

中文释义不再赘述。

* __default.js__:

该任务流会在grunt命令后不接任何参数时执行，其任务过程为：

compileAssets --》 linkAssets --》 watch，即编译，链接，运行时监听（这个工作过程学过__操作系统__的同学不会陌生）


* __build.js__:

构建任务流，因为这只是完成构建任务，故而其任务过程相比较__default.js__,不再监听文件改动，而是在任务流末尾执行clean，copy等任务打扫下战场。

* __buildProd.js__:

只是在产品环境（production）下的构建过程，相比较于__build.js__，会执行一些文件合并及压缩工作来提高页面访问体验和效率。

--------------

##扬帆起航（Sails lift）
OK，现在我们可以启动项目了

```
sails lift
```

不带参数lift过程默认会执行的grunt任务流为__default.js__。(lift 的详细参数使用说明可以通过 ```sails lift -h```查看)
接下来在浏览器输入http://localhost:1337，你应该能够看到如下页面：
![sais lift](http://7pulhb.com1.z0.glb.clouddn.com/sails-sais_lift.png)

--------
## 章节预告

在下一章当中，我们不会立即开始博客系统开发，而是先对sails再进行一定的配置和改造，使其能够更加方便我们的代码编写。