title: 谈谈 Observable（上）
date: 2018-05-13
tags:
- JavaScript
---

因为工作中的某个 ng2 的项目中使用到了 Observable（具体地说，是 ng2 http 模块中的请求方法的返回结果正是 Observable），所以做一些简单的学习，在此记录。

由于篇幅太大，本篇先讲述 Observable 的基本概念，~~（如果有时间的话）就继续写更复杂一些的内容~~

<!--more-->

### 什么是 Observable

#### tl;dr

简单的说：

> Observable 是一个可被订阅的对象，该对象将随着时间推移发送有限或无限个值供其订阅者消费。

在这个说法中 Observable 有两个特征：

1. 它是可被订阅的 (Pub/Sub)
2. 它的值是一个有限/无限的队列

以下将分别说明这两点。

#### 对比 EventEmitter

提到订阅，我们自然会想起经典的 Pub/Sub 模式（也有叫观察者模式），在 JS 中的实现就是 `EventEmitter`：

```js
// in Node.js v9.x
const EventEmitter = require('events')
let ev = new EventEmitter()
ev.on('foo', function (msg) {
    console.log(`I got you, ${msg}!`)
})

ev.on('foo', function (msg) {
    console.log(`I got you too, ${msg}!`)
})

// 在后续代码的任何位置:
ev.emit('foo', 'a monster')
```

在上述的代码片段中，我们创建了一个 EventEmitter 对象，并向该对象注册了一个事件和对应的监听函数。在后续的程序代码中，我们可以在任何位置，让 EventEmitter 对象触发（`emit`）一个事件，从而唤醒该事件关联的监听函数（可以是多个）。浏览器中的 DOM 元素也实现了类似 EventEmitter 的特性。

和 EventEmitter 一样，Observable 可以实现基本的发布订阅：

```js
import Observable from 'rxjs'
const source = new Observable(observer => {
  observer.next(1)
  observer.next(2)
})
source.subscribe(number => console.log(number))
```

而在实际的需求中，我们可能要处理一个 **队列的事件触发**，且队列可能不是有限的（如 IM 消息，弹幕，用户在页面上的操作），我们需要从代码组织层面上提供更加方便的处理方式，这就引出了 Observable。

比如我们用 Observable 处理 WebSocket 数据：

```js
import Observable from 'rxjs'
const socket = new WebSocket('ws://someurl')
const source = new Observable(observer => {
  socket.addEventListener('message', e => {
    observer.next(e) 
  })
  return () => socket.close()
})

// 打印 WebSocket 发送过来的数据
// 假设接收的数据都为数字, 对其中所有的偶数X输出 "this number is X"
let sub = source
  .filter(number => number % 2 === 0)
  .map(number => `this number is ${number}`).subscribe(message => {
    // 事实上可以对 message 做任何的处理
    console.log(message)
  })

// 在代码的任意位置
sub.unsubscribe()
```

在例子中我们新建并监听一个 WebSocket 连接，并将收到的信息进行处理。将监听逻辑封装成一个 Observable，让我们可以在后续使用 `.map()` 等操作符，来对收到的数据进行处理，并最后用 `subscribe()` 完成订阅。

#### Stage-1 提案

