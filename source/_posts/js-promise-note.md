title: JS Promise Note
date: 2017-04-20 17:50:00
tags:
- JS
- Frontend
---

一直对 Promise 这个概念感到迷迷糊糊，实在是受不了了，决定系统的过一次相关的知识点。
<!--more-->

以下笔记主要基于著名的 [JavaScript 迷你书(中文版)](http://liubin.org/promises-book/)，感谢原作者 @azu 和翻译者 @liubin!

本文将不定期陆续更新。

## API

**Constructor**

```js
let promise = new Promise((resolve, reject) => {
	// async processing...
	// call resolve / reject when done
})
```

**Instance Method**

```js
promise.then(onFulfilled, onRejected)
// OR
promise.then(onFulfilled).catch(onRejected)
```

**Static Method**

`Promise.all()`, `Promise.resolve()`, `Promise.reject()`

## Status

Promise 有三种状态：`has-resolution/Fulfilled`, `has-rejection/Rejected`, `unresolved/Pending`

## 用法

**`Promise.resolve()`**

```js
// 快速新建一个 Promise 对象, 并第一时间调用 then 方法
Promise.resolve(42).then((value) => {
  console.log(value)
})

// 将 Thenable 转换为 Promise 对象
Promise.resolve($.ajax('/json/comment.json')).then((value) => {
  console.log(value)
})
```

同理，**`Promise.reject()`** 会第一时间返回一个 promise 对象（如果有调用 `.catch(onReject)` 则会在下一 tick 中执行 `onReject`

### 使用 Promise 确保异步流程

```js
function onReadyPromise() {
  return new Promise((resolve, reject) => {
    let readyState = document.readyState
    if (readyState === 'interactive' || readyState === 'complete') {
      resolve()
    } else {
      window.addEventListener('DOMContentLoaded', resolve)
    }
  })
}

onReadyPromise().then((value) => {
  console.log('DOM fully loaded and parsed')
})

console.log('==Starting==')
```

### Promise Chain

`.then` 和 `.catch` 方法可以进行链式调用，比如：

```js
let promise = Promise.resolve()
promise
	.then(taskA)
	.then(taskB)
	.catch(onRejected)
	.then(finalTask)
```

在这个链式调用中，taskA 和 taskB 可以通过两种方式使执行流程经过 onRejected 函数：

1. 抛出一个异常
2. 返回一个 `rejected` 状态的 Promise 对象（推荐使用这个方法）

使用第二种方法的理由：

1. 更加直观，因为 `.catch` 方法本来的含义就是在 Promise 对象状态变为 `rejected` 时执行的回调。
2. 避免 `throw` 关键字造成的副作用（影响 debug 等）

在 `then` 中注册的回调函数可以通过 `return` 返回一个值，这个返回值会传给后续的 `then` 或者 `catch` 的回调函数

但 `then` 的结果总是一个新创建的 promise 对象。如果 `then` 中注册的回调函数的返回值就是一个 Promise 对象，则 `then` 的结果就是这个对象。

所以我们可以在 `then` 中返回一个带 `reject` 状态的 Promise 对象：

```js
let onRejected = console.error.bind(console)
let promise = Promise.resolve()

promise.then(() => {
  return Promise.reject(new Error('this promise is rejected'))
}).catch(onRejected)
```

#### Anti-pattern: 对同一个对象同时调用 then 方法

显然这样的处理是不能得到预想中的结果的，必须修改成使用 Promise Chain 的方式。

```js
var aPromise = new Promise(function (resolve) {
    resolve(100);
});
aPromise.then(function (value) {
    return value * 2;
});
aPromise.then(function (value) {
    return value * 2;
});
aPromise.then(function (value) {
    console.log("1: " + value); // => 100
})
```

### 处理 IE8 下 `catch` 保留字的问题

> 在ECMAScript 3中保留字是不能作为对象的属性名使用的。而IE8及以下版本都是基于ECMAScript 3实现的，因此不能将 catch 作为属性来使用，也就不能编写类似 promise.catch() 的代码，因此就出现了 identifier not found 这种语法错误了。

**解决方案**: 使用 `then` 而不使用 `catch`；如果一定要使用 `catch` 的话：

```js
var promise = Promise.reject(new Error('message'))
promise['catch'](function (error) {
  console.log(error)
})
```

### `Promise.all()`

Promise.all 接收 Promise 对象组成的数组作为参数，两个 promise 对象的初始化会同时进行，当所有的 promise 对象的状态转变为 `fulfilled` 或者 `rejected` 之后才会处理 Promise Chain 上的 then 函数，且得到的执行结果的顺序与原 promise 数组的顺序一致。

```js
function timerPromisefy(delay) {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve(delay)
    }, delay)
  })
}

var startDate = Date.now()

// 所有 promise 变为 resolve 后程序退出
Promise.all([
  timerPromisefy(1),
  timerPromisefy(32),
  timerPromisefy(64),
  timerPromisefy(128),
]).then((values) => {
  console.log(Date.now() - startDate + 'ms') // ~128ms
  console.log(values) // [1,32,64,128]
})
```

### `Promise.race()`

`race()` 方法和 `all()` 类似，接收一个 promise 对象数组为参数，但区别在于：

> Promise.race 只要有一个 Promise 对象进入 FulFilled 或者 Rejected 状态的话，就会继续进行后面的处理。

但 race 胜出的 Promise 对象不会阻止其他 Promise 对象的执行。

### `.catch()`

最好的理解方法就是将 catch 方法当做是 `promise.then(undefined, onRejected)`，它们在本质上没有区别。

## Test Promise

* 使用 Mocha 对 Promise 进行测试
* 在 `it()` 中直接返回 Promise 对象，则不需要使用 `done`
* 测试 Promise 对象时，应该覆盖 Promise 对象的两种状态（Fulfilled, Rejected），同时检查两种状态时的返回值。满足这样条件的测试叫做可控测试（controllable tests）

定义一个叫做 `shouldRejected` 的函数，用于测试期待返回状态为 `onRejected` 的 Promise：

```js
let assert = require('assert')
function shouldRejected (promise) {
  return {
  	catch () {
  	  return promise.then(() => {
  	    throw new Error('Expected promise to be rejected but it was fulfilled')
  	  }, (reason) => {
  	    fn.call(promise, reason)
  	  })
  	}
  }
}

it('should be rejected', () => {
  let promise = Promise.reject(new Error('human error')
  return showRejected(promise).catch((error) => {
    assert(error.message === 'human error')
  })
})
```

同理可以写一个 `shouldFulfilled` 的 helper...

## Advanced

一些实现了 Promise 的第三方类库：

#### Polyfills

* [stefanpenner/es6-promise](https://github.com/stefanpenner/es6-promise)
* [getify/native-promise-only](https://github.com/getify/native-promise-only/)

#### Promise Extensions

* [kriskowal/q](https://github.com/kriskowal/q)
* [petkaantonov/bluebird](https://github.com/petkaantonov/bluebird)

### Thenable

* Thenable 就是一个具有 `.then()` 方法的一个对象。
* 通过 `Promise.resolve()` 可以将一个 Thenable 对象转换为一个标准的 Promise 对象
* 很多第三方库提供了将 Thenable 对象转换为其实现的 Promise 对象的途径。所以在内部使用 Thenable，便于在不同的 Promise 类库之间进行相互转换。

实现一个 Thenable 对象非常简单：

```js
let thenable = {
	then (resolve, reject) => {
		// call resolve when everything is okay

		// call reject when things go wrong
	}
}
```

### Deferred & Promise

#### Deferred 和 Promise 的关系

![](http://liubin.org/promises-book/Ch4_AdvancedPromises/img/deferred-and-promise.png)

* Deferred 拥有 Promise
* Deferred 具备对 Promise 状态进行操作的特权方法

#### 使用 Promise 实现 Deferred

```js
// using ES5 style
function Deferred() {
	this.promise = new Promise(function (resolve, reject) {
		this._resolve = resolve
		this._reject = reject
	}.bind(this))
}

Deferred.prototype.resolve = function (value) {
	this._resolve.call(this.promise, value)
}

Deferred.prototype.reject = function (reason) {
	this._reject.call(this.promise, reason)
}

// 实现一个 getURL 函数
function getURL(URL) {
	var deferred = new Deferred();
    var req = new XMLHttpRequest();
    req.open('GET', URL, true);
    req.onload = function () {
        if (req.status === 200) {
            deferred.resolve(req.responseText);
        } else {
            deferred.reject(new Error(req.statusText));
        }
    };
    req.onerror = function () {
        deferred.reject(new Error(req.statusText));
    };
    req.send();
    return deferred.promise;
}

// 运行实例
var URL = "http://httpbin.org/get";
getURL(URL).then(function onFulfilled(value){
    console.log(value);
}).catch(console.error.bind(console));
```

这样写的好处有：

* 减少一层缩进
* 不需要一开始就将处理流程写成一大段代码，只需要先创建 deferred 对象，在任何时机调用 `resolve`, `reject` 方法。

> 如果说Promise是用来对值进行抽象的话，Deferred则是对处理还没有结束的状态或操作进行抽象化的对象，我们也可以从这一层的区别来理解一下这两者之间的差异。
>
> 换句话说，Promise代表了一个对象，这个对象的状态现在还不确定，但是未来一个时间点它的状态要么变为正常值（FulFilled），要么变为异常值（Rejected）；而Deferred对象表示了一个处理还没有结束的这种事实，在它的处理结束的时候，可以通过Promise来取得处理结果。

### 实现超时机制: `Promise.race()`

以下这段代码实现了一个简单的超时函数，当目标 Promise 中的任务在超过 ms 后未执行完（状态未变更），则由和其竞争的 `timeout` promise 抛出一个异常，从而调起后续 Promise Chain 中的 catch 方法。

```js
function delayPromise(ms) {
    return new Promise(function (resolve) {
        setTimeout(resolve, ms);
    });
}
function timeoutPromise(promise, ms) {
    var timeout = delayPromise(ms).then(function () {
    	// 也可以用 reject
    	throw new Error('Operation timed out after ' + ms + ' ms');
    });
    return Promise.race([promise, timeout]);
}

// 运行示例
var taskPromise = new Promise(function(resolve){
    // 随便一些什么处理
    var delay = Math.random() * 2000;
    setTimeout(function(){
        resolve(delay + "ms");
    }, delay);
});
timeoutPromise(taskPromise, 1000).then(function(value){
    console.log("taskPromise在规定时间内结束 : " + value);
}).catch(function(error){
    console.log("发生超时", error);
});
```

不过这里还有一个问题，就是如果业务 promise 在执行过程中出现了问题，抛出一个错误（或者调用 reject），那么在 Promise Chain 后续的 catch 函数中，其实我们无法分辨到底是系统超时了还是业务 promise 出现了问题。当然，检查 `error.msg` 具体的字符串值是可以勉强做到的，但这样的实现非常的不美观。

一种理想的方案是自定义一个 `TimeoutError` 类型的对象，通过检查 `error instanceof TimeoutError` 来判断捕获到的错误是否为一个超时错误。

使用 ES6 规范里面的 `class`, `extend` 自然是轻轻松松，但也无妨看下在 ES5 下的实现方案。

#### 插播：创建一个继承 Error 的类 TimeoutError

```js
// TimeoutError.js

function copyOwnFrom (target, source) {
    Object.getOwnPropertyNames(source).forEach(function (propName) {
        Object.defineProperty(target, propName, Object.getOwnPropertyDescriptor(source, propName));
    });
    return target;
}
function TimeoutError () {
    var superInstance = Error.apply(null, arguments);
    copyOwnFrom(this, superInstance);
}
TimeoutError.prototype = Object.create(Error.prototype);
TimeoutError.prototype.constructor = TimeoutError;
```

另一种思路，来源于 CoffeeScript 的实现：

```js
var __hasProp = {}.hasOwnProperty
var __extends = function (Child, Parent) {
	// 复制构造器上的属性
	for (var key in Parent) {
		if (__hasProp.call(Parent, key)) Child[key] = Parent[key]
	}

	// 构建原型链
	function ctor() { this.constructor = Child }
	ctor.prototype = Parent.prototype
	Child.prototype = new ctor()

	Child.__super__ = Parent.prototype

	return Child
}

var TimeoutError = (function (_super) {
	function TimeoutError () {
		_super.call(this, arguments)
	}

	__extends(TimeoutError, _super)

	return TimeoutError
})(Error)
```

在实现了简单的超时之后，我们希望能够在 XHR 超时后取消其请求操作（以免阻塞后面可能的 XHR 请求），需要用到 `xhr.abort()` 方法。

对上一节中实现的 `getURL` 函数稍加改进，改成返回一个带 promise 和 abort 方法的对象，配合 `timeoutPromise` 方法就可以完成整个业务逻辑。

### `Promise.prototype.done`

> 如果你使用过其他的 Promise 实现类库的话，可能见过用 done 代替 then 的例子。

> 这些类库都提供了 Promise.prototype.done 方法，使用起来也和 then 一样，但是这个方法并不会返回 Promise 对象。

> 虽然 ES6 Promises 和 Promises/A+ 等在设计上并没有对 Promise.prototype.done 做出任何规定，但是很多实现类库都提供了该方法的实现。

* `done` 不返回 Promise 对象
* `done` 发生的异常会直接抛到外面

**在开发中，如果忘记编写 catch 函数处理 Promise Chain 中运行时错误，那么这些错误会被 "内部消化"，而不会被外部所得知，这样就给 debug 造成了巨大的困难。使用 done 的意义在于避免这样的情况。**

**在 setTimeout 中抛出一个异常并不会被捕获!!!**

```js
// 这个 error 不会被捕获
try {
    setTimeout(function callback() {
        throw new Error("error");
    }, 0);
} catch (error) {
    console.error(error);
}
```

### Promise & method chain

TODO

### Promise & sequence

TODO

## Reference

### Promise

* [JavaScript Promise 迷你书](http://liubin.org/promises-book/)

---

* [Promise Objects - ECMAScript Language Specification](https://tc39.github.io/ecma262/#sec-promise-objects)
* [Writing Promise - Using Specifications // W3C](https://www.w3.org/2001/tag/doc/promises-guide)
* [JavaScript Promises - Thinking Sync in an Async World // Speaker Deck](https://speakerdeck.com/kerrick/javascript-promises-thinking-sync-in-an-async-world)
* [JavaScript Promise // Google Web Developer](https://developers.google.com/web/fundamentals/getting-started/primers/promises)
* [You're Missing the Point of Promises](https://blog.domenic.me/youre-missing-the-point-of-promises/)
* [es6-promise: A polyfill for ES6-style Promises](https://github.com/stefanpenner/es6-promise)
* [Promise Anti-patterns](http://taoofcode.net/promise-anti-patterns/)
* [Promises/A+](https://promisesaplus.com/)
* [You're Missing the Point of Promises](https://blog.domenic.me/youre-missing-the-point-of-promises/)
* [JavaScript 异步编程的 Promise 模式](http://www.infoq.com/cn/news/2011/09/js-promise)

### Deferred

* [Promise & Deferred objects in JavaScript Pt.1: Theory and Semantics.](http://blog.mediumequalsmessage.com/promise-deferred-objects-in-javascript-pt1-theory-and-semantics)
* [The Deferred Anti-Pattern // petkaantonov/bluebird Wiki](https://github.com/petkaantonov/bluebird/wiki/Promise-anti-patterns#the-deferred-anti-pattern)
* [Coming from jQuery // kriskowal/q Wiki](https://github.com/kriskowal/q/wiki/Coming-from-jQuery)