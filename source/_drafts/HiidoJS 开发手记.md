title: Hiido.js 开发手记
tags: JavaScript
---

### 0 - 前言
hiido.js 是为 YY 公司内部各 Web 网页提供的网站访问信息统计脚本，其作用等同于 Google Analytics，友盟，百度统计等第三方网站统计服务所提供的 JS SDK。hiido.js 由海度数据中心下的海度系统组开发并维护，除前端 JS SDK 外，海度系统组还负责网站统计数据的处理和后台展示等工作。

以下是对 hiido.js 开发的一个简要的总结，主要关注前端 JS SDK 开发中所考虑和遇到的问题。

首先简单介绍下 hiido.js 的由来：

1. 为什么会有 hiido.js
	* 数据敏感（不倾向于使用第三方服务）
	* 原有脚本（duowan.js）的问题
		* 代码较混乱且缺乏组织，难以维护
		* 缺乏必要的测试
		* 某些指标统计结果不准确且混乱（如：设备和操作系统混在一起统计）

2. hiido.js 的功能
	* 完全覆盖 duowan.js 原有的功能点（后面会详述）
	* **必须保证在绝大部分的浏览器上（包括 IE6）能够正常工作**
	* **绝对不能影响业务代码的正常执行，也不能影响所在业务网站的正常页面渲染**: 对于用户来说，统计脚本必须是不可知的

3. 和 duowan.js 对比：
	* 可读性，可维护性，模块化
	* **准确获取** 用户访问时的设备相关信息（设备类型，操作系统，浏览器类型及版本, etc...）
	* 尽可能多的测试覆盖
	* 在满足以上条件的情况下，代码体积尽可能的小

