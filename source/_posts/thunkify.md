title: thunkify
date: 2016-8-02 11:03:08
tags:

- JavaScript

categories: JavaScript 学习笔记

---



引子
-------

自己之前曾经撸过一个验证库，代码大致如下：
```javascript
function validate(data, rules, cb) {
	// ....
	// 一切完成后会触发回调函数
	cb(null, errMap);
}
```

出于性能上的考虑，该库被封装为了一个异步函数，需要提供一个回调函数 `cb` 来获得验证结果，该回调函数是一个满足 node 规范的 **error-first callback**。在其他系统中，该库使用良好，但是，新项目使用了 koa，我在中间件中使用这个库提供的 `validate` 方法， 却连编译期都没有通过：

<!--more-->

```javascript
const validator = require('../libs/validator');
module.exports = function () {
    return function *(next) {
        // 获得请求体
        const body = this.request.body;
        // 获得规则
        const rules = getRules(this.request.url);
        if (rules) {
	        validator.validate(body, rules, (error, errMap) => {
				const errors = Object.keys(errMap).reduce((errors, key) => {
	                const value = errMap连[key];
	                if (value.error) {
	                    errors.push(`参数 [${key}] 的规则 [\'${value.msg}\'] 不通过`);
	                }
	                return errors;
	            }, []);
	            if (errors.length) {
	                logger.error(`参数校验失败:${errors.toString()}`);
	                const error = new Error(`参数校验失败:${errors.toString()}`);
	                error.status = 400;
	                this.throw(error);
	            } else {
		            // ！注意，我在回调函数中使用了yield
	                yield next;
	            }
			});
        } else {
            yield next;
        }
    };
};
```

查阅了[ ECMAScript Wiki ](http://tc39wiki.calculist.org/es6/arrow-functions/)知道，`yield` 关键字并不能在箭头函数中使用。一再探索解决渠道的时候，我有两个考虑，一方面，`yield` 是使用 koa 中间件时绕不开的。另一方面，我也不想去改造库，这样必然会影响到其他系统。

thunk
----------

解决该问题的方法就是将我们原有库的 API 包装成一个 thunk，该 thunk 大致形态如下：
```javascript
function thunk() {
	return function(cb) {
		// ...
		cb(err, data);
	}
}
```

`thunk` 返回的函数接收一个回调函数 `cb(error, data)`。同样地，该函数是一个 **error-first callback**。现在，在 koa 中间件中，我们就能这样使用 `thunk`：

```javascript
// ...
try {
    // data源于回调的第二个参数`data`
	const data = yield thunk();
} catch(error) {
	// error handling: 其中错误来源于回调的第一个参数`error`
}
// ...
```

所以，我们可以这样包装我们的 `validate` 方法，使 `validate` 方法 thunk 化：

```javascript
const thunkedValidate = function(data, rules) {
 	return function(cb) {
		return validator.validate(data, rules, cb);
	}
}
```

现在，我们能在中间件顺畅的使用 API 了，并且代码更加直观：

```javascript
// ...
// 获得请求体
const body = this.request.body;
// 获得规则
const rules = getRules(this.request.url);
if (rules) {
	const errMap = yield thunkedValidate(body, rules);
}
// ...
```

封装
------------

上面我们改造 `validate` 的做法并不通用，如果我们还想改造其他 API，逐个封装显然容易让人疲惫不堪。借助于高阶函数，我们能够封装一个通用的方法用来 thunk 化一个函数，我们将其命名为 `thunkify`，他的大致结构如下：

```javascript
/**
 * 接受一个函数, 返回一个thunk
 * @param func 原函数
 * @return Function thunk
 */
function thunkify(func) {
	// 返回一个thunk
    return function(cb) {
		// ...
	}
}
```

我们再分析下，将一个函数改造为返回一个 thunk 的过程：
1. 获得原函数的参数，比如上例中的 `data`、`rules`
2. 新建一个高阶函数，该函数以这些参数作为参数，并且返回一个 thunk，在 thunk 中，才是我们运行 API 的过程：

```javascript
function(data, rules) {
	return thunk(done){
		// 通常，回调函数是最后一个传入的函数
		validator.validate(rules, data, ()=> {})
	}
}
```

为了更加通用，在 `thunkify` 中，我们创建的高阶函数不再显式声明需要的参数，而是借助于 `arguments` 来捕获参数：

```javascript
function thunkify(func) {
    return function() {
	    // 获得参数，借助闭包保存
		var args = Array.prototype.slice.call(arguments, 0);

		// 返回thunk
		return function(cb) {
			// 追加`cb`到args中
			args.push(function(err, data){
				cb.call(this, err, data);
			});
			// 调用 api， 完成逻辑
			func.apply(this, args);
		}
	}
}
```

测试一下，用例我参考的[ node-thunkify ](https://github.com/tj/node-thunkify)：

```javascript
function load(name, fn) {
	fn(null, name);
}

thunked = thunkify(load);

thunked('wxj')(function(err, name){
	console.log(name);
}); // 'wxj'
```

改进
----------

### 执行上下文

再看下面的一个测试 ：
```javascript
function load(fn) {
  fn(null, this.name);
}

var user = {
  name: 'wxj',
  load: thunkify(load)
};

user.load()(function(err, name){
  console.log(name);
}); // undefined
```

没有按照预期的输出`'wxj'`，因为我们的 `thunkify` 忘记考虑绑定执行上下文了，在原始的 API 执行时：

```
func();
```

其对应的 `this` 应该是 `func()` 执行时所处的上下文。所以，我们优化 `thunkify`，让 thunk化 的 API 能获得正确的执行上下文：

```javascript
function thunkify(func) {
    return function() {
	    // 获得参数，借助闭包保存
		var args = Array.prototype.slice.call(arguments, 0);
		// 缓存上下文
		var ctx = this;
		// 返回thunk
		return function(cb) {
			// 追加 `cb` 到args中
			args.push(function(err, data){
				cb.call(this, err, data);
			});
			// 调用 API，使用缓存了的上下文
			func.apply(ctx, args);
		}
	}
}
```

再跑一下上面的用例， 成功输出了 `'wxj'`。

### 错误处理
----------

假定，我们 API 是会抛出错误的，那么在 API 的执行过程中捕获到错误时，我们就应当向 `cb` 中注入该错误：

```javascript
function thunkify(func) {
    return function() {
	    // 获得参数，借助闭包保存
		var args = Array.prototype.slice.call(arguments, 0);
		// 缓存上下文
		var ctx = this;
		// 返回 thunk
		return function(cb) {
			// 追加 `cb` 到 args 中
			args.push(function(err, data){
				cb.call(this, err, data);
			});
			// 调用 API，完成逻辑
			try{
				func.apply(ctx, args);
			} catch(e) {
				cb(e);
			}
		}
	}
}
```

测试一下：

```javascript
function load(fn) {
  throw new Error('boom');
}

load = thunkify(load);

load()(function(err) {
	console.log(err);
});
```

总结
----------

我们封装了一个大致可用的 thunkify 函数，也再次体会了 JavaScript 多范式编程的魅力和高阶函数的强大。不过，在实际生产环境中，还是推荐使用更健壮的[ node-thunkify ](https://github.com/tj/node-thunkify)。
