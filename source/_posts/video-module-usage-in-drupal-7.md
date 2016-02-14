title: "「译」- Drupal 7 中 Video 模块的使用"
tags:
- drupal
- translation
---

注1：本文翻译自 Drupal 官网中 Video 模块的文档，并做部分删改，仅供参考。

注2：本文要求读者具备基本的 Drupal 7 知识，如基本架构，模块使用等。

原文:

- [Installing and Configuring the Video Module - Drupal 7](https://www.drupal.org/node/2281571)
- [Video: transcoding and playback](https://www.drupal.org/documentation/modules/video)

### 关于 Video 模块

Video 模块可以将几乎任何视频格式文件 “转码” 为 H.246, Theora, VP8（以及其他众多格式）。它使用 [Zencoder](https://zencoder.com/) 转码云服务或者 [FFMPEG](http://ffmpeg.org/) 这个开源项目（需要运行在你的服务器上）。网站建设者可以通过一个 video 字段将视频上传到节点（nodes）中（video 字段是 Video 模块中自带的），就和文件字段一样。当视频完成转码后，Video 模块将自动为视频创建缩略图，并且将一个视频播放器嵌入到视图层中。Video 模块是高度可配置的，还可以集成到 AWS S3 云存储服务中。

### 安装及配置

#### 系统要求

* 系统中必须要安装 **Curl**，并且可以通过 PHP 调用。在 phpinfo() 中查看 Curl 的选项，或者在 linux 命令行下执行：
  ```sh
  $ php -i | grep "curl"
  ```
  如果你看到有输出信息，那么说明配置好了。
* FFMPEG 在 Windows 下只支持有限的 codec，需要做额外的安装配置。

#### 依赖的模块

* [Libraries](https://drupal.org/project/libraries)

#### 转码器（transcoder）

Video 模块兼容 FFMPEG 和 Zencoder。根据你选择的转码器你需要做以下配置（必须选择其中一个）：

* **Zencoder**
  1. 从 Github 中下载 [zencoder-php 库](https://github.com/zencoder/zencoder-php)
  2. 将这个库放置到 *sites/all/libaries* 中
* **FFMPEG** - 根据你使用的平台下载并安装 FFMPEG（根据平台的不同，下载 FFMPEG 的方式也不同）。**请自行搜索在不同平台下安装的方法。**

FFMPEG 和 Zencoder 有什么区别么?

**FFMPEG** 必须安装在你的服务器上，它的安装会有些复杂。但一旦安装配置好后，FFMPEG 不需要任何额外的花销。但是，视频转码是一个 CPU 密集的操作，所以 FFMPEG 在高流量的单点服务器上，可能不是一个很好的解决方案。同时，在上传的视频文件体积较大或者网站接受大量视频文件时，也不大适合使用 FFMPEG。在以下情况下使用 FFMPEG 比较合适：

1. 网站上传视频的频率不高，且视频比较短；
2. 你（开发人员）有时间来研究如何安装编译配置 FFMPEG，但没有钱（购买服务）；
3. 你的服务器能够承载大量的 CPU 计算；
4. 网站的流量不高。

**Zencoder** 是一个速度快的云转码服务。它可以接受高流量的大视频文件上传且不会让你的服务器变慢。尽管 Zencoder 是一个商业服务（收费），它提供免费的试用账号，该账号由 Video 模块为你轻松建立。使用 Zencoder 的两个不足是你需要为使用服务来付费，同时目前在没有 IP 地址的情况下难以测试 Zencoder。例如，在本地开发时很难测试 Zencoder。在以下情形下你可以考虑使用 Zencoder：

1. 你的资金能够负担视频服务的费用，但（开发人员）没有时间来研究 FFMPEG；
2. 你的服务器（网站）需要承受高流量；
3. 你允许网站访问者自由地上传视频到你的网站中；
4. 你希望网站接受大视频上传；
5. 你希望在某一天能在云上存储不限量的视频。

#### 配置 Video 模块

Video 模块包含3个单独的子模块：Video, Video UI 和 Zencoder

配置方法如下：

1. 在 *admin/modules* 中启用 Video 和 Video UI 两个模块
2. 如果使用 Zencoder:
  1. 启用 Zencoder 模块
  2. 进入 *admin/config/media/video/transcoders* 页面
  3. 在 Video transcoder 的选项中，选择 Zencoder
  4. 输入你的邮箱账号，以获得一个 Zencoder 的试用账号（trial）
  5. 保存配置
  6. 稍等一会儿，你的账号将被建立，并自动保存到你的 Video 模块信息中。
3. 如果使用 FFMPEG
  1. 在服务器上安装 FFMPEG。具体操作详见 FFMPEG 的 [官网](https://www.ffmpeg.org/download.html)
  2. 进入 *admin/config/media/video/transcoders* 页面
  3. 在 Video transcoder 的选项中，选择 FFmpeg/avconv
  4. 在 *Path to FFmpeg or avconv executable* 中填入服务器中 FFMPEG 的可执行文件地址（可以用 `which ffmpeg` 查看）
  5. 保存配置
4. 在 *admin/config/media/video/presets/add* 页面中，添加一个 preset（至少需要一个），并且至少配置以下选项：
  * Preset name
  * Video output extension - 输出格式
  * Video codec - 视频编码
  * Audio codec - 音频编码
5. 在 *admin/structure/types* 中，编辑或添加一个 *content type*
6. 添加一个 video 字段，并选择你之前设定好的 preset

### 使用

现在你可以通过新建一个该 content type 下的 node 来上传视频了。注意，如果你使用的是 Zencoder，请确保你有一个公共的 IP 地址（而不是 localhost）
