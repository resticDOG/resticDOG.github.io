---
title: 基于纯开源方案的指纹浏览器和同步器方案
date: 2025-03-15
description: "基于纯开源方案的指纹浏览器和同步器方案"
layout: post

tags:
    - 折腾
    - Chrome
categories:
    - Airdrop
lightgallery: true

toc:
    auto: true
---

## 1. 背景

先叠一个甲，此方案并不适用于小白，且有很高的学习成本，在使用这个方案之前，你需要熟练使用 vim 的常用热键。

去年 3 月份接触了空投赛道，说来也奇怪，当时接触到的空投信息还是一个诈骗信息，当时无意间刷到一篇博客，说是在 Shiba（柴犬币）的 Telegram 频道发布的一个 bot 中签到，每 8 小时签到一次，累计到 50w 积分（具体积分数额记不清了，大致是这个数）之后，就可提现到链上。为此还有人写了个 telegram 的 bot 定时签到，我还运行了一段时间，当时提现需要创建链上钱包，而创建钱包不可避免要了解各大公链，尤其是以太坊，还要注册交易所入金，由此走上了空投之路，也因此经历了一轮币圈的沉浮，见证了 BTC 的 ATH，和 meme 季大爆发，然而我去年的空投并没有赚到钱，投入了 1000u 左右，在川普上任之后资产到达顶峰 1200u 左右，之后就一蹶不振，最终只能勉强算不赚不亏。究其原因，就是起的号太少了，全年只有一个主号，选择的项目也是冷门项目，有一些项目也没能坚持，总之就是种种原因导致效果不好。

扯远了，说回正题，今年我决定重来，这次就先起 5 个号。工欲善其事必先利其器，既然要操作多号，而碍于种种原因我又不想用商业指纹浏览器，那就只能自己想办法了，经过重重阻拦，最终摸索出了现今这套模式，且听我慢慢道来。

## 2. VirtualBrowser

