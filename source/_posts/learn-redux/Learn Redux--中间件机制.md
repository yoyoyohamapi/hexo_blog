title: Learn Redux---中间件机制（Middlewares）
tags: redux 中间件
categories: Learn Redux
----

### 引子
我们知道在redux中，dispatch的作用在于派发一个action， 改action会被reducer收到，reducer根据action的类型进行相应的状态（state）维护。大致过程可由下图反映：

![dispatch流程](https://www.lucidchart.com/publicSegments/view/104e5690-383f-4e9d-be8c-de24471ac904/image.png)

原生的dispatch方法被设计得非常轻量，我们不妨查看redux中的__createStore.js__源码(我已经添加了注释)：

```javascript
function dispatch(action) {

   // 需要action是一个纯对象
   if (!isPlainObject(action)) {
     throw new Error(
       'Actions must be plain objects. ' +
       'Use custom middleware for async actions.'
     )
   }

   // action需要声明类型
   if (typeof action.type === 'undefined') {
     throw new Error(
       'Actions may not have an undefined "type" property. ' +
       'Have you misspelled a constant?'
     )
   }

   // 检查是否正在dispatch中
   if (isDispatching) {
     throw new Error('Reducers may not dispatch actions.')
   }

   // 开始dispatch, 倘若dispatch遇到阻碍(exception), 允许再次dispatch
   try {
     isDispatching = true
     // 通过reducer来刷新状态
     currentState = currentReducer(currentState, action)
   } finally {
     // 修复dispatch状态,以便再次发送
     isDispatching = false
   }

   // 获取最新一次(next)的监听列表, 逐个响应该dispatch
   var listeners = currentListeners = nextListeners
   for (var i = 0; i < listeners.length; i++) {
     listeners[i]()
   }

   return action
}
```

我们发现，在原生的dispatch实现中，仅能接受和返回一个扁平的action对象，亦即如下形式的action对象：

```javascript
{
	type: CLICK
	text: 'submit'
}
```

## 中间件
在真实场景中，比如开发环境下，我们要在dispatch过程中输出一些日志（比如dispatch前后的状态变动），可以有如下方式（参考文章：http://redux.js.org/docs/advanced/Middleware.html）：

1. 直接在业务代码中添加日志打印：

```javascript
let action = addTodo('Use Redux')

console.log('dispatching', action)
store.dispatch(action)
console.log('next state', store.getState())
```

目的是达到了，但是并不智慧（代码侵入性太强），我们每需要一个日志打印，就得手动在业务代码中添加，这会使我们业务代码变得肮脏（到处是重复的console.log）而难以阅读。

2. 我们重新构造一个经过包裹的dispatch

```javascript
function dispatchAndLog(store, action) {
  console.log('dispatching', action)
  store.dispatch(action)
  console.log('next state', store.getState())
}
```

之后，在有日志需求的地方，我们用该dispatch方法替换原有的dispatch方法：

```javascript
dispatchAndLog(store, addTodo('Use Redux'))
```

看起来这样做不错，代码也清爽简洁，但设想这样一种情况，团队的其他人交给我们了一份老旧的代码片，当中全是原生的dispatch方法，为了日志需求，我们不得不定位到每一个dispatch进行替换，这口锅你愿意不愿意背？当然还得背，否则KPI就要捉急，为了背着不是那么累，还得往下思考。

3. Monkey Patching

在前一种解决方式中，壁垒就是我们得__重复__替换每一个dispatch调用，然而，dispatch作为store的一个方法，保存的是一个方法引用，所以我们可以让store的dispatch属性重新指向包裹后的引用：

```javascript
// 先暂存老的业务逻辑
let next = store.dispatch
function dispatchAndLog(action) {
	// 为老的业务逻辑打上补丁patching
	console.log('dispatching', action)
	let result = next(action)
	console.log('next state', store.getState())
	return result
}
// 新的逻辑替换掉老的逻辑
store.dispatch = dispatchAndLog
```

> Monkey Patching是一个很常见的技术手段，先在老的业务逻辑上打补丁patching（所以要暂时保存老的业务）形成新的逻辑，再用新的逻辑替换掉老的逻辑，而几乎不影响代码变动。

好像天亮了，在你充满希望的等着上级褒奖时，新的需求来了：在一些场景下，为了保证性能（如惰性求值），我们不需要action立即成为一个扁平对象，而应当是一个[Thunk](http://www.ruanyifeng.com/blog/2015/05/thunk.html)，为此，我们包裹一个新的dispatch:

```javascript
// 此时，store.dispatch已经拥有了日志功能
let next = store.dispatch // 快照老的dispatch
function dispatchThunk(action) {
	if(typeof action === 'function')
		return action(store)
	return next(action) // 调用老的dispatch
}
store.dispatch = dispatchThunk // 刷新dispatch
```

此时，我们验证Monkey Patching的解决策略在新的需求到来时依然能够从容应对，通过不断的打补丁，我们使得原有的dispatch方法不断强大，这个强大体现在两个方面：

1. 在dispatch的执行过程中注入了新的逻辑，例如日志打印
2. 支持更广泛的action类型，不仅是plain object，还是thunk，亦或是promise都行

但是，我们也发现了，代码仍然有重复，主要是如下几点：

1. 缓存老（相对的）的dispatch逻辑

```javascript
let next = store.dispatch
```

2. 替换老（相对的）的逻辑

```javascript
store.dispatch = dispatchThunk
```

对于第一个问题， 我们可以通过参数传递屏蔽掉显式地缓存老逻辑：

```javascript
function dispatchAndLog(store, next) {
	return function(action){
		console.log('dispatching', action)
		let result = next(action)
		console.log('next state', store.getState())
		return result
	}
}
```

现在，我们新的业务代码会变成如下形式：

```javascript
store.dispatch = dispatchAndLog(store, store.dispatch)
store.disptach = dispatchThunk(store, store.dispatch)
```

如果熟悉函数式编程，我们还可以通过[Curry化](https://zh.wikipedia.org/wiki/%E6%9F%AF%E9%87%8C%E5%8C%96)优化上述代码片:

```javascript
function logger(store) {
	return function(next) {
		return function(action) {
			console.log('dispatching', action)
			let result = next(action)
			console.log('next state', store.getState())
			return result
		}
	}
}
```

现在， 业务代码变成了：

```javascript
let dispatch = store.dispatch
dispatch = logger(store)(dispatch)
dispatch = thunk(store)(dispatch)
store.dispatch = dispatch
```

我们发现，对于dispatch的构造，是一个__链式（chain）__的修饰过程，每一个对于现有dispatch的改造都可以视为一个中间件（middleware）， 我们现在封装一个函数表示上述过程：

```javascript
/**
 * 为store设置中间件
 * @param store
 * @paran middlewares {Array} 中间件列表
 */
function applyMiddleware(store, middlewares) {
	// 保证middlewares是一个数组
	middlewares = middlewares.slice()
	// 我们声明的中间件序列最靠外的应当被最后响应
	// 所以最靠外的中间件应当最接近原生的dispatch
	middlewares.reverse()

	// 初始化dispatch
	let dispatch = store.dispatch
	// 迭代装饰dispatch
	middlewares.forEach(middleware =>
	  dispatch = middleware(store)(dispatch)
	)

	return Object.assign({}, store, { dispatch })
}
```

OK， 业务代码现在变成如下形式：

```javascript
store = applyMiddleware(store, [thunk, logger])
```

这已经是一个非常干练的解决方式了，也非常接近于Redux自身提供的applyMiddleware的函数实现，但是该函数存在如下两个缺陷：

1. 将整个store暴露给中间件是过剩的，因为这些中间件仅有（1）改造dispatch方法和（2）读取当前状态的需求， 所以仅需要暴露给他们__dispatch(action)__ 及 __getState()__ 就够了，这形成了Redux对中间件的约定。

2. 该方法存在这样一个隐患：可以无休止的应用中间件，如果开发者使用不当，将会产生对dispatch的重复包装：

```javascript
store = applyMiddleware(store, [thunk, logger])
// ...
store = applyMiddleware(store, [thunk, logger])
// 此时，日志会被嵌套且重复地打印
store.dispatch(action)
```

所以，更合理的一种做法是，把应用中间件到store的过程放到第一次返回store之前， 亦即，在用户拿到store对象前，store对象的dispatch方法已经被中间件序列包装完毕。

官方实现的applyMiddleware解决了这些问题：

```javascript
export default function applyMiddleware(...middlewares) {
 // 重构了createStore方法， 保证用户在拿到store对象前，
 // store对象的dispatch已经被中间件序列包装完毕
  return (createStore) => (reducer, initialState, enhancer) => {
    var store = createStore(reducer, initialState, enhancer)
    // 原始的dispatch
    var dispatch = store.dispatch
    var chain = []

	// 暴露有限的API给中间件
    var middlewareAPI = {
      getState: store.getState,
      // 通过闭包， 每个中间件仅调用最新的dispatch
      dispatch: (action) => dispatch(action)
    }
    // 利用curry化，初始化中间件链
    chain = middlewares.map(middleware => middleware(middlewareAPI))
    // 组合各个中间件至最终的dispatch形态 ，最内层的中间件(最后一个执行的中间件)需要最原始的dispatch
    dispatch = compose(...chain)(store.dispatch)

    // 返回一个通过中间件重构了dispatch的store对象
    return {
      ...store,
      dispatch
    }
  }
}
```

注意到， 该方法实际上是重构了createStore(), 在官方的createStore实现中，我们观察到如下源码：

```javascript
// createStore.js
// 如果传递了enhancer, 就必须保证enhancer是一个函数
if (typeof enhancer !== 'undefined') {
   if (typeof enhancer !== 'function') {
     throw new Error('Expected the enhancer to be a function.')
   }
   // 如果enhancer合法,那么创建store的行为就内联到enhancer中完成
   return enhancer(createStore)(reducer, initialState)
}
```

我们得知，一旦遇到enhancer（在这里是applyMiddleware），那么创建store的行为就内联到enhancer中完成，这样就解决了之前提到的问题2。

此外，在applyMiddleware中有如下一段代码片很重要：

```javascript
dispatch = compose(...chain)(store.dispatch)
```

这段代码片中利用了函数式编程中函数组合（Composing）的概念，从中间件链中不断抽出函数进行组合（一个reduce过程）， 其中起始函数被传递了初始参数store.dispatch， 为什么要进行函数组合呢呢，我们先回顾之前的代码片：

```javascript
let dispatch = store.dispatch
dispatch = logger(store)(dispatch)
dispatch = thunk(store)(dispatch)
store.dispatch = dispatch
```

该构造过程可以简化为：

```javascript
store.dispatch = thunk(store)(logger(store)(dispatch))
```

如果了解函数式编程（学习Redux必须了解），这样的形式就暗示了我们可以进行函数组合（composing）来组合各个中间件构造最终的dispatch， 官方的compose实现如下：

```javascript
// compose.js
/**
 * 组合函数
 * @param funcs 待组合函数列表
 */
export default function compose(...funcs) {
  if (funcs.length === 0) {
    return arg => arg
  } else {
    // 获得起始函数
    const last = funcs[funcs.length - 1]
    // 获得其余函数
    const rest = funcs.slice(0, -1)
    // 通过reduce, 不断组合函数
    return (...args) => rest.reduceRight((composed, f) => f(composed), last(...args))
  }
}
```

假如我们的中间件链条为：[mid1, mid2, mid3], 那么最终组合成的函数即为：__mid1(mid2(mid3(...args)))__， 在dispatch时，中间件的响应顺序也为[mid1,mid2,mid3]

## 总结

我们整理一下通过中间件机制增强dispatch的过程：

#### 撰写中间件
打补丁， 包裹老的业务逻辑到新的业务逻辑形成一个新的中间件， 同时将各个中间件按序进行存储，形成一个中间件链

#### 迭代组合中间件
通过函数组合（Composing），新的业务逻辑不断装饰老的业务逻辑

####  pipeline
action将在各个中间件间流动, 每次流动，都有可能产生新的形式或者新的内容的action，亦即新的dispatch将会成为一个容纳action的管道

现在，具有中间件参与的dispatch如下图所示：

![中间件业务流程](https://www.lucidchart.com/publicSegments/view/b1fe65ad-1d2f-4bbe-a35e-f287fce57267/image.png)

## 栗子

现在，完整的看一个在Redux使用中间件的例子---一个[简单的计数器](https://github.com/yoyoyohamapi/redux-middleware-example)

首先，我们定义两个中间件：

1. 日志中间件logger：输出dispatch过程中的一些信息
2. thunk中间件：允许action是一个thunk


#### middlewares.js

```javascript
/**
 * 日志中间件
 */
const logger = ({dispatch, getState}) => next =>action=> {
    console.log('current state', getState())
    console.log('dispatching...', action);
    let result = next(action)
    console.log('next state', getState())
    console.log('.........')
    return result
}


/**
 * thunk中间件
 */
const thunk=({dispatch, getState})=>next=>action=>
    typeof action === 'function'?action(dispatch, getState):next(action)

export {thunk,logger}
```


#### actions.js

```javascript
export const INCREMENT = 'INCREMENT'
export const DECREMENT = 'DECREMENT'

export function increment() {
    return {
        type: INCREMENT
    }
}

export function decrement() {
    return {
        type: DECREMENT
    }
}

export function incrementIfOdd() {
    return (dispatch, getState) => {
        const { counter } = getState();

        if (counter % 2 === 0) {
            return;
        }

        dispatch(increment());
    };
}

```

#### reducers.js

```javascript
import { INCREMENT, DECREMENT } from './actions'

export default function countReducer(state={counter:0}, action){
    switch (action.type) {
        case INCREMENT:
            return Object.assign({}, state, {
                counter: state.counter + 1
            })
        case DECREMENT:
            return Object.assign({}, state, {
                counter: state.counter - 1
            })
        default:
            return state
    }
}

```


#### index.js

```javascript
import 'babel-polyfill'
import { createStore, applyMiddleware } from 'redux'
import  { increment, decrement, incrementIfOdd } from './actions'
import { logger, thunk} from './middlerwares'
import rootReducer from './reducers'

const store = createStore(
    rootReducer,
    applyMiddleware(
        thunk,
        logger
    )
)

store.dispatch(increment())
store.dispatch(increment())
store.dispatch(decrement())

store.dispatch(incrementIfOdd())

```

程序运行结果如下图所示：

![栗子](http://yoyoyohamapi.qiniudn.com/redux-middleware-middleware.png)

我们用图描述__store.dispatch(incrementIfOdd())__这一过程, 假设当前的counter为奇数：

![计数器流程](https://www.lucidchart.com/publicSegments/view/46caf1d5-08b1-4adf-9016-65461f32a717/image.png)
