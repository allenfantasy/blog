title: iFrame 那些事
date: 2018-07-15
tags:
- HTML
---

基本上稍有一定开发经历的 Web 开发者都多少会和 `<iframe>` 打过交道，但在最近连续接触了几个相关的需求之后，我却突然意识到，自己对 iFrame 并不是非常的了解，这就是本文的由来。

<!--more-->

在介绍 iFrame 之前，我们首先来了解一下 **浏览上下文**。

## 浏览上下文（Browsing Context）

根据 [HTML 规范](http://w3c.github.io/html/browsers.html#sec-browsing-contexts)中的定义：

> 一个 **浏览上下文** 是一个浏览器向用户展示文档对象的环境。

浏览上下文包含以下内容：

1. 一个 WindowProxy 对象，它包裹了一个当前窗口对象（Window），在浏览器的 JS 环境中我们只能用 `window` 关键字来获取到其对应的窗口对象。
2. 一份用户的会话历史记录（*Session history*），记录了该浏览器上下文展示过的所有文档（Document）对象。
3. 记录一个当前活跃的文档（*active document*），这个文档就是上下文对应的窗口对象所关联的文档对象（*associated Document*），即当前打开的文档对象。这个文档对象在窗口对象被创建时被设置，且只有在页面导航发生变化时才会变化。我们可以用 `document` 全局对象拿到当前打开的文档对象。

从类型上，有几种不同的浏览上下文：

1. 浏览器的一个 tab 或者一个窗口
2. `<iframe>`
3. 在 `<frameset>` 元素内的一个 `<frame>`

其中 `<frameset>` 的做法已经过时，并已被移出 Web 标准（取而代之的实现方案是 `<iframe>`）。

如果从结构上来看，浏览上下文是可嵌套的：文档内的某些元素如 `<iframe>` 可以实例化一个新的浏览上下文，这些元素被称为 **浏览上下文容器（browsing context container）**，而实例化的浏览上下文则被称为是 **嵌套的浏览上下文（nested browsing context）**；而对于某个嵌套的浏览上下文，其容器所在的文档对应的浏览器上下文，则可以被称为 **父浏览上下文（parent browsing context）**，且一定是唯一的。

对于浏览器的 tab 或者窗口来说，不存在比它更高一级的父上下文，所以它们可以被称为 **顶级浏览上下文（top-level browsing context）**。

![](/images/browsing_context.png)

## iFrame 由来

我们现在了解了 **浏览上下文** 的概念，也知道 `<iframe>` 元素可以在一个文档内创建一个 **嵌套的浏览上下文**，但为什么要有 iFrame 的存在呢？这就要回顾一下 Web 早期的发展历程了。

在很久很久之前（咳咳），使用多窗口页面（frames）来创建网站是一种比较流行的手段，人们将一个大的网站拆分成多个 HTML 页面，每个页面独立的放到一个 `<frame>` 元素中，再通过 `<frameset>` 元素将这些 frame 元素包含在一起，开发者甚至可以用 `cols` 或者 `rows` 属性来控制每个 frame 页面所占据的位置大小，有点像 table 布局：

```html
<!-- 很遗憾 这段 HTML 代码在现代浏览器上已经失效了 -->
<frameset cols="50%,50%">
  <frame src="https://developer.mozilla.org/en/HTML/Element/frameset" />
  <frame src="https://developer.mozilla.org/en/HTML/Element/frame" />
</frameset>
```

这样做当然不是闲着蛋疼，而是因为当时有充分的证据证明，将一个网页切分成多个小的页面，在当时下载速度还比较慢的时候，是有利于页面加载的，至少可以保证某些页面的内容先加载好并展示出来。但随着时间推移，网速变快之后，这样的做法就显得没有任何必要了。（当然，到了 2010 年后，类似的做法又重新的被搬出来，有兴趣的同学可以搜索一下 bigpipe）

而到了上世纪90年代末期和21世纪初，Java Applets 和 Flash 技术盛行，允许 Web 开发者向页面中嵌入类似视频和动画的高级内容，当时是通过 `<object>` 和 `<embed>` 元素来完成的，并确实达到了一些效果，但后续接连冒出了安全性、可访问性、文件大小等问题，加上移动端浏览器不支持此类插件，于是该方案又渐渐被抛弃。

最终，是 `<iframe>` 元素，作为 HTML4 的标准，成为了事实上的嵌入页面的通用做法，这极大地方便 Web 开发者，可以在页面上嵌入来自第三方站点的内容，如优酷，Youtube 等：

```html
<!-- Youtube: 10 Things I Regret About Node.js - Ryan Dahl - JSConf EU 2018 -->
<iframe width="560" height="315" src="https://www.youtube.com/embed/M3BM9TB-8yA" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>
```

## iFrame 的特性

由于 `<iframe>` 实际上是一个独立的浏览上下文，所以它有以下几个特点：

1. 和父文档完全隔离的 CSS 和 JS
2. 同源的 HTTP 文档和其内嵌的 `<iframe>` 元素可以通过 JS 互相获取到对方的窗口对象，并进行任意的操作。（非同源的窗口间不可以，使用 HTTPS 的协议也不可以）
3. `<iframe>` 内部发生的页面跳转导航，不会对父浏览上下文（即父窗口）产生任何影响
4. `<iframe>` 事实上会创建一个新的 Viewport（文章后面会专门讨论）

```js
// 从顶部文档获取 iFrame 文档:
iframeElement.contentWindow.document

// 从 iFrame 文档获取父文档
window.top.document
```

## iFrame 的安全性

由于 iFrame 几乎是当前 HTML 标准唯一可以直接引入第三方页面（或者，被第三方页面引入）的方法。如何保证其中的安全性是一个非常重要的问题。

#### 同源策略

假设我们在文档内引入了一个 `<iframe>` 文档。根据 DOM 提供的 API，理论上在主文档的 JS 中，我们可以通过 `iframeElement.contentDocument` 或 `inframeElement.contentWindow` 来获取 iFrame 对应的文档对象或窗口，甚至通过操纵这两个对象的方法获取 `<iframe>` 内文档的 DOM 元素，并进行任意的操作。

但这样的操作是不能被接受的，因为这意味着任何页面都可以通过这样的方式，利用 JS 操纵另一个网站的页面。浏览器通过[同源策略（Same-origin Policy）](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy)来限制这样的行为。

所谓“同源”，是指两个网站的 `协议+域名+端口` 是完全一致的，只要有一个不一致，则视作两个地址是非同源的。

当一个主页面尝试用以上的 JS 方法访问其嵌套的非同源 `<iframe>` 时，浏览器会返回以下错误：

> Uncaught DOMException: Failed to read the 'contentDocument' property from 'HTMLIFrameElement': Blocked a frame with origin "http://localhost:3000" from accessing a cross-origin frame.

> Uncaught DOMException: Blocked a frame with origin "https://xxx.example.com" from accessing a cross-origin frame.
    at <anonymous>:1:18

#### （插播）iFrame 与主文档的通信

最初 NetScape 提出同源策略时，其出发点是保证主文档并不能随意的操纵第三方的网站，但这也的确给开发者带来了一定的困扰。因为同源策略的检查非常严格，甚至不允许两个一级域名相同的文档间互相直接访问（如 `a.example.com` 和 `b.example.com`），但在某些场景下，开发者同时拥有两个域名下的页面，且希望两个页面间可以进行数据通信。

于是在早期，人们提出了若干个解决方案：

1. 通过在主页面和 `<iframe>` 中设置 `document.domain` 为同一个一级域名，来绕过同源策略的限制
2. 利用 `location` 的特性，不同域的页面，可以写不可读，让父子页面互相写对方的 location 的哈希部分进行通讯：
    1. 新建iframe，使用iframe访问一个非同源的地址（发请求），参数里带上父页面url；
    2. 当页面加载完成后，iframe内脚本设置父页面的url并在哈希部分带上数据；
    3. 父页面的脚本循环检查哈希值的变化，如果检查到有值就取值并清空哈希值；

而当 `window.postMessage` 出现后，一切都变成了浮云。`postMessage` [支持 IE8+ 及所有现代浏览器](https://caniuse.com/#search=postMessage)，且使用方式非常简单：

```html
<!-- 主 HTML: https://a.example.com/index.html -->
<iframe id="child-iframe" src="https://b.example.com/index.html"></iframe>
<script>
    var iframeElement = document.getElementById('child-iframe')
    iframeElement.addEventListener('load', function () {
        iframeElement.contentWindow.postMessage({ data: 1 }, '*')
    })
</script>
```

```html
<!-- 被嵌入的 HTML: https://b.example.com/index.html -->
<script>
    window.addEventListener('message', function (event) {
        var origin = event.origin
        var data = event.data
        // 1. 根据业务逻辑判断 origin 的合法性
        // 2. 处理 data
    });
</script>
```

#### 被引入的困扰

那么，既然 iFrame 可以允许在一个网页中嵌入任意的一个第三方的页面，那就意味着，我们编写的网页，是完全有可能被任何一个第三方的网站通过 `<iframe>` 引入的。而这些引入方，很可能带有恶意的攻击目的。

**点击劫持（Clickjacking）** 就是一种非常经典的攻击方式，也叫 *界面伪装*，通过在网页中将部分内容通过隐藏在看似无害的内容（如按钮）下，诱使用户点击。配合 `<iframe>` 使用的套路非常简单，假设攻击者希望对 Facebook 进行点击劫持：

1. 将一个访客诱骗到一个钓鱼页面（方式可以有很多种）
2. 页面本身看上去人畜无害，且带有一些诱导用户点击的内容（比如 `点击这里，赚大钱`，或者 `想寻求一些♂刺激吗？点击这里`）
3. 实际上，钓鱼页面将一个 `src` 指向 Facebook 站点的 `<iframe>` 嵌入到页面中，且这个 iFrame 元素是透明的，覆盖在诱导用户点击的区域上方（但访客是看不到的）
4. 只要用户尝试去点击，就会事实上点击 `<iframe>` 中的某个按钮，比如 `点赞` 等等。

就是这么简单的攻击方式，在2009年造成了一次[小轰动](http://shiflett.org/blog/2009/twitter-dont-click-exploit)：在 Twitter 上突然有大量的人开始转发一条 Twiiter：

> Don't Click: http://tinyurl.com/amgzs6

当访客点入到这个页面时，会发现这个页面里只有简单的一个按钮，上面写着 `Don't Click!`，出于好奇心，多数的访客都会尝试点一下这个按钮，而当按钮被点下去的瞬间，用户所使用的 Twitter 账号便会转发相同的一条推。

读者可以戳下[这个例子](https://javascript.info/article/clickjacking/clickjacking/)，查看页面元素感受下具体的攻击方式。

幸亏，浏览器为我们提供了相关机制来避免自己的站点被第三方随意嵌入。通过在页面的返回报头中设置 `X-Frame-Options`，我们可以控制自己的页面被引入的限制：

```sh
X-Frame-Options: DENY           # 不允许任何站点引用
X-Frame-Options: SAMEORIGIN     # 仅允许同源站点引用
X-Frame-Options: ALLOW-FROM https://example.com # 允许某个站点引用
```

比如说 `https://google.com` 就设置了同源引用的策略：

![](/images/google_x_frame_options.png)


## 几种使用姿势

尽管在设计之初，iFrame 可能只是扮演一个嵌入第三方内容的角色，但在 Web 开发的实际发展历程中，有很多功能是凭借 iFrame 实现的。

#### 在线编辑器

相信很多人有使用过类似 Codepen 或者 JSFiddle 一类的在线编辑器。这类编辑器通常由两部分组成，一部分支持用户在编辑框中编写代码，另一部分实时展示用户写入代码所对应的页面。这个 "实时展示" 的部分就是采用了 `<iframe>` 元素，其中包裹的 HTML 页面及效果正是使用了用户编写的代码。

读者也许会问，用户在编辑框里编写的 HTML，CSS 和 JS 代码是如何作用于 `<iframe>` 的？以下以 Codepen 为例子，介绍基本的实现流程。

***实时更新 HTML 和 JS 代码***

若用户修改 HTML 或 JS 编辑框内的代码，则拼接出一段 HTML 字符串，并发起一个 POST 请求。POST 请求中还带有一个随机生成的 key：

<figure><img src="/images/codepen_step_1.png"><figcaption>用户输入的 HTML 和 JS 代码都被包含在请求的 `html` 字符串中</figcaption></figure>

在延迟大约半秒之后 `<iframe>` 元素的 `src` 值被修改为对应 key 的一个 URL 地址：

<figure><img src="/images/codepen_step_2.png"><figcaption>注意 iFrame 元素的 `id` 属性，和上图请求中的 key 参数一样</figcaption></figure>

此处一个合理的猜测是：服务端在接受到 POST 请求后，根据请求中的 key 值生成了一个新的文件目录，同时在该目录下新建一个名为 `index.html` 的 HTML 文件。这样 `<iframe>` 在刷新后，所访问的页面就正好是 POST 请求中带上的 HTML 文本。

***实时更新 CSS 代码***

若用户修改的是 CSS 代码，页面不会发起类似上述的请求，而是通过 `window.postMessage` 的方式，同 `<iframe>` 元素进行跨域通信。以下是 Codepen 的实现代码：

```js
var CSSReload = {
    head: null,
    init: function() {
        this._storeHead(),
        this._listenToPostMessages()
    },
    _storeHead: function() {
        this.head = document.head || document.getElementsByTagName("head")[0]
    },
    _shouldHandleMessage: function(e) {
        return e.origin.match(/codepen/)
    },
    _listenToPostMessages: function() {
        // 监听主窗口（即 Codepen 主页面）中用 window.postMessage 发出的事件
        var e = this;
        window[this._eventMethod()](this._messageEvent(), function(t) {
            try {
                // 判断是否处理事件
                if (!e._shouldHandleMessage(t))
                    return;
                var s = JSON.parse(t.data);
                "string" == typeof s.css && e._refreshCSS(s)
            } catch (e) {}
        }, !1)
    },
    _messageEvent: function() {
        return "attachEvent" === this._eventMethod() ? "onmessage" : "message"
    },
    _eventMethod: function() {
        return window.addEventListener ? "addEventListener" : "attachEvent"
    },
    _refreshCSS: function(e) {
        // 删除 iFrame 窗口文档中原有的 <style> 样式
        // 插入新的 <style> 样式
        var t = this._findPrevCPStyle()
          , s = document.createElement("style");
        s.type = "text/css",
        s.className = "cp-pen-styles",
        s.styleSheet ? s.styleSheet.cssText = e.css : s.appendChild(document.createTextNode(e.css)),
        this.head.appendChild(s),
        t && t.parentNode.removeChild(t),
        "prefixfree" === e.css_prefix && StyleFix.process()
    },
    _findPrevCPStyle: function() {
        for (var e = document.getElementsByTagName("style"), t = e.length - 1; t >= 0; t--)
            if ("cp-pen-styles" === e[t].className)
                return e[t];
        return !1
    }
};
CSSReload.init();
```

可能读者会问，既然 `<iframe>` 窗口和主窗口都是位于同一个主域名（`codepen.io`）下，为什么不可以尝试用设置 `document.domain` 的方式，让主窗口可以直接通过 JS 操作 iFrame 呢？理论上这可能是可行的，但由于 Codepen 是全站使用 HTTPS 的（作为一个成熟的网站，你当然 [应该](https://github.com/jasonGeng88/blog/blob/master/201705/https.md) [使用](https://zhuanlan.zhihu.com/p/29022279) [HTTPS](https://en.wikipedia.org/wiki/HTTPS)），浏览器会禁止主窗口和 `<iframe>` 之间任何可能的 JS 相互调用。

#### 解决跨域请求问题

这个估计也是很多人初识 iFrame（或者说，实战 iFrame）的实际场景了，由于过程实在是太 hack，以及确实除了 hack 的技巧本身外并没有任何工程价值，所以我做出了一个艰难的决定：

> 我不详细讨论这个问题了。

毕竟在跨域请求方案上，早就有 [CORS](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS) 和 [JSONP](https://zh.wikipedia.org/zh/JSONP) 了，不去使用这种成熟方案，而来纠结 iFrame 的奇技淫巧的，恕我直言都真的是浪费自己的青春...

有兴趣的读者可以看 SegmentFault 上的 [这篇文章](https://segmentfault.com/a/1190000014223524#articleHeader4)，总结的很全面了（话说 SegmentFault 是做的越来越不错了，果然好东西都要靠积累，做时间的朋友啊...）

或者也可以看另一篇博客：[浅谈几种跨域的方法](https://wangningbo93.github.io/2017/06/16/%E6%B5%85%E8%B0%88%E5%87%A0%E7%A7%8D%E8%B7%A8%E5%9F%9F%E7%9A%84%E6%96%B9%E6%B3%95/)

#### Comet 中的永久帧

**Comet** 一词最早是由 Alex Russell（Dojo 库的作者）在 2006 年的一篇博客 *[Comet: Low Latency Data for the Browser](http://infrequently.org/2006/03/comet-low-latency-data-for-the-browser/)*  中首先提出，描述从服务端向浏览器“推送”数据的一系列手段，包括协议和具体技术实现。

如果从今天的角度来回看”向服务器推送数据“这个诉求，很容易就想到 WebSocket，对吧？但事物是处于不断发展的阶段的...WebSocket 协议在 2011 年才成为标准，浏览器厂商也是在 2010~2011 年之间先后推出了支持该协议的版本。但需求，是一定要 **通过各种手段** 完成的，Comet 就是各种手段的一种统称，也被称为 "Ajax Push", "Reverse Ajax", "HTTP Server Push" 等等。

Comet 的实现有若干种具体的手段：

1. 长轮询（Long polling）
2. 永久帧（Forever Frame）
3. XHR 流（XMLHttpRequest Streaming）

这里要说的是永久帧的实现。所谓“永久帧”，是指在当前文档内创建一个 `<iframe>` 元素，其文档所指向的地址会返回一个 HTTP 1.1 的 [trunked 编码](https://zh.wikipedia.org/wiki/%E5%88%86%E5%9D%97%E4%BC%A0%E8%BE%93%E7%BC%96%E7%A0%81) 文档。根据 trunked 编码文档的特性，服务器可以将整个文档分成多个部分发送给浏览器端。

通过这种方式，我们可以将 `<iframe>` 的文档看做是一个不断增加内容的文档，那么只需要在增量文档中生成 `<script>` 标签调用预定义的回调函数即可。

具体的实现为，首先定义一个生成永久帧的函数：

```js
function foreverFrame (url, callback) {
  var iframe = document.createElement('iframe')
  var randomName = 'callback_' + (Math.random() + '').substring(2)
  iframe.style.display = 'none'
  iframe.src = url + '?callback=parent.' + randomName
  window[randomName] = callback
}
```

在调用该函数后，生成的 `<iframe>` 所对应的文档的返回内容为：

```html
<script>
parent.callback_1310442051852272('hello world');
</script>

<script>
parent.callback_1310442051852272('hello mars');
</script>

<!-- 不断的增加中... -->
```

原理上，只要 `<iframe>` 元素对应的 trunked 编码的文档一直在输出内容，它就可以是被视作是”永久“的，且可以保证服务端持续地向浏览器输出内容。

但这里也有一个明显的弊端：在 IE 和 Firefox 下，采用这样的方案会让浏览器的进度条一直显示加载中，且 IE 的 tab 图标会不断的转动，表示正在进行加载。 Google 通过采用类型为 `htmlfile` 的 ActiveXObject 的技巧来解决了这个问题。[传送门](https://infrequently.org/2006/02/what-else-is-burried-down-in-the-depths-of-googles-amazing-javascript/)

关于长轮询和 XHR 流的实现，这里不做赘述，有兴趣了解详情的可以阅读 [Comet - 服务器推送解决方案](http://imweb.io/topic/565abde9823633e31839fc0e)

#### 其他

除了以上几个例子，iFrame 还可以实现无刷新文件上传，浏览器多页面间的通信，或者是音乐播放器（同一浏览器多个tab共享一个播放器）等功能，具体可以看知乎上的 [这个回答](https://www.zhihu.com/question/20653055)

## Viewport 的小麻烦

当页面被嵌入在 `<iframe>` 时，页面上的某些元素的定位规则会受到相应的影响。在解释具体的影响之前，首先我们要解释一下包含块（containing block）的概念。

#### `containing block`

对于一个元素来说，它的大小和位置通常受这个元素的 **包含块** 所影响。比如说，如果该元素的 `width`, `height`, `padding`, `margin` 属性的值是百分比的话，那么在计算这些值的实际大小时，将使用包含块的内容区域的宽度或者宽度来作为计算参考；如果该元素是绝对定位的元素（即 `position` 属性为 `absolute` 或 `fixed`），则元素的偏移属性（`left`, `right`, `top`, `bottom`）的值将相对于包含块进行计算，从而直接影响元素所处的位置。

浏览器通过元素的 `position` 属性值，有不同地指定元素的包含块的策略，具体可以查看 [MDN 的文档](https://developer.mozilla.org/en-US/docs/Web/CSS/Containing_block)

#### `<iframe>` => Viewport

我们需要说明的是：`<iframe>` 元素事实上创建了一个新的 Viewport。根据 [CSS2.1 的规范](https://www.w3.org/TR/CSS21/visudet.html#containing-block-details)：

> The containing block in which the root element lives is a rectangle called the **initial containing block**. For continuous media, it has the dimensions of the viewport and is anchored at the canvas origin...

从这段说明中我们可以得到两个结论：

1. 文档的根元素（`<html>`）就是该文档的初始化包含块（initial containing block）
2. 同时这个元素关联一个 Viewport

而 [关于 Viewport 的规范](https://www.w3.org/TR/CSS21/visuren.html#viewport) 则有：

> Useragents for continuous media generally offer users a **viewport** (a window or other viewing area on the screen) through which users consult a document. User agents may change the document's layout when the viewport is resized (see the initial containing block)

我们可以简单的将 Viewport 理解为用户查看文档的一个窗口。而对于 `<iframe>` 这样的嵌入文档，根据规范的说法，这事实上创建了一个新的 Viewport，且由于出现了一个新的文档对象，自然有其独立的初始化包含块（initial containing block）。

P.S：关于 Viewport 的详细介绍，可以查看学弟 @Mactavish 写的 [这篇文章](https://macsalvation.net/2018/05/23/dive-into-viewport/)

有了这样的理论基础，我们来看看两个特殊的元素定位问题。

#### `position: fixed`

相信很多写过自定义弹窗或者 Modal 组件的同学，都会使用 `position:fixed` 配合相应的偏移属性来实现相对于可视窗口的绝对居中效果。

但如果弹窗的元素是在一个 `<iframe>` 中，而该 `<iframe>` 元素又恰好只占用了父文档其中一部分的空间，那么实际上这个弹窗的居中效果是相对于 `<iframe>` 元素的，比如以下的这个例子，虽然黄色区块已经被设置成了 `position: fixed`，但显然其显示的位置不会在当前整个页面的正中央。

<p data-height="320" data-theme-id="0" data-slug-hash="XBpqqv" data-default-tab="css,result" data-user="allenfantasy1018" data-embed-version="2" data-pen-title="XBpqqv" class="codepen">See the Pen <a href="https://codepen.io/allenfantasy1018/pen/XBpqqv/">XBpqqv</a> by Zeqiu Wu (<a href="https://codepen.io/allenfantasy1018">@allenfantasy1018</a>) on <a href="https://codepen.io">CodePen</a>.</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>

（当然，像上述例子的情况实际上是一个双重嵌套 iframe，读者可以打开 devtool 自己查看）

当然，这样的设定是浏览器有意为之的，所以如果确实有必要希望弹窗的位置在整个浏览器窗口的正中间，开发者需要使用 JS 的手段实现，具体做法可以参考 Andy Langton 的 [这篇博客](https://andylangton.co.uk/blog/development/get-viewportwindow-size-width-and-height-javascript)

#### Viewport-percentage length

顾名思义，**Viewport-percentage length** 指和当前 Viewport 相关的长度单位，如 `vw`, `vh`, `vmin`, `vmax`。根据 [CSS 规范](https://www.w3.org/TR/css-values/#viewport-relative-lengths) 的说法：

> The *viewport-percentage lengths* are relative to the size of the **initial containing block**. When the height or width of the initial containing block is changed, they are scaled accordingly.

呵！原来 `vh` 和 `vw` 的计算参考系并不是当前浏览器的窗口大小，而是初始化包含块的高度和宽度，那么问题来了：由于 `<iframe>` 的独立文档会有单独的初始化包含块（就是其文档的 `<html>` 元素），也就是说：

在 `<iframe>` 文档中的元素，其 `vw` 和 `vh` 等长度单位的计算是相对于 `<iframe>` 元素的。[StackOverflow](https://stackoverflow.com/questions/34057239/css-vh-units-inside-an-iframe/44634369#44634369) 上也有人就这个问题做了详细的解答。

## Recap

嗯！这就是所有关于 `<iframe>` 要讨论的内容了，让我们来简单的回顾一下：

* 一个 iFrame 对应一个独立的浏览上下文（Browsing Context）
* iFrame 是出于嵌套第三方页面以丰富页面内容展示的需要而出现的，但围绕它可以实现许多特殊的功能
* 浏览器通过同源策略避免 iFrame 和主页面间的互相直接调用，但可以利用 `window.postMessage` 来让 iFrame 和主页面间进行通讯
* 可以通过 `X-Frame-Options` 控制页面被嵌套的策略
* iFrame 页面元素的样式需要注意相对于 Viewport 的处理

## Reference

* https://developer.mozilla.org/en-US/docs/Glossary/Browsing_context
* https://html.spec.whatwg.org/multipage/browsers.html#windows
* http://w3c.github.io/html/browsers.html#sec-browsing-contexts
* https://msdn.microsoft.com/en-us/library/dn705664(v=vs.85).aspx
* http://netsekure.org/2015/12/06/chromium-internals-documents-windows-browsing-contexts/
* https://developer.mozilla.org/en-US/docs/Web/HTML/Element/iframe
* https://developer.mozilla.org/en-US/docs/Web/HTML/Element/frameset
* https://developer.mozilla.org/en-US/docs/Learn/HTML/Multimedia_and_embedding/Other_embedding_technologies
* https://en.wikipedia.org/wiki/HTML_element
* [https://en.wikipedia.org/wiki/Sandbox_(computer_security)][1]
* https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP
* https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
* https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security
* https://zh.wikipedia.org/wiki/WebSocket
* https://developer.mozilla.org/en-US/docs/Glossary/HSTS
* https://javascript.info/clickjacking
* https://blog.mozilla.org/security/2013/12/12/on-the-x-frame-options-security-header/
* https://heycam.github.io/webidl/
* https://www.ibm.com/developerworks/cn/web/wa-lo-comet/#N10101
* http://imweb.io/topic/565abde9823633e31839fc0e
* https://segmentfault.com/a/1190000014223524
* http://www.fedlab.tech/archives/395.html
* [https://en.wikipedia.org/wiki/Comet_(programming)][2]

[1]: https://en.wikipedia.org/wiki/Sandbox_(computer_security)
[2]: https://en.wikipedia.org/wiki/Comet_(programming\)