指纹浏览器选择 [VirtualBrowser](https://github.com/Virtual-Browser/VirtualBrowser) ，这是一款基于 Chromium 的指纹浏览器，目前只支持 Windows 平台，看 README 计划支持 Linux 和 Mac，不过目前尚未开发完毕，它支持的指纹环境如下：

- Operating System: Modify the operating system part in `userAgent`.
- Browser version: Modify the browser version in `userAgent`.
- Proxy settings: Modify the browser proxy which supports "Default", "Do not use proxy", "Custom".
- User Agent: Modify `userAgent`.
- Language: Modify `navigator.language`, `navigator.languages`, and it can be automatically matched based on IP.
- Time zone: Modify the time zone in `new Date()`, and it can be automatically matched based on IP.
- WebRTC
- Geolocation: Modify the latitude and longitude in `navigator.geolocation.getCurrentPosition()`, and it can be automatically matched based on IP.
- Resolution: Modify `screen.width`/`screen.height`.
- Font: Randomly modify the supported font list.
- Canvas: Randomly modify Canvas 2D drawing differential pixels.
- WebGL image: Randomly modify WebGL drawing differential pixels.
- WebGL metadata: WebGL vendor, WebGL rendering, etc.
- AudioContext: Randomly modify the differential data of `getChannelData` and `getFloatFrequencyData` in AudioContext.
- ClientRects
- Speech voices
- CPU: Modify `navigator.hardwareConcurrency` CPU core count.
- Memory
- Device name
- MAC address
- Do Not Track
- SSL
- Port scan protection
- Hardware acceleration

目前来看支持还挺全面的，因为我撸的空投较少，我以前的方案是谷歌多 profile + SwitchyOmega 简单实现 IP 隔离，这个方案是没有指纹环境的，web 端稍加检测就能知道是同一台机器，写这篇文章的阶段也爆出 SwitchyOmega 有恶意代码，窃取用户信息，所以这套方案千万别用！回到 VirtualBrowser, 官方 [github release](https://github.com/Virtual-Browser/VirtualBrowser/releases/tag/2.1.2) 页面就可以下载安装，这里建议安装 `2.0.5` 版本，最新 `2.1.2` 版本莫名跳登录有点离谱。

### 2.1 指纹效果

新建一个环境测试一下指纹，代理我填的是一个韩国的 IP，本地化时间和语言我选择了根据 IP 自动选择，这依赖一个官方的 API 来检测 IP-Location，注册一个账号，免费额度即可。

![](https://img.linkzz.eu.org/main/images/2025/11/9e054d2ecd2b19ec0595ac9493906261.png)

这是浏览器计算的指纹 hash

![](https://img.linkzz.eu.org/main/images/2025/11/3748b84e6a6d016b562fb0e00ba2dbfe.png)

这是 [ipcheck.ing](https://ipcheck.ing/#/browserinfo) 计算的浏览器指纹

![](https://img.linkzz.eu.org/main/images/2025/11/84b278ea0589ba896437872682a6423d.png)

这是实机浏览器的指纹

我的 CPU 是 AMD-4600H ，GPU 是 Vega 6， 可以看出他识别 CPU 和 GPU 的能力还是较弱，无论是实机还是指纹浏览器均未正确识别出，不过指纹的伪装效果还是很不错的。

![](https://img.linkzz.eu.org/main/images/2025/11/bc2324089bbbf99cbc6dde5b216c7b3b.png)

### 2.2 隐患

没错，这个浏览器 Github 上仅开源了相关的前端代码，核心的浏览器内核的代码并未开源，所以还是存在代码植入恶意代码的可能，我因为资金较小，现在也只是实验阶段所以采用了他，不过话说回来，几个大的商业指纹浏览器都被盗了，严格来说没有完全的安全，所以撸毛钱包万不能放大资金。

## 3. 同步器

同步器一般是商业指纹浏览器的配套功能，能大大提高多号的操作效率，对应需要手撸的项目来说往往事半功倍，而 VirtualBrowser 是没有这个功能的，还好在某大指纹浏览器被盗之后撸毛社区出现了很多 Chrome 多开的资料，也有大佬开发了基于 Chrome 多开的同步方案。

### 3.1 Chrome-Manager

[Chrome-Manager](https://github.com/devilflasher/Chrome-Manager) 是一个 Chrome 浏览器多窗口管理器，可以实现多窗口排列和鼠标按键同步功能，可以提高 Chrome 多窗口交互效率。

他使用 Python 开发，调用的 pywin32 的库，原生不支持指定 chrome 引擎，所以直接用是无法进行 VirtualBrowser 的窗口管理的，需要对源码做一点改动。

![image.png](https://img.linkzz.eu.org/main/images/2025/11/7a7d494aafb8a6dd3876d448135538f1.png)

如上 diff，仅需改动判断 path 的代码即可，下面我们看下效果：

![image.gif](https://img.linkzz.eu.org/main/images/2025/11/d5ab1bced4f2ce2fa5626ce40267d0e3.gif)

可以看到效果还是可以的，检测窗口，自动排列窗口，以及窗口同步均可以使用，以及下方的批量打开网页的功能也是正常的，快捷方式相关的功能是不能用的，因为 VirtualBrowser 使用快捷方式打开会失去指纹的作用，所以这块没改，如果要求不高的化，到这里已经可以使用了，但是这个同步器并不是完美的，还是有一些缺陷。

## 4. Vim 方式控制浏览器

上文提到的缺陷即其同步依赖于鼠标相对的窗口位置计算来同步点击的位置，但是鼠标滚动的时候页面移动的像素却不是相等的，这就导致了虽然点击的位置是相同的，但是该位置的元素却不总是相同，原作者提到的解决方案是利用 PageUp 和 PageDown 按键来翻页，以及方向上、下键来滚动页面，以此实现滚动条总是滚动相同的位置。

这样确实有所改善，但还是会有位置不一样的情况发生。为了解决这个问题，我想起了一个很极客的项目 - [vimium](https://github.com/philc/vimium) 使用 vim 来控制你的浏览器，这个项目自己的简介就表明了他的特色 - `The hacker's browser.` 黑客的浏览器，这也是文中我提到这个方案并不适合小白的原因，不过既然看客都看到这里了，想必也对这个门槛不那么在意了。

vimium 是一个以 vim 热键的方式来控制浏览器的项目，这个项目是那些不想使用鼠标的 vim 大神开发的浏览器插件，vim 这个项目自诞生以来就以他高效的编辑体验、完美的热键设计受到各路开发者的喜爱，很多开发者也为各种编辑软件开发适配了 vim 的快捷键，甚至有些编辑器自身自带 vim 功能，比如 Obsidian、Zed 编辑器等，而这个 vimium 也实现了基本的 vim 功能，如 hijk 移动，d/u 上下滚动，f 键高亮所有可点击的元素，按下导航的按键即可点击对应元素，这样就不用依赖鼠标的点击就可以完成网页交互，效率比鼠标要高不少，下面看下效果：

![](https://img.linkzz.eu.org/main/images/2025/11/98b9c0b5b1129b57ebee85f3e95e2215.gif)

## 5. 总结

总体来说来说这套方案可以消除鼠标滚轮同步时的不足，但是多开窗口对电脑配置要求的确较高，我的Windows是虚拟机 `10 核 10 G` 的配置只能同时控制4个窗口，且4个窗口同时打开网页卡的飞起，不得不说撸毛也是卷配置。

**2025年10月更新：撸毛频繁反撸，这套方案已经弃用，撸毛已死，大家别折腾了！**
