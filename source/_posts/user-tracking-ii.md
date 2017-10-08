title: 网站统计那些事（二）：统计脚本实现（上）
date: 2017-05-08 14:01
tags:
- JavaScript
- user tracking
---

网站统计系列的第二篇。一箩筐辣鸡代码和絮絮叨叨。
<!--more-->

在这一篇中，我们将详细讨论如何从业务实际需求出发，实现一个网站统计脚本。由于涉及细节众多，将分为上下两篇来详述。**为了表述方便，下文我们用 ut.js 来代指统计脚本。**

在 ut.js 的实际开发中，我们参考了相当部分的 GA 和百度统计的代码，在此特别致谢。

传送门：

* [网站统计那些事（一）：背景与基础概念](../user-tracking-i)
* [网站统计那些事（三）：统计脚本实现（下）](../user-tracking-iii)
* [网站统计那些事（四）：工程化，模块化与测试](../user-tracking-iv)

## Content

本篇将讨论到的实现要点：

* [获取用户的唯一标识](#1-获取用户的唯一标识)
* [如何进行上报](#2-如何进行上报)
* [获取设备信息](#3-获取设备信息)
* [获取用户点击信息](#4-获取用户点击信息)

## 1 获取用户的唯一标识

采用 cookie 加随机数的方法生成。这里值得一提的是关于 cookie 的设定。以 `www.baidu.com`
 为例：

```js
// 随机生成的一个 100 年后才过期的，支持 *.baidu.com 的 cookie
document.cookie = 'ui_token=[generated_random_ui_token]; expires=2117-04-10T07:08:33.000Z; domain=.baidu.com'
```

注意处理 `domain` 的值是 `.baidu.com`，这样可以保证在同域名下的其他二级域名中，根据 ui token 识别同一个用户，对于同一主域名下的多个站点统计，这是关键的一步。而 `expires` 字段表明了这个 cookie 在足够长的时间内不会过期。

## 2 如何进行上报

和已有的几家服务商一样，ut.js 采用了 `Image src` 的上报方式，具体代码如下（为了方便理解，做了部分修改及添加了注释）:

```js
// IP_LIST: 备用服务器的 IP 列表
// REPORT_URL: 服务器的域名地址
//
// 上报数据
// @param  {String} logType 上报日志的类型
// @param  {Object} data    上报的数据内容

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

## 3 获取设备信息

获取设备信息是 ut.js 的关键 - 新的设备乃至操作系统层出不穷，国内设备厂商及浏览器厂商众多，要写好一个足够好的脚本能够准确判别信息非常不易。这也是开发中的一个痛点。

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

关于第2点，我们在后续的系列文章中再详述。

## 4 获取用户点击信息

为了后续获取数据的方便，以及了解用户在页面上的点击分布，ut.js 监听了用户点击的事件，以获取相应的信息。

```js
// addEvent - 包装了 attachEvent 和 addEventListener 的监听事件函数
// isLink - 判断某个 DOM 元素是否为 <a> 元素
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

在下篇中，我们会集中讨论如何获取 **页面停留时间** 这个复杂的问题，以及其他一些技术实现的要点。

