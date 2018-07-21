====以下是各种草稿

On the X-Frame-Options Security Header:
https://blog.mozilla.org/security/2013/12/12/on-the-x-frame-options-security-header/

### 沙盒

> 沙盒（Sandbox）是一种安全机制，为运行中的程序提供的隔离环境。通常是作为一些来源不可信、具破坏力或无法判断程序意图的程序提供控制实验之用

由于 iFrame 在一个页面中创建了一个内嵌的浏览上下文（browser context）
*TODO*

====

`Strict-Transport-Security`, HSTS:

HSTS 技术指服务器通过返回 `Strict-Transport-Security` 报头，告知浏览器必须使用 HTTPS 协议请求数据，并在接下来的每次请求都使用 HTTPS 协议：

```
Strict-Transport-Security: max-age=<expire-time>
```

当浏览器第一次用 HTTPS 协议访问你的站点，你的站点返回一个 `Strict-Transport-Security` 报头，浏览器将会把该信息记录下来。之后这个浏览器再尝试访问你的站点时，会自动使用 HTTPS 协议（不管在地址栏中输入的 url 是否带有 HTTPS 协议）

You should only embed third party content when completely necessary.

### 使用 HTTPS

1. HTTPS reduce the chance that remote content has been tampered with in transit.
2. HTTPS prevents embedded content from accessing content in your parent document, and vice versa.

从第二点角度出发考虑，若使用 iFrame 嵌套某个第三方的 url，这个 url 必须是 HTTPS 的

### `sandbox` attribute

> ... A container for code where it can be used appropriately -- or for testing -- but can't cause any harm to the rest of the codebase (either accidental or malicious) is called a [sandbox](https://en.wikipedia.org/wiki/Sandbox_(computer_security))


### 1.1 window

The *Window* has an *associated Document*, which is a Document object, it is set when the Window object is created, and only ever changed during navigation from the initial about:blank Document

return the current window (itself), return this Window object's browsing context's WindowProxy object.

* `window.window`
* `window.frames`
* `window.self`

* `window.document` return the *associated Document*

* `window.defaultView`

window 对象可以用 `window[name]` 的语法指向某个对象，其中包括：

* *document-tree child browsing contexts*
* 某个含有 `name` 属性为 "name" 的 `<embed>`, `<form>`, `<frameset>`, `<img>`, `<object>` 元素
* 某个 `id` 为 "name" 的 DOM 元素

以上指向优先考虑获取子浏览器上下文，可以称为 `nested browsing context`

#### WindowProxy

`WindowProxy` 不是一个 JS API 或者方法，无法在浏览器中显式获得。

更像是一个虚拟或者外来的对象，持有一个对 `Window` 对象的引用。

> A WindowProxy is an exotic object that wraps a Window ordinary object, indirecting most operations through to the wrapped object.

### 1.2 Document

### active document

*TODO*

### 1.3 browsing context

*Definition:*

> A Browsing context is the environment in which a browser displays a *Document* (normally a tab nowadays, but possibly also a window or a frame within a page)
> Each browsing context has a specific *origin*, the origin of the active document, and a history that lists all the displayed documents in order.
> Communications between browsing contexts is severely restricted.

> Each embedded browsing context has its own *session history* and *active document*. The browsing context that contains the embedded content is called the parent browsing context. The topmost browsing context—the one without a parent—is usually the browser window, represented by the Window object.

Browsing context 有以下几种:

