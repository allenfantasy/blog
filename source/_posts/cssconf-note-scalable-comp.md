title: "CSSConf 笔记: Building Scalable Components"
date: 2018-05-25
tags:
- CSS
---

原分享地址： https://zellwk.github.io/scalable-components

Zell 用他的博客作为例子解释了如何去构建可伸缩的（Scalable）的组件：在不同尺寸的页面中，一个组件应该有不同的展现形式（大小，内容，内部位置关系等）。

Zell 提出了三个 Scaling 的概念：

1. 按比例的（Proportional Scaling）
2. 响应式的（Responsive Scaling）
3. 模块化的（Modular Scaling）

#### Proportional Scaling

在按比例缩放中，组件应当根据以下两个对照值进行伸缩（scale）：

* font-size
* viewport

其中我们要使用 *相对(字体大小)单位*，比如：

* `em` 相对当前对象内文本的字体大小
* `ch` 代表元素所用字体 font 中 "0" 这一字形的宽度（"0", U+0030）[MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/length)
* `%` 相对于父容器的 *同一属性* 的值的大小 [MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/percentage)

在编写同一个组件的不同大小尺寸时，可以将组件公共的 CSS 组织在一起，达到类似 bootstrap 一类框架的风格。也可以利用 Sass @mixin 可以实现更简练的风格：

```scss
@mixin button {
	display: inline-block;
  padding: 0.5em 0.75em;
  /* Other common properties */
}

.button-large {
  @include button;
  font-size: 2em;
}
```

```html
<button class="button-large">A large button</button>
```

另外一种控制相对大小的方法是使用相对于视口（viewport）比例的长度单位，如：

* `vw`
* `vh`
* `vmin`
* `vmax`

结合 `calc` 函数，我们可以 [实现响应式变动文字大小](https://zellwk.github.io/scalable-components/demo/rt.html)

Mike Riethmuller 在 [Responsive And Fluid Typography With vh and vw Units](https://www.smashingmagazine.com/2016/05/fluid-typography/) 中详细地描述了更加具体的实践；有兴趣的同学可以看看这个 [Demo](https://codepen.io/allenfantasy1018/pen/VxNJXw?editors=1100) 和这个 [Sass Snippet](https://www.sassmeister.com/gist/7f22e44ace49b5124eec)

#### Modular Scaling

*TODO*

#### Responsive Scaling

*TODO*

## Reference

* [Is the default font-size of every browser 16px? Why?](https://stackoverflow.com/questions/29511983/is-the-default-font-size-of-every-browser-16px-why)
* [Breakpoints And The Future Of Websites](https://www.smashingmagazine.com/2014/07/breakpoints-and-the-future-websites/)
* https://codepen.io/MadeByMike/pen/VvwqvW
* http://alistapart.com/article/more-meaningful-typography
* [Type Scale](http://type-scale.com/)
* [Modular Scale](http://www.modularscale.com/)

## TODO

* 读完 https://www.smashingmagazine.com/2016/05/fluid-typography/#maintaining-vertical-rhythm
