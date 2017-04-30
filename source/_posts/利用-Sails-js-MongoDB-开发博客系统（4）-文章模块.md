title: 利用 Sails.js + MongoDB 开发博客系统（4）-- 文章模块
date: 2015-05-04 11:10:44
tags:
- Sails
- Node
- MongoDB
categories: 利用 Sails.js + MongoDB 开发博客系统
---

章节概述
----------

在本章中，你将能学习到如下知识：

- 认识 MongoDB 的 `mapReduce` 函数，并利用其实现标签统计功能。
- 认识 ES6 中的 Promise，了解其实现库 blue bird，为我们解决 Node 中难看的嵌套回调。
- 如何利用 markdown 来创建文章并完成代码高亮显示。

<!--more-->

设计
---------

我们将设计如下两个 document：

* 文章（article）
* 分类（category）

而评论将使用[多说](http://duoshuo.com/)。

### article document

| 名称           | 说明        | 类型         | 限制           |
| ------------- | ----------- |-------------| -------------|
| title         | 文章标题     | string      |必选，长度1-20字符|
| content       | 文章内容     | string      |必选，不少于1个字符|
| author        | 作者         | User        |必选            |
| tags          | 标签         | array       |               |
| category      | 分类         | Category    |默认：未分类     |

### category document

| 名称           | 说明        | 类型         | 限制           |
| ------------- | ----------- |-------------| -------------|
| name          | 名称         | string      |必选,长度1-20字符|


### 额外需求
1. 用户具有默认分类：__未分类__。
2. 每创建一篇文章，需要完成标签统计。

初始化
-------

### 创建 api

```
sails generate api article
sails generate api category
```

### 创建 “未分类”

我们需要在 user 模型中的 `afterCreate` 这个生命期中存储一个默认分类：

api/models/Category.js：

```js
/**
 * Category.js
 *
 * @description :: 分类
 * @docs        :: http://sailsjs.org/#!documentation/models
 */
DEFAULT_NAME = "未分类";
module.exports = {

    attributes: {
        name: {
            type: "string",
            required: true,
            unique:true
        }
    },

    // 获得默认分类名
    getDefault: function(){
        return DEFAULT_NAME;
    }
};
```


api/models/User.js:

```js
    // ...
	// 创建（注册）用户前，对用户密码加密
    beforeCreate: function (values, cb) {
        bcrypt.genSalt(10, function (err, salt) {
            bcrypt.hash(values.password, salt, function (err, hash) {
                if (err) return cb(err);
                values.password = hash;
                // 执行用户定义回调
                cb();
            });
        });
    },

    // 创建用户后，自动为之生成默认分类-"未分类"，并更新站点信息
    afterCreate: function (createdUser, cb) {
        var thisModal = this;
        Category.create({name:Category.getDefault(),creator:createdUser})
            .exec(function(err,category){
                if(category){
                    thisModal.updateSite(createdUser);
                    cb();
                }
            });
    },

    // ..
```

标签统计功能
----------

### MapReduce

如何统计所有文章中标签的出现频度，很容易想到的办法是遍历所有文章来统计各个标签的出现的次数，但这样做是十分低效的，为了完成这个需求，我们将利用 MongoDB 中的 `mapReduce()` 函数，相应知识可以参考官方文档：[Map Reduce](http://docs.mongodb.org/manual/core/map-reduce/)

这里以一个例子简要的介绍一下 MongoDB 的 `mapReduce()`。考虑一个学生（Student）集合（collection），当中有如下三个学生文档（document）：张三，李四，王五

张三：

```json
{
	name: "张三",
	class: 1
}
```

李四：

```json
{
	name: "李四",
	class: 1
}
```

王五：

```json
{
	name: "王五",
	class: 2
}
```

现在我们利用 Map Reduce 来统计各个班级的学生人数，首先我们要进行一个 map 过程，该过程就是__根据一定条件将划分数据。

```js
var map = function(){
	emit(this.class,{count:1});
}
```

通过 `emit(参数1,参数2)` 方法，我们就能获得输出键值对（key-value）。其中，`参数1`可以看做我们数据的划分依据，例如，本例中我们是根据班级进行划分，而 `参数2` 则是要传入这个划分的值，本例中，每个班级下，我们需要学生人数统计 `count:1`。

本例中，map过程将会为我们产生两个文档，分别是：

__班级1__:

```json
{1,[{count:1},{count:1}]}
```

__班级2__:

```json
{2,{count:1}}
```

下面再通过 reduce 过程对 map 产生的键值对进行处理，输出每个班的人数：

```js
var reduce = function(key,values){
	var res = {count:0}
	values.forEach(function(value){
		res.count += value.count;
	});
	return res;
}
```

>!重要：reduce 函数中接收的 `values` 参数的形式，必须和 reduce 函数返回的结果 `res` 的形式一致。

本例中，reduce 过程将产生两个文档：

```json
{
	_id: 1
	value: {count:2}
}
```

```json
	_id: 2
	value: {count:1}
```

最后，通过 MongoDb 的 `mapReduce()` 函数将统计信息输出到 `statistics` 集合中：

```js
db.student.mapReduce(map,reduce,{out:"statistics"});
```

现在我们执行

```
db.statistics.find();
```

将能看到如下结果：

<div style="text-align:center">
<img src="http://7pulhb.com1.z0.glb.clouddn.com/sails-mapreduce_result.png" width="300"></img>
</div>

### 标签统计实现

借助sails中[ `native()` ](http://sailsjs.org/documentation/reference/waterline-orm/models/native)方法，我们可以封装 `mapreduce()` 函数到我们的业务逻辑中：

api/models/Article.js：

```js
/**
 * Article.js
 *
 * @description :: 文章模型
 * @docs        :: http://sailsjs.org/#!documentation/models
 */

/**
 * 定义Article集合的map，reduce函数,统计标签
 */
var map = function () {
    // 分类依据组成为："用户ID:标签"
    this.tags.forEach(function (tag) {
        emit(tag, 1);
    });
};
var reduce = function (k, values) {
    var total = 0;
    for (var i = 0; i < values.length; i++) {
        total += values[i];
    }
    return total;
};

module.exports = {

    attributes: {
        title: {
            type: 'string',
            required: true,
            minLength: 1,
            maxLength: 40
        },
        content: {
            type: 'string',
            required: true,
            minLength: 1
        },
        tags: {
            type: 'array'
        },
        category: {
            model: 'category',
            required: true
        }
    },

    // 每次文章创建完成，更新标签统计
    afterCreate: function (article, cb) {
        this.updateTags();
        cb();
    },
    //custom
    updateTags: function () {
        Article.native(function (err, collection) {
            if (err) return res.serverError(err);
            collection.mapReduce(map, reduce, {out: "tags"});
        });
    }
};

```

### 创建 tags api

```
sails generate api tags
```

添加路由及访问控制
---------

### 路由

___config/routes.js__:

```js
	//----------------Aticles
    // 默认显示全部文章
    '/': 'ArticleController.index',
    // 显示某篇文章
    'get /article/show/:id' : 'ArticleController.show',
    // 跳转到创建文章页
    'get /article/new': 'ArticleController.new',
    // 跳转到编辑文章页
    'get /article/edit/:id': 'ArticleController.edit',

    // 显示分类下的全部文章
    '/category/:id/articles/:page': 'CategoryController.getArticles',

    // 显示标签下的全部文章
    '/tag/:name/articles/:page': 'TagsController.getArticles',
```

### Policies

config/polies.js:

```js
// 文章显示逻辑不需要登录
    ArticleController: {
        index: 'userCreated',
        show: 'userCreated'
    },

    CategoryController: {
        getArticles: true
    },

    TagsController: {
        getArticles: true
    }
```


页面组织
---------

我是采用如下的页面组织方式:

<div style="text-align:center">
<img src="http://7pulhb.com1.z0.glb.clouddn.com/sails-article_views.png" width="300"></img>
</div>

其中编辑页面和创建页面因为逻辑相似，二者共用 save.swig，其中含有一个编辑器 _editor.swig 的子页面，建议所有内嵌的子页面都以下划线开头，以示区分。

业务逻辑撰写
---------

api/controllers/ArticleController.js：

```js
module.exports = {

    // 跳至创建文章
    new: function (req, res) {
        //获取分类
        Category.find()
            .exec(function (err, categories) {
                if (!err) {
                    return res.view(
                        'article/save',
                        {
                            categories: categories,
                            form: {action: '/article', method: 'POST'}
                        }
                    )
                }
            });
    },

    // 跳至修改文章:
    edit: function (req, res) {
        //获取文章
        var id = req.param('id');
        Article.findOne(id).exec(function (err, article) {
            // 如果不存在，404
            if (article) {
                //获取分类
                Category.find()
                    .exec(function (err, categories) {
                        if (!err) {
                            return res.view(
                                'article/save',
                                {
                                    article: article,
                                    categories: categories,
                                    form: {action: '/article/' + id, method: 'PUT'}
                                }
                            )
                        }
                    });
            } else {
                return res.notFound();
            }
        });

    }

};
```

### Promise 解决难看的嵌套回调

OK，可以看到，我们已经遇到了 Node 常见的多层嵌套回调了，不断横向延伸的代码感觉就像闷了口大翔在嘴里，非常不舒服，在 ES6 中，可以通过[ Promise ](https://promisesaplus.com/)来解决嵌套的回调，下面简要介绍一下 Promise。

顾名思义，Promise 代表一种“许诺”，也就是未来才会发生的东西，如同现实生活中的许诺一样，它会被“履行（fulfiled）” 或者 “拒绝履行（rejected）”，Promise 的核心方法为 `then(onFulfiled,onRejected)`，通过 `then`，我们就能构造一个不断向下的过程，而不是横向延伸。

看个栗子：

一个 Ajax 请求的嵌套回调:

```js
// 首先我要得到一篇文章
$.get('/article',function(err,article){
	if(err)
		return;
	else
	// 在回调中得到该文章所有的评论
		$.get(article.commentsURL,function(err,comments){
			//doSomething
		});
});
```

经 Promise 处理后的代码:

```js
$.get('/article')
	.then(function(article){
		return $.get(article.commentsURL);
	})
	.then(function(comments){
		// doSomething
	})
	.catch(function(err){
		// error handler
	})
```

显然，这种向下的逐级传递使得回调逻辑更加易读以及易维护。想更深入了解 Promise，可以参看[这篇文章](http://spion.github.io/posts/why-i-am-switching-to-promises.html).

Sails 所采用的 Waterline 也提倡我们进行 Promise 式的书写，其所采用的 Promise 实现库是[ blue bird ](https://github.com/petkaantonov/bluebird)。现在我们通过 blue bird 对编辑文章的业务逻辑进行重构：

```js
    // 跳至修改文章:
    edit: function (req, res) {
        //获取文章
        var id = req.param('id');
        Article.findOne(id).populate('category')
            .then(function (article) {
                return [article,Category.find()];
            })
            .spread(function (article, categories) {
                res.view(
                    'article/save',
                    {
                        article: article,
                        categories: categories,
                        form: {action: '/article/' + id, method: 'PUT'}
                    }
                );
            })
            .catch(function (err) {
                res.notFound();
            });
    }
```
下面我们访问[ localhost:1337/article/new ](http://localhost:1337/article/new)，进入如下页面:

<div style="text-align:center">
<img src="http://7pulhb.com1.z0.glb.clouddn.com/sails-%E5%88%9B%E5%BB%BA%E6%96%87%E7%AB%A0.png" width="800"></img>
</div>

编辑
---------

### markdown 支持

个人不喜欢用富文本编辑器，技术博客最好的写作工具还是 markdown，下面我们通过 bower 来为前端添加 markdown 解析，这里我用的[ marked ](https://github.com/chjj/marked)。

```
bower install marked --save
```

### 代码高亮支持

对于代码高亮，选择老牌的[ highlight.js ](https://highlightjs.org/)，它提供了不少很骚的主题。

```
bower install highlightjs --save
```

记得将喜欢的高亮主题 css 添加到页面，否则看不到加亮效果

views/article/layout.swig:

```twig
{% extends '../partial/layout.swig' %}
{% block stylesheets -%}
    <link rel="stylesheet" type="text/css" href="{{ path.style }}/article.css"/>
    <link href="{{ path.bower }}/highlightjs/styles/solarized_dark.css" type="text/css" rel="stylesheet"/>
{%- endblock %}
```

### 预览

现在，我们可以试试效果了，在表单创建相应内容，然后单击预览看看效果：

<div style="text-align:center">
<img src="http://7pulhb.com1.z0.glb.clouddn.com/sails-4_article_edit.png" width="800"></img>
</div>

<div style="text-align:center">
<img src="http://7pulhb.com1.z0.glb.clouddn.com/sails-4_article_preview.png" width="800"></img>
</div>

> 在创建/编辑文章前端逻辑中，新用到的 semantic-ui 的组件有[ modal ](http://www.semantic-ui.cn/modules/modal.html)

### 重构标签
因为表单发送的标签（tags）是字符串，而实际上我们的标签在数据库中的组织形式是数组，所以我们需要在 **每次后端验证 article 的表单前** 对发送过来的标签进行处理，将其转换为字符串形式，并保证每个标签的有效性和唯一性：

api/models/Article.js 中为 `beforeValidate` 生命期添加逻辑:

```js
//文章验证前，重构标签
    beforeValidate: function(article,cb){
        if(article.tags.length) {
            var rowTags = article.tags[0].split(" ");
            article.tags = [];
            // 去除空标签及重复标签
            for(var i=0;i<rowTags.length;i++){
                var tag = rowTags[i].replace(" ","");
                if(tag.length>0 && (article.tags.indexOf(tag)<0)){
                    article.tags.push(tag);
                }
            }
        }
        cb();
    },
```

显示文章
--------

### 分页

考虑到访问性能，我们要进行分页，分页采用 “下一页” 方式，操作方式为 “点击加载”。

Waterline 通过 `paginate()` 函数实现分页：

```js
Model.find().paginate({page: 2, limit: 10});
```

### 业务逻辑

在 api/controllers/ArticleController.js 添加文章显示的业务逻辑:

```js
// 文章查询顺序：以更新时间逆序
FIND_ORDER = 'updatedAt desc';
// 文章每页条目数
FIND_PER_PAGE = 2;

// 文章查询顺序：以更新时间逆序
FIND_ORDER = 'updatedAt desc';
// 文章每页条目数
FIND_PER_PAGE = 2;

module.exports = {

    /**
     * 首页显示文章
     * @param req
     * @param res
     */
    index:function(req,res){
        // 获得当前需要加载第几页
        var page = req.param('page') ? req.param('page') : 1;
        // 如果是第1页，则需要加载分类以及标签
        if( page==1 ) {
            Article.find({
                sort: FIND_ORDER,
            }).paginate({page: page, limit: FIND_PER_PAGE})
                .populate('category').then(function (articles) {
                    // 每篇文章转换
                    // 查找分类,及标签
                    return [
                        articles,
                        Category.find(),
                        Tags.find()
                    ];
                })
                .spread(function (articles, categories, tags) {
                    return res.view(
                        'article/index',
                        {
                            articles: articles,
                            categories: categories,
                            tags: tags,
                            page: page
                        });
                });
        }else{
            Article.find({
                    sort: FIND_ORDER,
                }).paginate({page: page, limit: FIND_PER_PAGE})
                .populate('category').exec(function (err, articles) {
                    if (!err) {
                        // 刷新下一页
                        return res.view(
                            'article/_article',
                            {
                                articles: articles,
                                page: page
                            });
                    }
                    else {
                        console.log(err);
                    }
                });
        }

    },

    // 显示某篇文章
    show:function(req,res) {
        var id = req.param('id');
        Article.findOne(id).populate('category').
            exec(function (err, article) {
            if(err){
                res.notFound();
            }else{
                res.view('article/show',{
                    articles: [article]
                });
            }
        });
    },

    //..............
 };

```

### markdown 显示优化

我自己小改了一下 git 的 markdown theme，主要是去掉了自带的代码高亮，来使得 markdown 解析出的内容效果更加舒适，相应文件在[这儿](https://github.com/yoyoyohamapi/blog/blob/master/assets/sass/default/markdown.scss)。

在 views/article/layout.swig 引入 css：

```twig
{% extends '../partial/layout.swig' %}
{% block stylesheets -%}
    <link rel="stylesheet" type="text/css" href="{{ path.style }}/article.css"/>
    <link href="{{ path.bower }}/highlightjs/styles/solarized_dark.css" type="text/css" rel="stylesheet"/>
    <link href="{{ path.style }}/markdown.css" type="text/css" rel="stylesheet"/>
{%- endblock %}
```

OK，访问[ localhost:1337 ](http://localhost:1337)看一下效果：

<div style="text-align:center">
<img src="http://7pulhb.com1.z0.glb.clouddn.com/sails-4_article_index.png" width="800"></img>
</div>

章节预告
------

下一章节中，我们将实现个人信息维护功能。
