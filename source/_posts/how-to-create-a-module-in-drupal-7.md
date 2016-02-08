title: 如何在 Drupal 7 中构建模块 [译]
tags:
- drupal
- translation
---

注1：本文翻译自 Drupal 官方网站关于构建模块的[指南][guide]，结合实际操作有部分修改和简化。

注2：阅读本文需要对 Drupal 有基本的了解：知道如何使用 Drupal 的管理后台，以及安装模块。

### Getting Started

#### 预备工作

1. 给模块取个名字。模块名建议用 “下划线+小写“ 的方式，以避免不必要的问题，比如 my_custom_module
2. 创建一个文件夹作为模块的文件夹
3. 在文件夹中新建两个 php 文件，分别是 `my_custom_module.module` 和 `my_custom_module.info`。module 文件是该模块的主文件入口，info 文件是该模块的定义，记录关于该模块的元信息。
4. 初始化 `my_custom_module.info` 文件。info 文件的通用模板为：

  ```
  name = My Custom Module
  description = A description of what your module does
  core = 7.x
  ```

  这里的 name 和 description 会显示在网站后台的模块管理界面中，作为模块的名字和介绍。关于 info 文件的详细介绍可以参考相应的 [Drupal 文档][info-doc].
5. 初始化 `my_custom_module.module` 文件。在没有添加任何功能前，先在文件中加入一行 php 的开头标签（不需要结尾，如果加入结尾标签可能会导致一些运行问题）

  ```php
  // my_custom_module.module
  <?php
  ```
6. 将模块的整个文件夹放入 `sites/all/modules` 文件夹中
7. 这时在 Drupal 的管理后台中，你应该能够看到一个名字为 "My Custom Module" 的模块。启用它并保存配置。

#### [实现第一个钩子][hook-guide]

接下来我们要利用通过 "钩子函数"（hooks） 来在模块中实现某些特定的功能。钩子函数的命名方式为：`{module_name}_{function_name}`，其中 `module_name` 是模块的名字，而 `function_name` 是预定义的钩子函数的后缀。Drupal 会调用这些钩子函数，并传入相应的特定数据。

为了便于展示，我们来实现一个简单的钩子：[`hook_help`](hook_help-doc)，这个钩子可以让我们向使用者提供关于这个模块的帮助信息。要实现这个函数，我们就要用模块名字来取代钩子命名中的 "hook"，也就是实现一个叫做 `my_custom_module_help` 的函数，代码如下：

```php
<?php
function my_custom_module_help($path, $arg) {
  switch($path) {
    case "admin/help#my_custom_module":
      return "<p>Do something</p>";
      break;
  }
}
```

这个钩子函数接受两个参数：

* `$path` 参数是用户查看模块帮助时所在的位置，即当前所处的 url 地址，这个参数可以是通配符
* `$arg` 当 `$path` 为通配符时的匹配参数

在这个例子中我们仅匹配特定的路径，所以仅使用 `$path` 参数（更多用法可以直接查看 Drupal 的[文档][hook_help-doc]）。

#### 检查钩子是否正确工作

在模块管理界面查看 "My Custom Module" 这个模块，这时应该能够看到一个 "help" 的链接（首先要确保启用了 Help 这个核心模块），点击进入模块的帮助页面，查看钩子函数返回的文字内容。

如果没有看到该链接，尝试先禁用模块，再重新启用。

到这里为止，我们就顺利的搭建模块了！实现钩子函数是模块最常用的使用方法。

### Reference

1. [Drupal 7 模块指南][guide]
2. [Drupal 7 模块指南 - 钩子函数][hook-guide]
3. [Drupal 7 模块的 info 文件][info-doc]
4. [Drupal 7 hook_help 函数][hook_help-doc]

[guide]: https://www.drupal.org/developing/modules/7 "Drupal 7 模块指南"
[hook-guide]: https://www.drupal.org/node/1095546 "Drupal 7 模块指南 - 钩子函数"
[info-doc]: http://drupal.org/node/542202 "info 文件的介绍"
[hook_help-doc]: https://api.drupal.org/api/drupal/modules%21system%21system.api.php/function/hook_help/7 "hook_help 钩子的文档"
