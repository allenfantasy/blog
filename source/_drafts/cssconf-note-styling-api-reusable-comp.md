
### [Styling API for Reusable Component](https://img.w3ctech.com/Styling-API-for-Reusable-Components.pdf)

个人最喜欢的是 E0 的演讲，因为这确实触及到了我们团队在设计组件库时遇到的困惑，即如何管理组件库的 CSS，既保证和 HTML 结构的足够内聚，但同时也提供方便简洁的外部修改方式。

E0 首先大谈了前端样式开发的历史进程：

* 史前 HTML：样式规则以 HTML 标签属性的方式（`bgcolor`）存在...存在可怕的耦合问题，无法有效的重用；Netscape 提出 JSSS（JavaScript Style Sheets），写起来竟然有点像 CSS-in-JS，但后来被抛弃。
* CSS 的诞生（1996）：在最初 HTML 更多作为 "文档" 的意义存在时，CSS 所起的作用更多的是允许读者对既有文档进行修改「注：可以理解为用各种浏览器插件来优化页面，如印象悦读等」，这达到了一个 **Author/reader balance**；这也意味着页面