具体做法参考借鉴了 [Google Analytics](https://developers.google.com/analytics/devguides/collection/analyticsjs/?hl=zh-cn) 和 [百度统计](https://tongji.baidu.com/web/welcome/login) 的方案。

---

### 1 - 背景知识

#### 网站统计如何工作？

![](./web-stat-process.png)

上图简要的说明了网站统计过程中的四个步骤：上报（log），数据处理（process），数据存储（store）和数据展示（display）。

我们采用了服务器集群来对上报请求进行负载均衡，所收集到的上报请求均以 **日志文件** 的形式储存在集群各机器中。

在数据处理阶段，我们采用定时的 Hive 任务读取各服务器上的日志，进行计算后写入到业务层的 MySQL 中，最终再由 MySQL 向 Web 后台站点提供数据，显示可视化报表。

#### 基本概念：PV, UV 和 IP

在网站统计中，PV (Pageview) 和 UV (Userview) 是必不可少的两个重要的指标。

##### PV

PV 顾名思义，是某个网页被访问的次数，[维基百科](https://en.wikipedia.org/wiki/Page_view) 的定义如下：

> A page view (PV) or page impression is a request to load a single HTML file (web page) of an Internet site

[Google Analytics](https://support.google.com/analytics/answer/6086080?hl=en) 的定义如下：

> An instance of a page being loaded (or reloaded) in a browser. Pageviews is a metric defined as the total number of pages viewed.

[百度统计](http://tongji.baidu.com/web/help/article?id=253&type=0) 对 PV 的定义：

> 即通常说的Page View(pv)，用户每打开一个网站页面就被记录1次。用户多次打开同一页面，浏览量值累计。

从维基百科的定义，可以理解为 PV 是 **某个网页的 url 被请求的次数**；GA 认为这个指标和 `loaded` 有关 —— 也就是说 **前提条件是页面的加载完成**；百度统计的描述其实非常模糊，并没有声明 "打开一个网站界面" 的关键步骤（时间点）究竟是什么。

如果从直观的效果来讲，个人认为应该等到用户访问的页面完成加载了之后（也就是用户已经看到了页面）再视作一次 PV，可能会更加准确一些。

不过从百度统计的代码上看，其代码的主函数是在 `DOMContentLoaded` 或 `onload` 事件（使用两个事件的原因是处于兼容性考虑）触发的回调函数中被唤起的，也就是说其上报策略是在用户看到页面加载的内容后才进行上报。

##### UV - 独立访客数

在查阅 GA 相应文档时，我并没有找到对应 UV 的说法，比较接近的 [User-ID Views](https://support.google.com/analytics/answer/3123669) 是针对已登录用户在含用户系统的站点中的用户访问情况。

[百度统计中的定义](http://tongji.baidu.com/web/help/article?id=253&type=0):

> 一天之内您网站的独立访客数(以Cookie为依据)，一天内同一访客多次访问您网站只计算1个访客(uv)。

这里所提到的 "独立访客数", 在技术实现上通常是由 JS SDK 生成一个唯一的 ui token 作为用户标识。所以 UV 数据的本质是将一定时间内的用户访问记录，根据 Cookie 所得的 ui token 来进行去重所得到的值，这个值是以 **单个设备上的浏览器** 为准的。

用 PV 数除以 UV 数得到的比值 PV/UV，可以较模糊地反映单个用户对某个页面的黏度。

##### IP - 独立 IP 数

根据访问站点的 IP 对 PV 数据进行去重后得到的数据。从目前的情况来看，所起到的作用是从宏观上让我们对页面的分布广度有一个模糊的了解。


值得一提的是，PV/UV/IP 这三个指标，必须要对限定时间内的所有数据进行计算才有意义，比如计算 "2017.5.1-2017.5.7" 的 UV，必须要将这7天的所有访问记录根据 ui token 来进行去重，所得到的数据才是有效的，不能够单纯的将 5.1-5.7 这7天的 UV 进行相加。

---

### 2 - 功能点及具体实现

接下来我们来逐个审视 hiido.js 需满足的功能点和实现。

#### 2-1 获取用户的唯一标识（ui token）

采用 cookie 加随机数的方法生成。这里值得一提的是关于 cookie 的设定。以 www.huya.com 为例：

```js
// 随机生成的一个 100 年后才过期的，支持 *.huya.com 的 cookie
document.cookie = 'hd_newui=[generated_random_ui_token]; expires=2117-04-10T07:08:33.000Z; domain=.huya.com'
```

注意处理 `domain` 和 `expires` 的值，以保证在同域名下的其他二级域名中，可以根据 ui token 识别同一个用户。

#### 2-2 上报策略

hiido.js 会在代码加载后立即获取用户信息并上报。这和百度统计的做法并不一致，为什么呢？

这是由于 **duowan.js 原来的策略采用了直接上报**。由于可能有一部分用户在页面未完成加载时关闭页面（特别是网络状况差的时候），所以 ”加载后上报“ 所得到的 PV 值必然低于直接上报所得到的 PV。

如果 hiido.js 参考百度使用相同的方式，则公司内各站点中的 PV 很可能出现一定的下滑。作为服务部门，经过反复讨论后，我们选择了遵循原有做法，以避免大量的向业务方解释数据下滑的麻烦。

#### 2-3 上报方法

和已有的几家服务商一样，我们采用了 `Image src` 的上报方式，具体代码如下（为了方便理解，做了部分修改及添加了注释）:

```js
// IP_LIST: 备用服务器的 IP 列表
// REPORT_URL: 服务器的域名地址
/**
  * 上报数据
  * @param  {String} logType 上报日志的类型，在 hiido.js 中 logType=webstat
  * @param  {[type]} data    上报的数据内容
  */
function log(logType, data) {
	var queryArr = [];
	for(var key in data) {
	  queryArr.push(encodeURIComponent(key) + '=' + encodeURIComponent(data[key]));
	}
	var queryString = queryArr.join('&');

	var uniqueId = "log_"+ (new Date()).getTime();
	var image = new Image(1,1);
	window[uniqueId] = image;   // use global pointer to prevent unexpected GC

	// 如果使用服务器的域名地址上报失败，则随机使用一个备用 IP 列表中的服务器进行上报
	image.onerror = function() {
	  var ip = IP_LIST[Math.floor(Math.random() * IP_LIST.length)];
	  image.src = window.location.protocol + '//' + ip + '/j.gif?act=' + logType + '&' + queryString;
	  image.onerror = function() {
	    window[uniqueId] = null;  // release global pointer
	  };
	};
	image.onload = function() {
	  window[uniqueId] = null; // release global pointer
	};

	image.src = REPORT_URL + '?act=' + logType + '&' + queryString;
}
```

通过动态生成一个 `Image()` 对象的方法的好处在于不会影响用户的正常使用（完全不可知），同时省去了使用 Ajax 的各种麻烦。但为什么要将新建的 Image 对象赋值给一个 `window` 对象下的属性呢？原因是在于浏览器的垃圾回收机制会积极地回收这个 Image 对象，且回收的时机很可能在 Image 根据 src 的值发起请求之前，这就导致了上报请求并没有发出。

但为什么浏览器垃圾回收会如此的主动呢？这就和具体的站点情况有关：

> 因为一个大脚本的运行回产生大量的“垃圾”，浏览器垃圾回收也会相应地更频繁的启动，从而造成LOG数据丢失

具体的分析和测试可以参考百度开发童鞋的[这篇博客](http://blog.csdn.net/fudesign2008/article/details/6772108)

#### 2-4 获取设备信息

获取设备信息是 hiido.js 的关键 - 新的设备乃至操作系统层出不穷，国内设备厂商及浏览器厂商众多，要写好一个足够好的脚本能够准确判别信息非常不易。这也是原 duowan.js 开发中的一个痛点。

通过 JS，我们唯一能够获取设备信息的来源就是 `navigator.userAgent`，所以这里的问题又集中到了两点：

1. 能否写出一个有效的检测方法，从 userAgent 准确地获取设备信息（包括：设备类型，型号，操作系统类型及版本，浏览器类型及版本）？
2. 能否有效地建立 userAgent 库，以满足后续的测试及维护？

关于第1点，鉴于这是一个非常广泛的需求，所以我们先从 browser detection 相关的第三方库入手：

* [ded/bowser](https://github.com/ded/bowser)
* [3rd-Eden/useragent](https://github.com/3rd-Eden/useragent)
* [piwik/device-detector](https://github.com/piwik/device-detector)
* [mobile-detect.js](https://github.com/hgoebl/mobile-detect.js/blob/master/mobile-detect.js)
* [zsxsoft/useragent.js](https://github.com/zsxsoft/useragent.js)
* [hotoo/detector](https://github.com/hotoo/detector)

经过了仔细调研之后，最后决定使用 [hotoo/detector](https://github.com/hotoo/detector) 有几个原因：

1. 完善的文档，测试和例子
2. 由支付宝的 @闲耘 维护，大厂使用还是有一定保障（只要他们在一直用这个库，那么我们就不用担心这个库被抛弃掉的问题）
3. 符合国情（并没有找到国人写的比这个更好的一个包）

具体的做法就是在开发中引入对 `detector` 的依赖（代码有部分改动，便于阅读）：

```js
var detector = require('detector');

function getOS(ua) {
  var os = detector.parse(ua).os;
  var osName = os.name;
  var osVer = os.fullVersion;

  //========================
  // 特定的一些自定义处理
  // 修改格式, 补充遗漏部分等
  // ...
  //========================

  return osName + '|' + osVer;
}

function getBrowser(ua) {
  //========================
  // 对 ua 进行预处理，识别具有明显特征但不在 detector 检测范围内的浏览器 vendor
  // ...
  //========================

  var bs = detector.parse(ua).browser;
  var bsName = bs.name;
  var bsVersion = bs.fullVersion;

  //========================
  // 特定的一些自定义处理
  // 修改格式, 补充遗漏部分等
  // ...
  //========================

  return bsName + '|' + bsVer;
}

// 对设备类型的检测类似，不赘述
```

关于第2点，我们在后面的 "测试" 一节中再详述。

#### 2-5 页面停留时长

上文提到过 `PV/UV` 可以较模糊地反映单个用户对某个页面的黏度，另一个有效反应用户黏度的值就是用户在该页面上的停留时间。在实际开发中，如何计算用户在页面上的停留时间，成为了一个非常复杂的问题。

##### 方案一：头尾相减

在计算停留时长时，一个简单的思路是使用 ”头尾相减“，即：`停留时长 = T(用户离开页面时间点) - T(用户进入页面时间点)`

这样乍看没有问题，但如果用户实际上并没有在看这个页面呢？有趣。

用户是有可能不在看这个页面的，具体有几种情况：

* 用户打开了另外一个标签页
* 用户打开了另一个应用（或者浏览器被最小化了）
* （移动端）用户将浏览器 app 切到了后台，打开了微信
* 用户只是单纯的走开了...

如果需要获取用户 **真正在浏览** 页面时所花的时间，我们不能够单纯的使用头尾相减的方式，因为在这种情况下，以上几种情况都会导致用户的停留时长高的可怕。

##### 方案二：定时心跳

针对上述提到的几种用户不在浏览页面的情况，hiido.js 采用了定时心跳的方法，具体策略如下（为表述方便，以下简称停留时长为 lifetime）：

* 从用户进入页面时，初始化 lifetime 为 0
* 初始化后，每一个 tick（一个 rAF 的 loop，如果浏览器不支持 rAF 则默认为每 16.67ms）进行一次心跳，每次心跳给 lifetime 加上这次 tick 的时间
* 在 **用户不在看页面** 时暂停心跳
* 在 **用户重新看页面** 时重启心跳
* 在 **用户离开页面** 时，直接上报 lifetime 的值

看上去这样的策略没问题，只要我们知道在 JS 中如何得知用户并不在看当前的页面。

##### Page Visibility API

考虑用户在看另一个标签页或者使用另一个程序的情况，使用 [Page Visibility API](https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API)：

> The Page Visibility API lets you know when a webpage is visible or in focus.

在浏览器的页面发生切换的时候，Page Visibility API 会触发一个 visibilitychange 的事件，通过监听这个事件，我们可以动态的控制 lifetime 的心跳：

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
  if (document[hidden]) {file:///Users/allenwu/Desktop/no-fuck-say.jpeg
    // 停止心跳...
  } else {
    // 启动心跳...
  }
}

// Handle page visibility change
document.addEventListener(visibilityChange, handleVisibilityChange, false);
```

找到这个之后我满心欢喜，直到看到了兼容性部分：

![](page-visibility-api-compatible-table.png)

这不得不让我们寻求其他解决方案。

![](no-fuck-say.jpeg)

##### 文档的 focus 事件

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

为什么不直接使用 `window.onfocus` 或者 `window.onblur` ? 因为 [有人提到](http://stackoverflow.com/a/3745804/1301194)：

> onfocus and onblur are buggy on the window object in IE. The alternative is to use the propagating onfocusin and onfocusout events

注意，这里不可以使用 `document` 对象的 focus 和 blur 事件，因为这两个事件是不冒泡的。

##### 如果用户只是单纯的走开了呢...?

其实这里的根本问题是：用户所在的页面一直处于 focus 的状态，而我们无法得知用户是否在真正浏览这个页面。

于是在这里我们给出一个假设：如果用户一直在浏览某个页面，那么他必须会在一定时间内对这个页面进行一定的操作（不管是何种类型的操作）。基于假设，我们设计了以下策略：

1. 设置一个变量 `inactiveTime`，记录用户不活跃的时间，初始值 为 0
2. 在每一次心跳中累积 `inactiveTime`，增加该次心跳的时间
3. 如果用户在页面上产生了操作（mousemove, click, touchstart），则重置 `inactiveTime` 为 0
4. 当 `inactiveTime` 的累计时间超过一个阈值时，停止心跳。

这个策略保证了在用户离开设备的一段时间内，单个页面的 lifetime 累积的最大值不会超过我们所设定的阈值。在 hiido.js 中我们将这个阈值设为了 **20s**，这个数字是没有科学依据的，仅仅是作者对于用户浏览一个屏幕内页面内容所需时间的一个估计。

这个策略也无法 **精准的估计** lifetime —— 你无法采用某个 JS API 来得知用户是不是看着看着网页然后离开了座位去喝口水。

##### lifetime 计算的最终实现

关于 lifetime 的计算到这里就告一段落，hiido.js 中最终的实现代码如下（为理解方便做了修改）：

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

##### 上报 lifetime 数据

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

##### 移动端的上报 - 页面到底走了没有？

出于节省流量等考虑，目前相当一部分的手机浏览器都采用了页面缓存的策略，在这个策略下，访问过的上一个页面的文档内容并不会被销毁（即使看上去用户已经关闭了页面），而是被放置到了浏览器自带的一个页面缓存中（详见 Webkit 团队的 [这篇博客](https://webkit.org/blog/427/webkit-page-cache-i-the-basics/)）。

在这种情况下，页面将永远不会触发 `beforeunload` 或者 `unload` 事件，按照原有的策略，我们无法进行 lifetime 的上报。这样的直接影响就是，在移动端我们永远无法得知用户在页面上的停留时间。

解决这个问题的思路有两个：

1. 利用 `pageshow` 和 `pagehide` 事件，当 `pagehide` 被触发时，视为用户离开了页面；当 `pageshow` 触发时，视作用户重新进入了页面
2. 利用 Page Visibility API，在页面隐藏起来时，视为用户离开了页面；否则视作用户重新进入了页面

两种思路在相应事件触发时，都上报 lifetime，并重置所有起始变量。

hiido.js 使用了第1种实践，具体代码如下：

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

#### 2-6 获取用户点击信息

为了后续获取数据的方便，以及了解用户在页面上的点击分布，hiido.js 监听了用户点击的事件，以获取相应的信息。

```js
// data - 储存用户信息的对象
// data.ot - 用户离开页面时点击的目标 url
// data.xy - 用户最后一次点击时的坐标地址
function captureLink(e) {
  var elem = e.target || e.srcElement;
  if (isLink(elem)) {
    data.ot = elem.href;
  }
  // update the x,y of the last link click
  data.xy = e.clientX + ',' + e.clientY;
}

addEvent(document, 'click', captureLink, true);
```

在这段代码中，hiido.js 获取了用户离开页面时点击的链接地址，以及点击的位置。值得一提的是 `addEvent` 方法的最后一个参数是 `true`，实际上是将 `addEventListener` 中的 `useCapture` 参数设为 true，其原因是为了在所有其他 click 事件的监听函数执行之前，抢先捕获这个事件并执行监听函数。[这篇博客](https://blog.othree.net/log/2007/02/06/third-argument-of-addeventlistener/) 简单的说明了这一点。

#### 2-7 内部版

为了满足内部业务的一些特殊需要，同时保留对外开放的可能性，hiido.js 项目包含内部版和外部版两个不同版本的 JS 文件。其中差异主要有：

* 获取站点 id 的方式不同
* 内部版额外支持了专题信息的上报：hiido.js 声明了一套与原 duowan.js 一致的使用方式，让业务方可以声明某个页面属于某个专题。

#### 2-8 其他

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

### 3 - 工程化，模块化及测试

#### 3-1 工程化

有别于原 duowan.js 的一个单一 JS 文件的开发模式，hiido.js 具备完整的工程以及开发流程。目录结构如下：

![](hiidojs-code-structure.png)

* `dist/` - 编译得到的生产环境下的脚本
* `src/` - 项目源代码
* `doc/` - 文档
* `mtq/` - *@deprecated* 无用文件
* `others/` - *@deprecated* 参考用的文件
* `test/` - 单元测试代码
* `vendor/` - 页面测试使用的第三方的测试库

由于我们采用了模块化开发，且依赖于 `detector`，所以我们采用 webpack 将 `src/` 中的代码编译到 `dist/` 中。又由于 `detector` 的源码采用 ES6 语法，所以我们加入了 `babel-loader`，具体的 webpack 配置可以参考项目源码，此处不赘述。

开发中使用的任务用 Gulp 管理，采用 ESLint 进行代码检查。

由于我们使用了 Babel 来编译含 ES6 语法的源代码，为了在 IE8 及以下的版本中顺利通过，我们采用了 Babel 提供的 [`loose-mode`](https://github.com/bkonkle/babel-preset-es2015-loose)

#### 3-2 模块化

原有的 duowan.js 的一个严重的问题就是，各个功能之间的代码全部混在一个文件里，对于后续维护来说非常的痛苦，同时也无法享受到直接引用第三方库的便利。hiido.js 采用了模块化开发的方式，功能代码结构如下：

![](hiidojs-code-src-structure.png)

* `hiido.js`, `hiido_internal.js` - 分别为外部版和内部版的统计代码（的主文件），两个都引用了 `tracker.js`
* `tracker.js` - 统计代码核心逻辑文件
* `lib/` 各个子功能模块的代码
* `tmp/` *@deprecated* 临时代码

#### 3-3 测试

##### 3-3-1 单元测试

在 `2-4 获取设备信息` 中我们提到过，为了提高代码的可持续维护，建立一套测试机制以及可维护的 userAgent 库是很有必要的。在模块化的基础上，我们使用 [Mocha](https://mochajs.org/) 建立了针对设备信息检测的单元测试。以下是一段真实的测试代码：

```js
var _ = require('lodash');
var osFixtures = require('./fixtures/devices');

describe('操作系统', function() {
  _.forEach(osFixtures, function(f) {
	var desc = f.name + ': ' + f.key + '|';
	if (f.key === 'linux') {
	  desc += '[发行版]';
	} else {
	  desc += '[版本号]';
	}
	describe(desc, function() {
	  _.forEach(f.cases, function(c) {
	    it(c[0], function() {
	      var ret = detector(c[0]);
	      assert.equal(ret.os, c[1].os);
	    })
	  });
	})
  });
});
```

在测试代码中引用的 `./fixtures/devices` 就是一个记录了近百条 userAgent 及其真实的设备信息（包括 OS 和浏览器）的 **json 文件**

在开发前期，我们从以下几个来源获取 userAgent：

* [UserAgentString](http://www.useragentstring.com/pages/useragentstring.php)
* [fynas.com](http://www.fynas.com/ua)
* [各种手机的 User-Agent](http://www.xuebuyuan.com/885759.html)
* 部分杂牌浏览器：
  * [115](http://blog.zsxsoft.com/post/4)
  * [2345](http://www.cnblogs.com/qq21270/p/3799124.html)
  * [userAgent string - 其中含百度盒子](https://github.com/hotoo/detector/issues/71)
  * [Baidu Spark](http://www.webapps-online.com/online-tools/user-agent-strings/dv/browser581791/baidu-spark-browser)
  * [Baidu Spark II](http://blog.csdn.net/ccclll1990/article/details/17006159)
  * [Yunhai / F1浏览器](https://www.aqtronix.com/useragents/?Action=ShowAgentDetails&Name=F1Browser+Yunhai+Browser)
  * [海豚浏览器](http://blog.csdn.net/ccclll1990/article/details/17006159)
  * [旗鱼浏览器](http://liulanmi.com/dl/10230.html/comment-page-2)
  * [hotoo/detector: 用户份额大于 0.01% 的客户端 ua 信息](https://github.com/hotoo/detector/issues/45)

在系统获取了一批 userAgent 后，我们根据预上线测试阶段所搜集到设备信息为空的请求日志，进行分析后又加入了一批新的 userAgent，而所有在 `./test/fixtures` 中的测试用例，我们都通过测试的方式进行覆盖并保证其输出是我们预期的结果。

##### 3-3-2 E2E 测试

由于 hiido.js 的实际使用场景将会是线上的各个站点，所以在发布之前，我们必须在本地进行 E2E 测试以保证其基本功能能够顺利进行。以下是 E2E 测试的结构图

![](hiidojs-e2e-test.png)

在测试中：

* 我们使用 Express 构建一个简单的 Mock Log Server：
	1. 提供一个 log 的 API 地址供 hiido.js 使用，这个 API 地址在测试时代替了 hiido log 服务器集群，因为我们需要了解实际上报的内容；
	2. 提供一个路由加载 E2E 测试用的网页
* 我们使用多个浏览器 Client 打开测试网页，同时页面上加载了 hiido.js 的代码
* 根据测试的具体需要，我们 **像真实用户一样操作** 并观察实际上报的数据内容是否符合预期，具体的数据内容由 Mock Log Server 输出到终端或者保存到日志。

---

### 4 - 额外的一些小坑

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

---

### 5 - Reference

* [Cross-browser Page Visibility API polyfill, Github Gist](https://gist.github.com/addyosmani/1122546)
* [HTML5 迟来的 API: Page Visibility, Tencent AlloyTeam Blog](http://www.alloyteam.com/2012/11/page-visibility-api/)
* [Webkit Page Cache I - The Basics, Webkit](https://webkit.org/blog/427/webkit-page-cache-i-the-basics/)
* [Webkit Page Cache II - The unload Event, Webkit](https://webkit.org/blog/516/webkit-page-cache-ii-the-unload-event/)
* [Don't lose user and app state, use Page Visibility, by Ilya Grigorik](https://www.igvita.com/2015/11/20/dont-lose-user-and-app-state-use-page-visibility/)
* [addEventListener 的第三个参数](https://blog.othree.net/log/2007/02/06/third-argument-of-addeventlistener/)
* [addEventListener](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener)
* [4ae9b8/browserhacks](https://github.com/4ae9b8/browserhacks)
* [识别特定浏览器最佳实践 #18, hotoo/detector](https://github.com/hotoo/detector/issues/18)
* [jQuery 的 ready 函数是如何工作的？](http://www.cnblogs.com/haogj/archive/2013/01/15/2861950.html)