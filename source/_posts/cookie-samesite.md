title: Cookie SameSite
date: 2020-02-15

---

对 `SameSite` 做些简单的研究探讨，没什么新鲜的，仅做记录参考。

<!--more-->

## 为何需要 SameSite: Cookie 现存的问题

在解释为何要 SameSite 时，首先我们解释下 first-party cookie 和 third-party cooki

**First-Party Cookies**: 和当前站点域名对应的 cookie
**Third-Party Cookies**: 来自其他域名的 cookie

以下我们探讨 cookie 现存的两个问题

### 占用请求带宽

第三方 cookie 在某些特定的场合的确是有用的，比如说你在页面中嵌入一个 YouTube 的播放器，那么如果用户的浏览器已经登录了 YouTube 那么，第三方 cookie 就可以保证用户可以在嵌入的播放器中使用已登录用户可使用的功能比如 `稍后再看`。

但从总体来看，第三方资源中的 cookie 并不 **总是** 有用的，很多 cookie 其实不是必须的，但在对第三方资源（CDN 等）发起请求时还是被带上了。

由于上传的带宽相比下载带宽更加的有限，这就增加了 TTFB (Time to first byte) 的时间。

### CSRF

首先有必要简单的介绍 CSRF：

Cross-site request forgery (CSRF) 是一种常见的网页攻击手段。通过诱导用户访问钓鱼网站，并在网站中嵌入对攻击目标站点（如银行，金融产品，社交网络）的表单，诱导用户点击触发表单请求，就可以在当前用户已经登录目标站点（即：浏览器中含有 cookie）的前提下，在用户不知情时发起相关的操作（如：转账）。

目前对 CSRF 的一个通用的防御方式，是在表单中增加一个随机的 token 字段，该 token 只能由表单所在的站点后台服务产生并派发到网页端；生成的 token 同时和当前用户的 session 关联；在接受表单请求时，服务端将校验当前用户的 session 是否与表单中携带的 token 是匹配的。如果是的话则允许表单通过。

由于 token 的生成是无法由第三方控制的，所以这就简单地杜绝了 CSRF 的可能性。

但利用第三方 cookie 对其域名服务进行攻击的可能性，理论上还是存在的，所以还是需要防备的。

综上所述，我们需要的是能够对 cookie 进行不同的区分处理：对于不需要在第三方使用到的 cookie，最好是可以让浏览器在第三方站点请求我方站点资源时不带上；同时也可以允许第三方请求我方资源时带上某些 cookie 以保证用户的体验。

## Here comes SameSite

`SameSite` 正是解决这个问题的方法。它是一个设置 Cookie 时用到的属性。可以用于声明你的 cookie 是否限定在同一个 site 下使用。

当然我们有必要说明下这里的 `site`（以下称为”站点“）的含义：

