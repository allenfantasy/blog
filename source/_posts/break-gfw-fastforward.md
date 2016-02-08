title: 科学上网之快速过关
tags:
- lifehack
---

本文旨在提供快速科学上网的通关方法。
废话少说，立刻开始。
<!--more-->

# 综述

科学上网的方式有若干种：

* VPN
* Shadowsocks（又称 ss 或影梭）
* Lantern（蓝灯）
* GoAgent
* 改 hosts

我自己比较喜欢用前两个方法。Lantern 我没有用过，GoAgent 不稳定而且局限程度比较大。

稍微做一个简单的比较表格吧

|方式        |平台         |费用 |备注|
|-----------|-------------| -----|----|
|VPN        |全平台|一百多到几百不等|不易被封杀|
|Shadowsocks|全平台|每月10-20美元(自建)/80-100元(付费服务)|iOS 待研究;也不易被封杀|
|Lantern    |全平台|**据说免费**|不要相信我那个“据说”|
|GoAgent    |PC|这个是真免费|移动端存疑;只能用在浏览器上|
|改 hosts    |PC|这个也免费|移动端我没试过;猫捉老鼠|

关于各种方法的具体优劣，Annie Wu 写了一篇很不错的 [比较文章][AnnieWu-VPN]，这里我只对比较熟悉的 VPN 和 Shadowsocks 进行详述。

# VPN

