title: 如何实现一个 redux-observable
date: 2018-08-21 15:06:18
tags:

- React
- Redux
- RxJS

categories: React

---

![](cover.png)

> 本文是 《使用 RxJS + Redux 管理应用状态》系列第二篇文章，将会介绍 redux-observable 的设计哲学和实现思路。返回第一篇：[使用 redux-observable 实现组件自治](http://yoyoyohamapi.me/2018/08/18/%E4%BD%BF%E7%94%A8_redux-observable_%E5%AE%9E%E7%8E%B0%E7%BB%84%E4%BB%B6%E8%87%AA%E6%B2%BB/)
>
> 本系列的文章地址汇总：
>
> - [使用 redux-observable 实现组件自治](http://yoyoyohamapi.me/2018/08/18/%E4%BD%BF%E7%94%A8_redux-observable_%E5%AE%9E%E7%8E%B0%E7%BB%84%E4%BB%B6%E8%87%AA%E6%B2%BB/)
> - [如何实现一个 redux-observable](http://yoyoyohamapi.me/2018/08/21/%E5%AE%9E%E7%8E%B0%E4%B8%80%E4%B8%AA_redux-observable/)

<!--more-->

## Redux

Redux 脱胎于 Elm 架构，其状态管理视角和流程非常清晰和明确：

![](redux.png)

1. dispatch 了一个 action
2. reducer 俘获 action，并根据 action 类型进行不同的状态更新逻辑
3. 周而复始地进行这个过程

这个过程是同步的，Redux 为了保护 reducer 的纯度是不推荐在 reducer 中处理副作用的（如 HTTP 请求）。因此，就出现了 redux-thunk、redux-saga 这样的 Redux 中间件去处理副作用。

这些中间件本质都是俘获 dispatch 的内容，并在这个过程中进行副作用处理，最终 dispatch 一个新的 action 给 reducer，让 reducer 专心做一个纯的状态机。

## 用 observable 管理副作用

假定我们在 UI 层能派发出一个数据拉取的 `FETCH` action，拉取数据后，将派发拉取成功的 `FETCH_SUCCESS` action 或者是数据拉取失败的 `FETCH_ERROR` action 到 reducer。

```
           FETCH
             |
       fetching data...
             |
            / \
           /   \
 FETCH_SUCCESS FETCH_ERROR
```

如果我们用 FRP 模式来思考这个过程，FETCH 就不是一个独立的个体，而是存在于一条会派发 FETCH action 的流上（observable）：

```
---- FETCH ---- FETCH ---- 

---- FETCH_SUCCESS ---- FETCH_SUCCESS ----

---- FETCH_ERROR ---- FETCH_ERROR ----
```

若我们将 FETCH 流定义为 `fetch$`，则 FETCH_SUCCESS 和 FETCH_ERROR 都将来自于 `fetch$`：

```ts
const fetch$: Observable<FetchAction> = //....
fetch$.pipe(
  switchMap(() => from(api.fetch).pipe(
    // 拉取数据成功
    switchMap(resp => ({
      type: FETCH_SUCCESS,
      payload: {
        // ...
      }
    }),
    // 拉取数据失败
    catchError(error => of({
      type: FETCH_ERROR,
      payload: {
        // ....
      }
    }))
  ))
)
```

除此之外，我们可以用一个流来承载页面所有的 action：

```ts
const action$: Observable<Action>
```

那么， `fetch$` 亦可以由 `action$` 流转得到：

```ts
const fetch$ = action$.pipe(
  filter(({type}) => type === FETCH)
)
```

这样，我们就形成了使用 observable 流转 action 的模式：

![使用 observable 流转 action](使用observable流转action.png)

接下来，我们尝试讲这个模式整合到 Redux 中，让 observable 来负责应用的 action 流转和副作用处理。

## 构建中间件

Redux 提供的中间件机制能让我们干预每个到来的 action， 借此处理一些业务逻辑，然后再返还一个 action 给 reducer：

![middleware](middleware.png)

中间件的函数构成如下：

```ts
const middleware: Middleware = store => {
  // 初始化中间件
  return next => action => { 
  	// do something
  }
}

const store = createStore(
  rootReducer,
  applyMiddleware(middleware)
)
```

现在，当中间件初始化时，我们进行 `action$` 。当新的 action 到来时：

1. 将 action 交给 reducer 处理
2. 想 `action$` 中放入 action
3. `action$` 可以转化另一个的 action 流

因此，`action$` 既是观察者又是可观察对象，是一个 Subject 对象：

```ts
const createMiddleware = (): Middleware => {
  const action$ = new Subject()
  const middleware: Middleware = store => next => action => {
    // 将 action 交给 reducer 处理
    const result = next(action)
    // 将 action 放到 action$ 中进行流转
    action$.next(action)
    return result
  }
  return middleware
}
```

## 流的转换器

现在，在中间件中，我们初始化了 `action$`，但是如何得到 `fetch$` 这些由 `action$` 派生的流呢？因此，我们还需要告知中间件如果通过 `action$` 生成更多的流，不妨定义一个转换器，由它负责 `action$` 的流转，并在当中处理副作用：

![](transformer.png)

```ts
interface Transformer {
  (action$: Observable<Action>): Observable<Action>
}

const fetchTransformer: Transformer = (action$) => {
  action$.pipe(
    filter(({type}) => type === FETCH),
    switchMap(() => from(api.fetch).pipe(
      switchMap(resp => ({
        type: FETCH_SUCCESS,
        payload: {
          // ...
        }
      }),
      catchError(error => of({
        type: FETCH_ERROR,
        payload: {
          // ....
        }
      }))
    ))
  )
}
```

应用中，我们可能定义不同的转换器，从而得到派发不同 action 的流：

![多个 transformer ](transformers.png)

```ts
const newActionsStreams: Observable<Action>[] = transformers.map(transformer => transformer(action$))
```

由于这些 action 还具有一致的数据结构，因此我们可以将这些流进行合并，由合并后的流负责派发 action 到 reducer：

![合并多个流](merge.png)

```js
const newAction$ = merge(newActionStreams)
```

那么，修改我们的中间件实现：

```ts
const createMiddleware = (...transformers): Middleware => {
  const action$ = new Subject()
  // 运行各个 transformer，并将转换的流进行合并
  const newAction$ = merge(tramsformer.map(transformer => transformer(action$)))
  const middleware: Middleware = store => {
    // 订阅 newAction$
    newAction$.subscribe(action => store.dispatch(action))
    return next => action => {
      // 将 action 交给 reducer 处理
      const result = next(action)
      // 将 action 放到 action$ 中进行流转
      action$.next(action)
      return result
    }
  }
  return middleware
}
```

## 优化：`ofType` operator

由于我们总是需要 `filter(action => action.type === SOME_TYPE)` 来过滤 action，因此可以封装一个 operator 来优化这个过程：

```ts
const ofType: OperatorFunction<Observable<Action>, Observable<Action>> = (type: String) => pipe(
  filter(action => action.type === type)
)
```

```ts
const fetchTransformer: Transformer = (action$) {
  return action$.pipe(
    filter(({type}) => type === FETCH),
    switchMap(() => from(api.fetch)),
    // ...
  )
}
```

再考虑到我们可能不只过滤一个 action type，因此可以优化我们的 `ofType` operator 为：

```ts
const ofType: OperatorFunction<Observable<Action>, Observable<Action>> = 
  (...types: String[]) => pipe(
    filter((action: Action) => types.indexOf(action.type) > -1)
  )
```

```ts
const counterTransformer: Transformer = (action$) {
  return action$.pipe(
    ofType(INCREMENT, DECREMENT),
    // ...
  )
}
```

下面这个测试用例将用来测试我们的中间件是否能够工作了：

```ts
it('should transform action', () => {
   const reducer: Reducer = (state = 0, action) => {
    switch(action.type) {
      case 'PONG':
        return state + 1
      default:
        return state
    }
  }

  const transformer: Transformer = (action$) => {
    return action$.pipe(
        ofType('PING'),
        mapTo({type: 'PONG'})
      )
    )
  }

  const middleware = createMiddleware(transformer)
  const store = createStore(reducer, applyMiddleware(middleware))
  store.dispatch({type: 'PING'})
  expect(store.getState()).to.be.equal(1)
})
```

## 优化：获得 state

在 action 的流转过程可能还需要获得应用状态，例如，`fetch$` 中获取数据前，需要封装请求参数，部分参数可能来自于应用状态。因此，我们可以考虑为每个 transformer 再传递当前的 store 对象，使它能拿到当前的应用状态：

```ts
interface Transformer {
  (action$: Observable<Action>, store: Store): Observable<Action>
}

// ...

const createMiddleware = (...transformers): Middleware => {
  const action$ = new Subject()
  const middleware: Middleware = store => {
    // 将 store 也传递给 transformer
    const newAction$ = merge(tramsformer.map(transformer => transformer(action$, store)))
    newAction$.subscribe(action => store.dispatch(action))
    return next => action => {
      const result = next(action)
      action$.next(action)
      return result
    }
  }
  return middleware
}
```

现在，当需要取用状态的时候，就通过 `store.getState()` 拿取：

```ts
const fetchTransformer: Transformer = (action$, store) {
  return action$.pipe(
    filter(({type}) => type === FETCH),
    switchMap(() => {
      const { query, page, pageSize } = store.getState()
      const params = { query, page, pageSize }
      return from(api.fetch, params)
    }),
    // ...
  )
}
```

## 优化：观察状态

在响应式编程体系下，一切数据源都应当是可被观察的，而上面我们对状态的取值确是主动的（proactive）的，正确的方式是应当观察状态的变化，并在变化时作出决策：

![state$](state$.png)

为此，类似 `action$`，我们也将 state 流化，使得应用状态成为一个可观察对象，并将 `state$` 传递给 transformer：

```ts
interface Transformer {
  (action$: Observable<Action>, state$: Observable<State>): Observable<Action>
}

// ...

const createMiddleware = (...transformers): Middleware => {
  const action$ = new Subject()
  const state$ = new Subject()
  const middleware: Middleware = store => {
    // 由各个 transformer 获得应用的 action$
    const newAction$ = merge(tramsformer.map(transformer => transformer(action$, state$)))
    // 新的 action 到来时，将其又 dispatch 到 Redux 生态
    newAction$.subscribe(action => store.dispatch(action))
    return next => action => {
      // 将 action 交给 reducer
      const result = next(action)
      // 获得 reducer 处理后的新状态
      state$.next(state)
		  // 将 action 放入 action$
      action$.next(action)
      return result
    }
  }
  return middleware
}
```

当业务流程需要状态时，就可以自由组合 `state$` 得到：

```ts
const fetchTransformer: Transformer = (action$, state$) {
  return action$.pipe(
    filter(({type}) => type === FETCH),
    withLatestFrom(state$),
    switchMap(([action, state]) => {
      const { query, page, pageSize } = state
      const params = { query, page, pageSize }
      return from(api.fetch, params)
    }),
    // ...
  )
}
```

乍看之下，似乎不如 `store.getState()` 来的方便，为了获得当前状态，我们还额外引入了一个 operator `withLatestFrom`。但是，要注意到，我们引入 `state$` 不只为了获得状态和统一模式，更重要是为了**观察**状态。

举个例子，我们有一个备忘录组件，每次内容变动时，我们就存储一下草稿。如果我们能观察状态变动，通过响应式编程模式，当状态变动时，自动形成草稿存储的业务：

```ts
const saveDraft$: Observable<Action> = state$.pipe(
  // 选出当前
	pluck('content'),
  // 只有当内容变动时才考虑存储草稿
  distinctUntilChanged(),
  // 只在 1 s 内保存一次
  throttleTime(1000),
  // 调用服务存储草稿
  switchMap(content => from(api.saveDraft(content)))
  // ....
)
```

大家也可以在回顾系列第一篇所介绍的内容，正是由于 redux-observable 在 1.0 版本引入了 `state$`，我们才得以解耦组件的业务关系，实现单个组件的自治。

## 优化：响应初始状态

现在，我们可以测试一下现在的中间件，看能否观察应用状态了：

```ts
it('should observe state', () => {
   const reducer: Reducer = (state = {step: 10, counter: 0}, action) => {
    switch(action.type) {
      case 'PONG':
        return {
          ...state,
          counter: action.counter
        }
      default:
        return state
    }
  }

  const transformer: Transformer = (action$, state$) => {
    return action$.pipe(
        ofType('PING'),
      	withLatestFrom(state$, (action, state) => state.step + state.counter),
        map(counter => ({type: 'PONG', counter}))
      )
    )
  }

  const middleware = createMiddleware(transformer)
  const store = createStore(reducer, applyMiddleware(middleware))
  store.dispatch({type: 'PING'})
  expect(store.getState().counter).to.be.equal(10)
})
```

遗憾的是，这个测试用例将不会通过，通过调试发现，当我们 dispatch 了 PING action 后，`withLatestFrom` 没有拿到最近一次的 state。这是为什么呢？原来是因为 Redux 的 init action 并没有暴露给中间件进行拦截，因此，应用的初始状态没能被送入 `state$` 中，观察者无法观察到初始状态。

为了解决这个问题，在创建了 store 后，我们可以尝试 dispatch 一个无意义的 action 给中间件，强制将初始状态先送入 `state$` 中：

```ts
const middleware = createMiddleware(transformer)
const store = createStore(reducer, applyMiddleware(middleware))
// 派发一个 action 去获得初始状态
store.dispatch({type: '@@INIT_STATE'})
```

这个方式虽然能让测试通过，但缺不是很优雅，我们让用户手动去派发一个无意义的 action，这会让用户感觉很困惑。因此，我们考虑为中间件单独设置一个 API，用以在 store 创建后，完成一些任务：

```ts
// 设置一个 store 副本
let cachedStore: Store
const createMiddleware = (...transformers): Middleware => {
  const action$ = new Subject()
  const state$ = new Subject()
  const newAction$ = merge(transformers.map(transformer => transformer(action$, state$)))
  
  const middleware: Middleware = store => {
    cachedStore = store
    
    return next => action => {
      // 将 action 交给 reducer
      const result = next(action)
      // 获得 reducer 处理后的新状态
      state$.next(state)
		  // 将 action 放入 action$
      action$.next(action)
      return result
    }
  }
  
  middleware.run = function() {
    // 1. 开始对 action 的订阅
    newAction$.subscribe(cachedStore.dispatch)
    // 2. 将初始状态传递给 state$
    state$.next(cachedStore.getState())
  }
  return middleware
}
```

现在，我们为中间件提供了一个 `run` 方法，来让中间件在 store 创建以后完成一些工作。当我们创建好 store 后，运行 `run` 方法来运行中间件：

```ts
const middleware = createMiddleware(transformer)
const store = createStore(reducer, applyMiddleware(middleware))
// 运行我们的中间件
middleware.run()
```

## 优化：相互关联的 transformer

再考虑一个更加场景，各个 transformer 之间可能存在关联，各个 trasformer 也可能直接发出 action，而不需要依赖于 `action$`：

```js
  it('should support synchronous emission by transformer on start up', () => {
    const reducer = (state = [], action) => state.concat(action)
    const transformer1 = (action$, state$) => of({ type: 'FIRST' })
    const transformer2 = (action$, state$) => action$.pipe(
      ofType('FIRST'),
      mapTo({ type: 'SECOND' })
    )

    const epicMiddleware = createEpicMiddleware(epic1, epic2)
    const store = createStore(reducer, applyMiddleware(epicMiddleware))
    epicMiddleware.run()

    const actions = store.getState()
    actions.shift() // remove redux init action
    expect(actions).to.deep.equal([
      { type: 'FIRST' },
      { type: 'SECOND' }
    ])
  })
```

在这个测试用例中，我们看到：

- `transformer1` 不依赖于 `action$`，就直接发出了 `FIRST` action
- `transformer2` 接收到 `FIRST` action 之后，会发出 `SECOND` action

因此，我们期待应用的 action 序列是：

```
FIRST
SECOND
```

但是，在当前的中间件实现中，你将得到：

```
FIRST
```

这并不符合预期。但是，问题又出在哪里呢？

我们分析下程序的执行流程，首先，我们通过 `run` 方法启动了中间件，在其中我们的 `newAction$` 订阅了 observer：

```ts
middleware.run = function() {
  newAction$.subscribe(cachedStore.dispatch)
  state$.next(cachedStore.getState())
}
```

假定我们令派发 `FIRST` action 和 `SECOND` action 的为 `first$`，和 `second$`，则：

```ts
newAction$ = merge($first, second$)
```

当 `newAction$` subscribe 后，就将发生：

```
fisrt$ subscribe
first$ emit FIRST
store.diapatch(FIRST)
action$.next(FIRST)
action$ emit FIRST
second$ subscribe
```

可以看到，由于 `first$` 是同步派发值的，它并不会等到 `second$` subscribe 才开始发出值，因此， `second$` 因为 subscribe 滞后，就没能响应 `action$` 中的 `FIRST`。transformer 之间的关联并不成功。

```
fisrt$ subscribe
second$ subcscribe
first$ emit FIRST
store.diapatch(FIRST)
action$.next(FIRST)
action$ emit FIRST
second$ emit SECOND
second$ subscribe
```

RxJS 中提供了 [`subscribeOn`](http://reactivex.io/rxjs/class/es6/Observable.js~Observable.html#instance-method-subscribeOn) 和 [ `observeOn`](http://reactivex.io/rxjs/class/es6/Observable.js~Observable.html#instance-method-observeOn) 两个 operator，二者都接收 [Scheduler](http://reactivex.io/rxjs/class/es6/Scheduler.js~Scheduler.html) 对象作为参数，前者能控制 source `subscribe` observer 的节奏，后者则是控制 source 发出值的节奏。

因此，从需求上看，我们既要控制 `newAction$` 的 subscribe 节奏，也要控制 `newAction$` 的中派发值的节奏。

redux-observable 中，为 merge 后的流使用了 [queue scheduler](http://reactivex.io/rxjs/variable/index.html#static-variable-queue) 进行速率控制：

```ts
const newAction$ = merge(transformers.map(transformer => transformer(action$, state$))).pipe(
	observeOn(queueScheduler),
  subscribeOn(queueScheduler)
)
```

这样就能保证上面的测试用例中 action 含有：

```
FIRST
SECOND
```

为什么使用 queue scheduler，这个故事如何向开发者讲好， redux-observable 的作者也在探索（参看 [github 上对此的讨论](https://github.com/redux-observable/redux-observable/pull/493)），因为牵涉了很复杂递归过程。遗憾的是，截止目前为止，本文也只能分析现象的成因，而没法简练地概括 queue scheduler 在这个场景下的调度过程，读者如果对此有较好的认知，建议到官方的讨论下面进行回复。当然，这一块我也会继续关注。

## 总结

截止目前，我们的中间件已经允许我们通过 FRP 模式梳理应用状态了，这个中间件的实现已经非常类似于 redux-observable 的实现了。当然，大家生产环境还是用更流行，更稳定的 redux-observable，本文旨在帮助大家更好的理解如何在 Redux 中集成 RxJS 更好的管理状态，通过一步一步对中间件的优化，也让大家理解了了 redux-observable 的设计哲学和实现原理。本文实现的 mini redux-observable 我也放到了我的 [github](https://github.com/yoyoyohamapi/toys/tree/master/redux-observable) 上，包含了一些测试用例和一个小的 demo。

接下来，我们将探索将 redux-observable 以及 FRP 这套模式集成到 dva 架构的前端框架中，dva 架构帮助砍掉 Redux 冗长的样板代码，而 redux-observable 则专注于副作用处理。

-------

## 参考资料

- [RxJS API document](http://reactivex.io/rxjs/class/es6/Observable.js~Observable.html)
- [PRIMER ON RXJS SCHEDULERS](https://staltz.com/primer-on-rxjs-schedulers.html)
- [redux-observable #493 pull request](https://github.com/redux-observable/redux-observable/pull/493)
- [Gerard Sans — Bending time with Schedulers and RxJS 5](https://www.youtube.com/watch?v=AL8dG1tuH40&t=2366s)

## 关于本系列

- 本系列将从介绍 redux-observable 1.0 开始，阐述自己在结合 RxJS 到 Redux 中的心得体会。涉及内容会有 redux-observable 实践介绍，redux-observable 实现原理探究，最后会介绍下自己当前基于 redux-observble + dva architecture 的一个 state 管理框架 reobservable。
- 本系列不是 RxJS 或者 Redux 入门，不再讲述他们的基础概念，宣扬他们的核心优势。如果你搜索 RxJS 不小心进到了这个系列，对 RxJS 和 FRP 程序设计产生了兴趣，那么入门我会推荐：
  - [learnrxjs.io](https://www.learnrxjs.io/)
  - Andre Staltz 在 [egghead.io](https://egghead.io/courses/introduction-to-reactive-programming) 上的一系列课程
  - 程墨的 [《深入浅出 RxJS》](https://item.jd.com/12336101.html)
- 本系列更不是教程，只是介绍自己在 Redux 中应用 RxJS 的一些思路，希望更多人能指出当中存在的误区，或者交流更优雅的实践。
- 由衷的感谢实践路上一些师兄的帮助，尤其感谢腾讯云的 questguo 学长在模式上的指导。reobservable 脱胎于腾讯云 questguo 主导的 React 框架 —— TCFF，期待未来 TCFF 的开源。
- 感谢小雨的设计支援。