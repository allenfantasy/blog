title: 网站统计那些事（四）：工程化，模块化与测试
date: 2017-05-08 14:03
tags:
- JavaScript
- User Tracking
---

网站统计系列的第四篇。没有代码。最后一篇了。
<!--more-->

完全没想到我可以就这个话题写四篇又长又臭的文章...万幸这是最后一篇了 —— 简单介绍项目开发的工程化实践以及如何进行测试。

传送门：

* [网站统计那些事（一）：背景与基础概念](../user-tracking-i)
* [网站统计那些事（二）：统计脚本实现（上）](../user-tracking-ii)
* [网站统计那些事（三）：统计脚本实现（下）](../user-tracking-iii)

## 工程化 & 模块化

在公司旧统计脚本的项目中，缺乏对整个工程的任务管理。对此，我们在项目中设计了较完整的工程以及开发流程。目录结构如下：

![](/images/hiidojs-code-structure.png)

* `dist/` - 编译得到的生产环境下的脚本
* `src/` - 项目源代码
* `doc/` - 文档
* ~~`mtq/` - *@deprecated* 无用文件~~
* ~~`others/` - *@deprecated* 参考用的文件~~
* `test/` - 单元测试代码
* `vendor/` - 页面测试使用的第三方的测试库

由于我们采用了模块化开发，且依赖于 [detector](https://github.com/hotoo/detector)，所以我们采用 webpack 将 `src/` 中的代码编译到 `dist/` 中。又由于 [detector](https://github.com/hotoo/detector) 的源码采用 ES6 语法，所以我们加入了 `babel-loader`，同时为了在 IE8 及以下的版本中顺利通过，我们采用了 babel 提供的 [loose-mode](https://github.com/bkonkle/babel-preset-es2015-loose)

开发中使用的任务用 Gulp 管理，采用 ESLint 进行代码检查。

重新审视公司的旧统计脚本的项目，另一个严重的问题就是，各个功能之间的代码全部混在一个文件里，对于后续维护来说非常的痛苦，同时也无法享受到直接引用第三方库的便利。我们采用了 CommonJS 模块化开发的方式。

在接下来的工作中，我们会采用 ES6 语法对代码进行重写，同时考虑用 [ES Modules](https://github.com/lukehoban/es6features#modules) 取代原有的 CommonJS 模式。

## 测试

为了保证统计脚本统计结果的准确性，以及具备良好的兼容性，我们在本地架设了完整的测试。

#### 单元测试

在 [获取设备信息](../user-tracking-ii#3-获取设备信息) 中我们提到过，为了提高代码的可持续维护，建立一套测试机制以及可维护的 userAgent 库是很有必要的。在模块化的基础上，我们使用 [Mocha](https://mochajs.org/) 建立了针对设备信息检测的单元测试。以下是一段真实的测试代码：

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

#### E2E 测试

由于 ut.js 的实际使用场景将会是线上的各个站点，所以在发布之前，我们必须在本地进行 E2E 测试以保证其基本功能能够顺利进行。以下是 E2E 测试的结构图

![](/images/hiidojs-e2e-test.png)

在测试中：

* 我们使用 Express 构建一个简单的 Mock Log Server：
    1. 提供一个 log 的 API 地址供 hiido.js 使用，这个 API 地址在测试时代替了 hiido log 服务器集群，因为我们需要了解实际上报的内容；
    2. 提供一个路由加载 E2E 测试用的网页
* 我们使用多个浏览器 Client 打开测试网页，同时页面上加载了 hiido.js 的代码
* 根据测试的具体需要，我们 **像真实用户一样操作** 并观察实际上报的数据内容是否符合预期，具体的数据内容由 Mock Log Server 输出到终端或者保存到日志。