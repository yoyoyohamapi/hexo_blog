title: callbag，一个有趣的规范

date: 2018-02-14 17:23

tags:

- 函数式编程
- JavaScript

categories: JavaScript 函数式编程

---



## push 和 pull 模型

如果你了解 RxJs，在响应式编程中，Observable 和 Obsever 是 push 模型，与之对应的，还有一个 pull 模型：

![](https://i.stack.imgur.com/HwvQv.png)

- **Pull（`f(): B`）**：返回一个值。
- **Push（`f(x: A): void`）**：响应式的，当有值产生时，会发出一个事件，并携带上这个值。订阅了该事件的观察者（Observer）将获得反馈。

JavaScript 中的 `Math.random()`、`window.outerHeight` 等都是 pull 模型：

```js
const height = window.outerHeight();
// 或者是迭代器写法
function* getWindowHeight() {
    while(true) {
        yield window.outerHeight;
    }
}
var iter = getWindowHeight()
iter.next()
```

pull 模型包含两个部分：

- **生产者**：负责生产数据，是数据源
- **消费者**：负责消费数据，是数据的使用方

在 pull 模型中，数据是**按需索取**的。

再通过 RxJs 看一个 push 模型的例子：

```js
Rx.Observable
    .fromEvent(document, 'click')
	.map(event => `Event time: ${event.timeStamp}`)
    .subscribe(function observer(val) {
    	console.log(val);
	})
```

push 模型的组成包含了两个部分：

- **可观察（可监听）对象**：是数据来源
- **观察者（监听者）**：是数据的使用方

与 pull 模型不同，观察者**不能主动索取数据**，而是观察数据源，当数据源有数据时，才可消费和使用。

push 模型有这么一些优点：

- **高度复用的可观察对象**：通过对源可观察对象使用不同的运算子，可构建出新的可观察对象。
- **延迟执行**：可观察对象只有被观察者订阅，才会派发数据。
- **声明式、描述未来的代码**：我们只用声明数据源和数据消费方式，而不用关心数据交付时的细节。

[Cycle.js](https://cycle.js.org/) 的作者 Andre Staltz 长久以来面对一个问题，Cycle.js 及其推荐使用的响应式编程库 [xstream](https://github.com/staltz/xstream) 都是 push 模型的，这让框架的模型和业务代码都受益于 push 模型的优点。但是，实际项目中，我们还是有不少 pull 模型下的需求，Andre Staltz 也开了一个 [issue](https://github.com/cyclejs/cyclejs/issues/581) ，讨论如何更好的使用代码描述 pull 模型。

<!--more-->

## push 与 pull 可以是同型的

stalz 看到，我们的 Observable 和 Observer：

```typescript
interface Observer {
  next(x): void;
  error(e): void;
  complete(): void;
}

interface Observable {
  subscribe(observer): Subscription;
  unsubsribe(): void;
}
```

可以通过函数进行描述：

```js
function observable(msgType, msgPayload) {}
```

- `msgType == 0`：payload 是 observer，意味着 observer 向 observable 问好，需要订阅这个 observerble。（subscribe）
- `msgType == 1`：意味着 observer 将取消对 observable 的订阅。（unsubscribe）

```js
function observer(msgType, msgPayload) {}
```

当：

- `msgType == 1`：对应 `observer.next(payload)`，即 observable 交付数据给 observer，此时 payload 携带了数据。
- `msgType == 2` 且 payload 为 `undefined`：对应于 `observer.complete()`。
- `msgType == 2` 且 payload 含有值：对应于 `observer.error(payload)`，此时 payload 描述了错误。

进一步概括就是：

**Observer**:

- ```
  observer(1, data): void
  ```

  - 数据交付 ：observable 将数据交付给 observer

- ```
  observer(2, err): void
  ```

  - 出错：observable 将错误告知 observer

- ```
  observer(2): void
  ```

  - 完成：observable 不再有数据，告知 observer 任务完成


**Observable**:

- ```
  observable(0, observer): void
  ```

  - 问好：observer 订阅了 observable

- ```
  observable(2): void
  ```

  - 结束：observer 取消对 observable 的订阅


这么概括下来，我们发现，pull 模型也可以进行类似的概括：

**Consumer**：

- ```
  consumer(0, producer): void
  ```

  - 问好：**在 pull 模型中，producer 需要向 consumer 问好，告诉 consumer 有需要时，从哪里取值**

- ```
  consumer(1, data): void
  ```

  - 数据交付：producer 将数据交付给 consumer

- ```
  consumer(2, err): void
  ```

  - 出错：producer 将错误告知 consumer

- ```
  consumer(2): void
  ```

  - 完成：producer 告知 consumer 任务已完成

**Producer**：

- ```
  producer(0, consumer): void
  ```

  - 问好：consumer 确定和哪个 producer 交互

- ```
  producer(1, data): void
  ```

  - 数据交付：**在 pull 模型中，consumer 需要主动向 producer 取值**

- ```
  producer(2): void
  ```

  - 结束：consumer 结束了和 producer 的交互

综上，我们发现，push 和 pull 模型是同型的（具有一样的角色和函数签名），因此，可以通过一个规范同时定义二者。

## callbag

staltz 为 push 和 pull 模型创建了一个名为 callbag 的[规范](https://github.com/callbag/callbag)，这个规范的内容如下：

````typescript
(type: number, payload?: any) => void
````

### 定义（Defination）

- Callbag：一个函数，函数签名为： `(type: 0 | 1 | 2, payload?: any) => void`
- Greet：如果一个 callbag 以 `0` 为第一个参数被调用，我们就说 `该 callbag 被问好了`。此时函数执行的操作是： “向这个 callbag 问好”。
- Deliver：如果一个 callbag 以 `1` 为第一个参数被调用，我们就说 “这个 callbag 正被交付数据”。此时函数执行的操作是：“交付数据给这个 callbag”。
- Terminate：如果一个 callbag 以 `2` 为第一个参数被调用，我们就说 “这个 callbag 被终止了”。此时函数执行的操作是：“终止这个 callbag”。
- Source：一个负责交付数据的 callbag。
- Sink：一个负责接收（消费）数据的 callbag。

### 协议（Protocal）

**问好（Greets）**: `(type: 0, cb: Callbag) => void`

当第一个参数是 `0`，而第二个参数是另外一个 callbag（即一个函数）的时候，这个 callbag 就被问好了。

**握手（Handshake）**

当一个 source 被问好，并被作为 payload 传递给了某个 sink，sink **必须**使用一个 callbag payload 进行问好，这个 callbag 可以是他自己，也可以是另外的 callbag。换言之，问好是相互的。相互间的问好被称为**握手**。

**终止（Termination）**: `(type: 2, err?: any) => void`

当第一个参数是 `0`，而第二个参数要么是 undefined（由于成功引起的终止），要么是任何的真实值（由于失败引起的终止），这个 callbag 就被终止了。

在握手之后，source **可能**终止掉 sink，sink 也**可能**会终止掉 source。如果 source 终止了 sink，则 sink **不应当**终止 source，反之亦然。换言之，终止行为**不应该**是相互的。

**数据交付（Data delivery）** `(type: 1, data: any) => void`

交付次数：

- 一个 callbag（source 或者 sink）**可能**会被一次或多次交付数据

有效交付的窗口：

- 一个 callbag **一定不能**在被问好之前被交付数据
- 一个 callbag **一定不能**在终止后被交付数据
- 一个 sink **一定不能**在其终止了它的 source 后被交付数据

## 创建自己的 callbag

callbag 的组成可以简单归纳为：

- handshake：一次握手过程，source 和 sink 如何握手
- talkback：对讲对象，sink 和 source 正在和谁沟通

### listener（observer）sink

- **定义问好过程**：在问好阶段，可以知道在和谁对讲：

  ```js
  function sink(type, data) {
    if (type === 0) {
      // sink 收到了来自 source 的问好
      // 问好的时候确定 source 和 sink 的对讲方式
      const talkback = data;
      // 3s 后，sink 终止和 source 的对讲
      setTimeout(() => talkback(2), 3000);
    }
  }
  ```

* **定义数据处理过程**

  ```js
  function sink(type, data) {
      if (type === 0) {
          const talkback = data;
          setTimeout(() => talkback(2), 3000);
      }
      if (type === 1) {
          console.log(data);
      }
  }
  ```

* **定义结束过程**

  ```js
  let handle;
  function sink(type, data) {
      if (type === 0) {
          const talkback = data;
          setTimeout(() => talkback(2), 3000);
      }
      if (type === 1) {
          console.log(data);
      }
      if (type === 2) {
          clearTimeout(handle);
      }
  }
  ```

  可以再用工厂函数让代码干净一些：

  ```js
  function makeSink() {
    let handle;
    return function sink(type, data) {
      if (type === 0) {
        const talkback = data;
        handle = setTimeout(() => talkback(2), 3000);
      } 
      if (type === 1) {
        console.log(data);
      }
      if (type === 2) {
        clearTimeout(handle);
      }
    }
  }
  ```

### puller（consumer）sink

puller sink 则可以向 source 主动请求数据：

```js
let handle;
function sink(type, data) {
    if (type === 0) {
        const talkback = data;
        setInterval(() => talkback(1), 1000);
    }
    if (type === 1) {
        console.log(data);
    }
    if (type === 2) {
        clearTimeout(handle);
    }
}
```

### listenable（observable）source

- 定义问好过程：

  ```js
  function source(type, data) {
    if (type === 0) {
      // 如果 source 收到 sink 的问好，
      // 则 payload 即为 sink，source 可以向 sink 发送数据了
      const sink = data;
      setInterval(() => {
        sink(1, null);
      }, 1000);
    }
    // 让 source 也和 sink 问好，完成一次握手
    sink(0, /* talkback callbag here */)
  }
  ```

- 当 sink 想要停止观察，需要让 source 有处理停止的能力，另外，listenable 的 source 不会理会 sink 主动的数据索取。因此，我们这么告知 sink 沟通方式：

  ```js
  function source(type, data) {
      if (type === 0) {
          const sink = data;
          let handle = setInterval(() => {
              sink(1, null);
          }, 1000);
      }
      const talkback = (type, data) => {
          if (type === 2) {
              clearInterval(handle);
          } 
      }
      sink(0, talkback);
  }
  ```

- 优化一下代码可读性：

  ```js
  function source(start, sink) {
    if (start !== 0) return;
    let handle = setInterval(() => {
      sink(1, null);
    }, 1000);
    const talkback = (t, d) => {
      if (t === 2) clearInterval(handle);
    };
    sink(0, talkback);
  }
  ```

### pullable（iterable）source

pullable source 中，值时按照 sink 的需要获取的，因此，只有在 sink 索取值时，source 才需要交付数据：

```js
function source(start, sink) {
    if (start !== 0) retrun;
    let i = 10;
    const talkback = (t, d) => {
        if (t == 1) {
            if (i <= 20) sink(1, i++);
            else sink(2);
        }
    }
    sink(0, talkback)
}
```


## 创建运算子

借助于 operator，能够不断的构建新的 source，operator 的一般范式为：

```js
const myOperator = args => inputSource => outputSource
```

借助于管道技术，我们能一步步的声明新的 source：

```js
pipe(
  source,
  myOperator(args),
  iterate(x => console.log(x))
)
// same as...
pipe(
  source,
  inputSource => outputSource,
  iterate(x => console.log(x))
)
```

下面我们创建了一个乘法 operator：

```js
const multiplyBy = factor => inputSource => {
    return function outputSource(start, outputSink) {
        if (start !== 0) return;
        inputSource(start, (type, data) => {
            if (type === 1) {
                outputSink(1, data * factor);
            } else {
                outputSink(1, data * factor);
            } 
        })
    }
}
```

使用：

```js
function source(start, sink) {
    if (start !== 0) return;
    let i = 0;
    const handle = setInterval(() => sink(1, i++), 3000);
    const talkback = (type, data) => {
        if (type === 2) {
            clearInterval(handle);
        }
    }
    sink(0, talkback);
}

let timeout;
function sink(type, data) {
    if (type === 0) {
        const talkback = data;
        timetout = setTimeout(() => talback(2), 9000);
    }
    if (type === 1) {
        console.log('data is', data);
    }
    if (type === 2) {
        clearTimeout(handle);
    }
}

const newSource = multiplyBy(3)(source);
newSource(0, sink);
```

## 总结

通过 callbag ，我们可以近乎一致的处理**数据源和数据源的消费**：

例如，下面是 listenable 数据源，我们用 `forEach` 消费：

```js
const {forEach, fromEvent, map, filter, pipe} = require('callbag-basics');

pipe(
  fromEvent(document, 'click'),
  filter(ev => ev.target.tagName === 'BUTTON'),
  map(ev => ({x: ev.clientX, y: ev.clientY})),
  forEach(coords => console.log(coords))
);
```

下面则是 pullable 数据源，我们仍可以用 `forEach ` 进行消费：

```js
const {forEach, fromIter, take, map, pipe} = require('callbag-basics');

function* getRandom() {
  while(true) {
    yield Math.random();
  }
}

pipe(
  fromIter(getRandom()),
  take(5),
  forEach(x => console.log(x))
);
```

## 参考资料

- [WHY WE NEED CALLBAGS](https://staltz.com/why-we-need-callbags.html)
- [callbag](https://github.com/callbag/callbag)
- [Creating your own utilities](https://github.com/callbag/callbag/blob/master/getting-started.md)