title: Hexo + Github Page 搭建博客
tags:
---

在 第2届广州创客马拉松 的守夜期间，为了不让自己睡着，突然想起了要搭建自己的博客。

有几个可选的方式：

* jekyll
* octopress
* hexo
* 从头搭建一个

由于最近用 Node 多一点（Ruby 的相当生疏），所以对 jekyll 和 octopress 不大有信心；而从头搭建一个又比较累（懒），加上[鑫磊也干了](http://threegoldstone.com/)，所以我还是用 hexo 吧。

[Hexo](https://hexo.io/) 是一个基于 Node 的博客生成工具，支持 Markdown 语法写作。可以阅读 [这篇文章](http://ibruce.info/2013/11/22/hexo-your-blog/) 来了解详细的各种折腾方法。

对于 Windows 用户我会更加推荐 Hexo，因为在 Windows 上基于 Ruby 的应用可能会有些坑。~~虽然我是一个 Mac 党，并没有这样的问题。~~

本文主要记录我搭建本博客的基本步骤和流程，基于 Hexo 和 Github Page

### 购买域名

如果没有购买域名的话，基于 Github 的博客就只能跑在 `[username].github.io` 上了，而我的 Github 用户名又很长。。所以还是买一个域名吧。我用的是 [namecheap](https://www.namecheap.com/)，只需要登陆后，在首页输入想要的关键字，就可以查看到可选的域名列表，以及具体的价格。推荐大家买 `.me` 或者 `.us` 的域名，另外 `.ninja` 也挺好玩的。

### 安装 Hexo

Hexo 是基于 Node 的博客工具。还没有安装 Node 的童鞋可以根据 [这个教程](https://cnodejs.org/topic/5338c5db7cbade005b023c98) 来安装。

安装指令：`npm install -g hexo`

### 利用 Hexo 搭建博客原型

在命令行中，通过以下的指令来生成博客：

```sh
hexo init blog  # 生成一个博客的原型，存放在 blog 文件夹下
cd blog
npm install     # 安装 npm 包依赖
hexo generate   # 生成静态页面
hexo server     # 启动 hexo 服务器，访问 localhost:4000 查看效果
```

这样就得到了博客，存放在 blog 文件夹下。接下来看看文件夹内部的构造：

```sh
.
├── _config.yml         # 配置文件
├── db.json             # （可以忽略）
├── node_modules/       # node 模块
├── package.json        # （可以忽略）
├── public/             # 生成的静态页面存放路径
├── scaffolds/          # 页面/文章的模板
├── source/             # 页面/文章的源码（markdown 文件）
└── themes/             # 主题目录
```

### 文章写作

首先生成一篇新的文章：

```sh
hexo new "The very new start"
```

执行后在 `source/_posts/` 目录中会添加一个 markdown 文件：`The-very-new-start.md`。编辑这个文件来进行博客的写作。

插播一句，不会用 Markdown 写作的童鞋可以查看 [这里](https://help.github.com/articles/markdown-basics/) 学习。（[这里](http://wowubuntu.com/markdown/) 是中文版）

用以下指令来生成静态文件，在本地观察效果：

```sh
hexo generate
hexo server     # watch localhost:4000/
```

### 部署到 Github

1. `npm install --save hexo-deployer-git`
2. 修改 `_config.yml` 文件中的 `deploy` 值（将 repository 改为你的 github repo 的地址）
  ```yaml
  deploy:
     type: git
     repository: git@github.com:allenfantasy/allenfantasy.github.io.git
     branch: master
  ```
3. `hexo deploy --generate` 生成静态文件，并将生成的内容推到 github 仓库中。

然后...访问 http://allenfantasy.github.io 耶！一切顺利！

接下来要考虑域名的事情了。

### 域名绑定 & DNS

我需要将部署在 Github Page 的博客和 http://afantasy.ninja 绑定起来，同时在访问原来的 .io 域名时也可以跳转到 .ninja 域名。

幸好 Github 早已看穿了一切，给出了完善的指南：

* [设置 CNAME](https://help.github.com/articles/adding-a-cname-file-to-your-repository/)
* [在域名提供商的后台设置中配置DNS](https://help.github.com/articles/tips-for-configuring-an-a-record-with-your-dns-provider/)

其中，注意 CNAME 文件要创建在本地的 `source/` 目录下，这样在生成静态页面时，CNAME 会复制一份到生成后的目录下，从而在部署时可以部署到 Github.

而 DNS 在设置后需要一定时间才能生效，快的话半小时左右，慢的可能要几个小时（但不会超过24小时）。

在花了很多时间摸索得到以上的正解后，我终于在敲下 http://afantasy.ninja 后看到了和本地一模一样的博客首页，感动啊T^T

但是...我想换个主题...

### 主题

* [Github Wiki](https://github.com/hexojs/hexo/wiki/Themes)
* [官网的主题](https://hexo.io/themes/)

自己找去吧...

通常比较好的主题都有教如何安装的。如果需要在原主题基础上修改的话，最好是 fork 一份再慢慢改。

（我果然是个懒人...）

详细的内容就还是参考 [Bruce 的文章](http://ibruce.info/2013/11/22/hexo-your-blog/) 吧，也是够详细了...