1. 浏览器的一个 tab 或者是一个窗口
2. `<iframe>`
3. frames in a `<frameset>

一个 Browsing context 有包含以下内容：

1. 一个 WindowProxy 对象
2. 一份 session 历史记录, which lists the Document objects that the browsing context has presented, is presenting, or will present. A browsing context's active document is its WindowProxy object's [[window]] internal slot value's * associated Document*

> A browsing context has a strong reference to each of its Documents and its WindowProxy object, and the user agent itself has a strong reference to its top-level browsing contexts.

#### nested browsing context
https://html.spec.whatwg.org/multipage/browsers.html#nested-browsing-context

* 某些元素如 `<iframe>` 可以实例化一个新的 browser context，这些元素被成为 **browsing context container**
* 每个 container 都持有一个 **nested browsing context**，或者是一个 browser context 对象，或者为 null

#### child browsing context

如果一个 browsing context 是一个 browsing context container 的 nested browsing context，且这个 container 对应的 DOM 元素是 *连接的*[1]，则称其为 **child browsing context**

如果这个 DOM 元素（节点）尚未连接（到 Document，即还没有添加到文档中），则称对应的 browsing context 是 *document-tree child browsing contexts*


*origin*
https://developer.mozilla.org/en-US/docs/Glossary/origin

`window.open` 可以打开一个新的窗口
https://developer.mozilla.org/zh-CN/docs/Web/API/Window/open

*active document*
https://html.spec.whatwg.org/multipage/browsers.html#active-document

*delaying load events mode*
https://html.spec.whatwg.org/multipage/parsing.html#delay-the-load-event

*content document*
https://html.spec.whatwg.org/multipage/browsers.html#concept-bcc-content-document

* `window.top`
* `window.parent`
* `window.frameElement`

## 1.4 `<iframe>`

`<iframe>` 标准的说法应该叫 **HTML Inline Frame Element**，我们对他通常的印象是可以通过 `<iframe>` 在一个页面中嵌入另一个页面。以下是 MDN 的解释：

> The HTML Inline Frame Element (`<iframe>`) represents a nested browsing context, effectively embedding another HTML page into the current page. You can include any number of `<iframe>` elements within a document, each of which embeds another document inside `<body>` of a page.

https://stackoverflow.com/questions/25098021/securityerror-blocked-a-frame-with-origin-from-accessing-a-cross-origin-frame

https://stackoverflow.com/questions/7570496/getting-the-document-object-of-an-iframe
https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/iframe

iFrame CSS things

> It is possible but only if the iframe's domain is the same as the parent
> The domain, port and protocol have to be the same, doesn't work with subdomains either.


https://stackoverflow.com/questions/217776/how-to-apply-css-to-iframe

iFrame is a Viewport

https://stackoverflow.com/a/44634369/1301194
https://www.w3.org/TR/css-values/#viewport-relative-lengths
https://www.w3.org/TR/CSS21/visudet.html#containing-block-details

**Same Origin Policy**

Theoretically one could access an `<iframe>`'s window object by using `iframe.contentDocument`, but some error occurs when trying so:

> Uncaught DOMException: Failed to read the 'contentDocument' property from 'HTMLIFrameElement': Blocked a frame with origin "http://localhost:4200" from accessing a cross-origin frame.

So there exist some cross-origin problem when the embedded document's url is actually not coming from the same origin as the outer page (BI's page) - [*Same-origin security policy*](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy). [This SO answer](https://stackoverflow.com/a/25098153/1301194) explains it well, also mentioning the solution:

> ... **if you own both the pages, you can work around this problem using window.postMessage and its relative message event**

In main page:

```js
var frame = document.getElementById('your-frame-id'); 

frame.contentWindow.postMessage(/*any variable or object here*/, '*');
```

In the `<iframe>`: 

```js
window.addEventListener('message', function(event) { 

    // IMPORTANT: Check the origin of the data! 
    if (~event.origin.indexOf('http://yoursite.com')) { 
        // The data has been sent from your site 

        // The data sent with postMessage is stored in event.data 
        console.log(event.data); 
    } else { 
        // The data hasn't been sent from your site! 
        // Be careful! Do not use it. 
        return; 
    }
});
```
## TOC

1. Browsing Context (ok)
2. `<iframe>` 由来 (ok)
3. iFrame 的特点
    * 安全性?
    * 带 origin 的独立窗口 => Same Origin Policy
        * embedded limit
        * 和外界的通信: 如何与父窗口进行相互控制? interact with parent window
            * `window.postMessage()`
    * 带文档的独立窗口 => CSS problem
4. some hacky usage

## iFrame 的安全性

分两个方面来谈：

* 引入第三方的 iFrame

* only embed when necessary
* use HTTPS
    * 草稿 "使用HTTPS"
* use `sandbox` attribute
    * sandbox: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/iframe#attr-sandbox
    * https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/iframe
* Configure CSP directives
    * CSP directives
        * `Content-Security-Policy` 报头 https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy
        * CSP1.0 就说用 `frame-src`, CSP2 里又说 `frame-src` deprecated 要用 `child-src`, WTF >>>
        * `frame-src` https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/frame-src
        * `child-src` https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/child-src
        * *TODO* 写一个demo说明
        * *TODO* HTML5Rocks 写 CSP 的文章: https://www.html5rocks.com/en/tutorials/security/content-security-policy/
    * X-Frame-Options https://developer.mozilla.org/en-US/docs/Web/HTTP/X-Frame-Options: avoid clickjacking attacks
        * Clickjacking https://en.wikipedia.org/wiki/Clickjacking

## Code

* [Window.open](https://codepen.io/allenfantasy1018/pen/rrNKEm?editors=1010)
