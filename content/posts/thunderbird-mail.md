---
title: 用 Thunderbird 统一管理我的邮件
date: 2024-03-21
description: "用 Thunderbird 统一管理我的邮件"
layout: post

tags:
  - Windows
categories:
  - tools
lightgallery: true

toc:
  auto: true
---

## 1. 前言

因为国内工作一直是用 **微信** 或者 **钉钉**，所以我一直不怎么使用邮件客户端来收发和管理邮件，就是收验证码的时候打开网页临时用一下，这就导致了我的邮件长期以来未读 **99+** ，这其中大部分是 `Twitter`、`Nintendo`、`Steam` 的推送，还好我不是一个强迫症，所以就还好。但是最近我还是决定用一个邮件客户端来统一管理电子邮件，不是因为换了要用电子邮件办公的公司，而是收验证码的时候一个个邮件账号要切换，光是谷歌邮箱就有 3 个，是在是麻烦。还有一个重要的点，有时候绑定的账号用了一个不常用的邮箱，比如 **Steam** 小号，当小号异地登录（被盗）的时候发送的邮件我收不到提醒，可想而知小号就危险了。

## 2. 客户端选择

所以我需要一个邮件客户端来同意管理我的所有邮箱账号，由于我使用的系统为Windows，客户端的选择也并不多，我对邮件客户端的要求有：

1. 免费
2. 颜值高，简洁
3. 内存占用尽量少，要常驻后台
4. 方便集成常用的邮箱 QQ、Google、Outlook 等
5. 不想用国产

基于以上选择我选了几个客户端来尝试一下，先放个比较图表：

| 客户端 | 优点 | 缺点 | 适用性 |
| --- | --- | --- | --- |
| Outlook For Windows | 微软官方支持，界面一致 | 字体渲染差，阅读体验不佳 | 适合微软生态用户 |
| 邮件 (UWP) | 简洁，内存占用小 | 功能有限，UI 设计一般 | 适合简单邮件收发 |
| Thunderbird | 开源，功能丰富，支持扩展 | UI 略显紧凑 | 适合需要高级功能的用户 |

### 2.1 Outlook For Windows

微软专门推出的邮件客户端，功能界面和 Web 版 outlook 一致，支持 Google、QQ、outlook 等常用邮箱，界面简洁，缺点就是在 Windows 下字体渲染巨差，在我使用了 MacType 替换了字体之后还是没有丝毫优化，阅读体验为负分，所以 Pass 。

![png](https://img.linkzz.eu.org/main/images/2024/03/a6790566ce0bfc7f277ec33574b8a0d1.png)

### 2.2 邮件（UWP）

微软原装 UWP 应用，可以方便的添加 outlook 邮箱，布局简洁，内存可忽略不计，如果只使用 **收验证码** 这个朴素的任务的化是完全可以胜任的，但是对我来说还是功能太少，而且 UI 对我实在不是很有眼缘，所以 Pass 。

![Untitled](https://img.linkzz.eu.org/main/images/2024/03/eaaeb7e90ed9b70b8328605413f6dbf1.png)

### 2.3 Thunderbird

[Thunderbird](https://www.thunderbird.net/zh-CN/) 是由 [Mozilla](https://mozilla.org/) 开源的一个邮件客户端，所以 UI 和火狐很像，支持常用的邮箱，支持插件系统，UI 界面有点紧凑，支持 rss 订阅，Windows 下字体渲染尚可，支持 rss 让我可以在里面阅读订阅的微博博主，这点比较吸引我。

![Untitled](https://img.linkzz.eu.org/main/images/2024/03/59affcdcf77825657568ecbecc3f73e0.png)

内存占用嘛，虽然不及原生应用，但在 electron 泛滥的今天也已经算不错了。

![Untitled](https://img.linkzz.eu.org/main/images/2024/03/bb288964d3d5a05df34d01179fa61257.png)

## 3. 体验

接下来说一下 Thunderbird 使用中的一些体验

### 3.1 添加邮箱账户

添加账户很方便，支持 Google OAuth 登录，有自己的 SMTP 数据库，无需自己配置 SMTP 等信息，我添加的几个大邮箱都可以方便的添加。

QQ 邮箱需要自己开启 SMTP 功能并开启授权码，用授权码当作密码登录才可以。

### 3.2 RSS 订阅

可以添加 RSS 订阅一些新闻，以下是我用 Rsshub 订阅的微博，因为 App 阅读体验真的很烂，所以常看的博主我都会通过 rss 去订阅。

![Untitled](https://img.linkzz.eu.org/main/images/2024/03/a344113a465e05aba317e177fd885d82.png)

## 4. 总结

Windows 因为众所周知的字体渲染问题，在低分屏下选择应用的时候总是会受困于难看的字体渲染，今天的邮件客户端也是，在选择的时候第一眼看到锯齿感十足的字体我就不想再使用下去了，Thunderbird 开源免费、功能齐全、UI 美观、支持拓展功能，是现阶段下我找到的最合适的邮件客户端了，感谢 Mozilla！感谢维护 Thunderbird 的各路大佬！
