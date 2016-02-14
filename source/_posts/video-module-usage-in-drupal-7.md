title: "「译」- Drupal 7 中 Video 模块的使用"
tags:
- drupal
- translation
---

注1：本文翻译自 Drupal 官网中 Video 模块的文档，并做部分删改，仅供参考。

注2：本文要求读者具备基本的 Drupal 7 知识，如基本架构，模块使用等。

### 安装及配置

原文:

- [Installing and Configuring the Video Module - Drupal 7](https://www.drupal.org/node/2281571)
- [Video: transcoding and playback](https://www.drupal.org/documentation/modules/video)

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

[TODO](Video: transcoding and playback)

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

#### 使用

现在你可以通过新建一个该 content type 下的 node 来上传视频了。注意，如果你使用的是 Zencoder，请确保你有一个公共的 IP 地址（而不是 localhost）