[VPN](https://zh.wikipedia.org/zh/%E8%99%9B%E6%93%AC%E7%A7%81%E4%BA%BA%E7%B6%B2%E8%B7%AF) 是一种简单的全局翻墙方式。特殊的国情造就了大量的 VPN 服务。

需要注意的是，VPN 是一种全局的翻墙方式，所以如果你希望同时访问国内外两边的网站（比如，看 youtube 同时看 youku），那你需要留意 VPN 厂商是否提供了智能分流的方式。否则要频繁切换也是很烦躁的。

电脑和移动端操作系统都预装了 VPN 客户端，相比别的翻墙操作更简单，适合不折腾人士。

以下列举我了解的 VPN 服务：

* [云梯](https://www.ytpub.com/) - 这个是去年我在用的方式，不过半年前我换成了 Shadowsocks 了。需要注意的是云梯有很多假冒者，请小心分辨
  * 优点：国内可以访问，有完善的各平台教程，有智能分流，客服不错（提交工单就会回复），价格还算亲民~~（真不是打广告）~~
  * 缺点：部分地区网络无法访问（其实这个是所有 VPN 都可能有的问题）
* [Astrill](https://www.astrill.com/) - 这个是很多老外喜欢用的 VPN
  * 优点：据老外和 Annie 说好，比较稳。支付方式多。
  * 缺点：贵。而且好像网站本身也要翻墙上，有点鸡生蛋蛋生鸡的感觉。
* 红杏
  * 据 Annie 说是死了。Sad story.
  * 刚搜索了一下，现在也有叫「红杏」的服务（不全是 VPN），但真假莫辩：
    * [一枝红杏](http://www.yizhihongxing.com/)
    * [红杏](http://www.hongxingchajian.com/)
  * 然后我又搜索了一下，发现官方有 [解释这个问题](http://help.honx.in/hc/kb/article/49540/)，敬请大家不要上当。

其他没用过的我就不提了。或者大家如果有找到好的也可以用。

# Shadowsocks

江湖人称 ss 或影梭。也是一种可全局翻墙的方式（原理与 VPN 不同，使用 Socks 协议）

Shadowsocks 分两个部分：Server & Client

Server 指服务器端，Client 指客户端（你的设备），两端通过 Socks 协议沟通，由于协议本身并不是普通的 HTTP 协议，所以更难被墙。

在服务器端有两种方式可以实现：自建（用 VPS）

## 自建

推荐阅读 Shadowsocks 的 **[官方文档](https://github.com/shadowsocks/shadowsocks/wiki/Shadowsocks-%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E)**

当然也可以参考这些教程：

* For linux: [安装配置 Shadowsocks](https://xiaoxiao.im/2014/06/02/shadowsocks.html)
* [使用 Digital Ocean 和 Shadowsocks 来科学上网](http://jerryzou.com/posts/shadowsocks-with-digitalocean/)

至于 VPS 的选择方面，有几个选项：

* Vultr - 这个有很多朋友推荐，目前在用这个
* [Conoha](https://www.conoha.jp/zh) - 据说国内速度很不错哦（但是我没用过。特别鸣谢某18岁骚年的推荐。
* Linode
* Digital Ocean

当然你也可以用你喜欢的 VPS 架设。

## 使用服务

作为程序员我是自己搭建的，所以没买过这种服务。

Annie 推荐的是 [这个](https://shadowsocks.com/)，有兴趣的同学可以试试。（不行不要打我）

## 客户端的问题

这里有一个比较完整的 [列表](https://shadowsocks.com/client.html)

### Mac 下的使用

由于我自己用的是 Mac，所以只能整理出 Mac 下的教程了。
Windows 和 Linux 的用户实在抱歉。

* [Windows 教程](https://github.com/shadowsocks/shadowsocks-windows/wiki/Shadowsocks-Windows-%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E)
* [Linux 教程](http://jerryzou.com/posts/shadowsocks-with-digitalocean/)

具体步骤：

1. 购买或在服务器上自建好 Shadowsocks 服务。
2. [下载安装 Shadowsocks OS X](https://github.com/shadowsocks/shadowsocks-iOS/wiki/Shadowsocks-for-OSX-%E5%B8%AE%E5%8A%A9)

  貌似 Shadowsocks OS X 的那个下载链接...是被墙的嗯。[百度盘链接](http://pan.baidu.com/s/1skb1yQp)
3. 配置：打开 Shadowsocks OS X，这时屏幕顶部任务栏会有一个纸飞机的图案。点击之后选择「服务器」-「服务器设定」，然后会出现以下窗口：

  ![](http://7xjvtw.com1.z0.glb.clouddn.com/ss-mac-os-x.png)

  地址栏填服务商提供的（或者自建的）IP 和端口；填写密码和备注。

4. 配置好后点击纸飞机：「打开 Shadowsocks」，并切换到自动代理模式；
5. 用 Chrome 浏览器，并在浏览器上安装 SwitchyOmega 插件，并把插件调整为 `auto switch` 模式
6. 尽情的上网吧。

# 那你在用什么?

* Mac 上用 Shadowsocks （非常完美）
* iOS 上用云梯 VPN。我用的是移动 4G，只能说时灵时不灵吧。

# Reference

* [科学上网专题 - 简书](http://www.jianshu.com/collection/b6b16295fc83) - ~~话说简书这么干，我告诉你，吃枣药丸啊~~
* [V＊P＊N使用大全丨当我想看看世界的时候，我要做些什么。][AnnieWu-VPN] - Annie 的呕心沥血之作
* [使用 Digital Ocean 和 Shadowsocks 来科学上网](http://jerryzou.com/posts/shadowsocks-with-digitalocean/) - 很全面的翻墙教程

[AnnieWu-VPN]: https://mp.weixin.qq.com/s?__biz=MzAwOTU3MDMwNg==&mid=402146885&idx=1&sn=b8b69ca32f4b49fbe42f1effa39e095d&scene=1&srcid=01130kK0bCb7pKtQwV2Vfpq8&key=710a5d99946419d9a313aac034600058a65e11dd5741d34c5ac15fc11b7ed24df3e5a616096eaaff56b058df8903ca4b&ascene=0&uin=MjYyODA0MjQ4Mw%3D%3D&devicetype=iMac+MacBookPro11%2C2+OSX+OSX+10.11.3+build(15D21)&version=11020201&pass_ticket=px%2FKbwHJYExj3Fb52LK%2FlNK4T1Q7IOubUmdFgDsYhcXx9BtQCsMI3oIxktwDz3MB