首先是 [Public Suffix List](https://publicsuffix.org/list/public_suffix_list.dat) 列举了所有的顶级域名后缀，如：

  * `.dev`
  * `.com`
  * `.github.io`

那么 `站点` 就是指顶级域名后缀加上在后缀之前的那一部分域名，如：

* `web.dev` 
* `xxx.github.io`

在绝大部分的情况下，我们可以将 `站点` 简单理解为大家常说的 `一级域名`。那么 `SameSite` 所约束的同一个 site 其实就是指：

> 同一个 `一级域名` 下的各个子域名间的资源属于同一个站点

### 使用 SameSite

我们可以在 `Set-Cookie` 时指定 `SameSite` 属性：

```
Set-Cookie: <cookie-name>=<cookie-value>; Domain=<domain-value>; HttpOnly; Secure; SameSite=<same-site-value>
```

`SameSite` 可以设三个值：`Strict`，`Lax` 和 `None`。用一个图来清晰表达它们的使用场景：

![](https://web.dev/samesite-cookies-explained/samesite-none-lax-strict.png)

#### `SameSite=Strict`

如果设置 `SameSite` 为 `Strict`，则完全禁止第三方 cookie，只有用户 **已经在站点** 上时发起的请求才可以带上这个 cookie。

那就意味着任何从外部（我的理解是 `Referer` 不是站点本身，或者没有 `Referer`）访问这个站点的请求都不会带上 cookie。

盲目地使用该值最可能触发的问题是影响用户 **第一次访问（top-level navigation）** 站点的体验，比如说如果把维持用户当前登录状态的 cookie 设成 `SameSite` 的，那么用户即便之前已经登录过站点，下一次直接从浏览器里输入地址（或者通过书签）访问，都是无法看到登录后内容的。又或者说用户在一个第三方站点里点击一个微博的链接，假如微博将登录 cookie 设成 `SameSite` 用户跳转后将发现自己处于未登录状态。

所以在 [RFC6265](https://tools.ietf.org/html/draft-ietf-httpbis-rfc6265bis-03#section-5.3.7) 规范中也提到：

> ... As discussed in [Section 8.8.2](https://tools.ietf.org/html/draft-ietf-httpbis-rfc6265bis-03#section-8.8.2), this might or might
> not be compatible with existing session management systems.

所以如果是涉及到必须要用户在站点上操作的功能，将 cookie 设置为 `Strict` 比较合适，而对于第一次访问时使用的 cookie，一个更切实的做法是使用 `Lax` 属性值。

#### `SameSite=Lax`

相比于 `Strict` 模式，`SameSite=Lax` 相对宽松一些，允许第一次导航到目标网址也带上 cookie。导航到目标网址有以下几种情况：

* 链接：`<a href="...">...</a>`
* 预加载：`<link ref="prerender" href="..." />`
* GET 表单：`<form method="GET" action="...">`

这里值得留意的是 [predender](https://www.chromium.org/developers/design-documents/prerender) 的方式，在 `Lax` 模式下也是可以带上 cookie 的，如 [规范中提到的](https://tools.ietf.org/html/draft-ietf-httpbis-rfc6265bis-03#section-5.3.7)，站点的开发者应该小心处理关于用户权限相关的 cookie，只允许控制读权限的 cookie 使用 `Lax` 模式，以降低 CSRF 攻击风险。

> ... Features like `<link rel='prerender'>` prerendering can be
> exploited to create "same-site" requests without the risk of user
> detection.
> ...
> When possible, developers should use a session management mechanism
> such as that described in [Section 8.8.2](https://tools.ietf.org/html/draft-ietf-httpbis-rfc6265bis-03#section-8.8.2) to mitigate the risk of CSRF
> more completely.

#### 默认模式

在规范中说明的，如果在设置 cookie 时并未显式声明 `SameSite` 则浏览器会认为该 cookie 可以在任何地方的请求中带上，这样的模式也可以通过 `SameSite=None` 显式地声明。注意声明时需要同时加上 `Secure` 否则声明无效：

```
Set-Cookie: widget_session=123; SameSite=None; Secure
```

虽然规范是允许我们不声明的，但值得一提的是在 Chrome 中，目前如果出现了没有声明 `SameSite` 的 cookie，会弹出一个警告信息：

> A cookie associated with a cross-site resource at http://xxx.yourwebsite.com was set without the `SameSite` attribute. A future release of Chrome will only deliver cookies with cross-site requests if they are set with `SameSite=None` and `Secure`. You can review cookies in developer tools under Application>Storage>Cookies and see more details at https://www.chromestatus.com/feature/5088147346030592 and https://www.chromestatus.com/feature/5633521622188032.

我简单翻译下：Chrome 在未来的某个版本中会只允许显式声明了 `SameSite=None` 和 `Secure` 标识的 cookie 在跨域中使用。所以为了保证后续的稳定性，建议开发者还是显式地声明自己的 cookie 比较好。

## 那怎么做

开发者需要对自己站点所提供的 cookies 做一次全面的检查（不管 cookie 是由服务端通过返回的报头中下发的，还是通过 JS 脚本的 `document.cookies = xx` 声明的），对不同情况下使用的 cookies 做 `SameSite` 配置：

1. 对于仅可以在自己站点页面上触发的请求所依赖的 cookie，设 `SameSite=Strict`
2. 对于仅和展示内容相关（比如登录状态）所依赖的 cookie，设 `SameSite=Lax`
3. 对于完全不需要在请求中使用，但又确实需要用到 cookie（比如统计脚本中种的），设为 `SameSite=None`

虽然规范中的 SameSite 的默认值是 None，但目前已经有相应的 [提案](https://tools.ietf.org/html/draft-west-cookie-incrementalism-00) 建议将默认值修改为 `SameSite=Lax` 以及要求 `SameSite=None` 必须指定 `Secure`，且各个浏览器将陆续跟进：

* Chrome 将从 v80 开始[将 cookie 的默认值设为 `SameSite=Lax`](https://www.chromestatus.com/feature/5088147346030592)，[禁止不安全的 SameSite=None cookie](https://www.chromestatus.com/feature/5633521622188032) -- v80 在 2020.2.4 已进入稳定版。
* Firefox 从 [v69 开始作为默认行为](https://groups.google.com/d/msg/mozilla.dev.platform/nx2uP0CzA9k/BNVPWDHsAQAJ)
* Edge 也[计划跟进](https://groups.google.com/a/chromium.org/d/msg/blink-dev/AknSSyQTGYs/8lMmI5DwEAAJ)

所以配置 `SameSite` 属性是很有必要的！建议开发者都做起来。

## Reference

* https://web.dev/samesite-cookies-explained/
* https://tools.ietf.org/html/draft-ietf-httpbis-rfc6265bis-03#section-4.1