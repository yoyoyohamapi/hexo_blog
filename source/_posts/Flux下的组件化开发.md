title: Flux 下的组件化开发
date: 2016-06-20 21:20:30
tags:
	- React
	- Flux
	- JavaScript
categories: React
---

Flux 构成
----------

首先要明确的是，Flux 并不是一个前端框架，而是前端的一个设计模式，一个状态管理机制，其把前端的一个交互流程简单的模拟成了一个单向数据流。

<div style="text-align:center">
<img src="http://facebook.github.io/flux/img/flux-simple-f8-diagram-1300w.png" width="500"></img>
</div>

在上图中，我们可以看到 Flux 的四个核心构成：

<!--more-->

### Action

一个交互动作,更多的时候代表一个时间，比如用户在页面组件上的点击（click），失焦（blur），双击（doubleclick）等等。一个 Action 往往由如下两个部分组成：

* 交互类型（type）：例如创建、删除、更新等
* 交互体（payload）：或者说交互的携带信息，例如创建的文本

### Dispatcher

Action 分发器。从上图的数据流中，我们可以看到，用户每次产生的 Action 将被送入 Dispatcher，Dispatcher 对 Action 进行简单的包裹之后将其派发到**所有** Store 中。

> 注意！Dispatcher 的这种广播行为有别于 **Pub/Sub** 模型，在 Pub/Sub 模型中，需要声明订阅的消息类型，然后发布者会像订阅者广播特定类型的消息。而在 Dispatcher 中，Store 向其注册的任意回调接口都不要声明订阅的 Action 类型，即 Store 只告诉 Dispatcher “如果 Action 到来，请你把它发送给我”。当 Dispatcher 派发 Action 时，所有注册到 Dispatcher 的 callback 都会得到响应。回调可以通过简单的 switch-case 来针对不对类型的 Action 做出不同的响应。

Store
------------

数据仓库，保存了我们某个前端 App 的数据以及对数据的操作。Store 会向 Dispatcher 注册一个回调函数，该回调函数接受一个 Action 作为参数。当 Action 被派发到 Store 时，该回调函数被调用，借由 Action 中描述的**交互类型（type）**，Store 进行不同处理，这些处理都将被持久化到Store维护的数据对象上。

Store 完成数据的变更后，由于 Flux 并不是双向数据绑定的，因而即便我们已经持久化了 Store 中的数据，但组件的数据并未得到更新，组件也不会重新渲染。所以，每次数据变动后，为了告知组件去更新数据，Store 会 emit 一个 change 事件。当监听到 change 事件发生，注册到监听器上的回调去完成各个组件的状态更新。

View
-----------

顾名思义，这就是用户所能看到的视图。有别于传统的 MVC，在 Flux 中，View 并不会和数据模型（Model）产生交互，其只会产生各种交互行为（Actions），这些行为将会被送到Dispatcher中，如下图所示：

<div style="text-align:center">
<img src="http://facebook.github.io/flux/img/flux-simple-f8-diagram-with-client-action-1300w.png" width="500"></img>
</div>

当 View 中维护的状态变动时，View 需要被重新渲染。

Todo 栗子
--------------

