title: 利用 Sails.js + MongoDB 开发博客系统（1）-- 创建项目
date: 2015-04-30 08:42:25
tags:
- Sails
- Node
- MongoDB
categories: 利用 Sails.js + MongoDB 开发博客系统
---

> 框架教程具有时效性，尽可能以官方文档为主

Sails.js 简介
-----------

Sails 是构建于 [Express](http://expressjs.com/) 之上的一个实时 Node MVC 框架，其整个风格来源于 [Ruby on Rails](http://rubyonrails.org/)。Sails 提供了类似于 Rails 的脚手架功能，同时又吸纳了不少现代 Web app 工具和功能，比如 Grunt 和 websocket 等。

显然，Sails 的最佳应用场景会是一些实时性较强的场景，比如聊天室，游戏等，但是官方也笃定的认为 Sails 适用于任何 web app 开发。对于 Web 项目开发，之前我已经使用过了的 PHP 的 Symfony2 和 Ruby 的 Rails，但在学习了 Node 之后，我需要一个 Node 的框架进行项目实战，因此，我因为那只小章鱼和官网健全的文档选择了 Sails，这一点都不机智。

本教程（也可以说是开发日志）将帮助各位开发一个基本的个人技术博客站点，旨在让大家熟悉 Sails 的开发流程，并且再好好地串联一下有关 JavaScript，也有关前端，有关 MongoDB 的相关知识，真正要搭建一个健壮的博客系统，教程上的内容还远远不够，需要各位自己努力。

[项目源码](https://github.com/yoyoyohamapi/blog)已经托管到了 Github，大家可以 Clone 测试。

![sails](http://www.sailsjs.org/images/bkgd_squiddy.png)

<!--more-->

前驱知识
--------------

在完成博客系统搭建前，你需要去认识或者学习以下罗列出的几门知识，个人认为，每次新技能 get 是会带来如同网游中技能树成长的快感的。

- [**Node.js**](https://nodejs.org/)

要注意，Node 并非 JavaScript 一个框架，相比框架，Node 更加底层。其实 Node.js 归为前驱知识也并不必要，因为大部分想要利用 Sails 来构建博客系统的读者在之前已经对 Node.js 有所接触了。

任何工程技术最好的教学资料一定是官方文档(理由不是官方文档写的多么精彩，而是足够时效)，除了官方文档外，我的 Node 启蒙还有[《Node.js实战》](http://item.jd.com/11457487.html)。

- [**Grunt（智慧野猪）**](http://Gruntjs.com/)

一个前端自动化构建工具，从现在开始，应当把以前你那混乱不堪，毫无优化的前端代码扔到历史的垃圾桶了，我们需要 Grunt 让我们的前端更加智慧。Grunt 本意是"咕噜声",官方也是贴切的采用了野猪作为其 Logo，猪是一种大智若愚的动物，而 Grunt “愚” 在一定学习成本，配置撰写，“智” 在撰写完成后行云流水的自动化构建体验。

- [**Bower（美丽园丁鸟）**](http://bower.io/)

Bower 是一个前端插件管理器，类似于 PHP 的[ composer ]()。Bower 能够帮助开发者脱离手动下载包，手动管理包的依赖关系等等繁重业务劳动。Bower 的 Logo 设计精良，看见小鸟嘴上衔着的叶纸了吗，是不是就是那片璀璨的 Jquery 哇，名字也是诗意盎然，勤劳如园丁鸟，衔取树枝木条，造就景观大厦，相比之下，composer 那残念的指挥家简直不能看。

对于 Grunt 和 Bower 的学习，除了官方资料外，[ materliu ](https://github.com/materliu)的[视频教程](http://www.imooc.com/learn/30)十分推荐，PPT 制作精良，讲解也充满文艺气息，风格非常对我胃口。

- [**Sass**](http://sass-lang.com/) & [**Compass**](http://compass-style.org/)

Sass 能够程序化我们的CSS，提高了我们CSS代码的重用性，而 Compass 则又让 Sass 变得无比强大。借助于 Compass，我们现在甚至都只用一句话就能写出一个健壮的 `box-shadow`，而不用在疲惫地去为该属性添加一个个内核前缀。

- [**RequireJS**](http://requirejs.org/)

其实我是国内[ Seajs ](https://github.com/seajs/seajs)的忠实拥趸，但是考虑到更好的兼容性，前端模块化开发这次选择了 RequireJS，如果你曾经接触过 Seajs，那么学习 RequireJS 并不是难事。

环境搭建
------------

### 基础设施

1. Node 环境搭建：[戳我](https://github.com/joyent/node/wiki/Installing-Node.js-via-package-manager)
2. Ruby 环境搭建：[戳我](https://www.ruby-lang.org/en/documentation/)
3. MongoDB 安装：[戳我](https://www.mongodb.org/downloads)


### 文本编辑器 or IDE
如果是想用文本编辑器开发，[ sublime ](http://www.sublimetext.com/)是不错的选择，git 官方出品的[ Atom ](https://atom.io/)目前也发布了稳定版，大家可以尝鲜。如果是 IDE，作为 jetbrain 家的铁粉，会强烈推荐[ Webstorm ](https://www.jetbrains.com/webstorm/)，各种愉悦，爽快加轻松不一一列举辣。

## 初始化项目

### 安装sails

```
sudo npm -g install sails
```

### 创建项目

```
sails new blog
```

如此这般，我们就在当前目录下创建了一个 Sails 项目（如果你知道[ Yeoman ](http://yeoman.io/),那也不建议用 Yeaman 来创建项目，但是上面 Sails 的构建器（generator）已经许久没有更新了）。

目录结构
----------

<div style="text-align:center">
<img src="http://7pulhb.com1.z0.glb.clouddn.com/sails-%E7%9B%AE%E5%BD%95%E7%BB%93%E6%9E%84.png" width="300"></img>
</div>

..**api**：存放业务逻辑，数据模型，安全策略等

..**assets**： 用过web开发框架应该对此很熟悉，就是存放图像，js，CSS 等静态资源的目录。值得一提的是，Sails 项目运行起来后，静态资源会被 Grunt 中配置的响应任务压缩并转移到 .tmp 文件下，即网站上对这些资源的寻址实际也是定位到 .tmp 下，这样做有两个好处：

	1. 压缩的资源能够提高网页的加载和访问效率。
	2. 保留了源文件，便于开发者调试和修改。

..**config**：配置文件存放目录

..**node_modules**：node 包存放目录

..**tasks**： Grunt任务存放目录,我们可以将自己撰写的 Grunt 任务放到 tasks/config 目录下，然后在 tasks/register 注册任务，就能够通过 Grunt 执行我们的构建过程了。

..**views**： 视图存放目录

Grunt Tasks 说明
------------

### tasks/config/：任务配置

- **clean.js**：

dev 模式下，清除 .tmp 目录下文件，亦即给项目一个干净的启动状态。build 模式下，清除 www 目录。

-  **coffee.js**:

将 assets/js 下的 JavaScript 文件转化成 CoffeScript 并输出至 .tmp/public 目录。

- **concat.js**:

文件合并，将已经注入到页面的各个 JavaScript 文件 和 CSS 文件合并，并输出至 .tmp/public 目录。

- **copy.js**:

文件复制， dev 模式下，将**assets 下的资源复制到 .tmp/public目录 。 build 模式下，将 .tmp/public 目录下文件复制到 www 目录下。

- **CSSmin.js**:

将 concat.js 合并好的 CSS 文件压缩。

- **jst.js**:

预编译模板，模板引擎为 jst。

- **less.js**:

因为我们用了 sass，所以不再考虑用 less。

- **sails-linker.js**:

Sails链接器，当你在 html 文件中插入

```html
<!--SCRIPTS-->
<!--SCRIPTS END-->
```

等代码包裹块后，该任务能够自动注入在 pipeline.js 中设置好的js文件，CSS 文件的注入也类似。注入后的文件输出至 .tmp/public 目录及 views 目录。

- **sync.js**:

同步两个目录，这里同步的是 assets 目录和 .tmp/public 目录，该任务和 copy.js 任务非常类似，不同的是，同步仅发生在文件改变时。

- **uglify.js**:

这个大家不会陌生，用来压缩 JavaScript 文件的，这里的压缩对象是 concat.js 合并好的文件。

- **watch.js**:

文件监听，当文件改变时触发相应地任务，如在 Sails 中，aseests 等目录下的文件变化时，会触发 [syncAssets, linkAssets] 这一任务流，该任务流中的这两个任务会在之后介绍。

### tasks/register/：任务注册

- **compileAssets.js**:

顾名思义,该任务旨在编译 assets 目录下文件，该任务流的任务执行过程为:

```
clean:dev --> jst:dev --> less:dev --> copy:dev --> coffee:dev
```

亦即：

```
缓存清除 --> 编译 jst 至 .tmp/public --> 编译 less 至 .tmp/publc --> 剩余文件（字体，图片等）拷贝至 .tmp/public --> JavaScript 转换成 CoffeeScript 至 .tmp/public

```

* **linkAssetsXXXX.js**:

利用 sails-linker，完成一系列的注入操作

* **syncAssets.js**:

同步 assets 目录下文件的任务流，其过程为：

```
jst:dev --> less:dev --> sync:dev --> coffee:dev
```

* **default.js**:

该任务流会在 Grunt 命令后不接任何参数时执行，其任务过程为：

```
compileAssets --> linkAssets --> watch
```

* **build.js**:

构建任务流，因为这只是完成构建任务，故而其任务过程相比较 default.js，不再监听文件改动，而是在任务流末尾执行 clean、copy 等任务打扫下战场。

* **buildProd.js**:

只是在产品环境（production）下的构建过程，相比较于 build.js，会执行一些文件合并及压缩工作来提高页面访问体验和效率。

--------------

扬帆起航
------------

OK，现在我们可以启动项目了

```
sails lift
```

不带参数 lift 过程默认会执行的 Grunt 任务流的 default.js。

接下来访问 [http://localhost:1337](http://localhost:1337)，你应该能够看到如下页面：

<div style="text-align:center">
<img src="http://7pulhb.com1.z0.glb.clouddn.com/sails-sais_lift.png" width="800"></img>
</div>

章节预告
---------------

在下一章当中，我们不会立即开始博客系统开发，而是先对 Sails 再进行一定的配置和改造，使其能够更加方便我们的代码编写。
