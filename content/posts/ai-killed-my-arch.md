---
title: AI 弄崩了我使用已久的ArchLinux系统
date: 2026-03-20
description: "AI 弄崩了我使用已久的ArchLinux系统"
layout: post

tags:
    - AI
categories:
    - AI
lightgallery: true

toc:
    auto: true
---

## 起因

我这套 ArchLinux 的系统是安装在 esxi 里面的虚拟机，还是3年前装的系统，作为我的远程开发机器一直稳定使用，期间很少更新，前几天弄了一个 zed 的教育优惠，我琢磨这弄个 zed 来使用，况且宿主机还有一个闲置的 `Nvidia 1060`的亮机卡，将它直通来使用岂不美哉，于是启动了我的升级之旅。

## 开始升级

我这套系统估计 2 年没更新了，而 ArchLinux 这样滚动更新的版本更新频率那是相当的快，好处嘛自然是能用到最新的软件包，但是稳定性就无法保障了，虽然但是，为了用到免费的 `Claude 4.6 Sonnet`还是硬着头皮更新了，期间遇到的第一个问题就是 `nvidia` 的驱动，nvidia 最终还是大发善心维护了开源的驱动，但是 `nvidia-open`的驱动却是不支持 10 系的显卡了，于是升级的时候需要屏蔽 nvidia 驱动的更新，这更新时问题就来了，当然在 AI 高速发达的今天，小孩子都知道遇到问题就找 AI，于是我打开了大家公认最为靠谱的 `claude ai`，免费用户可以使用 `Claude 4.6 Sonnet`模型，看似很良心，问了一通问题之后 AI 给出了建议，也都解决了一些问题，让我颇为满意，但在最后一步 AI 让我使用以下命令：

```shellscript
sudo pacman -Rdd linux-firmware
sudo pacman -Syu --overwrite '*'
```

我隐约觉得有点不对，但是前面都很流程的解决了问题，让我暂时打消了疑虑，所以就执行了，执行之后看似问题解决了，但是当我重启之后：

![img.png](https://img.linkzz.eu.org/main/images/2026/03/41c4079b1e84e9f253545cef69905c05.png)

啪，这还是我第一次见到 Kernel Panic，证明还是折腾的不够多哈哈。

然后我就开始问起 AI：

![img.png](https://img.linkzz.eu.org/main/images/2026/04/98089fbdaaa39cfde4678bdbc0b88540.png)

不是，你还好意思笑！

无奈，只能挂 ISO 盘开始修复。

挂了之后发现由于我删除了 `linux-firmware` 模块，导致很多指令执行不了，pacman 报错， yay 报错， 只能一步步修复，好在 curl 可以使用，下载了 pacman-static 之后才将 `linux-firmware` 包安装回来，可是此包的版本已经是最新了，又和 nvidia 的驱动不兼容，所以只能来一次系统的大更新，好在系统是救回来了。

## 感悟

这两年来 AI 确实进化了很多，但即使强如 claude 这样的大模型，依然无法直接上生产还是有一定的原因的，最近也确实听到过不止一例的 AI 导致的生产事故，AI 的确可以带来未知领域的知识快速获取，但是碳基生命的消化速度跟不上获取速度，所以对 AI 的建议，尤其是生产环境的建议，还是要谨慎再谨慎，不要贪图一时爽快而弄崩了大盘。
