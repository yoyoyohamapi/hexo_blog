title: 一步步写一个 co

date: 2017-02-03 19:29:34

tags:

-	co
-	JavaScript
-	ES6

categories: 一步步写一个 co

---



> 文章已发布至 [GitBook](https://www.gitbook.com/book/yoyoyohamapi/-co/details)

现在，我们有三个 markdown 文件 file1.md,file2.md,file3.md，我们想要统计这三个文件的大小信息，并输出为以下格式：

```js
{file1: 5384, file2: 2712, file3: 13942}
```

<!--more-->

从最传统的回调开始，我们不断优化解决该问题的方式，最后完成了一个简化版的 [co V2](https://github.com/tj/co/tree/2.0.0)，实现了用更优雅地方式来组织和编写异步流程。

> 本文对应的项目地址，包含了文中所有涉及到的代码片: [戳我](https://github.com/yoyoyohamapi/write-a-co)
