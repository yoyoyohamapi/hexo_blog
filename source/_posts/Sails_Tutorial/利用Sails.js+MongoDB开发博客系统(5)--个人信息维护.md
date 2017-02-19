title: 利用Sails.js+MongoDB开发博客系统(5)--个人信息维护
tags: sails,mongodb,nodejs,博客系统
categories: 利用Sails.js+MongoDB开发博客系统

---------
## 章节概述
在本章中，你将能学到如下知识:

* 利用skipper来上传和存储文件

## 设计
对于个人信息维护模块，我们暂时实现如下功能：

* 头像更改,站点的头像仅支持jpeg格式，并保存到：__/assets/images/avatar.jpg__
* 站点信息维护：包括站点名称以及站点介绍

其他诸如密码修改等功能留给读者自行完成。

## 配置路由

__config/routes.js__添加:

```js
//---------------User Profile
    '/user/profile': 'UserController.index',

    '/user/profile/avatar': 'UserController.setAvatar',
```

## 业务逻辑撰写

sails中通过[skipper](https://github.com/balderdashy/skipper)实现了[文件上传](http://www.sailsjs.org/documentation/concepts/file-uploads)，其核心函数是__upload()__，其第一个参数可接受如下配置:

* dirname：上传文件保存目录
* maxBytes：总的上传（不是单个文件）大小限制
* saveAs：如果该参数是字符串，表示单个文件的保存名，如果是函数，则对每个上传文件进行设置
* onProcess：每此上传过程中的回调逻辑

__api/controllers/UserController.js__:

```js
module.exports = {
    // 个人信息修改首页
    index: function(req,res){
        if(req.method === 'GET'){
            return res.view('user/index');
        }

    },

    // 跳转到设置头像页面
    setAvatar: function (req, res) {
        if (req.method === 'GET')
            return res.view('user/avatar');

        // POST请求进行文件上传
        // 判断是否有文件上传
        if (!req.file('avatar')._files[0]) {
            return res.badRequest('No file was uploaded');
        }

        // 判断文件类型是否争取
        var fileType = req.file('avatar')._files[0].stream.headers['content-type'];
        if (fileType != 'image/jpeg') {
            return res.badRequest('文件类型错误,仅支持JPG文件格式');
        }
        req.file('avatar').upload({
            maxBytes: 10000000,
            dirname: '../../assets/images',
            saveAs: 'avatar.jpg'
        }, function whenDone(err, uploadedFiles) {
            if (err) {
                return res.negotiate(err);
            }

            return res.redirect('/');
        });
    }
};

```
### 界面显示

![profile](http://7pulhb.com1.z0.glb.clouddn.com/sails-5_article_profile.png)

> 图像预览我选用的插件是[JavaScript-Load-Image
](https://github.com/blueimp/JavaScript-Load-Image)

---------------
## 总结
利用sails+mongodb开发个人博客系统的教程就暂告段落，其实教程中很多东西未必是最佳实践（Best Practice），很多功能也尚未开发，本教程的写作动机和目的也不在于搭建一个博客系统，因为博主认为技术博客的最佳实践仍是静态博客，本教程的出发点还是在于让各位能够了解sails搭建web app的开发流程、少部分优化手段以及借此让各位接触一下bower，grunt，semantic-ui这些前端工具。

真心希望各位能够对文章提出宝贵的意见与批评----吴小蛆.