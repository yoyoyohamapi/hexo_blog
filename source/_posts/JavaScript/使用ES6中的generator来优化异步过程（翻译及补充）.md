title: 使用ES6中的generator来优化异步过程（翻译及补充）
tags: ES6,generator,await,promise,异步优化 
categories: JavaScript
---

> 原文：[Going Async With ES6 Generators](https://davidwalsh.name/async-generators)
> 在作者文章的基础上，适当补充了一些代码及说明

ES6中的generator能够帮助我们提供一种类似同步一样的代码风格，来将异步过程的具体实现隐藏。这样的好处就是我们能够更加自然的表达自己的业务流程（work flow），而避免陷入异步的麻烦中。换言之，借助于generator,我们既拥有了异步执行的能力，又不用再绞尽脑汁的去维护异步代码。

继续阅读本文，你会发现这么做的结果简直太美妙了，以前那些糟糕的异步代码现在讲会想同步代码那样变得__易于阅读__和__可维护__。（这个同步只是代码风格上的同步，他的执行过程仍然是异步的。）

说了那么多，仍然有些抽象，现在我们由浅入深的看看到底怎么通过ES6来优化异步过程。

### 一个最简单的异步
假设我们的程序原来拥有这样的异步代码，这是最为朴实和原始的js异步流程控制:
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

可以看到，对于一次异步请求，我们获取异步结果的过程放到了回调当中。然而，借助generator来完成相同的任务：
```javascript
function request() {
    // 我们将真正的异步功能掩藏在`request`中，这样我们在generator中能专注同步写法
    // 通过`it.next(..)` 来获得异步结果，并让generator的流程继续
	makeAjaxCall(url, (result)=>{
		it.next(result);
	});
	// 注意，这里没有返回任何值，也就是说request的执行结果会返回undefined
}

function *main() {
	// 在generator中，我们的异步处理流程摇身一变成了同步执行过程
	const result1 = yield request('http://some.url.1');
	const data =  JSON.parse(result1);

	const result2 = yield request( "http://some.url.2?id=" + data.id );
    const resp = JSON.parse( result2 );
    console.log( "The value you asked for: " + resp.value );
}

// `main`方法执行后，generator进入暂态，当makAjaxCall异步任务完成后，会让main继续
it = main();
it.next();
```

可以看到，generator函数`*main(..)`自身非常纯净清晰，我们在其中撰写业务流程就像我们在php或者java等语言撰写业务流程，看不到任何的回调。

下面解释一下以上代码片是如何工作的：

帮助函数`request(..)`简单的包裹了异步任务`makeAjaxCall(..)`，一旦`makeAjaxCall(..)`取得了结果，就调用generator迭代器的`next(..)`方法使generator继续运行。

要注意的是，`request(..)`并没有显式的返回任何值，所以最终该函数的执行结果会返回`undefined`。那么，难道`yield request(..)`返回的结果是`undefined`吗（要知道，默认情况下的`yield`就是返回`undefined`）？

当`*main`运行到`yield ..`后，他会被暂停在`yield`发生的位置，直到遇到了在`makeAjaxCall(..)`的回调中声明的`it.next(..)`才会继续执行。注意到，我们把Ajax请求到的结果`result`传递给了`it.next(..)`，那么之后，`result`就会被返回到的`*main`暂停了的位置，作为`yield ..`表达式的输出，所以，`result1`不会是`undefined`，而是拿到的异步结果。

这就是真正伟大和牛逼的地方。语句`result1 = yield request(..)`所表达的意图是要去请求一个值，但是，这个请求过程却被隐藏了。利用`yield`实现的__暂停（pause）__功能，并将__继续（resume）__功能放到generator函数以外地方，更精准的说，是放到了generator以外的异步回调中，从而保证了我们能够在generator中利用__同步__方式撰写业务流程。

> __暂停-继续__这样串行执行的过程模拟了__同步__的过程，使得这条语句在语法风格上实现了同步，但其内部实现又是异步的。

别高兴的太早，上面的代码还存在一些问题。在上面的代码中，我们总是执行一个异步Ajax调用，但是，如果我们之后将Ajax的返回结果缓存到了内存来提升性能，这意味着我们下一次请求不再需要去服务端获得数据，而可以立即从内存上获取。为了满足这个需求，我们可能就会将代码改成如下形式:
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

注意：这里用到了`setTimeout(..0)`这个小技巧来强行进入“异步过程”，如果我们直接调用`it.next(cacheUrl)`，就会出错，原因在于执行语句`yield request(..)`时，我们先执行`request(..)`,之后generator函数才会暂停（后执行`yield`）。所以，如果我们直接调用`it.next(cacheUrl)`：则流程如下：
```bash
next()->request()->next()
```

由于此时generator已经运行了，程序会抛出错误`Generator is already running`。而通过`setTimeout(..0)`包裹后，我们的执行流程如下：
```bash
next()->request()->yield->next()->继续
```

整个业务才能继续执行。

现在，我们的generator仍然是这样的:
```javascript
const result1 = yield request( "http://some.url.1" );
const data = JSON.parse( result1 );
```

牛逼吧？尽管我们新添加了缓存的逻辑，但丝毫不影响我们的generator函数，仍旧是在专心的写业务。在`*main()`中，其过程仍然是非常清晰的业务流：
```bash
请求值-->暂停（等待请求完成）-->获得值-->继续
```

> 在该场景下，暂停的持续时间变得很微妙，他可能很长（比如向服务器请求值），也可能很短（比如从内存缓存中请求值），但在我们的`*main()`中，还是只关注工作流（flow），无论异步过程的实现细节是否变得复杂。

### 更好的异步流程控制
上面的代码已经满足了一些简单的异步场景。但是很快，他的功能就会显得捉襟见肘，我们需要一个更加强大的异步机制来结合我们的generator去满足更大的业务场景。这个机制就是__Promises__。

> 对于ES6中Promise尚存疑惑的读者可以看下作者关于此的[博客](http://blog.getify.com/promises-part-1/)。

首先，我们反思一下之前的设计缺陷：

1. 缺乏清晰的错误处理。在[之前作者撰写的文章](https://davidwalsh.name/es6-generators-dive#error-handling)中，我们能够知道一些在Ajax调用过程中检测错误的手段：通过`it.throw(..)`将错误返回的generator中，而在generator中，我们又通过`try..catch`来俘获错误，进行错误处理:
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
    这样做的坏处就是，我们把错误抛出耦合到了Ajax流程中，设想，我们有还有其他的generator也用到了`request(..)`，我们的错误控制就会变成这样:
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
    这样，`request(..)`变得难以复用。
2. 如果`makeAjaxCall(..)`是一个并不受我们的控制的第三方库，换言之，我们如果要在其中做诸如`it.next(..)`这样对generator的控制，就不得不修改这个库的实现，耗费人力不说，随意破坏第三方库也会使得代码难以移植。
3. 总有一些时候我们需要__并行的（in paralle）__做一些任务（例如同时发送两条Ajax请求）。由于generator中的`yield`是一个单步暂停点，同一时刻就只能跑一个任务。所以，我们仍然渴望一个新的方式去实行并行任务，而不需要太多的人工介入。

要解决上述的问题就需要我们探索新的设计模式了，结合这个新的设计模式，能让我们的基于generator的异步过程变得更加优雅。这个新的设计模式将会引入__Promise__，其流程大致如下：
> `yield`一个promise对象后暂停，直至这些promise对象被__履行（fulfill）__的时候才继续我们的generator。由于并行的`Promise.all([..])`也是一个promise对象，所以在这种设计模式下，也能执行并行任务。

让我们对之前的`request(..)`函数加以修改，使之基于promise：
```javascript
function request(url) {
    return new Promise(function(resolve,reject){
	    // 现在，`makeAjaxCall(..)`不再耦合`it.next(..)`
        makeAjaxCall( url, (result)=>resolve(result));
    } );
}
```

`request(..)`构造了一个promise对象并返回，该promise将会在Ajax请求完成后被resolved。现在，generator中的`yield`最终也将产出这个promise对象。我们还需要一个工具函数来控制我们的generator的迭代器，完成我们generator函数的自动执行。我们暂且将这个工具函数称之为`runGenerator(..)`：
```javascript
// `runGenerator`函数将运行一个generator函数`g`直至其完成
function runGenerator(g) {
    const it = g(), ret;
    // 执行迭代过程的函数，首次立即执行的目的是为了启动generator
    (function iterate(val){
        // 获得最近迭代结果, 启动时val是undefined
        ret = it.next( val );

		// 如果generator没有执行完毕
        if (!ret.done) {
            // 是否`ret`仍然是一个promise对象，如果是，意味着generator还在不断yield
            if ("then" in ret.value) {
                // 将`iterate(val)`注册为该promise的`then(..)`的回调，
                // 借此，获得一个promise链
                ret.value.then( iterate );
            }
            // 如果不是promise对象，而是立即数, 将该结果返回
            else {
                // avoid synchronous recursion
                setTimeout( function(){
                    iterate( ret.value ); // 相当于it.next(立即数)
                }, 0 );
            }
        }
    })();
}
```

我们可以分析一下该工具函数的执行过程：
1. 我们首先初始化了传入的generator的迭代器`it`，并且创建了一个迭代函数`iterate`，该迭代函数用来__继续__generator的流程，从而实现generator的自动执行至完毕。
2. 每次我们执行`iterator(val)`，就会调用`it.next(val)`，并且获得结果`ret`。假设我们generator中的执行语句是`yield request( "http://some.url.1" )`，`request(..)`会返回一个promise对象，此时，`ret`也就是该promise对象，我们向其`then(..)`方法注册`iterator`，使得该promise对象完成后进入下一个promise对象的流程，并且每次完成都会继续generator。
3. 当`iterator(val)`不停流转，直至`val`是一个立即数时，暗示promise链执行完毕，获得了结果，将其返回到generator使generator得以继续执行。

> 简言之，结合了promise的generator异步流程就是：每次`yield`一个promise进入暂停态，在promise完成后generator得以继续执行。

下面我们看看怎么使用`runGenerator`：
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

设想，如果我们不做`runGenerator`函数，就需要手动控制generator的流程：
```javascript
const it = main(); // 获得generator的迭代器
// 不断用`then(..)`修饰promise
it.next().value.then((result1)=>{
	it.next(result1).value.then((result2)=>{
		// 最终的结果返回
		it.next(result2);
	})
});
```
如果业务流非常漫长，则撰写的嵌套是非常恐怖的。

现在，我们已经使用了promise来管理基于generator的异步流程，它将我们从充满了诸如回调陷阱（callback hell）的回调书写模式中解放了出来。通过generators+promise这个设计模式，我们阐述一下如何解决上面提到的三个问题：
1. 现在，我们拥有内置的错误处理。虽然这点没有在上面的`runGenerator(..)`进行揭示，但是，后文会讲到，在新的设计模式下，从promise中监听所有的错误并不困难。最终通过将错误绑定到`it.throw(..)`，我们就可以放心的在generator中使用`try..catch`语句来捕获和处理错误。
2. 我们拥有了promise提供的[control/trustability](https://blog.getify.com/promises-part-2/#uninversion)。
3. promise已经做了大量抽象帮助我们方便的操纵多个“并行的”任务。

例如，`yield Promise.all([..])`将会利用传入的并行的任务数组（数组元素都是promise对象），产出单一的promise对象供generator操纵，generator会等待所有的子promise对象完成（无论完成顺序是怎样的）才继续进行。最后，我们真正返回给generator流程的是所有子promise的响应构成的数组，数组元素的顺序会与请求顺序一致。

先让我们看看在generators+promise下的__错误处理__是怎么做的:
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
    // 现在，传入了err作为第一个参数
    (function iterate(err, val){
        // generator迭代过程中遇到错误就throw
        if(err) {
	        setTimeout(()=>{
		        it.throw(err);
	        },0);
	        return;
	    }
        ret = it.next( val );

		// 如果generator没有执行完毕
        if (!ret.done) {
            // 是否`ret`仍然是一个promise对象，如果是，意味着generator还在不断yield
            if ("then" in ret.value) {
                // 将`iterate(val)`注册为该promise的`then(..)`的回调，
                // 借此，获得一个promise链
                ret.value.then( iterate );
            }
            // 如果不是promise对象，而是立即数，暗示promise链已经获得最终结果，将该结果返回
            else {
                // avoid synchronous recursion
                setTimeout( function(){
                    iterate( ret.value ); // 相当于it.next(立即数)
                }, 0 );
            }
        }
    })();
}

runGenerator( function *main(){
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

如果一个promise的reject发生，那么该reject对应到的错误会映射到generator中的能够捕获的一个错误（这个映射过程是通过`runGenerator(..)`中声明的`it.throw(..)`来完成的）。

再让我们看看新的设计模式下的并行任务处理:
```javascript
function request(url) {
    return new Promise((resolve,reject)=>{
        makeAjaxCall( url, resolve );
    } )
    // 当获得返回的text，可以做一些后置处理
    .then( function(text){
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


在上面代码中，`Promise.all([ .. ])`创建了一个promise对象，该对象会等待三个子promise对象完成。最终，返回的到generator的，恢复generator执行的，会是该promise对象的执行结果。

### ES7中的`async`
尚未发布的ES7标准中提出了一个`async`函数，该函数就像我们上面撰写被`runGenerator(..)`所包裹的generator。通过该函数，你能够发出promise对象，他会等待这些对象完成后才继续下去（我们甚至都不再需要借助迭代器了）。

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

正如你所看到的那样，一个`async function`能够被直接调用，而不需要再包裹上`runGenerator(..)`。其次，我们将用新的关键字`await`来替代`yield`告诉`async function`在继续前需要等待当前的promise处理完成。

### 总结
简言之，generator+promise的设计模式集成了强大而优雅的同步式的异步流程控制的优势。通过简单的wrapper函数，我们能够自动地运行我们的generator直至完成，包括清晰明了的同步式的错误控制。

而在ES7以上的版本，我们还能有`async function`来完成同样的任务，而不再需要借助wrapper函数来驱动generator的执行。