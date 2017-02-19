title: JavaScript中的递归优化
tags: 函数式编程 递归 TCO Trampoline 蹦床
categories: JavaScript函数式编程
-----
### 引子
说到递归，我们先来看一个最常见的递归用例-- `求取阶乘(factorial)`
```javascript
function factorial(n) {
	if(n==1) return 1;
	return n*factorial(n-1);
}
```

测试一下：
```javascript
factorial(5); // => 120
```

似乎一切正常，5的阶乘120被正确计算出来了，我们试着把数字调大一些：
```javascript
factorial(70000);
// Uncaught RangeError:Maximum call stack size exceeded(…)
```

浏览器提示我们 `栈溢出` 了（测试环境：chrome 51），究竟发生了什么呢?

----

### 尾调用
其实，上面的 `factorial` 的实现等同于下面这个形式：
```javascript
function factorial(n) {
	if(n==1) return 1;
	var total = factorial(n-1);
	return n*total;
}
```

这个函数有个什么特点呢， 就是该函数中的调用 `factorial(n-1)` 并没有发生在函数最后。因此，为了在执行完该调用还能返回到函数中执行后续操作:

1. 赋值给局部变量 `total`：`var total = factorial(n-1)`
2. 计算 `n*total` 并返回。

那么，当前的运行环境在进入这个调用前，会现将__调用位置__以及周围的一些环境保存成一个 `调用帧（call frame）` ，并将该调用帧 `压入（push）` 一个栈空间中，这个栈被称为 `调用栈（call stack）` 。当再无调用帧入栈时，就开始逐个 `压出（pop）` 调用帧，调用取值。比如上面的 `factorial(5)` 的执行过程就是一个呈 `金字塔` 形的过程，金字塔的峰值就反应了 `factorial` 对栈空间需求的峰值：
```javascript
push factorial(5)
push 5*factorial(4)
push 5*( 4*factorial(3) )
push 5*( 4* (3*factorial(2)) )
push 5*( 4*( 3* (2*factorial(1)) ) )
pop 5*4*3*(2*1)
pop 5*4*(3*2)
pop 5*(4*6)
pop 5*24
pop 120
```

很明显，这次调用之于调用栈的空间复杂度是 `O(n)`，亦即，随着我们 `n` 取值的不断变大，栈空间不断被消耗，当 `n` 很大时，栈溢出就发生了，毕竟内存总是有限的，不允许我们这么无节制的消耗栈空间。


那么要如何优化，才能不发生栈溢出呢？我们先来看另外一个非常简单的函数：
```javascript
function a() {
	var n = 100;
	return b(100);
}
```

