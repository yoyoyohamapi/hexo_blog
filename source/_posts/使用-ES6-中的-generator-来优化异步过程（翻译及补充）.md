title: 使用 ES6 中的 generator 来优化异步过程
date: 2016-08-01 19:58:27
tags:
    - JavaScript
    - ES6
categories: JavaScript 学习笔记
---

> 原文：[Going Async With ES6 Generators](https://davidwalsh.name/async-generators)
> 本文在作者文章的基础上，适当补充了一些代码及说明

ES6 中的 generator 能够帮助我们提供一种类似同步一样的代码风格，来将异步过程的具体实现隐藏。这样的好处就是我们能够更加自然的表达自己的业务流程（work flow），而避免陷入异步的麻烦中。换言之，借助于 generator，我们既拥有了异步执行的能力，又不用再绞尽脑汁的去维护异步代码。

继续阅读本文，你会发现这么做的结果简直太美妙了，以前那些糟糕的异步代码现在讲会想同步代码那样变得**易于阅读**和**可维护**。需要知道的是，这个同步只是代码风格上的同步，他的执行过程仍然是异步的。

说了那么多，仍然有些抽象，现在我们由浅入深地看看到底怎么通过 ES6 来优化异步过程。

<!--more-->

一个最简单的异步
----------

假设我们的程序原来拥有这样的异步代码，这是最为朴实和原始的 JavaScript 异步流程控制：

```javascript
function makeAjaxCall(url,cb) {
    // do some ajax fun
    // call `cb(result)` when complete
}

makeAjaxCall( "http://some.url.1", function(result1){
    var data = JSON.parse( result1 );

    makeAjaxCall( "http://some.url.2/?id=" + data.id, function(result2){
        var resp = JSON.parse( result2 );
        console.log( "The value you asked for: " + resp.value );
    });
});
```

可以看到，对于一次异步请求，我们获取异步结果的过程放到了回调当中。然而，借助 generator 来完成相同的任务：

```javascript
function request() {
    // 我们将真正的异步功能掩藏在 `request` 中，这样我们在 generator 中能专注同步写法
    // 通过 `it.next(..)` 来获得异步结果，并让 generator 的流程继续
	makeAjaxCall(url, (result)=>{
		it.next(result);
	});
	// 注意，这里没有返回任何值，也就是说 `request()` 的执行结果会返回 `undefined`
}

function *main() {
	// 在generator中，我们的异步处理流程摇身一变成了同步执行过程
	const result1 = yield request('http://some.url.1');
	const data =  JSON.parse(result1);

	const result2 = yield request( "http://some.url.2?id=" + data.id );
    const resp = JSON.parse( result2 );
    console.log( "The value you asked for: " + resp.value );
}

// `main()` 方法执行后，generator 进入暂态，当 `makAjaxCall` 异步任务完成后，会让 `main()` 继续
it = main();
it.next();
```

可以看到，generator 函数 `*main(..)` 自身非常纯净，我们在其中撰写业务流程就像我们在 PHP 或者 Java 等语言撰写业务流程，看不到任何的回调。

下面解释一下以上代码片是如何工作的：

helper 函数 `request(..)` 简单的包裹了异步任务 `makeAjaxCall(..)`，一旦 `makeAjaxCall(..)` 取得了结果，就调用 generator 迭代器的 `next(..)` 方法使 generator 继续运行。

当 `*main` 运行到 `yield ..` 后，他会被暂停在 `yield` 发生的位置，直到遇到了在 `makeAjaxCall(..)` 的回调中声明的 `it.next(..)` 才会继续执行。注意到，我们把 Ajax 请求到的结果 `result` 传递给了`it.next(..)`，那么之后，`result` 就会被返回到的 `*main` 暂停了的位置，作为 `yield ..` 表达式的输出，所以，`result1` 不会是`undefined`（默认情况下，`yield` 返回 `undefined`），而是拿到的异步结果。

这就是真正牛逼的地方。语句 `result1 = yield request(..)` 所表达的意图是要去请求一个值，但是，这个请求过程却被隐藏了。利用 `yield` 实现 **暂停** 功能，然后将 **继续** 功能放到 generator 函数以外地方，更准确地说，是放到了 generator 以外的异步回调中，从而保证了我们能够在 generator 中利用 **同步** 方式撰写业务流程。

> **暂停-继续** 这样串行执行的过程模拟了 **同步** 的过程，使得这条语句在语法风格上实现了同步，但其内部实现又是异步的。

别高兴的太早，上面的代码还存在一些问题。在上面的代码中，我们总是执行一个异步 Ajax 调用，但是，如果我们之后将 Ajax 的返回结果缓存到了内存来提升性能，这意味着我们下一次请求不再需要去服务端获得数据，而可以立即从内存上获取。为了满足这个需求，我们可能就会将代码改成如下形式:

```javascript
const cache = {};
function request(url) {
	if(cache[url]) {
		setTimeout(()=>{
			it.next(cacheUrl);
		},0);
	} else {
		makeAjaxCall(url, (resp)=>{
			it.next(resp);
			cache[url] = resp;
		});
	}
}
```

注意：这里用到了 `setTimeout(..0)` 这个小技巧来强行进入异步过程，如果我们直接调用 `it.next(cacheUrl)`，就会出错，原因在于执行语句 `yield request(..)` 时，我们先执行 `request(..)`，之后 generator 函数才会暂停（后执行 `yield` ）。所以，如果我们直接调用 `it.next(cacheUrl)`，则流程如下：

```bash
next()->request()->next()
```

由于此时 generator 已经运行了，程序会抛出错误 `Generator is already running`。而通过 `setTimeout(..0)` 包裹后，我们的执行流程如下：

```bash
next()->request()->yield->next()->继续
```

整个业务才能继续执行。

现在，我们的generator是这样的:

```javascript
const result1 = yield request( "http://some.url.1" );
const data = JSON.parse( result1 );
```

牛逼吧？尽管我们新添加了缓存的逻辑，但丝毫不影响我们的 generator 函数，仍旧是在专心的写业务。在 `*main()` 中，其过程仍然是非常清晰的业务流：

```bash
请求值-->暂停（等待请求完成）-->获得值-->继续
```

> 在该场景下，暂停的持续时间变得很微妙，他可能很长（比如向服务器请求值），也可能很短（比如从内存缓存中请求值），但在我们的 `*main()` 中，还是只关注工作流（flow），无论异步过程的实现细节是否变得复杂。

更好的异步流程控制
----------

上面的代码已经满足了一些简单的异步场景。但是很快，他的功能就会显得捉襟见肘，我们需要一个更加强大的异步机制来结合我们的 generator 去满足更大的业务场景。这个机制就是 **Promises**。

> 对于 ES6 中 Promise 尚存疑惑的读者可以看下作者关于此的[博客](http://blog.getify.com/promises-part-1/)。

首先，我们反思一下之前的设计缺陷：

- 缺乏清晰的错误处理

在[作者之前撰写的文章](https://davidwalsh.name/es6-generators-dive#error-handling)中，我们能够知道一些在 Ajax 调用过程中检测错误的手段：通过 `it.throw(..)` 将错误返回的 generator 中，而在 generator 中，我们又通过 `try..catch` 来俘获错误，进行错误处理:

```javascript
function request(url) {
	makeAjaxCall(url, (err,result)=>{
		if(err) it.throw(err);
		else it.next(result);
	});
}

function *main(){
    try {
        const result1 = yield request( "http://some.url.1" );
    }
    catch (err) {
        console.log( "Error: " + err );
        return;
    }
    const data = JSON.parse( result1 );

    try {
        const result2 = yield request( "http://some.url.2?id=" + data.id );
    } catch (err) {
        console.log( "Error: " + err );
        return;
    }
    const resp = JSON.parse( result2 );
    console.log( "The value you asked for: " + resp.value );
} );
```

这样做的坏处就是，我们把错误抛出耦合到了 Ajax 流程中，设想，我们有还有其他的 generator 也用到了 `request(..)`，我们的错误控制就会变成这样：

```javascript
function request(url) {
    makeAjaxCall(url, (err,result)=>{
        if(err) {
            it1.throw(err);
            it2.throw(err);
            it3.throw(err);
            // ..
            itn.throw(err);
        }
        // ...
    });
}
```

这使得 `request(..)` 难以复用。

- 如果 `makeAjaxCall(..)` 是一个并不受我们的控制的第三方库

我们如果要在其中做诸如 `it.next(..)` 这样对 generator 的控制，就不得不修改这个库的实现，耗费人力不说，随意破坏第三方库也会使得代码难以移植。

- 并行任务。

由于 generator 中的 `yield` 是一个单步暂停点，同一时刻就只能跑一个任务。所以，我们渴望一个新的方式去执行并行任务，并且不需要太多的人工介入。

要解决上述的问题就需要我们探索新的设计模式了，结合这个新的设计模式，能让我们的基于 generator 的异步过程变得更加优雅。这个新的设计模式将会引入 **Promise**，其流程大致如下：

`yield` 一个 Promise 对象后暂停，直至这些 Promise 对象被 **履行（fulfill）** 的时候才继续我们的 generator。由于并行的 `Promise.all([..])` 也是一个 Promise 对象，所以在这种设计模式下，也能执行并行任务。

让我们对之前的 `request(..)` 函数加以修改，使之基于 Promise：

```javascript
function request(url) {
    return new Promise(function(resolve,reject){
	    // 现在，`makeAjaxCall(..)` 不再耦合 `it.next(..)`
        makeAjaxCall( url, (result)=>resolve(result));
    } );
}
```

`request(..)` 构造了一个 Promise 对象并返回，该 Promise 对象将会在 Ajax 请求完成后被 resolved。现在，generator 中的 `yield` 最终也将产出这个 Promise 对象。我们还需要一个工具函数来控制我们的 generator 的迭代器，完成我们 generator 函数的自动执行。我们暂且将这个工具函数称之为 `runGenerator(..)`：

```javascript
// `runGenerator` 函数将运行一个 generator 函数 `g` 直至其完成
function runGenerator(g) {
    const it = g(), ret;
    // 执行迭代过程的函数，首次立即执行的目的是为了启动 generator
    (function iterate(val){
        // 获得最近迭代结果, 启动时 val 是 undefined
        ret = it.next( val );

		// 如果 generator 没有执行完毕
        if (!ret.done) {
            // 是否 `ret` 仍然是一个 Promise 对象，如果是，意味着 generator 还在不断 yield
            if ("then" in ret.value) {
                // 将 `iterate(val)` 注册为该 Promise 的 `then(..)` 的回调，
                // 借此，获得一个 Promise 链
                ret.value.then( iterate );
            }
            // 如果不是 Promise 对象，而是立即数, 将该结果返回
            else {
                // avoid synchronous recursion
                setTimeout( function(){
                    iterate( ret.value ); // 相当于 `it.next(立即数)`
                }, 0 );
            }
        }
    })();
}
```

我们可以分析一下该工具函数的执行过程：
1. 我们首先初始化了传入的 generator 的迭代器 `it`，并且创建了一个迭代函数 `iterate`，该迭代函数用来**继续** generator 的流程，从而让 generator 的自动执行至完毕。
2. 每次我们执行 `iterator(val)`，就会调用 `it.next(val)`，并且获得结果 `ret`。假设我们 generator 中的执行语句是 `yield request( "http://some.url.1" )`，`request(..)` 会返回一个 Promise 对象，此时，`ret` 也就是该 Promise 对象，我们向其 `then(..)` 方法注册 `iterator`，使得该 Promise 对象完成后能够进入下一个 Promise 对象的流程，并且每次完成都会继续 generator。
3. 当 `iterator(val)` 不停流转，直至 `val` 是一个立即数时，暗示 Promise 链执行完毕，获得了结果，将其返回到 generator 使 generator 得以继续执行。

> 简言之，结合了 Promise 的 generator 异步流程就是：每次 `yield` 一个 Promise 进入暂停态，在 Promise 完成后 generator 得以继续执行。

下面我们看看怎么使用 `runGenerator`：

```javascript
runGenerator( function *main(){
    const result1 = yield request( "http://some.url.1" );
    const data = JSON.parse( result1 );

    const result2 = yield request( "http://some.url.2?id=" + data.id );
    const resp = JSON.parse( result2 );
    console.log( "The value you asked for: " + resp.value );
} );
```

简直碉堡了，有没有！我们的业务逻辑仍然没什么变化！

设想，如果我们不做 `runGenerator` 函数，就需要手动控制 generator 的流程：

```javascript
const it = main(); // 获得 generator 的迭代器
// 不断用 `then(..)` 修饰 Promise
it.next().value.then((result1)=>{
	it.next(result1).value.then((result2)=>{
		// 最终的结果返回
		it.next(result2);
	})
});
```

如果业务流非常漫长，则撰写的嵌套是非常恐怖的。

现在，我们已经使用了 Promise 来管理基于 generator 的异步流程，它将我们从充满了诸如回调陷阱（callback hell）中解放了出来。通过 generators+promise 这个设计模式，我们阐述一下如何解决上面提到的三个问题：
1. 现在，我们拥有内置的错误处理。虽然这点没有在上面的 `runGenerator(..)` 进行揭示，但是，后文会讲到，在新的设计模式下，从 Promise 中监听所有的错误并不困难。最终通过将错误绑定到 `it.throw(..)`，我们就可以放心的在 generator 中使用 `try..catch` 语句来捕获和处理错误。
2. 我们拥有了 Promise 提供的 [control/trustability](https://blog.getify.com/promises-part-2/#uninversion)。
3. Promise 已经做了大量抽象帮助我们方便的操纵多个 “并行的” 任务。

例如，`yield Promise.all([..])` 将会利用传入的并行的任务数组（数组元素都是 Promise 对象），产出单一的 Promise 对象供 generator 操纵，generator 会等待所有的子 Promise 对象完成（无论完成顺序是怎样的）才继续进行。最后，我们真正返回给 generator 流程的是所有子 Promise 的响应构成的数组，数组元素的顺序会与请求顺序一致。

generators+promise 下的错误处理
-----------

```javascript
function request(url) {
    return new Promise( (resolve,reject)=>{
        makeAjaxCall( url, (err,text)=>{
            if (err) reject( err );
            else resolve( text );
        } );
    } );
}

function runGenerator(g) {
    const it = g(), ret;
    // 现在，传入了 `err` 作为第一个参数
    (function iterate(err, val){
        // generator 迭代过程中遇到错误就 `throw`
        if(err) {
	        setTimeout(()=>{
		        it.throw(err);
	        },0);
	        return;
	    }
        ret = it.next( val );

		// 如果 generator 没有执行完毕
        if (!ret.done) {
            // 是否 `ret` 仍然是一个 Promise 对象，如果是，意味着 generator 还在不断 `yield`
            if ("then" in ret.value) {
                // 将 `iterate(val)` 注册为该 Promise 的 `then(..)` 的回调，
                // 借此，获得一个 Promise 链
                ret.value.then( iterate );
            }
            // 如果不是 Promise 对象，而是立即数，暗示 Promise 链已经获得最终结果，将该结果返回
            else {
                // avoid synchronous recursion
                setTimeout( function(){
                    iterate( ret.value ); // 相当于it.next(立即数)
                }, 0 );
            }
        }
    })();
}

runGenerator(function *main(){
    try {
        const result1 = yield request( "http://some.url.1" );
    }
    catch (err) {
        console.log( "Error: " + err );
        return;
    }
    const data = JSON.parse( result1 );

    try {
        const result2 = yield request( "http://some.url.2?id=" + data.id );
    } catch (err) {
        console.log( "Error: " + err );
        return;
    }
    const resp = JSON.parse( result2 );
    console.log( "The value you asked for: " + resp.value );
} );
```

如果一个 Promise 的 reject 发生，那么该 reject 对应到的错误会映射到 generator 中的能够捕获的一个错误，这个映射过程是通过 `runGenerator(..)` 中声明的 `it.throw(..)` 来完成的。

generators+promise 下的并行任务
-------------

```javascript
function request(url) {
    return new Promise((resolve,reject)=>{
        makeAjaxCall( url, resolve );
    } )
    // 当获得返回的 `text`，可以做一些后置处理
    .then(function(text){
        // did we just get a (redirect) URL back?
        if (/^https?:\/\/.+/.test( text )) {
            // make another sub-request to the new URL
            return request( text );
        }
        // otherwise, assume text is what we expected to get back
        else {
            return text;
        }
    });
}

runGenerator( function *main(){
    const search_terms = yield Promise.all( [
        request( "http://some.url.1" ), // 每个元素也是promise对象
        request( "http://some.url.2" ),
        request( "http://some.url.3" )
    ] );

    const search_results = yield request(
        "http://some.url.4?search=" + search_terms.join( "+" )
    );
    const resp = JSON.parse( search_results );

    console.log( "Search results: " + resp.value );
} );
```

在上面代码中，`Promise.all([ .. ])` 创建了一个 Promise 对象，该对象会等待三个子 Promise 对象完成。最终，返回的到 generator 的，恢复 generator 执行的，会是该 Promise 对象的执行结果。

### ES7 中的 `async`
尚未发布的 ES7 标准中提出了一个 `async` 函数，该函数就像我们上面撰写被 `runGenerator(..)` 所包裹的 generator。通过 `await` 关键字，你能够发出 Promise 对象，他会等待这些对象完成后才继续下去（我们甚至都不再需要借助迭代器了）。

aysnc 函数的大致使用过程如下：

```javascript
async function main() {
    const result1 = await request( "http://some.url.1" );
    const data = JSON.parse( result1 );

    const result2 = await request( "http://some.url.2?id=" + data.id );
    const resp = JSON.parse( result2 );
    console.log( "The value you asked for: " + resp.value );
}

main();
```

正如你所看到的那样，一个 `async function` 能够被直接调用，而不需要再包裹上 `runGenerator(..)`。其次，我们将用新的关键字 `await` 来替代 `yield` 告诉 `async function` 在继续前需要等待当前的 Promise 处理完成。

总结
-------

generator+promise 的设计模式集成了强大而优雅的同步式的异步流程控制的优势。通过简单的 wrapper 函数，我们能够自动地运行我们的 generator 直至完成，包括清晰明了的同步式的错误控制。

而在 ES7 以上的版本，我们还能有 `async function` 来完成同样的任务。
