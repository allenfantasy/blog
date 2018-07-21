title: 我不知道的XSS
---

* [经验分享 | XSS手工利用方式](https://cloud.tencent.com/developer/article/1043862)
* [Why is the Samy Worm considered XSS?](https://security.stackexchange.com/questions/37362/why-is-the-samy-worm-considered-xss?utm_medium=organic&utm_source=google_rich_qa&utm_campaign=google_rich_qa)
* [XSS 蠕虫 & 病毒](https://wizardforcel.gitbooks.io/xss-worms-and-viruses/content/0.html)

> XSS的本质是破坏文档、代码的结构。使用的漏洞是程序没有对用户输入（可以是提交表单、也可以是url）的字符编解码，导致把用户输入的内容以代码形式输出到页面显示或执行。

### XSS 的分类

一种常见的说法是：

* 存储型XSS：数据库中已有数据返回给客户端，若数据未转义就被浏览器渲染，就可能导致 XSS
* 反射型XSS：用户输入的存在XSS攻击的数据，发送给后台，后台未进行过滤又返回给客户端，被浏览器渲染导致 XSS
* DOM-XSS：纯粹发生在客户端的 XSS，如在 url 中带上攻击脚本 `<script>alert(document.cookie)</script>` 再在页面代码中利用 `document.write` 来在页面中渲染该 script 标签从而执行攻击代码（`alert`）

~~但其实我觉得这看的我头大~~

而 [贺老说](https://www.zhihu.com/question/26628342/answer/33572866)：

> XSS就是XSS。所谓“存储型”、“反射型”都是从黑客利用的角度区分的。对于程序员来说意义不大，反而是误导。只有“DOM-based”型略有不同。
>

基于对贺老的回答，我的理解是：

由于 XSS 起作用总是通过在页面上输出 HTML 或者执行 JS 来实现的，那么我们有两种区分方法：

* 按输出内容区分：用户输入最终被输出成 HTML 的一部分，还是 JS 代码的一部分？
* 按输出方式区分：拼接 HTML 模板或者 JS 代码时，是在服务端（如 PHP）完成的，还是在客户端由 JS 完成的？

如果我们按输出方式区分，那么所谓的 DOM-XSS 无非就是：

> 在客户端对获取的数据（可能是页面上的用户输入「反射型」，或者 API 接口返回的数据「存储型」）进行页面渲染的过程中出现的 XSS

贺老给出了一个很经典的例子，按照概念来说，既是 "DOM-XSS" 也是 "存储型XSS"：

> 百度翻译不久前爆了一个xss漏洞。百度翻译除了给出翻译结果之外，如果这个词有百度百科词条的话，也会通过ajax拉百度百科的内容并显示在页面上。问题他们是直接把得到的百度百科内容通过innerHTML插入页面，并且没有做escape，这导致百度百科的内容中的html标签（比如那些html、css、js相关的词条上的代码sample）直接就生效了。可想而知，你只要修改一下百度百科上的词条就能轻松让对应的百度翻译页面带有XSS漏洞……

### HTML escape

简单的解释：将 HTML 字符转义，使其正确的显示在网页中。

*TODO* 需要转义的字符
*TODO* HTML escape 和 XSS 的关系

### 防御

1. 对用户请求参数进行变量类型控制
2. 用户输入的数据若为字符串，则对 `<`, `>`, `/`, `'`, `"`, `&` 做实体化转义