这个函数有个什么特点？就是其内部的函数调用 `b(100)` 发生在了函数末尾，也就是说，该函数调用之后不再有任何后续操作，通常，把这样的调用形式称之为[尾调用](https://zh.wikipedia.org/wiki/%E5%B0%BE%E8%B0%83%E7%94%A8)。尾调用因为脱离对后续操作的依赖，也就没有必要再去创建更多的调用帧消耗宝贵的栈空间了。

那么我们尝试着把之前的 `factorial` 改写成尾调用的形式，再次分析 `factorial` 的执行过程，可以发现，它能够被简化成如下形式：
```javascript
1*factorial(5)
5*factorial(4)
20*factorial(3)
60*factorial(2)
120*factorial(1)
120*1
120
```

即每次的执行过程总是，`total` 是当前可被计算出的值：
```javascript
total*factorial(n)
```

进一步，我们将 `total` 当做 `factorial` 的参数，这样就将后续过程也融入到了函数调用中，最终获得了一个尾调用形式的 `factorial` ：
```javascript
function factorial(n, total) {
	if(n==1) return total;
	return factorial(n-1, n*total);
	// 不再有后续操作
}
```

`factorial` 的执行过程就会变成：
```javascript
factorial(5,1)
factorial(4,5)
factorial(3,20)
factorial(2,60)
factorial(1,120)
120
```

已经不是金字塔形了，之于栈的空间复杂度由 `O(n)` 减少到了 `O(1)` ，仅最外层的函数需要一个调用帧，因为外层函数执行完成后需要返回到外部空间进行后续操作。

> 这种优化方式被称为 `尾调用优化（Tail Call Optimization）` ，简称 `TCO` 。

-------------

### 现实却很残酷
我们再来测试一下被优化过 `factorial` 函数：
```javascript
factorial(70000,1);
// => VM16052:1 Uncaught RangeError: Maximum call stack size exceeded(…)
```

悲剧！还是栈溢出了，这是为什么呢？原因就在于尾调用优化虽然在其他语言里面的得到了支持，__但在 `ES5` 中还没有得到支持__，换言之，`ES5` 根本不管你函数调用是否发生在函数末尾。那么还有没有其他的解决方式？难道我们永远无法在 `ES5` 中实现一个大的阶乘运算？目前来说，我们可能想到的办法会是：干脆不要递归了，重写一遍 `factorial` 的非递归的形式。

对于阶乘来说，非递归的实现复杂度也不高，但试想，如果面对一个更加复杂的业务，难道也得替换成冗长的非递归形式吗？

-----------

### Trampoline：蹦床
另一个解决该问题方式是引入 `Trampoline` ，翻译过来就是 `蹦床`，先看一下利用蹦床解决栈溢出的递归的代码：
```javascript
// 阶乘函数
function factorial(n, total) {
	if(n==1) return total;
	// 返回一个thunk，避免函数调用
	return factorial.bind(null, n-1, n*total);
}

// tramponline函数
function trampoline(fn){
	while(fn && typeof fn === 'function'){
		fn = fn();
	}
	return fn;
}

function fac(n,total) {
	return trampoline(function(){
		return factorial(n, total);
	});
}
```

通过以上代码我们可以看到，`Trampoline` 的核心在于通过__循环（loop）__来替换__递归（recursion）__过程，从而规避递归过程中占空间的申请和消耗。为了实现之一目标，大致需要如下几步：

1. 改造原来的递归，迫使他不再返回一个函数调用，而只返回一个[thunk](https://en.wikipedia.org/wiki/Thunk)，以备调用：
```javascript
// 原来的阶乘函数
function factorial(n,total) {
	// ...
	return factorial(n-1, n*total);
}

// 现在的阶乘函数
function factorial() {
	// ...
	return factorial.bind(null, n-1, n*total);
}
```

2. 一个蹦床函数，由他来通过循环控制原递归函数的执行过程：接受一个参数 `fn` ，若 `fn` 是一个 `thunk` ，执行 `fn` ，直至 `fn` 不再需要被执行，就获得了最后的结果。该过程相当于抑制了递归函数的执行，我们一次次的抽出子过程进行执行。
```javascript
function trampoline(fn){
	while(fn && typeof fn === 'function'){
		fn = fn();
	}
	return fn;
}
```

3. 一个wrapper函数，用来创建新的执行过程：
```javascript
function fac(n,total) {
	return trampoline(function(){
		return factorial(n, total);
	});
}
```

4. 现在，这个wrapper函数将是我们最终的调用对象，之后求取阶乘我们将调用 `fac` ，而不再是 `factorial` :
```javascript
fac(70000, 1);
// => Infinity
```

可以看到，现在不再出现栈溢出的情况了。另外，为了避免混淆，更好的一种方式是将 `factorial` 封装到 `fac` 内部:
```javascript
function fac(n,total) {
    function factorial(n, total) {
		if(n==1) return total;
		// 返回一个thunk，避免函数调用
		return factorial.bind(null, n-1, n*total);
	}
	return trampoline(function(){
		return factorial(n, total);
	});
}
```

--------------

### ES6中的尾递归优化
在 `ES6` 下，尾递归优化获得了支持，但是在 `Babel` 最新的 `v6` 版本中，去掉了对于尾递归函数的transform，`Babel` 官方给出了理由如下：

> Only explicit self referencing tail recursion was supported due to the complexity and performance impact of supporting tail calls globally. Removed due to other bugs and will be re-implemented.

早先版本的 `Babel` 对于尾递归的优化可以参看这篇文章：[Functional JavaScript – Tail Call Optimization and Babel](https://taylodl.wordpress.com/2015/08/09/functional-javascript-tail-call-optimization-and-babel/) 

-------------

### 参考资料
- [Tramponline in Javascript](http://raganwald.com/2013/03/28/trampolines-in-javascript.html)
- [Wiki 尾调用](https://zh.wikipedia.org/wiki/%E5%B0%BE%E8%B0%83%E7%94%A8)
- [阮一峰-尾调用优化](https://zh.wikipedia.org/wiki/%E5%B0%BE%E8%B0%83%E7%94%A8)
- [Understanding recursion in functional JavaScript programming](http://www.integralist.co.uk/posts/js-recursion.html)
- [Functional JavaScript – Tail Call Optimization and Babel](https://taylodl.wordpress.com/2015/08/09/functional-javascript-tail-call-optimization-and-babel/) 
