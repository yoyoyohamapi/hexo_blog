title: 实现一个 redux-observable
date: 2018-08-21 15:06:18
tags:

- React
- Redux
- RxJS

categories: React

---

![](cover.png)

# 使用 redux-observable 实现组件自治

> 本文是 《使用 RxJS + Redux 管理应用状态》系列第二篇文章，将会介绍 redux-observable 的设计哲学和实现思路。返回第一篇：[使用 redux-observable 实现组件自治]()

## 状态流动

Elm 架构

dispatch 了一个 action，reducer 会劫持到 action，并根据 action 的 type 和 payload 更新状态。

这是一个同步过程，Redux 为了保护 reducer 的纯度是不推荐在 reducer 中处理副作用的（如 HTTP 请求）。因此，就出现了 redux-thunk、redux-saga 这样的 Redux 中间件去处理副作用。

这些中间件本质都是俘获 dispatch 的内容，然后处理副作用，最终 dispatch 新的 action 到 reducer，让 reducer 专心做一个纯的状态机。

## 用 observable 管理副作用

假定我们在 UI 层能派发出一个数据拉取的 `FETCH` action，拉取数据后，将生成拉取成功的 `FETCH_SUCCESS` action 或者是数据拉取失败的 `FETCH_ERROR` action。

![fetch]()

```
           FETCH
             |
         fetching....
             |
            / \
           /   \
 FETCH_SUCCESS FETCH_ERROR
```

如果我们用 FRP 模式来思考这个过程，FETCH 就不是一个独立的个体，而是存在于一条会派发 FETCH 的流上（observable）。FETCH_SUCCESS 于 FETCH_ERROR 也类似：

```
---- FETCH ---- FETCH ---- 

---- FETCH_SUCCESS ---- FETCH_SUCCESS ----

---- FETCH_ERROR ---- FETCH_ERROR ----
```

我们将 FETCH 流定义为 `fetch$`，则 FETCH_SUCCESS 和 FETCH_ERROR 都来自于 `fetch$`：

```ts
const fetch$: Observable<FetchAction> = //....
fetch$.pipe(
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
```

并且，我们同样可以用一个流来承载页面所有的 action：

```ts
const action$: Observable<Action>
```

那么， `fetch$` 亦可以由 `action$` 流转得到：

```ts
const fetch$ = action$.pipe(
  filter(({type}) => type === FETCH)
)
```

## 构建中间件

Redux 提供的中间件机制能让我们干预每个到来 action ：

![middleware]()

```ts
const middleware: Middleware = store => next => action => {
  // do something
}

const store = createStore(
  rootReducer,
  applyMiddleware(middleware)
)
```

中间件初始化时，我们就可以创建 `action$`。我们先思考下 `action$` 的职责：

- 在中间件拿到 action 后，向 `action$` 放入值
- `action$` 可以流转为新的 action 流，新的 action 流可以被 reducer 订阅

因此，`action$` 既是观察者又是可观察对象，是一个 Subject 对象。在中间件捕获到 action 时，我们向 `action$` 放入值：

```ts
const createMiddleware = (): Middleware => {
  const action$ = new Subject()
  const middleware: Middleware = store => next => action => {
    action$.next(action)
    const result = next(action)
    return result
  }
  return middleware
}
```

## 流的转换器

现在，在中间件中，我们初始化了 `action$`，我们还需要告知中间件如果通过 `action$` 生成更多的流。我们可以定义一个转换器，由它负责 `action$` 的流转：

```ts
interface Transformer {
  (action$: Observable<Action>): Observable<Action>
}

const fetchTransformer: Transformer = (action$) {
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

![多个 transformer ]()

```ts
const newActionsStreams: Observable<Action>[] = transformers.map(transformer => transformer(action$))
```

由于这些 action 具有一致的数据结构，因此我们可以将这些流进行合并：

![合并多个流]()

```js
const newAction$ = merge(newActionStreams)
```

现在，我们的中间件长这个样子：

```ts
const createMiddleware = (...transformers): Middleware => {
  const action$ = new Subject()
  const newAction$ = merge(tramsformer.map(transformer => transformer(action$)))
  const middleware: Middleware = store => {
    newAction$.subscribe(action => store.dispatch(action))
    return next => action => {
      action$.next(action)
      const result = next(action)
      return result
    }
  }
  return middleware
}
```

## 优化：`ofType` operator

因为我们时长需要 `filter(action => action.type === SOME_TYPE)` 这个过程，因此可以封装一个 operator 来优化这个过程：

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

再考虑到我们可能不只过滤一个 action type，因此可以优化我们的 `ofType` operator：

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

## 优化：获得 state

在我们的 transformer 过程中，可能需要获得应用状态，因此，我们可以考虑为 transformer 再传递当前的 store 对象，借此拿到应用状态：

```ts
interface Transformer {
  (action$: Observable<Action>, store: Store): Observable<Action>
}

// ...

const createMiddleware = (...transformers): Middleware => {
  const action$ = new Subject()
  const middleware: Middleware = store => {
    const newAction$ = merge(tramsformer.map(transformer => transformer(action$, store)))
    newAction$.subscribe(action => store.dispatch(action))
    return next => action => {
      action$.next(action)
      const result = next(action)
      return result
    }
  }
  return middleware
}
```

现在，当需要使用状态的时候，就通过 `store.getState()` 拿取：

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

## 优化：`state$`

在 FRP 下，一切数据源都应当是可被观察的，而上面我们对状态的取值确是主动的（proactive）的，而不是观察 state 变动。类似 action$，我们也可以将 state 流化：

```ts
interface Transformer {
  (action$: Observable<Action>, state$: Observable<State>): Observable<Action>
}

// ...

const createMiddleware = (...transformers): Middleware => {
  const action$ = new Subject()
  const state$ = new Subject()
  const middleware: Middleware = store => {
    const newAction$ = merge(tramsformer.map(transformer => transformer(action$, state$)))
    newAction$.subscribe(action => store.dispatch(action))
    return next => action => {
      action$.next(action)
      state$.next(state)
      const result = next(action)
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

大家乍看之下，这么做似乎不如 `store.getState()` 来的方便，为了获得当前状态，我们还引入了一个 operator `withLatestFrom`。但是，要注意到，我们引入 `state$` 不只为了获得状态和统一需求，更重要是为了观察状态：


1. 

## 优化：响应初始状态

## 优化：action 顺序