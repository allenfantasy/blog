title: 网站统计那些事（三）：统计脚本实现（下）
date: 2017-05-08 14:02
tags:
- JavaScript
- User Tracking
---

网站统计系列的第三篇。坑爹的兼容性。
<!--more-->

本篇将继续讨论统计脚本的具体功能实现：

* [统计停留时长](#5-统计停留时长)
* [其他细节](#6-其他细节)
* [一些小坑](#7-一些小坑)

传送门：

* [网站统计那些事（一）：背景与基础概念](../user-tracking-i)
* [网站统计那些事（二）：统计脚本实现（上）](../user-tracking-ii)
* [网站统计那些事（四）：工程化，模块化与测试](../user-tracking-iv)

## 5 统计停留时长

在[系列的第一篇](../user-tracking-i#UV-独立访客数)中我们提到， `PV/UV` 可以较模糊地反映单个用户对某个页面的黏度，另一个有效反应用户黏度的值就是用户在该页面上的停留时间。在实际开发中，如何计算用户在页面上的停留时间，成为了一个非常复杂的问题。

#### 方案一：头尾相减

在计算停留时长时，一个简单的思路是使用 ”头尾相减“，即：`停留时长 = T(用户离开页面时间点) - T(用户进入页面时间点)`

这样乍看没有问题，但如果用户实际上 **并没有在看这个页面呢** ？有趣。

用户是有可能不在看这个页面的：

* 用户打开了另外一个标签页
* 用户打开了另一个应用（或者浏览器被最小化了）
* （移动端）用户将浏览器 app 切到了后台，打开了微信
* 用户只是单纯的走开了...

如果需要获取用户 **真正在浏览** 页面时所花的时间，我们不能够单纯的使用头尾相减的方式，因为在这种情况下，以上几种情况都会导致用户的停留时长高的可怕。

#### 方案二：定时心跳

针对上述提到的几种用户不在浏览页面的情况，ut.js 采用了定时心跳的方法，具体策略如下（为表述方便，以下简称停留时长为 lifetime）：

* 从用户进入页面时，初始化 lifetime 为 0
* 初始化后，每一个 tick（一个 rAF 的 loop，如果浏览器不支持 rAF 则默认为每 16.67ms）进行一次心跳，每次心跳给 lifetime 加上这次 tick 的时间
* 在 **用户不在看页面** 时暂停心跳
* 在 **用户重新看页面** 时重启心跳
* 在 **用户离开页面** 时，直接上报 lifetime 的值

看上去这样的策略没问题，只要我们知道在 JS 中如何得知用户并不在看当前的页面。

#### Page Visibility API

考虑用户在看另一个标签页或者使用另一个程序的情况，使用 [Page Visibility API](https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API)：

> The Page Visibility API lets you know when a webpage is visible or in focus.

在浏览器的页面发生切换的时候，Page Visibility API 会触发一个 `visibilitychange` 的事件，通过监听这个事件，我们可以动态的控制 lifetime 的心跳：

```js
// Set the name of the hidden property and the change event for visibility
var hidden, visibilityChange;
if (typeof document.hidden !== "undefined") { // Opera 12.10 and Firefox 18 and later support
  hidden = "hidden";
  visibilityChange = "visibilitychange";
} else if (typeof document.msHidden !== "undefined") {
  hidden = "msHidden";
  visibilityChange = "msvisibilitychange";
} else if (typeof document.webkitHidden !== "undefined") {
  hidden = "webkitHidden";
  visibilityChange = "webkitvisibilitychange";
}

function handleVisibilityChange() {
  if (document[hidden]) {
    // 停止心跳...
  } else {
    // 启动心跳...
  }
}

// Handle page visibility change
document.addEventListener(visibilityChange, handleVisibilityChange, false);
```

找到这个之后我满心欢喜，直到看到了兼容性部分：

![](/images/page-visibility-api-compatible-table.png)

这不得不让我们寻求其他解决方案。

![](/images/no-fuck-say.jpeg)

#### 文档的 focus 事件

我们可以监听网页的 `focus` 事件，来判断用户当前是否在使用这个标签页。

```js
function onFocus() {
  // 启动心跳...
}

function onBlur() {
  // 停止心跳...
}

if ('onfocusin' in document) {
  document.onfocusin = onFocus;
  document.onfocusout = onBlur;
} else {
  window.onfocus = onFocus;
  window.onblur = onBlur;
}
```

* [focusin](https://developer.mozilla.org/en-US/docs/Web/Events/focusin)
* [focusout](https://developer.mozilla.org/en-US/docs/Web/Events/focusout)
* [focus](https://developer.mozilla.org/en-US/docs/Web/Events/focus)
* [blur](https://developer.mozilla.org/en-US/docs/Web/Events/blur)

为什么不直接使用 `window.onfocus` 或者 `window.onblur` ? 因为 [出现了兼容性问题](http://stackoverflow.com/a/3745804/1301194)：

> onfocus and onblur are buggy on the window object in IE. The alternative is to use the propagating onfocusin and onfocusout events

注意，这里不可以使用 `document` 对象的 focus 和 blur 事件，因为这两个事件是不冒泡的。

#### 如果用户只是单纯的走开了呢...?

这里的根本问题是：

> 用户所在的页面一直处于 focus 的状态，而我们无法得知用户是否在 **真正浏览** 这个页面。

为了解决这个问题，我们对用户的行为给出了一个关键假设：

> 如果用户一直在浏览某个页面，那么他必须会在一定时间内对这个页面进行一定的操作（不管是何种类型的操作）。

基于假设，我们设计了以下策略：

1. 设置一个变量 `inactiveTime`，记录用户不活跃的时间，初始值 为 0
2. 在每一次心跳中累积 `inactiveTime`，增加该次心跳的时间
3. 如果用户在页面上产生了操作（mousemove, click, touchstart），则重置 `inactiveTime` 为 0
4. 当 `inactiveTime` 的累计时间超过一个阈值时，停止心跳。

这个策略保证了在用户离开设备的一段时间内，单个页面的 lifetime 累积的最大值不会超过我们所设定的阈值。

在 ut.js 中我们将这个阈值设为了 **20s** —— 这个数字是没有科学依据的，仅仅是作者对于用户浏览一个屏幕内页面内容所需时间的一个估计。

这个策略也无法 **精准的估计** lifetime —— 你无法采用某个 JS API 来得知用户是不是看着看着网页然后离开了座位去喝口水。

#### lifetime 计算的最终实现

关于 lifetime 的计算到这里就告一段落，ut.js 中最终的实现代码如下（为理解方便做了修改）：

```js
// addEvent() 是包装了 `window.addEventListener` 和 `window.attachEvent` 的事件监听函数

var r = window.requestAnimationFrame;
var c = window.cancelAnimationFrame;
var h;

var lt = 0;
var ltStart;
var inActiveTime;
var inActiveThreshold = 2E4;

// 创建一个心跳闭包，负责向 lifetime 增加累计时间
var h = (function() {
  var timer;

  function beat() {
    var now = new Date()
    var diff = now - ltStart;

    lt = lt + diff;
    inActiveTime = inActiveTime + diff;
    ltStart = now;

    if (inActiveTime <= inActiveThreshold) {
      timer = r(beat);
    } else {
      timer = null;
    }
  }

  return {
    start: function() {
      if (!timer) {
        ltStart = new Date();
        timer = r(beat);
      }
    },
    stop: function() {
      if (timer) {
        c(timer);
        timer = null;
      }
    }
  };
})();

function onFocus() {
  h.start()
}
function onBlur() {
  h.stop();
}

// 在 PC 端使用 focusin / focusout / focus / blur 事件
if ('onfocusin' in document) {
  document.onfocusin = onFocus;
  document.onfocusout = onBlur;
} else {
  window.onfocus = onFocus;
  window.onblur = onBlur;
}

// 在移动端使用 Page Visibility API 检查页面是否 active
var prefixes = ['', 'webkit', 'moz', 'ms', 'o'];
var pf;
var hiddenKey;
var eventKey;

if (/* current env is mobile */) {
  for(var i = 0; i < prefixes.length; i++) {
    pf = prefixes[i];
    hiddenKey = pf ? (pf + 'Hidden') : 'hidden';
    if (hiddenKey in document) {
      eventKey = pf + 'visibilitychange';
      break;
    }
  }

  if (eventKey) {
    addEvent(document, eventKey, function() {
      document[hiddenKey] ? onBlur() : onFocus();
    });
  }
}

inActiveTime = 0;
h.start(); // 开始计算 lifetime
```

#### 上报 lifetime 数据

在用户结束对网页的访问时，此时我们需要上报 lifetime 数据。单独采用 `onbeforeunload` 或者 `onunload` 都是不可取的，因为两个事件都有浏览器不支持的情况出现：

* [Opera not supporting load and unload events](https://www.quirksmode.org/bugreports/archives/2004/11/load_and_unload.html)
* [Mozilla - window.onunload don't work when is page is in a popup](https://bugzilla.mozilla.org/show_bug.cgi?id=681636)
* [onunload not working in Safari...](https://webkit.org/blog/516/webkit-page-cache-ii-the-unload-event/)

...这就很吓人了。为了尽可能的实现兼容，保险的做法是对两个事件同时进行监听，只要有一个事件监听到了就上报 lifetime 的数据。具体代码如下（和实际代码有调整）：

```js
var leaveReportSent = false;
function reportLeave() {
  if (leaveReportSent) { // 如果已经发送过了就不再发送了
    return;
  }
  // 上报 lifetime 数据...
  // 上报其他数据...
  // ...
  leaveReportSent = true;
},

window.onbeforeunload = function() {
  reportLeave();
};

window.onunload = function() {
  reportLeave();
};
```

但是，还有一个问题...

#### 移动端的上报 - 页面到底走了没有？

出于节省流量等考虑，目前相当一部分的手机浏览器都采用了页面缓存的策略，在这个策略下，访问过的上一个页面的文档内容并不会被销毁（即使看上去用户已经关闭了页面），而是被放置到了浏览器自带的一个页面缓存中（详见 Webkit 团队的 [这篇博客](https://webkit.org/blog/427/webkit-page-cache-i-the-basics/)）。

在这种情况下，页面将永远不会触发 `beforeunload` 或者 `unload` 事件，按照原有的策略，我们无法进行 lifetime 的上报。这样的直接影响就是，在移动端我们永远无法得知用户在页面上的停留时间。

解决这个问题的思路有两个：

1. 利用 `pageshow` 和 `pagehide` 事件，当 `pagehide` 被触发时，视为用户离开了页面；当 `pageshow` 触发时，视作用户重新进入了页面
2. 利用 `Page Visibility API`，在页面隐藏起来时，视为用户离开了页面；否则视作用户重新进入了页面。[via Ilya Grigorik](https://www.igvita.com/2015/11/20/dont-lose-user-and-app-state-use-page-visibility/)

两种思路在相应事件触发时，都上报 lifetime，并重置所有起始变量。

ut.js 使用了第1种实践，具体代码如下：

```js
// lt - 前文提到的储存 lifetime 的变量
// reportEnter() - 进入页面时上报信息的方法
// reportLeave() - 离开页面时上报 lifetime 及其他内容的方法

var leaveReportSent = false;
var enterReportSent = false;

addEvent(window, 'pageshow', function(event) {
  if (event.persisted) {
    // like a new page, reset all related attributes
    leaveReportSent = false;
    enterReportSent = false;
    lt = 0;

    reportEnter();
  }
});

addEvent(window, 'pagehide', function(event) {
  if (event.persisted) {
    reportLeave();
  }
});
```

## 6 其他细节

* 使用了 `try ... catch` 进行处理错误捕获
* 暴露一个 `hd()` 的全局方法，让业务方使用该方法进行 **自定义的上报** ，这一点和 GA 以及百度统计的实现是类似的。
* 自定义一个 `addEvent()` 方法封装了 `attachEvent` 和 `addEventListener`
* **防止业务方重复加载执行该脚本**：设置一个全局变量作为判定标识
* 获取额外的设备信息：
    * 屏幕颜色深度：`window.screen.colorDepth`
    * 屏幕尺寸：`window.screen.width`,`window.screen.height`
    * Flash 版本：`window.ActiveXObject`（随着 Flash 逐渐被淘汰，考虑去掉这个支持）
    * Cookie：`navigator.cookieEnabled`
    * Java 支持：`navigator.javaEnabled()`
    * 系统语言：`navigator.language`

## 7 一些小坑

#### IE 中使用 `console` 对象

在脚本部署逐步上线使用的过程中，`console` 对象是调试时的一个重要方法，但当实际上线时，我们惊讶地发现正是 `console` 引起了一些额外的问题：

* 旧版的 IE 中 `window` 对象并没有 `console` 属性，
* 即便有 `console` 属性，直接使用 `console` 关键字并不能生效。

当然，解决方案是很简单的，写一个简易的 polyfill 即可：

```js
if(!window.console) {
  var console = {};
  console.log = function() {};
  console.dir = function(obj) {
    for(var i in obj) {
        console.log(i + " ", obj[i]);
    }
  };
  window.console = console;
}
if(!window.console.dir) {
  window.console.dir = function(obj) {
    for(var i in obj) {
      window.console.log(i + " ", obj[i]);
    }
  }
}
// 如果希望在后续的代码中使用 console 关键字
var console = window.console;
```

#### 360 浏览器在大部分情况下无法检测

* 援引 [百度统计](http://tongji.baidu.com/data/browser)

> 奇虎360浏览器份额在2010年10月至2011年3月，和2012年9月以来，两次大幅下降，是因为360浏览器去掉了原本的浏览器特征（User-Agent），而表现为IE等浏览器特征所致。

* [detector 包作者的分析](https://github.com/hotoo/detector/issues/6)
* [某位网友的分析, 指出偏方只能够在 360.cn 上起作用](http://bbs.csdn.net/topics/390212770#post-394546176)
* Cnzz 之所以能够检测是因为 360 浏览器内核在 cnzz 的域名下的 userAgent 会带有 360SE/EE 标记，在别的域名下不会...
详见 [知乎上的回答](https://www.zhihu.com/question/20556578/answer/15472681)

## RECAP

到这里我们完成了统计脚本实现细节的归纳。下一篇会着重关注具体的工程化实践，以及如何进行测试以保证质量。