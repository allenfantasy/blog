title: co 源码分析
tags:
- JavaScript
---

心血来潮看了下 co 的代码，两百来行并不算多，简单的做个分析。

<!--more-->

### tl;dr

对于没时间看详解的同学，只需要知道这个事实：

> co 遍历执行一个 generator 函数，并返回了一个 Promise 对象，在 generator 函数执行结束后返回的 Promise 对象的状态将变成 `resolved`。

以及：

> Promise 链中，当链中反复在 `.then()` 方法中返回新的 Promise 对象，且最外围的 Promise 的状态一直保持在 `pending` 时，会造成内存泄漏的问题。

又及：

> 没事不要乱看规范...真的够烧脑的...2333

### brief

实际上 co 的代码早已不是寥寥几行了（可能一开始是），但通篇下来其实也就是两百多行的代码，但功能上已经非常完备了。

co 具体做的事情：

1. 接受一个 generator 作为输入，输出一个 Promise 对象
2. 遍历整个 generator（即不断的调用 next）
* 在遍历结束时（即 next 返回的对象 `done: false`）进行 resolve，resolve 所持有的值是最后一个 next 输出的 `value`
* 在遍历过程中出现错误则 reject
1. 仅支持 generator 函数中 yield 非空对象（不支持 primitive types 如 number, string 等），具体查看 co 文档中 [Yieldables](https://github.com/tj/co#yieldables) 部分

来看看核心代码（省去了一些无关的注释，实际核心代码只有几十行）：

```js
var slice = Array.prototype.slice;
// ...
// co.wrap 的实现（这个我们待会再说）
// ...
function co(gen) {
  var ctx = this;
  var args = slice.call(arguments, 1);
  
  // 一开始就返回一个 Promise 对象
  return new Promise(function(resolve, reject) {
    // 如果输入是一个 GeneratorFunction，则先得到其执行后的 Generator 对象
    if (typeof gen === 'function') gen = gen.apply(ctx, args);
    // 如果 gen 不是一个 Generator，则 Promise 的状态变成 fulfilled，并将 gen 作为返回值
    if (!gen || typeof gen.next !== 'function') return resolve(gen);

    // 启动遍历 Generator 的过程
    onFulfilled();

    function onFulfilled(res) {
      var ret;
      try {
        // 获取 Generator 中下一个值
        ret = gen.next(res);
      } catch (e) {
        // 在执行过程中出现任何错误, 都直接让外围 Promise 的状态变成 rejected
        return reject(e);
      }
      next(ret);
      return null;
    }

    // 退出 Generator, 并让外围的 Promise 的状态变成 rejected
    function onRejected(err) {
      var ret;
      try {
        ret = gen.throw(err);
      } catch (e) {
        return reject(e);
      }
      next(ret);
    }

    // 这个是 co 中最关键的函数
    // 接收一个 Generator 遍历出来的值 { value, done }
    // 并将 value 作为下一个 .next() 方法的输入
    // 这里造成的效果是, yield 语句后面跟着的值(即 value)会成为上一个 yield 语句的返回值
    function next(ret) {
      if (ret.done) return resolve(ret.value);
      // 封装成 Promise
      var value = toPromise.call(ctx, ret.value);
      // 继续进行 Generator 的遍历
      if (value && isPromise(value)) return value.then(onFulfilled, onRejected);
      
      // 如果 value 的值的类型不是 Function/Promise/Generator/GeneratorFunction/Array/Object 的话
      // 则中断整个 Generator 并让 Promise 状态为 rejected
      return onRejected(new TypeError('You may only yield a function, promise, generator, array, or object, '
        + 'but the following object was passed: "' + String(ret.value) + '"'));
    }
  })
}

// 将输入封装成 Promise
// 如果输入类型不符, 则返回原类型(返回值肯定不是一个 Promise)
function toPromise(obj) {
  if (!obj) return obj;
  if (isPromise(obj)) return obj;
  if (isGeneratorFunction(obj) || isGenerator(obj)) return co.call(this, obj);
  if ('function' == typeof obj) return thunkToPromise.call(this, obj);
  if (Array.isArray(obj)) return arrayToPromise.call(this, obj);
  if (isObject(obj)) return objectToPromise.call(this, obj);
  return obj;
}

//=== 以下省略 thunkToPromise, arrayToPromise, objectToPromise 的实现
```

co 的实现的流程：

1.  整个函数返回一个 Promise 对象
2.  将调用 `Generator.next` 的操作封装在一个函数 onFulfilled 中
3.  将每次 `next` 返回的值封装成一个新的 Promise 对象，并在其状态变成 `fulfilled` 时调用 onFulfilled，让 Generator 函数继续执行
4.  当本次 `next` 返回的 `done: true` 时，将要返回的 Promise 状态变为 `fulfilled`，将当前的 value 的值作为 Promise 所持有的值
5.  在出现以下情况时，将要返回的 Promise 的状态变为 `rejected`：
    1.  Generator 函数执行过程中抛出任何错误
    2.  某个 `yield` 语句中如果有 Promise 对象，而该 Promise 对象的状态为 `rejected`
    3.  某个 `yield` 语句中的值的类型不是 Function/Promise/Generator/GeneratorFunction/Array/Object

**TODO** 这里的 5.3 是一个令我很不解的地方，为什么要对 yield 关键字后面跟着的值的类型做要求呢？

然后我们再来看 `co.wrap`：

```js
/**
 * Wrap the given generator `fn` into a
 * function that returns a promise.
 * This is a separate function so that
 * every `co()` call doesn't create a new,
 * unnecessary closure.
 *
 * @param {GeneratorFunction} fn
 * @return {Function}
 * @api public
 */
co.wrap = function (fn) {
  createPromise.__generatorFunction__ = fn;
  return createPromise;
  function createPromise() {
    return co.call(this, fn.apply(this, arguments));
  }
};
```

`co.wrap` 做的事情是：接受一个 `[GeneratorFunction]` 函数对象，返回一个 "临时函数" —— 这个 "临时函数" 可以在任何时间被执行并返回一个 Promise：

>   Convert a generator into a regular function that returns a `Promise`.

这看上去似乎有些令人摸不着头脑，但正是这个方法构成了 `koa` 1.x 中处理 middleware 的基础。有兴趣的同学可以看这里的代码：

1. `koa-compose` 中的 compose 方法: https://github.com/koajs/compose/blob/master/index.js
2. `co.wrap` in `app.callback` in `application.js` https://github.com/koajs/koa/blob/v1.x/lib/application.js#L127

### Promise memory leak?

在阅读源码过程中，我发现了一段很有趣的注释：

```javascript
function co(gen) {
  //...省略代码
  
  // we wrap everything in a promise to avoid promise chaining,
  // which leads to memory leak errors.
  // see https://github.com/tj/co/issues/180

  //...省略代码
}
```

于是我查看了一下[这个 issue](https://github.com/tj/co/issues/180)，提出 issue 的人认为 co 在某些情况下可能会造成内存泄漏，而具体使用的情况与 Promise 有关。

这个 issue 已经[被修复](https://github.com/tj/co/issues/180#issuecomment-68094905)，具体做法是使用一个 Promise 实例，在所有需要变更状态的情况下都调用该实例所对应的 `resolve` 和 `reject` 方法。但我产生了一个新的疑问，就是为什么这样改就可以修复 co 的内存泄漏问题呢？于是我决定继续研究这个 issue。

```js
function co (gen) {
  // blah...

  return new Promise(function(resolve, reject) {
     // 在后续的代码中直接调用 resolve 和 reject 方法
     // 不采用 Promise.resolve, Promise.reject
  })
}
```

在这个 issue 中，首先有人提出了一段测试代码：

```javascript
var co = require('co');
function* sleep() {
    return new Promise(function(resolve) {
        setTimeout(resolve, 1);
    });
};
co(function*() {
    for(var i = 0; true; ++i) {
        yield sleep();

        if (i % 10000 === 0) {
            global.gc();
            console.log(process.memoryUsage());
        }

    }
}).then(function() {
    console.log('finished')
}, function(err) {
    console.log('caught error: ', err.stack);
});
```

执行这段代码可以观察到明显的内存泄漏的情况：

```
{ rss: 17420288, heapTotal: 9620736, heapUsed: 3590768 }
{ rss: 44822528, heapTotal: 49288192, heapUsed: 12972200 }
{ rss: 70955008, heapTotal: 58575616, heapUsed: 21688912 }
{ rss: 80048128, heapTotal: 66831104, heapUsed: 30531560 }
{ rss: 89157632, heapTotal: 76118528, heapUsed: 39490184 }
{ rss: 98275328, heapTotal: 85405952, heapUsed: 48445040 }
{ rss: 107368448, heapTotal: 93661440, heapUsed: 57410024 }
{ rss: 116477952, heapTotal: 102948864, heapUsed: 66365712 }
{ rss: 125591552, heapTotal: 112236288, heapUsed: 75330040 }
{ rss: 134684672, heapTotal: 120491776, heapUsed: 84285144 }
{ rss: 143798272, heapTotal: 129779200, heapUsed: 93250072 }
{ rss: 152907776, heapTotal: 139066624, heapUsed: 102205152 }
{ rss: 162000896, heapTotal: 147322112, heapUsed: 111170352 }
{ rss: 171114496, heapTotal: 156609536, heapUsed: 120125032 }
```

而 @fengmk2 利用 `heapdump` 将内存泄漏的原因 [锁定在了 Promise 上](https://github.com/tj/co/issues/180#issuecomment-68022246)，项目维护者 @jonathanong 也提出了一个和 co 无关的[测试用例](https://github.com/tj/co/issues/180#issuecomment-68094905)来说明 Promise 的问题：

```javascript
var i = 0
next()
function next() {
  return new Promise(function (resolve) {
    i++
    if (i % 100000 === 0) {
      global.gc();
      console.log(process.memoryUsage());
    }
    setImmediate(resolve)
  }).then(next)
}
```

```
{ rss: 142749696, heapTotal: 128759296, heapUsed: 93098624 }
{ rss: 234614784, heapTotal: 218537728, heapUsed: 182771736 }
{ rss: 325664768, heapTotal: 308316160, heapUsed: 272393200 }
{ rss: 416694272, heapTotal: 397062656, heapUsed: 361990640 }
{ rss: 507744256, heapTotal: 486841088, heapUsed: 451476544 }
{ rss: 598794240, heapTotal: 576619520, heapUsed: 541064768 }
{ rss: 688660480, heapTotal: 666397952, heapUsed: 630666888 }
{ rss: 779710464, heapTotal: 756176384, heapUsed: 720264424 }
{ rss: 870760448, heapTotal: 845954816, heapUsed: 809866824 }
{ rss: 961794048, heapTotal: 934701312, heapUsed: 899464696 }
{ rss: 1052844032, heapTotal: 1024479744, heapUsed: 989066688 }
{ rss: 1143898112, heapTotal: 1114258176, heapUsed: 1078667208 }
{ rss: 1234948096, heapTotal: 1204036608, heapUsed: 1168269624 }
{ rss: 1325998080, heapTotal: 1293815040, heapUsed: 1257867232 }
{ rss: 1417052160, heapTotal: 1383593472, heapUsed: 1347469472 }
```

在这个 issue 得到修复之后，后续的讨论集中到了 Promise 实现的问题上...

首先是 co 的维护者 @jonathanong 在 Promise A+ 规范上提了 issue: [chain of never resolved promises create memory leaks](https://github.com/promises-aplus/promises-spec/issues/179)，随后贺老 @hax 也提出了[对规范的疑问](https://github.com/promises-aplus/promises-spec/issues/183)，认为控制内存泄漏和控制 Promise 执行顺序之间是无法同时满足的。

在第一个 issue 中，bluebird 的作者 @petkaantonov [提到](https://github.com/promises-aplus/promises-spec/issues/179#issuecomment-68209678):

>   Well the whole 2.3.2 needs to be redone.
>
>   As far as I can tell we both fixed the memory leak in our implementation by turning `promise` into a proxy for `x`: any operation performed on `promise` will be redirected to `x`. Any operation already performed on `promise` are immediately redirected to `x` (which is possible because both must still be pending at this point).
>
>   This is different from what the spec currently says, `promise` is now never pending, fulfilled or rejected, it doesn't have its own state at all. If it had, that means `x` would need to inform `promise` when it changed state so that `promise` can change its state and that means `x` holding reference to `promise` which leads to the original leak.

简单翻译一下：

>   （Promise A+ 规范中的）整个 2.3.2 都需要重新设计。
>
>   据我所知我们（译者注：指的是 `then` 和 `bluebird` 的作者）都在我们各自的实现中将 `promise` 变成了 `x` 的一个 proxy: 任何对 `promise` 的操作都会重定向到 `x`。任何已经在 `promise` 上进行的操作（译者注：根据 Promise 的规范，`promise` 必须等待 `x` 的状态确定之后才知道自己的状态，所以对 `promise` 的操作如 `then` 等都必须要等待 `x` 的状态确定之后才可以调用，所以这里有一个 "延迟" 的情况）会立刻重定向到 `x` 上（这是可能的，因为在这时两个 promise 对象的状态都是 pending 的。
>
>   这和当前规范中的说法不一致，`promise` 现在（的状态）永远不会是 pending, fulfilled 或者是 rejected，它根本就没有自己的状态。如果它有的话，那就意味着 `x` 必须在状态改变时通知 `promise` ，这样 `promise` 才可以修改它自身的状态，这就意味着 `x` 必须要保留对 `promise` 的引用，这样就导致了最初的（内存）泄漏。

@petkaantonov 提出：[then](https://github.com/then/promise/pull/67) 和 [bluebird](https://github.com/petkaantonov/bluebird/commit/8c1edaf0a77a6d46a7527b2873e456d3ff62fab8) 的实现都规避了内存泄漏的问题，但实际的做法与 Promise A+ 规范有冲突。

为了彻底理解上述说法，我们需要研究一下 Promise A+ 规范。

**Promise A+ spec, 2.3.2:**

*[Promise A+ 标准](https://promisesaplus.com/)* 中对于 `then` 方法有以下规定：

> ...
> 2.2.7 `then` must return a promise [3.3]
```
promise2 = promise1.then(onFulfilled, onRejected);
```
>   2.2.7.1 If either `onFulfilled` or `onRejected` returns a value `x`, run the Promise Resolution Procedure `[[Resolve]](promise2, x)`
>   ...
>   ...
>   2.3 The Promise Resolution Procedure
>   The **promise resolution procedure** is an abstract operation taking as input a promise and a value, which we donate as `[[Resolve]](promise, x)`. If `x` is a thenable, it attempts to make `promise` adopt the state of `x`, under the assumption that `x` behaves at least somewhat like a promise. Otherwise, it fulfills `promise` with the value `x`.
>   ...
>   2.3.2 If `x` is a promise, adopt its state [3.4]:
>   2.3.2.1 If `x` is pending, `promise` must remain pending until `x` is fulfilled or rejected.
>   2.3.2.2 If/when `x` is fulfilled, fulfill `promise` with the same value.
>   2.3.2.3 If/when `x` is rejected, reject `promise` with the same reason.
>   .....

这里我尝试翻译一下：

>   ...
>
>   2.2.7 `then` 必须返回一个 Promise 对象 [3.3]

```javascript
promise2 = promise1.then(onFulfilled, onRejected)
```

>   2.2.7.1 如果 `onFulfilled` 或者 `onRejected` 返回一个值 x，则运行下面的 **Promise 解析过程**: `[[Resolve]](promise2, x)`
>
>   ...
>
>   ...
>
>   2.3 Promise 解析过程
>
>   **Promise 解析过程** 是一个抽象的操作，其需输入一个 Promise 和一个值，我们表示为 `[[Resolve]](promise, x)`，如果 `x` 是一个 Thenable（注：持有 `then` 方法的对象），解析过程尝试去让 `promise` 接受 `x` 的状态，基于以下假设：`x` 的表现至少有某些部分像是一个 Promise；否则，解析过程将让 `promise` 的值变成 `fulfilled` 且采用 x 的值。
>
>   ...
>
>   2.3.2 如果 `x` 是一个 Promise，则使 `promise` 接受 `x` 的状态 [3.4]
>
>   2.3.2.1 如果 `x` 状态为 `pending`，则 `promise` 也将保持为 `pending` 直到 `x` 状态变成 `fulfilled` 或者是 `rejected`
>
>   2.3.2.2 如果/当 `x` 状态为 `fulfilled`，则 `promise` 状态也为 `fulfilled` 且持有的值与 `x` 相同
>
>   2.3.2.3 如果/当 `x` 状态为 `rejected`，则 `promise` 状态也为 `rejected` 且理由(reason) 与 `x` 相同

我们再来重新看 bluebird 作者的原话，就不难理解 Promise A+ 规范的问题是什么了：

>   这和当前规范中的说法不一致，`promise` 现在（的状态）永远不会是 pending, fulfilled 或者是 rejected，它根本就没有自己的状态。如果它有的话，那就意味着 `x` 必须在状态改变时通知 `promise` ，这样 `promise` 才可以修改它自身的状态，**这就意味着 `x` 必须要保留对 `promise` 的引用，这样就导致了最初的（内存）泄漏。**

#### 所以，co 是怎么修复内存泄露的？

回到最初提出的关于 co 的问题，通过 [diff](https://github.com/tj/co/pull/182/files) 我们可以看到，修复的关键在于修改 `onFulfilled` 和 `onRejected` 两个方法，让它们不要返回一个新的 Promise，这样就不会触发 Promise 解析过程，也就规避掉了刚才提到的内存泄漏的问题。

#### Promise Order ??

在 [promise A+ 规范的 issue](https://github.com/promises-aplus/promises-spec/issues/179#issuecomment-68213192) 中，@petkaantonov 提出了一个很有趣的例子，然而我没有看懂…我们来看这段代码:

```js
var resolveA
var a = new Promise(function (resolve, reject) {
  // resolveA = resolve
  resolveA = arguments[0]
})

a.then(function () {
  console.log('first')
})

var resolveB
var b = new Promise(function (resolve, reject) {
  // resolveB = resolve
  resolveB = arguments[0]
})

b.then(function () {
  console.log('second')
})

resolveA(b)

b.then(function () {
  console.log('third')
})

resolveB()
```

这段代码的输出顺序应该是？

bluebird 作者 `@petanntonov` 的结论是:

> 遵循规范实践的版本(before fix) 和 Q 的实践中, 以上代码的顺序是 second, third, first
> 而 bluebird 修复 memory leak 问题之后的执行顺序是 second, first, third

然而我想了很久也不是特别明白这里的处理方式...

#### 原生 Promise 实现?

我们现在知道，在 Node 及浏览器未能支持 Promise 规范的情况下，根据 Promise A+ 标准实现的第三方 Promise 库，可能会出现内存泄漏。使用 co 的 issue#180 中的[测试代码](https://github.com/tj/co/issues/180#issue-52777951)：

```js
var co = require('co');
function* sleep() {
    return new Promise(function(resolve) {
        setTimeout(resolve, 1);
    });
};
co(function*() {
    for(var i = 0; true; ++i) {
        yield sleep();

        if (i % 10000 === 0) {
            global.gc();
            console.log(process.memoryUsage());
        }

    }
}).then(function() {
    console.log('finished')
}, function(err) {
    console.log('caught error: ', err.stack);
});
```

在 Node v8.5.0 环境下测试（执行时需要启用 gc 的选项：`node —expose-gc test.js`）结果如下：

```
{ rss: 22249472, heapTotal: 10485760, heapUsed: 4095864, external: 13316 }
{ rss: 28135424, heapTotal: 11010048, heapUsed: 4547568, external: 8224 }
{ rss: 28520448, heapTotal: 11010048, heapUsed: 4573504, external: 8224 }
{ rss: 28835840, heapTotal: 11010048, heapUsed: 4540072, external: 8224 }
{ rss: 28966912, heapTotal: 11010048, heapUsed: 4547504, external: 8224 }
{ rss: 29106176, heapTotal: 11534336, heapUsed: 4543120, external: 8224 }
{ rss: 29138944, heapTotal: 11534336, heapUsed: 4550552, external: 8224 }
{ rss: 29282304, heapTotal: 11534336, heapUsed: 4545632, external: 8224 }
{ rss: 29351936, heapTotal: 11534336, heapUsed: 4553064, external: 8224 }
```

测试代码使用了 `process.memoryUsage()` 方法来获得当前 Node 环境下内存使用情况：

*   `heapTotal` 和 `heapUsed` 指的是 V8 的内存使用情况，其中 `heapUsed` 指程序执行过程中实际使用的内存
*   `external` 指 V8 管理的 JS 对象所绑定的 C++ 对象所使用的内存大小
*   `rss`（Resident Set Size，驻留集）指的是主内存设备（main memory device）所占用的内存空间，其中包括了堆，代码片段和栈调用。

可以看到 `heapUsed` 并没有明显的增长（从第二行日志开始一直维持在 455w 左右波动，并没有明显的递增趋势），那是否意味着 Node 中 Promise 的实现没有问题呢？

由于代码中使用了 co，所以我们需要排除掉 co 的影响，于是使用第二个测试例子，这个例子没有用到 co，是一个纯 Promise 的测试：

```js
var i = 0
next()
function next() {
  return new Promise(function (resolve) {
    i++
    if (i % 100000 === 0) {
      global.gc();
      console.log(process.memoryUsage());
    }
    setImmediate(resolve)
  }).then(next)
}
```

测试代码的思路很清楚，就是通过递归的方式，实现一个 Promise 链：每一个新建的 Promise 对象的 `.then` 调用中，回调函数里总是会返回一个新的 Promise，这就重现了 Promise A+ 规范中 2.2.7 和 2.3.2 的情况：

>   2.2.7 `then` must return a Promise [3.3]
>
>   2.2.7.1 If either `onFulfilled` or `onRejected` returns a value `x`, run the Promise Resolution Procedure `[[Resolve]](promise2, x)`
>
>   2.3.2 If `x` is a promise, adopt its state [3.4]:

其运行结果如下：

```
{ rss: 94011392, heapTotal: 79167488, heapUsed: 44926712, external: 8224 }
{ rss: 132673536, heapTotal: 119013376, heapUsed: 86040960, external: 8224 }
{ rss: 181956608, heapTotal: 164102144, heapUsed: 126798032, external: 8224 }
{ rss: 220876800, heapTotal: 202899456, heapUsed: 167640600, external: 8224 }
{ rss: 257863680, heapTotal: 243793920, heapUsed: 208447880, external: 8224 }
{ rss: 303140864, heapTotal: 285212672, heapUsed: 249248816, external: 8224 }
{ rss: 344813568, heapTotal: 326631424, heapUsed: 290051784, external: 8224 }
{ rss: 386400256, heapTotal: 368050176, heapUsed: 330850536, external: 8224 }
{ rss: 432185344, heapTotal: 413663232, heapUsed: 371600120, external: 8224 }
{ rss: 470716416, heapTotal: 451936256, heapUsed: 412433776, external: 8224 }
{ rss: 511557632, heapTotal: 492830720, heapUsed: 453250368, external: 8224 }
```

看来，似乎目前 Node v8.5.0 版本内对 Promise 的实现仍然会存在这个问题。嗯，看来编码中要注意了……

#### 更多研究

哼哧哼哧写完之后才发现，早有人很详细的研究了这个问题，惭愧哪……

*   Maya 大神写的关于 Promise 链的详细研究（太长了，暂时看不动）：https://github.com/xieranmaya/blog/issues/5
*   关于 Promise 内存泄漏的问题 by 腾讯 AlloyTeam http://www.alloyteam.com/2015/05/memory-leak-caused-by-promise/

## Reference

*   [tj/co, The ultimate generator based flow-control goodness for node.js](https://github.com/tj/co)
*   [Aggressive Memory Leak, tj/co](https://github.com/tj/co/issues/180#issuecomment-68094905)
*   [chain of never resolved promises create memory leaks, promises-aplus/promises-spec](https://github.com/promises-aplus/promises-spec/issues/179)
*   [Which behavior is correct? ](https://github.com/promises-aplus/promises-spec/issues/183)
*   [Promise A+](https://promisesaplus.com/)
*   [Promise A+ 规范中文翻译](https://segmentfault.com/a/1190000002452115)
*   [「翻译」Promises/A+ 规范](http://www.ituring.com.cn/article/66566)
*   [Generator 函数的语法, ECMAScript6 入门 by 阮一峰](http://es6.ruanyifeng.com/#docs/generator)