下面我们分析一个用 React+Flux 实现的一个 Flux 栗子，其源码托管在 [github](https://github.com/facebook/flux/tree/master/examples/flux-todomvc)上。

在项目实践中，面向组件化开发的最佳场景我认为是 **交互驱动型的开发**，该定义可能不够准确，其描述的是一旦一个完善的交互设计稿产生后，我们就可以从交互稿中 **分割** 出组件，并进行组件的状态**分析**。假设我们得到了 Todo 的交互原型：

<div style="text-align:center">
<img src="http://7pulhb.com2.z0.glb.clouddn.com/Flux-Todo%E4%BA%A4%E4%BA%92%E5%8E%9F%E5%9E%8B.png" width="800"></img>
</div>

> 这是交互设计师的给我们的原稿，并且，原稿可能远不止这样一幅简单的图像，可能还包括更多的交互效果

我们将会把这个应用拆分为如下组件：

### TodoApp

<div style="text-align:center">
<img src="http://7pulhb.com2.z0.glb.clouddn.com/Flux-TodoApp.png" width="500"></img>
</div>

通常，在前端面向组件化的开发过程中，我们往往需要一个顶部容器包裹住我们的组件，一个页面可以存在若干个这样的顶部容器，这个容器类似一个集装箱或者盒子，封装了某个页面应用的所有组件和状态。例如，在某视频网站中，视频播放窗口可以作为一个顶部容器，其包裹了播放窗口，进度条，播放选项等各个组件，同时，评论部分也可以作为一个顶部容器，其包裹了评论列表，评论框等组件。

在 Todo 中，TodoApp 作为一个顶部容器，包裹了所有 Todo 应用需要的组件，这样，我们在应用入口只需要从 TodoApp 开始渲染，进而逐个渲染其子组件。但更为重要的是，TodoApp将会封装其下各个组件需要用到的状态，通过数据流，各个组件将会收到状态，并且在状态改变时，重新开始渲染自己，最终更新页面内容。

### Header

<div style="text-align:center">
<img src="http://7pulhb.com2.z0.glb.clouddn.com/Flux-TodoHeader.png" width="500"></img>
</div>

这是一个头部组件，根据交互设计，他除了将保有静态的 “todos” 文字标题以外，还将会具有如下行为：

* 右侧输入框失焦或者按下回车键：创建新的 todo 任务

可以看到，由于 Header 不维护任何状态，所以 Header 是一个 **无状态（Stateless）** 的组件

### Footer

<div style="text-align:center">
<img src="http://7pulhb.com2.z0.glb.clouddn.com/Flux-TodoFooter.png" width="500"></img>
</div>

这是一个底部组件，它将显示未完成 todo 数，并能删除所有已完成todo。首先他需要维护这些 **状态**:

* 所有任务：

	* 通过遍历任务的完成情况，能获得未完成 todo 任务数
	* 通过遍历任务的完成情况，统计已完成 todo 任务的信息
	* 如果当前无任务，隐藏 Footer

因此，在初步的设计中，Footer 是一个**有状态（Stateful）**的组件。后面我们会谈到该做法的不恰当。

并且，他具有如下行为：

* 单击右侧按钮（Clear completed）: 清除所有已完成 todo 任务

### MainSection

<div style="text-align:center">
<img src="http://7pulhb.com2.z0.glb.clouddn.com/TodoMainSection.png" width="500"></img>
</div>

该组件将会负责渲染所有的以创建任务，因而他需要维护的状态为：

* 所有任务

其具有的行为：

* 点击顶部左侧图标按钮：完成/取消完成所有任务，具体根据 **所有任务** 是否都完成了决定

因此， MainSection 也是一个有状态的组件。

### TodoItem

<div style="text-align:center">
<img src="http://7pulhb.com2.z0.glb.clouddn.com/TodoItem.png" width="500"></img>
</div>

这是 todo 项目，每个项目来源于 MainSection 中的迭代，并且该组件具有如下行为：

* 单击左侧按钮：完成/取消完成该任务
* 单击右侧按钮：删除该 todo
* 双击 todo 文本：进入下面的编辑模式

<div style="text-align:center">
<img src="http://7pulhb.com2.z0.glb.clouddn.com/%E5%8F%8C%E5%87%BB%E8%BF%9B%E5%85%A5%E7%BC%96%E8%BE%91.png" width="500"></img>
</div>

我们不难发现，“是否处于编辑模式” 实际上可作为该组件的一个状态，该状态的切换直接影响了该组件的展示和行为，所以，TodoItem 组件应当维护一个状态：

* 是否编辑模式

在编辑模式中，具有如下行为：

* 输入框失焦或者按下回车键：更新任务

可以看到，Header 组件及 TodoItem 组件的中的输入框具有一致的交互行为，所以，我们可以将该输入框提出来作为单独的组件，这也侧面体现了，一份完善的交互设计原型将预测到实现过程中的复用和抽象，避免了一些代码重构的时间。

### TodoTextInput
现在，我们抽象出一个可复用的输入组件 TodoTextInput 供 Header 和 TodoItem 使用，他需要维护如下状态：

* 输入值

他具有如下行为：

* 输入框失焦或者按下回车键：调用存储过程（创建，更新等等）


### 组件结构

<div style="text-align:center">
<img src="https://www.lucidchart.com/publicSegments/view/0f5014ca-ef30-4eac-8e6a-74a76dfc18d6/image.png" width="500"></img>
</div>

### 状态维护

<div style="text-align:center">
<img src="https://www.lucidchart.com/publicSegments/view/1c6988eb-cb2c-4092-8f7f-2cbc11e07c9e/image.png" width="500"></img>
</div>

我们发现在 MainSection 和 Footer 组件中都需要维护 **allTodos** 这一状态。由于 MainSection 与 Footer 属于平级的组件，所以，当 MainSection 中的 allTodos 这一状态发生改变时，为使 Footer 中的状态也发生改变，MainSection 中需要保存有 Footer 的引用才能更新到 Footer 的状态，同理，Footer 中也需要保存有 MainSection 的引用。这样，两个组件将会是强耦合的，如下图所示：

<div style="text-align:center">
<img src="https://www.lucidchart.com/publicSegments/view/210b9a7b-f8db-4642-bee0-62b8a62dbb33/image.png" width="500"></img>
</div>

设想，如果以后还有更多的组件需要 allTodos 这一状态，这一设计模式将会是十分糟糕的，任何一个组件的脱离将可能导致整个引用网络的崩溃。

既然 allTodos 被多个组件共享，那么我们可以将该状态提升到更上一次的组件中，然后通过 `props` 传递给子组件。所以，在本例中，最终将 allTodos 提到了顶部容器 TodoApp 中进行维护，这样，通过 TodoApp 的 `setState()` 方法，所有绑定到 TodoApp 的组件都获得了状态更新，避免了组件间的相互引用，如下图所示：

<div style="text-align:center">
<img src="https://www.lucidchart.com/publicSegments/view/0140d060-b037-4ab8-9357-61029d6a14ef/image.png" width="500"></img>
</div>

在 React 中，我们应当尽量创建多的无状态（Stateless）的组件，而把共享状态放到上层组件中，使上层组件成为一个有状态（Stateful）的组件。这样，有状态组件封装了交互行为以及与行为互动的状态，子组件通过 `props` 共享状态并进行数据渲染。更多 `state` 与 `props` 的关系和区别可以参看[官方文档](https://facebook.github.io/react/docs/interactivity-and-dynamic-uis-zh-CN.html)

### 实现

#### 目录结构

<div style="text-align:center">
<img src="http://7pulhb.com2.z0.glb.clouddn.com/Flux-%E7%9B%AE%E5%BD%95%E7%BB%93%E6%9E%84.png" width="500"></img>
</div>

其中 app.js 为应用的入口文件，从入口开始，逐步构造我们的 App。

#### Dispatcher

**js/AppDispatcher.js**

```javascript
var Dispatcher = require('flux').Dispatcher;
module.exports = new Dispatcher();
```

可以看到，TodoMVC 中的 Dispatcher 实现来自于于官方的[实现](https://www.npmjs.com/package/flux)。我们可以看下 Flux 中的 Dispatcher 源码，首先看到 `Dispatcher()` 构造函数：

```javascript
function Dispatcher() {
    _classCallCheck(this, Dispatcher);

    this._callbacks = {}; // 保存向Dispatcher注册回调函数
    this._isDispatching = false; // 是否正在分派Action
    this._isHandled = {}; // 已经完成执行的回调列表
    this._isPending = {}; // 正在执行中的回调列表
    this._lastID = 1; // 回调Id的起始标志
}
```

再看注册方法 `register(callback)` ,每个向 Dispatcher 的注册的回调（callback）都拥有唯一 Id 进行标识：

```javascript
/**
 * 向Dispatcher注册回调函数,每个回调函数都有唯一id进行标识
 * @param callback
 * @returns {string} 注册回调的id
 */

Dispatcher.prototype.register = function register(callback) {
    var id = _prefix + this._lastID++;
    this._callbacks[id] = callback;
    return id;
};

/**
 * 根据id删除回调
 */

Dispatcher.prototype.unregister = function unregister(id) {
    !this._callbacks[id] ? process.env.NODE_ENV !== 'production' ? invariant(false, 'Dispatcher.unregister(...): `%s` does not map to a registered callback.', id) : invariant(false) : undefined;
    delete this._callbacks[id];
};
```

执行一个注册了的回调函数将经历如下过程：

1. 标识当前正在执行的回调为进行中（Pending）状态
2. 将当前待处理的用户行为（payload）送至回调执行
3. 执行完成，标识该回调已经完成（Handled）

```javascript
/**
 * 执行回调函数,该过程为:
 * 1. 标识当前正在执行的回调为Pending状态
 * 2. 将payload送入回调执行
 * 3. 执行完成,标识该回调已经完成
 * @internal
 */

Dispatcher.prototype._invokeCallback = function _invokeCallback(id) {
    this._isPending[id] = true;
    this._callbacks[id](this._pendingPayload);
    this._isHandled[id] = true;
};
```

派发 `dispatch(payload)` 指定的用户行为 `payload` 到所有的 callback 将经历如下过程：

首先，需要明确的是能够进行派发的前提是当前 Dispatcher 为空闲状态，接下来

1. 派发前的预处理 `startDispatching()`
	- 初始化所有回调的状态
	- 设置当前正在分发的 `payload`
  - 标识当前的 Dispatcher 状态为 “正在进行派发”
2. 根据注册顺序依次执行回调 `invokeCallback(id)`
3. 派发结束后的收尾工作 `stopDispatching()`
 - 清除派发对象
 - 标识当前的 Dispatcher 状态为"结束派发"

```javascript
/**
 * 派发一个payload到所以已注册的callback中
 */

Dispatcher.prototype.dispatch = function dispatch(payload) {
    !!this._isDispatching ? process.env.NODE_ENV !== 'production' ? invariant(false, 'Dispatch.dispatch(...): Cannot dispatch in the middle of a dispatch.') : invariant(false) : undefined;
    this._startDispatching(payload);
    try {
        for (var id in this._callbacks) {
            if (this._isPending[id]) {
                continue;
            }
            this._invokeCallback(id);
        }
    } finally {
        this._stopDispatching();
    }
};


/**
 * 分发payload前的初始化:
 * 1. 初始化所有回调的状态
 * 2. 设置当前正在分发的payload
 * 3. 标识当前"正在进行派发"
 * @internal
 */

Dispatcher.prototype._startDispatching = function _startDispatching(payload) {
    for (var id in this._callbacks) {
        this._isPending[id] = false;
        this._isHandled[id] = false;
    }
    this._pendingPayload = payload;
    this._isDispatching = true;
};

/**
 * 结束派发时的收尾工作
 * 1. 清除派发对象
 * 2. 标识当前"结束派发"
 * @internal
 */

Dispatcher.prototype._stopDispatching = function _stopDispatching() {
    delete this._pendingPayload;
    this._isDispatching = false;
};
```

**waitFor**

再看 Dispatcher 中一个很重要的方法: `waitFor(ids)`, 顾名思义，该方法的作用是：等待其他向 Dispatcher 注册了的回调执行完成。因而，该方法主要保证了 dispatch 时，待响应的回调函数的执行的 **顺序性**。

例如，在一个航班订票系统中，我们首先要选择完国家（Country），才能选择城市（City），所以，当一个类型为 “更新所选国家”的交互被送到 `CityStore` 所注册的回调时，为了保证能正确的选择更新后国家的城市，我们需要这样做：

```javascript
CityStore.dispatchToken = flightDispatcher.register(function(payload) {
     if (payload.actionType === 'country-update') {
       /*
        * 如果不执行waitFor(),由于程序的异步性，那么可能CityStore的回调先于ContryStore的回调执行
        * 此时的国家尚未更新，得到的默认城市是错误的，而并不是最新的
        * */
       flightDispatcher.waitFor([CountryStore.dispatchToken]);
       // waitFor()保证了ContryStore先响应了'country-update'，即保证了国家更新先于城市更新

       // 此时我们能正确的选择该国家的城市
       CityStore.city = getDefaultCityForCountry(CountryStore.country);
     }
});
```

下面我们看 `waitFor()` 的源码实现：

```javascript
/**
 * 等待指定的回调完成
 */

Dispatcher.prototype.waitFor = function waitFor(ids) {
    !this._isDispatching ? process.env.NODE_ENV !== 'production' ? invariant(false, 'Dispatcher.waitFor(...): Must be invoked while dispatching.') : invariant(false) : undefined;
    for (var ii = 0; ii < ids.length; ii++) {
        var id = ids[ii];
        if (this._isPending[id]) {
            !this._isHandled[id] ? process.env.NODE_ENV !== 'production' ? invariant(false, 'Dispatcher.waitFor(...): Circular dependency detected while ' + 'waiting for `%s`.', id) : invariant(false) : undefined;
            continue;
        }
        !this._callbacks[id] ? process.env.NODE_ENV !== 'production' ? invariant(false, 'Dispatcher.waitFor(...): `%s` does not map to a registered callback.', id) : invariant(false) : undefined;
        this._invokeCallback(id);
    }
};
```

#### Store

在 **js/stores/TodoStore.js** 中：

首先，我们维护我们的数据对象，并提供若干对于该数据的操作：

```javascript
// 保存TODO列表
var _todos = {};
/**
 * 创建一个 Todo
 * @param text {string} Todo内容
 */
function create(text) {
    // ...
}

/**
 * 更新一个 TODO item
 * @param id {string}
 * @param updates {object} 待更新对象的属性
 */
function update(id, updates) {
    // ...
}

/**
 * 根据一个更新属性值对象更新所有 Todo
 * @param updates {object}
 */
function updateAll(updates) {
	// ...
}

/**
 * 删除 Todo
 * @param id {string}
 */
function destroy(id) {
    // ...
}

/**
 * 删除所有的已完成的 TODO items
 */
function destroyCompleted() {
	// ...
}
```

然后导出一个全局单例，该单例提供了常用的外部访问接口，并且通过 node 提供的 `EventEmitter` 来实现事件的派发和监听：

```javascript
var TodoStore = assign({}, EventEmitter.prototype, {
    /**
     * 是否所有TODO 都已完成
     * @return {boolean}
     */
    areAllComplete: function () {
        // ...
    },

    /**
     * 获得所有的TODO
     * @returns {object}
     */
    getAll: function () {
        // ...
    },

    /**
     * 发送变更事件
     */
    emitChange: function () {
        // ...
    },

    /**
     * 添加变更事件监听
     * @param callback
     */
    addChangeListener: function (callback) {
        // 一旦受到变更事件, 触发回调
        /*
         *   例如, 当我们创建一条todo时,
         *   TodoStore将会发出一条变更事件,
         *   上游的状态维护器将会调用callback进行状态更新
         */
        this.on(CHANGE_EVENT, callback);
    },

    /**
     * 删除变更事件监听
     * @param callback
     */
    removeChangeListener: function (callback) {
        this.removeListener(CHANGE_EVENT, callback);
    }
});
```

最后，我们需要向 `AppDispatcher` 注册回调函数，以便在 `payload` 被分发到 TodoStore 时，TodoStore 能做出响应：

```javascript
AppDispatcher.register(function callback(action) {
    var text;

    // 根据不同的action类型(即不同的交互逻辑), 执行不同过程
    switch (action.actionType) {
        case TodoConstants.TODO_CREATE:
            text = action.text.trim();
            if( text!=='') {
                create(text);
                // 一旦变更,发出变更事件,
                TodoStore.emitChange();
            }
            break;

        case TodoConstants.TODO_TOGGLE_COMPLETE_ALL:
            // ...
            break;

        case TodoConstants.TODO_UNDO_COMPLETE:
            // ...
            break;

        case TodoConstants.TODO_COMPLETE:
            // ...
            break;

        case TodoConstants.TODO_UPDATE_TEXT:
           // ...
            break;

        case TodoConstants.TODO_DESTROY:
            // ...
            break;

        case TodoConstants.TODO_DESTROY_COMPLETED:
            // ...
            break;
        default:
        // no op
    }
});
```

> !注意, 在回调执行过程中，如果发生状态的变动，需要抛出change事件，这样才能将组建的状态也更新（通过回调）。

#### Actions
我们将 TodoApp 中常见的 Action 都封装到了 **js/TodoActions.js** 中, 通过其中的 `AppDispatcher` 单例，我们可以将 Action 派发出去:

```javascript
var TodoActions = {
    /**
     * 创建行为
     * @param text {string}
     */
    create: function (text) {
        // 将创建行为送到Dispatcher, Dispatcher派发这个行为(action对象)到各个Store
        AppDispatcher.dispatch({
            actionType: TodoConstants.TODO_CREATE,
            text: text
        });
    },

    /**
     * 更新行为
     * @param id {string}
     * @param text {string}
     */
    updateText: function (id, text) {
        // ...
    },

    /**
     * 全部设置为完成
     * @param todo
     */
    toggleComplete: function (todo) {
       // ...
    },

    /**
     * 标记所有的Todo为已完成
     */
    toggleCompleteAll: function () {
      // ...
    },

    /**
     *
     * @param id
     */
    destroy: function (id) {
       // ...
    },

    /**
     * 删除所有已完成的Todo
     */
    destroyCompleted: function() {
      // ...
    }
};

```

#### Components

下面开始实现各个组件，个人偏向的流程是先在组件目录下创建好各个空白组件，之后再依序进行装填：

```javascript
var React = require('react');

var Header = React.createClass({

    render: function () {
      // TODO::render
    },
});

module.exports = Header;
```

装填顺序我会选择先装填顶部容器（此例中即为 TodoApp ），之后按照 DOM 树 **自底向上** 地进行装填:

**TodoApp.react.js**:

```javascript
var Footer = require('./Footer.react');
var Header = require('./Header.react');
var MainSection = require('./MainSection.react');
var React = require('react');
var TodoStore = require('../stores/TodoStore');

// 在根DOM下维护状态,
// 这样的状态往往是共享状态(会向下传递的状态)
function getTodoState() {
    return {
        allTodos: TodoStore.getAll(),
        areAllComplete: TodoStore.areAllComplete()
    };
}

var TodoApp = React.createClass({
    getInitialState: function () {
        return getTodoState();
    },

    /**
     * 绑定生命期--挂载
     */
    componentDidMount: function () {
        // 挂载时再为TodoStore添加监听器
        TodoStore.addChangeListener(this._onChange);
    },

    componentWillUnmount: function () {
        TodoStore.removeChangeListener(this._onChange);
    },

    render: function () {
        return (
            <div>
                <Header />
                <MainSection
                    allTodos={this.state.allTodos}
                    areAllComplete={this.state.areAllComplete}
                />
                <Footer allTodos={this.state.allTodos}/>
            </div>
        );
    },

    /**
     * Event handler for 'change' events coming from the TodoStore
     */
    _onChange: function() {
        this.setState(getTodoState());
    }
});

module.exports = TodoApp;
```

> 为了方便，TodoApp 不仅维护 allTodos 这个状态，还维护 areAllComplete，该状态主要服务于 MainSection 中的 “完成所有/取消完成所有任务” 这一用例，避免重复遍历 allTodos 的开销。

我们可以看到，TodoApp 提供了一个 `onChange()` 方法作为 TodoStore 的 `change` 事件的回调，当 TodoStore 发出 change 事件时，TodoApp 将刷新状态，借此通知其下组件如 MainSection 等重新渲染。

更多组件的实现不再赘述。下面着重介绍 Flux 的工作流程

### 工作流程

我们以 创建新的 Todo 这一工作流程为例展示 Flux 的工作过程。在 Flux 中，该流程如下图所示：

<div style="text-align:center">
<img src="https://www.lucidchart.com/publicSegments/view/49534565-8e62-4836-a6ca-e616269ba094/image.png" width="500"></img>
</div>

（1） 我们在 TodoTextInput 中敲入数据，在输入框上，我们监听了 **失焦(onBlur)** 和 **按下键盘按键(onKeyDown)** 的事件：

```javascript
// js/components/TodoTextInput.react.js
 /**
   * @return {object}
   */
  render: function() /*object*/ {
    return (
      <input
        className={this.props.className}
        id={this.props.id}
        placeholder={this.props.placeholder}
        onBlur={this._save}
        onChange={this._onChange}
        onKeyDown={this._onKeyDown}
        value={this.state.value}
        autoFocus={true}
      />
    );
  },
```

当事件发生时，调用 `save()` 方法进行处理：

```
_save: function() {
    this.props.onSave(this.state.value);
    this.setState({
      value: ''
    });
 },
```

（2） 在 Header 组件中，我们通过为 TodoTextInput 指定 `onSave` 属性（props）来确定当输入域发生变化后的执行逻辑，使得我们在 TodoTextInput 的状态发生改变时，能够发出一个 “创建行为” 到 Dispatcher：

```javascript
// js/components/Header.react.js
/**
   * @return {object}
   */
  render: function() {
    return (
      <header id="header">
        <h1>todos</h1>
        <TodoTextInput
          id="new-todo"
          placeholder="What needs to be done?"
          onSave={this._onSave}
        />
      </header>
    );
  },

  /**
   * Event handler called within TodoTextInput.
   * Defining this here allows TodoTextInput to be used in multiple places
   * in different ways.
   * @param {string} text
   */
  _onSave: function(text) {
    if (text.trim()){
      TodoActions.create(text);
    }
  }
```
我们之所以不在 TodoTextInput 中写死 `TodoActions.create(text)` 主要是考虑到组件的可扩展性。“输入域变动后的存储逻辑”更应当被设计为一种配置，通过在不同场景下指定其 `onSave` 属性（prop），使得 TodoTextInput 更加通用。

（3） 在 `TodoActions.create()` 中，我们将 Action 送到 Dispatcher，并由其派发一个 “创建 Action”到
TodoStore：

```javascript
// js/actions/TodoActions.js
 /**
   * @param  {string} text
   */
  create: function(text) {
    AppDispatcher.dispatch({
      actionType: TodoConstants.TODO_CREATE,
      text: text
    });
  },
```

（4） TodoStore 在接收到 Dispatcher 派发来的 Action 之后，其向 Dispatcher 注册的回调被调用, 新的 todo 会被持久化，并因此引起了 TodoStore 维护的 todos 的改变，所以 TodoStore 会抛出一个 change 事件：

```javascript
// js/stores/TodoStore.js
AppDispatcher.register(function(action) {
  var text;

  switch(action.actionType) {
    case TodoConstants.TODO_CREATE:
      text = action.text.trim();
      if (text !== '') {
        create(text);
        TodoStore.emitChange();
      }
      break;

	// ...  

    default:
      // no op
  }
});
```

（5）由于 TodoApp 向 TodoStore 订阅了 change 事件：

```javascript
// js/components/TodoApp.react.js
componentDidMount: function() {
    TodoStore.addChangeListener(this._onChange);
},
```
此时，change 事件发生，回调 `onChange()` 被触发, TodoApp 维护的状态得到更新：
```javascript
 /**
   * Event handler for 'change' events coming from the TodoStore
   */
  _onChange: function() {
    this.setState(getTodoState());
  }
```

（6） 由于 MainSection 及 Footer 组件中的属性（prop）绑定了 TodoApp 维护的状态，所以在 TodoApp 刷新状态 `setState()` 后，二者将会被重新渲染。
