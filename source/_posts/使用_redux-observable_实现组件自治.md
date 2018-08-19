title: 使用 redux-observable 实现组件自治
date: 2018-08-18 10:35
tags:

- React
- Redux
- RxJS

categories: React

---

![](cover.png)

# 使用 redux-observable 实现组件自治

> 本文是 《使用 RxJS + Redux 管理应用状态》系列第一篇文章，旨在介绍 redux-obervable v1 版本为 React + Redux 带来的组件自治能力。

## redux-observable 简介

[redux-observable](https://github.com/redux-observable/redux-observable) 是 redux 一个中间件，使用了 RxJs 来驱动 action 副作用。与其目的类似的有大家比较熟悉的 [redux-thunk](https://github.com/reduxjs/redux-thunk) 和 [redux-saga](https://github.com/redux-saga/redux-saga)。通过集成 redux-observable，我们可以在 Redux 中使用到 RxJS 所提供的函数响应式编程（FRP）的能力，从而更轻松的管理我们的异步副作用（前提是你熟悉了 RxJS）。

<!--more-->

**Epic** 是 redux-observable 的核心概念和基础类型，几乎承载了 redux-observable 的所有。从形式上看，Epic 是一个函数，其接收一个 **action stream**，输出一个新的 action stream：

```ts
function (action$: Observable<Action>, state$: StateObservable<State>): Observable<Action>
```

可以看到，Epic 扮演了 stream 转换器的能力。

在 redux-observable 的视角下，Redux 作为中央状态收集器，当一个 action 被 dispatch，历经某个同步或者异步任务，将 dispatch 一个新的 action，携带着它的负载（payload）到 reducer，如此反复。这么看的话，Epic 定义了 action 因果关系。

同时，FRP 模式的 RxJS 还带来了如下能力：

- **竞态处理能力**
- **声明式地任务处理**
- **测试友好**
- **组件自治**（redux-observable 1. 0 开始支持）

## 实践

本系列是假定读者有了 FRP 和 RxJS 的基础，因此，关于 RxJS 和 redux-observable 不再赘述。

现在，我们实践一个常见的业务需求 —— 列表页。先罗列列表页的诉求：

- 间隔一段时间轮询数据列表
- 支持搜索，触发搜索时，重新轮询
- 支持字段排序，排序状况变动，重新轮询
- 支持分页，页面容量修改，分页状况变动，重新轮询
- 组件卸载时，结束轮询

在前端组件化开发的思路下，我们可能会设计如下容器组件（Container），其中基础组件基于 [ant design]()：

- **数据表格（含分页）**：基于 **Table** 组件

- **搜索框：**：基于 **Input** 组件

- **排序选择框：**基于 **Select ** 组件

在 React + Redux 的架构下，容器组件通过 `connect` 方法从状态树上采摘自己所需要的状态，因此，先要认识到，这些容器组件必定存在一个**耦合**—— Redux：

![列表页](rxjs_redux_list_page.png)

借下将会讨论两种不同的模式下，列表应用的状态管理和副作用处理，它们分别是基于 redux-thunk 或者 redux-saga 的传统模式，以及基于 redux-observable 的 FRP 模式。大家可以看到不同模式下，除了基础的对于 Redux 的耦合，组件及其数据生态（状态与副作用）上耦合状况的差异。

当然，为了让大家更好的理解文章传达的，我也撰写了一个 **[demo](https://github.com/yoyoyohamapi/self-government-component-with-redux-observable)**，大家可以 clone & run。接下来的代码也都来源于这个 demo。demo 一个 github 小应用，其中你看到用户列表背后是基于 FRP 模式的，Repo 列表则是基于传统模式的：

![demo screenshot](rxjs_redux_demo.png)

## 传统模式下组件的耦合

在传统的模式下，我们需要面对一个现实，对于状态的获取是**主动式（proactive）**的：

```ts
const state = store.getState()
```

亦即我们需要**主动取用状态**，而无法监听状态变化。因此，在这种模式下，我们组件化开发的思路会是：

- 组件挂载，开启轮询
  - 搜索时，结束上次轮询，构建新的请求参数，开始新的轮询
  - 排序变动时，结束上次轮询，构建新的请求参数，开始新的轮询
  - 分页变动时，结束上次轮询，构建新的请求参数，开始新的轮询
- 组件卸载，结束轮询

在这种思路下，我们撰写搜索，排序，分页等容器时，当容器涉及的取值变动时，不仅需要在状态树上更新这些值，还需要去重启一下轮询。

![组件耦合](rxjs_redux_traditional.png)

假定我们使用 redux-thunk 来处理副作用，代码大致如下：

```ts
let pollingTimer: number = null

function fetchUsers(): ThunkResult {
  return (dispatch, getState) => {
    const delay = pollingTimer === null ? 0 : 15 * 1000
    pollingTimer = setTimeout(() => {
      dispatch({
        type: FETCH_START,
        payload: {}
      })
      const { repo }: { repo: IState } = getState()
      const { pagination, sort, query } = repo
      // 封装参数
      const param: ISearchParam = {
        // ...
      }
      // 进行请求
      // fetch(param)...
  }, delay)
}}

export function polling(): ThunkResult {
  return (dispatch) => {
    dispatch(stopPolling())
    dispatch({
      type: POLLING_START,
      payload: {}
    })
    dispatch(fetchUsers())
  }
}

export function stopPolling(): IAction {
  clearTimeout(pollingTimer)
  pollingTimer = null
  return {
    type: POLLING_STOP,
    payload: {}
  }
}

export function changePagination(pagination: IPagination): ThunkResult {
  return (dispatch) => {
    dispatch({
      type: CHANGE_PAGINATION,
      payload: {
        pagination
      }
    })
    dispatch(polling())
  }
}

export function changeQuery(query: string): ThunkResult {
  return (dispatch) => {
    dispatch({
      type: CHANGE_QUERY,
      payload: {
        query
      }
    })
    dispatch(polling())
  }
}

export function changeSort(sort: string): ThunkResult {
  return (dispatch) => {
    dispatch({
      type: CHANGE_SORT,
      payload: {
        sort
      }
    })
    dispatch(polling())
  }
}
```

可以看到，涉及到**请求参数**的几个组件，如筛选项目，分页，搜索等，当它们 dispatch 了一个 action 修改对应的业务状态后，**还需要手动 dispatch 一个重启轮询的 action 结束上一次轮询，开启下一次轮询**。

或许这个场景的复杂程度你觉得也还能接受，但是假想我们有一个更大的项目，或者现在的项目未来会扩展得很大，那么组件势必会越来越多，参与协作的开发者也会越来越多。协作的开发者就需要时刻关注到自己撰写的组件是否会是其他开发者撰写的组件的影响因子，如果是的话，影响有多大，又该怎么处理？

> 这里提到的组件不单纯指 UI Component，还包括了组件涉及的数据生态。因为绝大部分前端开发者撰写业务组件时，除了 UI，还要实现 UI 涉及的业务逻辑。

我们归纳下使用传统模式梳理数据流以及副作用面临的问题：

1. **过程式编程**，代码啰嗦
2. **竞态处理**需要人为地通过标志量等进行控制
3. **组件间耦合**大，彼此牵连。

## FRP 模式与组件自治

在 FRP 模式下，遵循 **passive** 模式，state 应当被观察和响应，而不是主动获取。因此，redux-observable 从 [1.0](https://github.com/redux-observable/redux-observable/blob/master/CHANGELOG.md#100-alpha0-2018-04-04)  开始，不再推荐使用 `store.getState()` 进行状态获取，Epic 有了新的函数签名， 第二个参数为 `state$`：

```ts
function (action$: Observable<Action>, state$: StateObservable<State>): Observable<Action>
```

state$ 的引入，redux-observable 也到了里程碑，现在，我们能在 Redux 中更进一步地实践 FRP。

回过头来，我们还可以将列表页的需求概括为：

- 间隔一段时间轮询数据列表
- 参数（排序，分页等）变动时，重新发起轮询
- 主动进行搜索时，重新发起轮询
- 组件卸载时结束轮询

在 FRP 模式下，我们定义一个轮询 epic：

```ts
const pollingEpic: Epic = (action$, state$) => {
  const stopPolling$ = action$.ofType(POLLING_STOP)
  const params$: Observable<ISearchParam> = state$.pipe(
    map(({user}: {user: IState}) => {
      const { pagination, sort, query } = user
      return {
        q: `${query ? query + ' ' : ''}language:javascript`,
        language: 'javascript',
        page: pagination.page,
        per_page: pagination.pageSize,
        sort,
        order: EOrder.Desc
      }
    }),
    distinctUntilChanged(isEqual)
  )

  return action$.pipe(
    ofType(LISTEN_POLLING_START, SEARCH),
    combineLatest(params$, (action, params) => params),
    switchMap((params: ISearchParam) => {
      const polling$ = merge(
        interval(15 * 1000).pipe(
          takeUntil(stopPolling$),
          startWith(null),
          switchMap(() => from(fetch(params)).pipe(
            map(({data}: ISearchResp) => ({
              type: FETCH_SUCCESS,
              payload: {
                total: data.total_count,
                list: data.items
              }
            })),
            startWith({
              type: FETCH_START,
              payload: {}
            }),
            catchError((error: AxiosError) => of({
              type: FETCH_ERROR,
              payload: {
                error: error.response.statusText
              }
            }))
          )),
          startWith({
            type: POLLING_START,
            payload: {}
          })
      ))
      return polling$
    })
  )
}
```

下面是对这个 Epic 的一些解释。

- 首先我们声明轮询结束流，当轮询结束流有值产生时，轮询会被终止：

```ts
const stopPolling$ = action$.ofType(POLLING_STOP)
```

- 参数来源于状态，由于现在状态可观测，我们可以从状态流 `state$` 派发一个下游 —— **参数流**：

```ts
const params$: Observable<ISearchParam> = state$.pipe(
  map(({user}: {user: IState}) => {
    const { pagination, sort, query } = user
    return {
      // 构造参数
    }
  }),
  distinctUntilChanged(isEqual)
)
```

> 我们预期参数流都是最新的参数，因此使用了 `dinstinctUntilChanged(isEqual)` 来判断两次参数的异同

- 主动进行搜索，或者参数变动时，将创建轮询流（借助到了 `combineLatest` operator），最终，新的 action 仰仗于数据拉取结果：

```ts
return action$.pipe(
  ofType(LISTEN_POLLING_START, SEARCH),
  combineLatest(params$, (action, params) => params),
  switchMap((params: ISearchParam) => {
    const polling$ = merge(
      interval(15 * 1000).pipe(
        takeUntil(stopPolling$),
        // 自动开始轮询
        startWith(null),
        switchMap(() => from(fetch(params)).pipe(
          map(({data}: ISearchResp) => {
            // ... 处理响应
          }),
          startWith({
            type: FETCH_START,
            payload: {}
          }),
          catchError((error: AxiosError) => {
            // ...
          })
        )),
        startWith({
          type: POLLING_START,
          payload: {}
        })
      ))
    return polling$
  })
)
```

OK，我们现在**只需要**在数据表格这个容器组件挂载时 dispatch 一个 `LISTEN_POLLING_START` 事件，即可开始我们的轮询，在其对应的 Epic 中，它完全知道什么时候去结束轮询，什么时候去重启轮询。我们的分页组件，排序选择组件都不再需要关心重启轮询这个需求。例如分页组件的状态变动的 action 就只需要修改状态即可，而不用再去关注轮询：

```ts
export function changePagination(pagination: IPagination): IAction {
  return {
    type: CHANGE_PAGINATION,
    payload: {
      pagination
    }
  }
}
```

在 FRP 模式下，passive 模型让我们观测了 state，声明了轮询的诱因，让轮询收归到了数据表格组件中， 解除了轮询和数据表格与分页，搜索，排序等组件的耦合。实现了数据表格的**组件自治**。

> **组件自治**：组件只用关注如何治理自己。

![](rxjs_redux_frp.png)

总结，利用 FRP 进行副作用处理带来了：

- **声明式地（declarative）**描述异步任务，代码简洁
- 使用 `switchMap` operator 处理**竞态**任务
- 尽可能减少组件耦合，来达到**组件自治**。利于多人协作的大型工程。

其带来的利好拳拳打到了传统模式的痛处。下图是一个更直观的对比，同样的业务逻辑，考上的是 redux-saga 实现，考下则是 redux-observable 实现。你一眼就能感受到谁更简洁明了：

![](rxjs_redux_saga.jpg)

![](rxjs_redux_observable.jpg)

## 接入 redux-observable

redux-observable 只是 redux 一个中间件，因此它可以和你现在的 redux-thunk，redux-saga 等共存，redux-observable 的作者你可以渐进地接入 redux-observable 去处理一些复杂的业务逻辑，当你基本熟悉了 RxJS 和 FRP 模式，你会发现它可以做一切。

后续，考虑到整个工程的风格控制，还是建议只选择一套模型，FRP 在复杂场景下表现力卓著，在简单场景下，也不会大炮打蚊子。

## 参考资料

- [redux-observable official docs](https://redux-observable.js.org/docs)

-------------------------------------------

## 关于本系列

- 本系列将从介绍 redux-observable 1.0 开始，阐述自己在结合 RxJS 到 Redux 中的心得体会。涉及内容会有 redux-observable 实践介绍，redux-observable 实现原理探究，最后会介绍下自己当前基于 redux-observble + dva architecture 的一个 state 管理框架 reobservable。

- 本系列不是 RxJS 或者 Redux 入门，不再讲述他们的基础概念，宣扬他们的核心优势。如果你搜索 RxJS 不小心进到了这个系列，对 RxJS 和 FRP 程序设计产生了兴趣，那么入门我会推荐：

  - [learnrxjs.io](https://www.learnrxjs.io/)
  - Andre Staltz 在 [egghead.io](https://egghead.io/courses/introduction-to-reactive-programming) 上的一系列课程
  - 程墨的 [《深入浅出 RxJS》](https://item.jd.com/12336101.html)

- 本系列更不是教程，只是介绍自己在 Redux 中应用 RxJS 的一些思路，希望更多人能指出当中存在的误区，或者交流更优雅的实践。

- 由衷的感谢实践路上一些师兄的帮助，尤其感谢腾讯云的 questguo 学长在模式上的指导。reobservable 脱胎于腾讯云 questguo 主导的 React 框架 —— TCFF，期待未来 TCFF 的开源。

- 感谢小雨的设计支援。

  





