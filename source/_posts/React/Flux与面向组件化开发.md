title: Flux下的组件化开发
tags: react,todomvc,flux
categories: React
---


首先要明确的是，Flux并不是一个前端框架，而是前端的一个设计模式，其把前端的一个交互流程简单的模拟成了一个单向数据流。

![Flux](http://facebook.github.io/flux/img/flux-simple-f8-diagram-1300w.png)

在上图中，我们可以看到Flux的四个核心构成：

### Action
一个交互动作,更多的时候代表一个时间，比如用户在页面组件上的点击（click），失焦（blur），双击（doubleclick）等等。一个Action往往由如下两个部分组成：

* 交互类型 ，例如创建、删除、更新等
* 交互体，或者说交互的携带信息， 例如创建的文本

### Dispatcher

Action分发器，从上图的数据流中，我们可以看到，用户每次产生的Action将被送入Dispatcher，Dispatcher对Action进行简单的包裹之后将其派发到__所有__Store中。

> ！注意，Dispatcher的这种广播行为有别于__Pub/Sub__模型，在__Pub/Sub__模型中，需要声明订阅的消息类型，然后发布者会像订阅者广播特定类型的消息。而在Dispatcher中，Store向其注册的任意回调接口都不要声明订阅的Action类型，即Store只告诉Dispatcher“如果Action到来，请你把它发送给我”。当Dispatcher派发Action时，所有注册到Dispatcher的callback都会得到响应。回调可以通过简单的switch-case来针对不对类型的Action做出不同的行为。

### Store

数据仓库，保存了我们某个前端App的数据以及对数据的操作。Store会向Dispatcher注册一个回调函数，该回调函数接受一个action作为参数。当Action被派发到Store时，该回调函数被调用，借由Action中描述的__交互类型（type）__，Store进行不同处理，这些处理都将被持久化到Store维护的数据对象上。

Store完成数据的变更后，由于Flux并不是双向数据绑定的，因而即便我们已经持久化了Store中的数据，但组件（View）的数据并未得到更新，组件也不会重新渲染。所以，每次数据变动后，为了告知组件去更新数据，Store会emit一个change事件。当监听到change事件发生，注册到监听器上的回调去完成各个组件的状态更新。

### View

顾名思义，这就是用户所能看到的视图，有别于传统的MVC，在Flux中，View并不会和数据模型（Model）产生交互，其只会产生各种交互行为（Actions），这些行为将会被送到Dispatcher中，如下图所示：

![Action被送入Dispatcher](http://facebook.github.io/flux/img/flux-simple-f8-diagram-with-client-action-1300w.png)

当View中维护的状态变动时，View需要被重新渲染。

### TODO栗子

下面我们分析一个用React+Flux实现的一个Flux栗子，其源码托管在[github](https://github.com/facebook/flux/tree/master/examples/flux-todomvc)上。

在项目实践中，面向组件化开发的最佳场景我认为是 __交互驱动型的开发__，可能描述不够准确，准确点说就是一旦一个完善的交互设计稿产生时，我们就可以去__分割__和__分析__组件了，我们现在来分析Todo的交互原型：

![Todo交互](http://7pulhb.com2.z0.glb.clouddn.com/Flux-Todo%E4%BA%A4%E4%BA%92%E5%8E%9F%E5%9E%8B.png)

> 这是交互设计师的给我们的原稿，并且，原稿可能远不止这样一幅简单的图像，可能还包括更多的交互效果

我们将会把这个应用拆分为如下组件：

#### TodoApp

![TodoApp](http://7pulhb.com2.z0.glb.clouddn.com/Flux-TodoApp.png)

通常，在前端面向组件化的开发过程中，我们往往需要一个顶部容器包裹住我们的组件，一个页面可以存在若干个这样的顶部容器，这个容器类似一个集装箱或者盒子，封装了某个页面应用的所有组件和状态。例如，在某视频网站中，视频播放窗口可以作为一个顶部容器，其包裹了播放窗口，进度条，播放选项等各个组件，同时，评论部分也可以作为一个顶部容器，其包裹了评论列表，评论框等组件。

在Todo例子中，TodoApp作为一个顶部容器，包裹了所有Todo应用需要的组件，这样，我们在应用入口只需要从TodoApp开始渲染，进而逐个渲染其子组件。但更为重要的是，TodoApp将会封装其下各个组件需要用到的状态，通过数据流，各个组件将会收到状态，并且在状态改变时，重新渲染自己，最终更新页面内容。

-----------

#### Header

![TodoHeader](http://7pulhb.com2.z0.glb.clouddn.com/Flux-TodoHeader.png)

这是一个头部组件，根据交互设计，他除了将保有静态的“todos”文字标题以外，还将会具有如下行为：

* 右侧输入框失焦或者按下回车键：创建新的todo

因此，Header是一个__无状态（Stateless）__的组件

----------

#### Footer

![TodoFooter](http://7pulhb.com2.z0.glb.clouddn.com/Flux-TodoFooter.png)

这是一个底部组件，它将显示未完成todo数，并能删除所有已完成todo，故而，首先他需要获得如下__状态__:

* 所有任务：

	* 通过遍历任务的完成情况，能获得未完成任务数
	* 通过遍历任务的完成情况，统计已完成任务的信息
	* 如果当前无任务，隐藏Footer

因此，在初步的设计中，Footer是一个__有状态（Stateful）__的组件。后面我们会谈到该做法的不恰当。

并且，他具有如下行为：

* 单击右侧按钮（Clear completed）: 清除所有已完成任务

------------

#### MainSection

![MainSection](http://7pulhb.com2.z0.glb.clouddn.com/TodoMainSection.png)

该组件将会负责渲染所有的以创建任务，因而他需要维护的状态为：

* 所有任务

其具有的行为：

*  点击顶部左侧图标按钮：完成/取消完成所有任务，具体根据__所有任务__是否都完成了决定

因此， MainSection也是一个有状态的组件。

----------------

#### TodoItem

![TodoItem](http://7pulhb.com2.z0.glb.clouddn.com/TodoItem.png)

这是Todo项，其Todo对象来源于MainSection的迭代，并且该组件具有如下行为：

* 单击左侧按钮：完成/取消完成该任务
* 单击右侧按钮：删除该Todo
* 双击Todo文本：进入如下的编辑模式

![编辑模式](http://7pulhb.com2.z0.glb.clouddn.com/%E5%8F%8C%E5%87%BB%E8%BF%9B%E5%85%A5%E7%BC%96%E8%BE%91.png)

我们不难发现，“是否处于编辑模式”实际上可作为该组件的一个状态，该状态的切换直接影响了该组件的展示和行为，所以，组件应当维护一个状态：

* 是否编辑模式

在编辑模式中，具有如下行为：

* 输入框失焦或者按下回车键：更新任务

可以看到，在__Header__组件及__TodoItem__组件的输入框组件具有一致的交互行为，所以，我们可以将其提出来作为单独的组件，这也侧面体现了，一份完善的交互设计原型将预测到实现过程中的复用和抽象，避免了一些代码重构的时间。

----------------

#### TodoTextInput
现在，我们抽象出一个可复用的输入组件TodoTextInput供Header和TodoItem使用，他需要维护如下状态：

* 输入值

他具有如下行为：

* 输入框失焦或者按下回车键：调用存储过程（创建，更新等等）

----------------

综上，我们以一个简单的示意图表示如上的划分：

#### 组件结构：

![组件结构](https://www.lucidchart.com/publicSegments/view/0f5014ca-ef30-4eac-8e6a-74a76dfc18d6/image.png)

#### 状态维护(仅展示部分)：

![状态维护](https://www.lucidchart.com/publicSegments/view/1c6988eb-cb2c-4092-8f7f-2cbc11e07c9e/image.png)

通过上图，我们发现在__MainSection__和__Footer__组件中都需要维护__所有todo__（allTodos）这一状态，由于MainSection与Footer属于平级的组件，所以，当MainSection中的__allTodos__这一状态发生改变时，为使Footer中的状态也发生改变，MainSection中需要保存有Footer的引用才能更新到Footer的状态，同理，Footer中也需要保存有MainSection的引用。这样，两个组件将会是强耦合的，如下图所示：

![状态不共享](https://www.lucidchart.com/publicSegments/view/210b9a7b-f8db-4642-bee0-62b8a62dbb33/image.png)

设想，如果以后还有更多的组件需要__allTodos__这一状态，这一设计模式将会是十分糟糕的，任何一个组件的脱离将可能导致整个引用网络的崩溃。

既然__allTodos__被多个组件共享，那么我们可以将该状态提升到更上一次的组件中，然后通过props传递给子组件。所以，在本例中，最终将状态提到了顶部容器TodoApp中进行维护，这样，通过TodoApp的__setState()__方法，所有绑定到TodoApp的组件都获得了状态更新，避免了组件间的相互引用，如下图所示：

![状态共享](https://www.lucidchart.com/publicSegments/view/0140d060-b037-4ab8-9357-61029d6a14ef/image.png)

> 在React中，我们应当尽量创建多的无状态（Stateless）的组件，而把共享状态放到上层组件中，使上层组件成为一个有状态（Stateful）的组件。这样，有状态组件封装了交互行为以及与行为互动的状态，子组件通过props共享状态并进行数据渲染。更多state与props的关系和区别可以参看[官方文档](https://facebook.github.io/react/docs/interactivity-and-dynamic-uis-zh-CN.html)

#### 封装

![目录结构](http://7pulhb.com2.z0.glb.clouddn.com/Flux-%E7%9B%AE%E5%BD%95%E7%BB%93%E6%9E%84.png)

其中app.js为应用的入口文件，从入口开始，逐步构造我们的App。

下面，开始实现我们的逻辑，顺着Flux的单向数据流，逐个分析Todo例子中的实现。

##### Dispatcher
__js/AppDispatcher.js__

```javascript
var Dispatcher = require('flux').Dispatcher;
module.exports = new Dispatcher();
```

可以看到，TodoMVC中的Dispatcher实现来自于于官方的[实现](https://www.npmjs.com/package/flux)。我们可以看下flux中的Dispatcher源码，所有解说都放在代码注释中：

首先看到__Dispatcher__的构造函数：

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

再看注册方法__register(callback)__,每个向Dispatcher的注册的回调（callback）都拥有唯一Id进行标识：

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
3. 执行完成,标识该回调已经完成（Handled）

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

派发__dispatch(payload)__指定的用户行为payload到所有的callback将经历如下过程：

首先，需要明确的是能够进行派发的前提是当前Dispatcher为空闲状态，接下来

1. 派发前的预处理___startDispatching()__
	- 初始化所有回调的状态
	- 设置当前正在分发的payload
  - 标识当前的Dispatcher状态为"正在进行派发"

2. 根据注册顺序依次执行回调___invokeCallback(id)__

3. 派发结束后的收尾工作___stopDispatching()__
 - 清除派发对象
 - 标识当前的Dispatcher状态为"结束派发"

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
__waitFor__

再看Dispatcher中一个很重要的方法:__waitFor(ids)__, 顾名思义，该方法的作用是：等待其他向Dispatcher注册了的回调执行完成。因而，该方法主要保证了dispatch时，待响应的回调函数的执行的__顺序性__。

例如，在一个航班订票系统中，我们首先要选择完国家（Country），才能选择城市（City），所以，当一个类型为“更新所选国家”的交互被送到CityStore所注册的回调时，为了保证能正确的选择更新后国家的城市，我们需要这样做：

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

下面我们看__waitFor()__的源码实现：

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

##### Store实现

在__js/stores/TodoStore.js__中：

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

然后导出一个全局单例，该单例提供了常用的外部访问接口，并且通过node提供的EventEmitter来实现事件的派发和监听：

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

最后，我们需要向__AppDispatcher__注册回调函数，以便在payload被分发到TodoStore时，TodoStore能做出响应：

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

##### Actions
我们将TodoApp中常见的Action都封装到了__js/TodoActions.js__中, 通过其中的__AppDispatcher__单例，我们可以将Action派发出去:

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

##### Components
下面开始实现各个组件， 个人偏向的流程是先在组件目录下创建好各个空白组件，之后再依序进行装填

```javascript
var React = require('react');

var Header = React.createClass({

    render: function () {
      // TODO::render
    },
});

module.exports = Header;
```

装填顺序我会选择先装填顶部容器（此例中即为__TodoApp__），之后按照DOM树自底向上的进行装填:
__TodoApp.react.js__:

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

> 为了方便，TodoApp不仅维护__allTodos(所有任务)__这个状态，还维护__areAllComplete(是否所有任务都已完成)__，该状态主要服务于__MainSection__中的---”完成所有/取消完成所有任务“这一用例，避免重复遍历__allTodos__的开销。

我们可以看到，TodoApp提供了一个___onChange()__方法作为TodoStore的__change__事件的回调，当TodoStore发出change事件时，TodoApp将刷新状态，借此通知其下组件如MainSection等重新渲染。

更多组件的实现不再赘述。下面着重介绍flux的工作流程

#### 工作流程

我们以__创建新的Todo__这一工作流程为例展示Flux的工作过程。在Flux中，该流程如下图所示：

![创建Todo工作流程](https://www.lucidchart.com/publicSegments/view/49534565-8e62-4836-a6ca-e616269ba094/image.png)

(1). 我们在TodoTextInput中敲入数据，在输入框上，我们监听了__失焦(onBlur)__和__按下键盘按键(onKeyDown)__的事件

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

当事件发生时，调用___save()__方法进行处理：

```
_save: function() {
    this.props.onSave(this.state.value);
    this.setState({
      value: ''
    });
 },
```

(2). 在__Header__组件中，我们通过为__TodoTextInput__指定__onSave__属性（props）来确定当输入域发生变化后的执行逻辑，使得我们在TodoTextInput的状态发生改变时，能够发出一个__“创建行为”__到Dispatcher

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

> 我们之所以不在TodoTextInput中写死__TodoActions.create(text)__主要是考虑到组件的可扩展性。“输入域变动后的存储逻辑”更应当被设计为一种配置，通过在不同场景下指定其__onSave__属性（prop），使得TodoTextInput更加通用。

(3). 在__TodoActions.create()__中，我们将action送到Dispatcher，并由其派发一个“创建action”到__TodoStore__:

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

(4). TodoStore在接收到Dispatcher派发来的Action之后，其向Dispatcher注册的回调被调用, 新的todo会被持久化，并因此引起了TodoStore维护的___todos__的改变，所以TodoStore会抛出一个change事件：

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

(5). 由于TodoApp向TodoStore订阅了change事件

```javascript
// js/components/TodoApp.react.js
componentDidMount: function() {
    TodoStore.addChangeListener(this._onChange);
},
```

此时，change事件发生， 回调___onChange()__被触发, TodoApp维护的状态得到更新:

```javascript
 /**
   * Event handler for 'change' events coming from the TodoStore
   */
  _onChange: function() {
    this.setState(getTodoState());
  }
```

(6). 由于MainSection及Footer组件中的属性（prop）绑定了TodoApp维护的状态，所以在TodoApp刷新状态（setState()）后，二者将会被重新渲染。