Observable 目前是 ECMAScript 的新提案 ([Stage-1](https://github.com/tc39/proposal-observable))。在 ECMAScript 的提案中，Observable 的定义如下：

> The **Observable** type can be used to model push-based data sources such as DOM events, timer intervals, and sockets. In addition, observables are:
> * *Compositional*: Observables can be composed with higher-order combinators
> * *Lazy*: Observables do not start emitting data until an **observer** has subscribed.

简单翻译如下：

> **Observable** 类型可以用于表示基于推送的数据源模型，例如 DOM 事件，计时器，或者 socket。此外，Observable 还具有（以下特征）：
> * *可组合的*: Observable 可以使用高阶的连接符进行拼接组合
> * *惰性*: Observable 仅当一个 **observer** 订阅时才会开始发送数据。

文章的后续会解释这两个属性，在这里读者可以先跳过概念部分，或者大概有个印象就可以了。

当前对 Observable 比较流行的实现有 [RxJS](https://github.com/ReactiveX/rxjs), [Bacon.js](https://baconjs.github.io/), [zen-observable](https://github.com/zenparsing/zen-observable) 等。接下来本文将基于 RxJS 中的实现来介绍 Observable 的基本概念。

### 创建和订阅

有别于固定长度的数组，Observable 的值是随时间发送的一连串的值，像水流一样，所以也有说法称 Observable 是一个流（Stream），为了更好的直观理解所谓的 “流”，首先我们来了解 Observable 的一种表示方法：*Marble Diagram*。

#### Marble Diagram

Marble Diagram 由两个部分组成：**timeline** 和 **item**。timeline 表示时间轴，item 表示在时间轴上触发的元素（类型可以是任意的）。下图表示一个事件流先后触发（emit）了三个 item，最后成功结束，出现错误的符号为叉：

![](https://cdn-images-1.medium.com/max/1600/1*brbCs4smjZfqitE0kHSHTQ.png)

我们也可以用类似 ASCII 的绘画方式来表达 Marble Diagram：

```js
// 用 - 表示一小段时间(可以理解为一个时间单位，如秒)，串起来表示一个时间轴，若某个时间中有发送值的话则用具体的值代替
---3-----2-----1----0---

// 小括号代表着 Observable 是同步送值
---(123)-----2-----1----0---
  
// X 表示有错误发生
-----------------------X

// | 表示 Observable 结束
-----------------------|
```

接下来我们将了解 RxJS 中 Observable 相关的 API，其中将会用 Marble Diagram 的 ASCII 绘画来表示 Observable。如果有不太明白的地方，可以使用 [RxViz](https://rxviz.com/) 将代码实际运行一下，观察其 Marble Diagram 的具体形态。

#### 创建和订阅

在上文中我们知道可以用 `Observable` 构造函数直接初始化一个 Observable 实例；RxJS 还提供了相同效果的静态方法 `Observable.create`：

```js
let observable = Rx.Observable.create(observer => {
  observer.next('Jerry')
  observer.next('Anna')
  observer.complete()
  observer.next('xxx') // don't work
})
```

在创建了 Observable 之后，可以通过 `subscribe` 方法订阅该 Observable：

```js
observable.subscribe({
  next (value) {
    // 处理 Observable 发出的值
  },
  error (err) {
    // 当 Observable 出错时执行
  },
  complete () {
    // 当 Observable 正常结束时执行（状态为 completed）
  }
})

// 另一种语法:
observable.subscribe(value => {
  // next 方法
}, err => {
  // err 方法
}, () => {
  // complete 方法
})
```

如上述代码所示，调用 `subscribe` 方法时我们传入了一个对象，该对象我们可以称为 **观察者（Observer）**。

观察者具有三个方法，每当 Observable 发生事件时便会呼叫观察者相对应的方法：

* `next`: 每当 Observable 发送新的值，触发该方法
* `complete`: 当 Observable 不再获得新的值时，complete 方法就会被触发，该方法被触发后，next 方法将不会再起作用。
* `error`: 每当 Observable 内发生错误时，error 方法被触发。

可以查看另一个 [观察者的例子](https://gist.github.com/allenfantasy/340d1237180440a20551a532dd632ff6#file-basic-js-L10)

和 EventEmitter 不同，Observable 在内部没有一个订阅者清单，订阅 Observable 的行为实际上是执行一个函数，这个函数接收一个 **Observer 对象** 并在函数体内触发 Observer 对象的方法（next, complete, error），也就是说，对于某个 Observable，其在构建时传入的回调函数，必须要在该 Observable 被订阅之后，才会调用执行。可以看一下这个例子：

```js
let ob = Rx.Observable.create(observer => {
  console.warn('now we start calling observer')
  observer.next(1)
})

// 必须要订阅之后才会执行
ob.subscribe(value => {
  console.warn('1st subscribe', value)
})

// another subscribe
ob.subscribe(value => {
  console.warn('2nd subscribe', value)
})

//=== output:
// now we start calling observer
// 1st subscribe 1
// now we start calling observer                                
// 2nd subscribe 1 
```

执行这段代码我们会发现，Observable 构造方法的回调函数实际上被调用了两次，这是因为这个 Observable 有两个订阅者，且回调函数是在 subscribe 时才被触发的。 如果我们将代码片段中 `subscribe` 的语句注释掉，执行时不会有任何输出。

在 RxJS 中，`subscribe` 方法会返回一个类型为 `Subscription` 的对象，可以用对象的 `unsubscribe` 方法可以停止对 observable 对象的监听（订阅）。

### Creation Operator

除了用 `Observable.create` 方法之外，RxJS 还提供了很多便捷的创建 Observable 的 API，我们统称为 *creation operator*，其中包括：

* `of`: `of(1,2,3,4)`
* `from`: `from([1,2,3,4])`
* `fromEvent`
* `fromPromise`
* `never`: 永远不会结束但什么都不做的
* `empty`: 空的且立刻结束的
* `throw`: 抛出错误
* `interval`
* `timer`

利用这些 operator 我们可以快速实现一些常见的功能，如点击监听事件：

```js
var el = document.getElementById('target')

Rx.Observable.fromEvent(el, 'click').subscribe(e => {
  // handle click events...
})
```

或者是数数：

```js
// 每秒钟在控制台输出一个数字, 从0开始每次+1
Rx.Observable.interval(1000).subscribe(number => {
  console.log(number)
})

// 等1秒后送出0, 然后每隔5秒送出一个值(1,2,3,4..)
Rx.Observable.timer(1000, 5000).subscribe(number => {
  console.log(number)
})
```

### Transform Operator

前面提到，Observable 可以发送有限个或无限个值，我们可以将一定时间内 Observable 发出的值看做是一个数组，那么对这些值我们可以应用数组的所有方法如 `map()`, `filter()`, `pluck()` 等。事实上 Observable 确实提供了一系列的操作符（operator），允许我们链式调用。

Observable 本质上就是表示随时间发展而不断发送的一系列的值（流），我们可以像对待数组一样去对 Observable 进行操作，这样的操作方式，我们称为 Transform Operator。Transform Operator 可以分为几类（我的理解）：

1. 处理单个流的:
   * 简单的队列映射: `map`, `pluck`, `filter`, `scan`, `reduce`, `take`, `first`, `distinctUntilChanged` ...
   * 和时序有关的: `debounce`, `debounceTime`, `throttle`, `throttleTime`
2. 处理多个流之间关系的: `merge`, `concat`, `combineLatest`, `zip`, `withLatestFrom`
3. 降维的(源 observable 所释放的每个值又是一个 observable): `concatAll`, `megeAll`, `combineAll`, `switch`
4. 映射+降维(源 observable 通过映射生成一个二维的 observable, 然后再降维): `concatMap`, `mergeMap`, `switchMap`
5. 其他: `catch`, `every`, `defaultEmpty`, `sequenceEqual`, `delay` 等

分类方法还有一种是按照 [RxJS Marbles](http://rxmarbles.com/) 中的分类。有兴趣的读者也可以查看。

在介绍具体的 operator 之前，首先我们先看 operator 的运作方式。

##### operators 运作方式

和数组的 operators 相比，Observable 的 operators 有两个特点：

* 延迟运算：只有在 observable 被订阅时，operators 才开始对元素进行运算
* 渐进式取值：

  > 每次的运算是一个元素运算到底，而不是将全部元素运算完再返回

数组的取值方式：

![](http://i.giphy.com/l0HlPZeB9OvFu7QwE.gif)

Observable 的取值方式：

![](http://i.giphy.com/3o6ZtqrBfUyHvMDQ2c.gif)

为了理解方便，以下介绍 operator 时会使用 ASCII Marble Diagram。

##### 处理单个流

**`map`**

```js
let source = Rx.Observable.interval(1000)
let newest = source.map(x => x + 2)

source: ----0----1----2----3---...
         map(x => x + 2)
newest: ----2----3----4----5---...
```

**`take`**

```js
let example = Rx.Observable.interval(1000).take(3)
// TODO subscribe it ..

source: ----0----1----2----3---...
         take(3)
newest: ----0----1----2|
```

**`distinctUntilChanged`**

如果要发送的元素和最后一次发送的元素相同，则过滤掉该元素

```js
let source = Rx.Observable.from(['a', 'b', 'c', 'c', 'b'])
let example = source.distinctUntilChanged()

example.subscribe({
  next: (value) => { console.log(value) },
  error: (err) => { console.log('Error: ', err) },
  complete: () => { console.log('complete') }
})

// Result:
// a
// b
// c
// b
// complete
```

##### 处理多个流之间的关系

**`merge`**

合并 observable，在时序上两个 observable 同时执行，但当两个 observable 同时触发元素时，被 merge 的 observable 所触发的元素在后面。

`merge` 的逻辑有点像 OR，就是当两个 observable 其中一个被触发时都可以被处理。

例子：

```js
let source = Rx.Observable.interval(500).take(3)
let source2 = Rx.Observable.interval(300).take(6)
let example = source.merge(source2)

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
})

/* Marble Diagram
source : ----0----1----2|
source2: --0-1---2--3--4--5|
            merge()
example: --0-(01)--21-3--(24)--5|
*/
```

**`concat`**

`concat` 可以把多个 observable 合并成一个：

```js
var source = Rx.Observable.interval(1000).take(3);
var source2 = Rx.Observable.of(3)
var source3 = Rx.Observable.of(4,5,6)
var example = source.concat(source2, source3);

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
// 0
// 1
// 2
// 3
// 4
// 5
// 6
// complete

/* Marble Diagram
source : ----0----1----2|
source2: (3)|
source3: (456)|
            concat()
example: ----0----1----2(3456)|
*/
```

##### 降维操作

和数组类似，Observable 也可能出现类似二维数组这样的 "高维" 情况，即 Observable 中所发出的每项元素又是一个单独 Observable。RxJS 提供了一系列的 API 允许我们将其转换为普通的 "一维 Observable"。

**`concatAll`**

比如对应数组中 `concat` 操作，RxJS 也有 `concatAll` operator，会将所有的 Observable "拼接" 起来：

```js
let obs1 = Rx.Observable.interval(1000).take(5) // 0, 1, 2, 3, 4
let obs2 = Rx.Observable.interval(500).take(2)
let obs3 = Rx.Observable.interval(2000).take(1)

let source = Rx.Observable.of(obs1, obs2, obs3)

let example = source.concatAll()

example.subscribe({
    next (value) => { console.log(value) },
    error (err) => { console.log('Error: ' + err) },
    complete () => { console.log('complete') }
})
/* Marble Diagram

obs1  : ----0----1----2----3----4|
obs2  : --0--1|
obs3  : --------0| 
source: (o1,                           o2,        o3)
          \                             \          \
           ----0----1----2----3----4|    --0--1|    --------0|
           
                       concatAll()
                       
example: ----0----1----2----3----4--0--1--------0|
*/

let source = Rx.Observable.of(1,2,3,4).map(number => {
  let req = await fetch(`/search?${number}`)
  return Rx.Observable.fromPromise(httpReq1)
})
```

观察上面的 Marble Diagram，我们可以发现，`concatAll` 将 source 中的三个 Observable 按顺序拼接起来依次输出。

**`switch`**

swtich 总是会将返回的 Observable 的 "控制权" 交给原 Observable 中 **最近返回** 的一个 Observable。

尝试理解下这个例子：

<iframe style="width:100%;height:400px;" src="https://stackblitz.com/edit/observable-switch?embed=1"></iframe>

再看看这个例子对应的 [Marble Diagram](https://rxviz.com/v/7Jag5mwo)：

<svg width="562" height="167" style="display: block; font-size: 14px; font-family: Arial, sans-serif; dominant-baseline: central; text-anchor: middle; cursor: default; user-select: none;"><line x1="21" y1="37" x2="21" y2="89" stroke="#767676" stroke-width="1" stroke-dasharray="3,3"></line><line x1="153.8031" y1="37" x2="153.8031" y2="141" stroke="#767676" stroke-width="1" stroke-dasharray="3,3"></line><g transform="translate(0, 11)"><!-- react-empty: 11 --><g transform="translate(21, 0)"><line x1="0" y1="26" x2="531" y2="26" stroke-width="2" stroke="#000000" style="shape-rendering: crispEdges;"></line><path transform="translate(531, 21)" d="M0 0 L10 5 L0 10 z" fill="#000000" style="transition: fill 0.2s ease-in-out;"></path><!-- react-empty: 16 --><g><g style="transform: translate(0px, 26px) scale(1); transition: transform 0.5s ease-in-out;"><circle cx="0" cy="0" r="15" stroke-width="0" stroke="#000000" fill="#767676"></circle></g><g style="transform: translate(132.803px, 26px) scale(1); transition: transform 0.5s ease-in-out;"><circle cx="0" cy="0" r="15" stroke-width="0" stroke="#000000" fill="#767676"></circle></g></g><!-- react-empty: 20 --></g></g><g transform="translate(0, 63)"><!-- react-empty: 22 --><g transform="translate(21, 0)"><line x1="0" y1="26" x2="531" y2="26" stroke-width="2" stroke="#767676" style="shape-rendering: crispEdges;"></line><path transform="translate(531, 21)" d="M0 0 L10 5 L0 10 z" fill="#767676" style="transition: fill 0.2s ease-in-out;"></path><!-- react-empty: 27 --><g><g style="transform: translate(53.2062px, 26px) scale(1); transition: transform 0.5s ease-in-out;"><circle cx="0" cy="0" r="15" stroke-width="2" stroke="#767676" fill="#ffffff"></circle><text x="0" y="0" style="fill: rgb(118, 118, 118); dominant-baseline: central;">0</text></g><g style="transform: translate(106.625px, 26px) scale(1); transition: transform 0.5s ease-in-out;"><circle cx="0" cy="0" r="15" stroke-width="2" stroke="#767676" fill="#ffffff"></circle><text x="0" y="0" style="fill: rgb(118, 118, 118); dominant-baseline: central;">1</text></g><g style="transform: translate(159.831px, 26px) scale(1); transition: transform 0.5s ease-in-out;"><circle cx="0" cy="0" r="15" stroke-width="2" stroke="#767676" fill="#ffffff"></circle><text x="0" y="0" style="fill: rgb(118, 118, 118); dominant-baseline: central;">2</text></g><g style="transform: translate(213.25px, 26px) scale(1); transition: transform 0.5s ease-in-out;"><circle cx="0" cy="0" r="15" stroke-width="2" stroke="#767676" fill="#ffffff"></circle><text x="0" y="0" style="fill: rgb(118, 118, 118); dominant-baseline: central;">3</text></g><g style="transform: translate(266.403px, 26px) scale(1); transition: transform 0.5s ease-in-out;"><circle cx="0" cy="0" r="15" stroke-width="2" stroke="#767676" fill="#ffffff"></circle><text x="0" y="0" style="fill: rgb(118, 118, 118); dominant-baseline: central;">4</text></g><g style="transform: translate(319.821px, 26px) scale(1); transition: transform 0.5s ease-in-out;"><circle cx="0" cy="0" r="15" stroke-width="2" stroke="#767676" fill="#ffffff"></circle><text x="0" y="0" style="fill: rgb(118, 118, 118); dominant-baseline: central;">5</text></g><g style="transform: translate(372.974px, 26px) scale(1); transition: transform 0.5s ease-in-out;"><circle cx="0" cy="0" r="15" stroke-width="2" stroke="#767676" fill="#ffffff"></circle><text x="0" y="0" style="fill: rgb(118, 118, 118); dominant-baseline: central;">6</text></g><g style="transform: translate(426.287px, 26px) scale(1); transition: transform 0.5s ease-in-out;"><circle cx="0" cy="0" r="15" stroke-width="2" stroke="#767676" fill="#ffffff"></circle><text x="0" y="0" style="fill: rgb(118, 118, 118); dominant-baseline: central;">7</text></g><g style="transform: translate(479.546px, 26px) scale(1); transition: transform 0.5s ease-in-out;"><circle cx="0" cy="0" r="15" stroke-width="2" stroke="#767676" fill="#ffffff"></circle><text x="0" y="0" style="fill: rgb(118, 118, 118); dominant-baseline: central;">8</text></g></g><!-- react-empty: 29 --></g></g><g transform="translate(0, 115)"><!-- react-empty: 41 --><g transform="translate(21, 0)"><line x1="132.8031" y1="26" x2="531" y2="26" stroke-width="2" stroke="rgba(118, 118, 118, 0.2)" style="shape-rendering: crispEdges;"></line><line x1="132.8031" y1="26" x2="319.662" y2="26" stroke-width="2" stroke="#767676" style="shape-rendering: crispEdges;"></line><path transform="translate(531, 21)" d="M0 0 L10 5 L0 10 z" fill="rgba(118, 118, 118, 0.2)" style="transition: fill 0.2s ease-in-out;"></path><line x1="319.662" y1="3.5" x2="319.662" y2="48.5" stroke-width="2" stroke="#767676" style="opacity: 1; transition: opacity 0.5s ease-in-out;"></line><g><g style="transform: translate(170.132px, 26px) scale(1); transition: transform 0.5s ease-in-out;"><circle cx="0" cy="0" r="15" stroke-width="2" stroke="#767676" fill="#ffffff"></circle><text x="0" y="0" style="fill: rgb(118, 118, 118); dominant-baseline: central;">0</text></g><g style="transform: translate(207.568px, 26px) scale(1); transition: transform 0.5s ease-in-out;"><circle cx="0" cy="0" r="15" stroke-width="2" stroke="#767676" fill="#ffffff"></circle><text x="0" y="0" style="fill: rgb(118, 118, 118); dominant-baseline: central;">1</text></g><g style="transform: translate(244.844px, 26px) scale(1); transition: transform 0.5s ease-in-out;"><circle cx="0" cy="0" r="15" stroke-width="2" stroke="#767676" fill="#ffffff"></circle><text x="0" y="0" style="fill: rgb(118, 118, 118); dominant-baseline: central;">2</text></g><g style="transform: translate(282.226px, 26px) scale(1); transition: transform 0.5s ease-in-out;"><circle cx="0" cy="0" r="15" stroke-width="2" stroke="#767676" fill="#ffffff"></circle><text x="0" y="0" style="fill: rgb(118, 118, 118); dominant-baseline: central;">3</text></g><g style="transform: translate(319.45px, 26px) scale(1); transition: transform 0.5s ease-in-out;"><circle cx="0" cy="0" r="15" stroke-width="2" stroke="#767676" fill="#ffffff"></circle><text x="0" y="0" style="fill: rgb(118, 118, 118); dominant-baseline: central;">4</text></g></g><!-- react-empty: 48 --></g></g><g style="text-anchor: start; dominant-baseline: text-before-edge;"></g></svg>

##### 映射+降维

为了更加方便操作，RxJS 还提供了一些复合 operator，可以同时完成映射（成一个 Observable）和降维的操作。

**`switchMap`**

```js
var source = Rx.Observable.fromEvent(document.body, 'click');

// map + switch
var example = source.switchMap(
  e => Rx.Observable.interval(100).take(3)
);
                
example.subscribe({
  next: (value) => { console.log(value); },
  error: (err) => { console.log('Error: ' + err); },
  complete: () => { console.log('complete'); }
});
/** Marble Diagram
source : -----------c--c------------------...
        concatMap(c => Rx.Observable.interval(100).take(3))
example: -------------0-1-2-0-1-2---------...
*/
```

##### 其他 operator

**`catch`**

`catch` 可以处理 observable 处理过程中出现的异常，可以通过返回一个新的 observable 来发送新的值：

```js
let source = Rx.Observable.from(['a', 'b', 'c', 'd', 2, '...'])
    .zip(Rx.Observable.interval(500), (x, y) => x)

let example = source
                .map(x => x.toUpperCase())
                .catch(error => Rx.Observable.of('h', 'e', 'l', 'l', 'o'))

example.subscribe({
    next: (value) => { console.log(value) },
    error: (err) => { console.log('Error: ' + err) },
    complete: () => { console.log('complete') }
})
```

这段代码对应的 Marble Diagram 是这样的：

```
source : ----a----b----c----d----2|
        map(x => x.toUpperCase())
         ----a----b----c----d----X|
        catch(error => Rx.Observable.of('h'))
example: ----a----b----c----d----h----e----l----l---o|
```

可以在 `catch` 的回调函数中，通过返回一个空的 observable（如：`Rx.Observable.empty()`）来让原有的 observable 结束。

### Practical Example

为了让读者更加直观理解 Observable 的具体使用，来几个例子：这里要鸣谢 "30天精通 RxJS" 教程的作者 @jerryhong，以下例子均出自他的教程。

* [拖拽方块](https://jsfiddle.net/c5azpk87/)
* [类似 Youku 的滚动视频窗口+可拖拽效果](https://jsfiddle.net/s6323859/ochbtpk5/3/)
* [简单加减](https://jsfiddle.net/s6323859/yf02gt9j/1/)
* [AutoComplete](https://jsbin.com/quvajux/1/edit?js,console,output): 使用了 `debounceTime`, `switchMap`, `filter`, `map`, `fromEvent`

### Recap

在了解了 RxJS 和实践了几个例子之后，我们对 Observable 有一个基本的认识：

* 从编程思想上来说，Observable 可以说是 `Reactive Programming` 和 `Functional Programming` 两种思想的结合，关于两种思想读者可以自行查阅。
* 从内容上看，Observable 实现了随时间变化的队列数据的发布订阅。
* 从使用场景上看，Observable 适合处理需要结合多个事件源（如：DOM事件）的复杂逻辑（应用适合的 operator），也适合处理弹幕，IM 聊天等无限数据流的需求。ng2 中大量的使用了 Observable 来管理其内部的 UI 状态。在下篇中我会提到这部分的内容 ~~（挖了个坑，逃~~

### Reference

* [Understanding Marble Diagrams for Reactive Streams](https://medium.com/@jshvarts/read-marble-diagrams-like-a-pro-3d72934d3ef5) 注: 文章中使用的还是 Rx 4.x 的语法
* [RxMarbles](http://rxmarbles.com/)
* [30天精通 RxJS](https://ithelp.ithome.com.tw/users/20103367/ironman/1199)
* [rxjs document](http://reactivex.io/rxjs/)
* https://rxjs-cn.github.io/learn-rxjs-operators/operators/transformation/switchmap.html
* [Becoming more reactive with RxJS flatMap and switchMap](https://medium.com/@w.dave.w/becoming-more-reactive-with-rxjs-flatmap-and-switchmap-ccd3fb7b67fa)
* https://angular.io/guide/observables
* https://medium.com/@luukgruijs/understanding-creating-and-subscribing-to-observables-in-angular-426dbf0b04a3
* https://blog.thoughtram.io/angular/2016/01/06/taking-advantage-of-observables-in-angular2.html
* [Comprehensive Introduction to @ngrx/store](https://gist.github.com/btroncone/a6e4347326749f938510#projecting-data)
* [@ngrx/store](https://github.com/ngrx/store)