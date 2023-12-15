---
title: pve 安装 Bliss OS - Android-x86 并配置显卡直通
date: 2023-12-15
description: "pve 安装 Bliss OS - Android-x86 并配置显卡直通"
layout: post

tags:
  - pve
  - Android
  - 虚拟机
categories:
  - pve
  - Android
  - 虚拟机
lightgallery: true

toc:
  auto: true
---

## 1. 前言

我的 [这篇文章](../android-x86-pve/) 中介绍了在 pve 环境中安装 `Android-x86` 实现 `x86` 架构的云手机，可惜 `Android-x86` 的版本停留在了 `Android 9.0` ，这个版本的 `Android` 无法实现音频的串流，`scrcpy` 的音频串流最低支持到 `Android 10` ，无法听到音频的云手机是不完美的，今天我们就来试下另一个 `Android-x86` 的项目 - [BlissOS](https://blissos.org/) ，看它是否可解决以上问题，成为真正可用的云手机。

## 2. 安装

### 2.1 ROM 选择

简单介绍一下 `BlissOS` ，它是基于 `AOSP` 优化定制，集成了很多专为 `PC` 优化的功能，比 如 `Dock`，`Desktop` 模式，兼容 `arm` 应用，也就是说他的 ROM 定制得更像一个平板系统，而不是原生 `AOSP` 那样，`AOSP` 如果作为远程使用，没有触摸屏的情况下还是有很多操作上的不便，但是它定制的版本则很好的解决了鼠标操作的问题，因为它目标就是专为Chromebook、桌面PC、平板等设备优化。

目前官方提供了两个稳定的分支，`BlissOS 14` 基于 `安卓 11` ，另一个 `BlissOS 15` 基于 `Android 12L` ，我这里选择了 `BlissOS 14`。

![image.png](https://img.linkzz.eu.org/main/images/2023/12/65ead7462dc848dbf65ed0df74d767d8.png)

BlissOS 14 提供了几个 ROM，我们下载带谷歌框架的这个版本：

![image.png](https://img.linkzz.eu.org/main/images/2023/12/321f7262c069ab6ca83bf2cc9a971eb6.png)

### 2.2 显卡加速

Bliss OS 由开源技术构建，内核基于 `Linux` 所以支持大部分显卡加速，然而由于 `Nvidia` 专用驱动对很多开源软件的支持问题，BlissOS 仅支持极少的 Nvidia 显卡，我的上篇文章中使用了 `VirtualGL` 为 `Android-x86` 提供显卡加速，那个方案是可行的，而且性能也足够，但缺点时没有物理输出，今天我们换个方案，参照 [这篇文章](../pve-igd-passthrough/) 中提取出的核显 `vbios` ，就能实现核显直通并具有物理输出，我们不仅可以用来当作云手机，甚至可以外接显示器作为 `HDPC` 使用。

### 2.3 安装 BlissOS

> 以下安装需要你确定你的显卡 `vbios` 正常工作，比如通过这个 `vbios` 能正常的安装 `Windows` 并点亮屏幕，显卡的兼容性可参考 `BlissOS` 的官网，正常来说所有 `Intel` 和 `AMD` 的核显均支持，以下安装使用的是 i5-10400 的核显 - `UHD630` 。

- **创建虚拟机**

```bash
# 以下参数看情况修改
qm create 115 \
--cores 6 \
--cpu host \
--bios ovmf \
--machine pc \
--memory $((8 * 1024)) \
--net0 virtio,bridge=vmbr0,firewall=1 \
--name BlissOS-14 \
--scsihw virtio-scsi-single \
--efidisk0 file=local-vzb:0,format=qcow2 \
--scsi0 file=local-vzb:32,format=qcow2 \
--ide2 file=ugreen:iso/Bliss-v14.10-x86_64-OFFICIAL-opengapps-20230704.iso,media=cdrom \
--boot order="scsi0;ide2" \
--hostpci0 0000:00:02,legacy-igd=1,romfile=igd.rom \
--hostpci1 0000:00:1f.3 \
--args '-set device.hostpci0.addr=02.0 -set device.hostpci0.x-igd-gms=0x2 -set device.hostpci0.x-igd-opregion=on' \
--vga none
```

记得添加 usb 键盘和鼠标设备，核显连接显示器之后开机安装。

![image.png](https://img.linkzz.eu.org/main/images/2023/12/e2c55d31464e52084bebdc1b22592b6a.png)

安装界面 `grub` 菜单，有很多 Live 的选项，可以先选择 Live 选项看下是否正常启动，测试一下显卡兼容性，安装就选择高级选项 - 自动安装即可。

![image.png](https://img.linkzz.eu.org/main/images/2023/12/0cb0927bff3188d4a354926cd3b21ecc.png)

如果 OS boot 的时候一直卡在这个界面就是显卡不兼容，无法运行 `BlissOS` 。

![image.png](https://img.linkzz.eu.org/main/images/2023/12/7588827d83484307018364ab6966b5a6.png)

如果没有其他问题正常安装即可，下面看下 ROM 的配置。

## 3. 配置

ROM 已经打开了 `5555` 端口的 `adb` 调试，我们可以直接 `scrcpy` 连接：

```bash
# adb 连接
adb connect 192.168.5.134

# scrcpy 连接
scrcpy -e
```

由于我们直通了主板声卡，所以该版本可以进行 `scrcpy` 音频串流了，连接信息中也没有了音频警告信息。

![image.png](https://img.linkzz.eu.org/main/images/2023/12/9dbb0294e281728e94cdd5da540d3f1b.png)

### 3.1 桌面模式

打开 `Desktop Mode` 进入桌面模式，该模式下和 `PC` 的操作逻辑很像，应用以浮窗模式运行，也可以进行全屏，窗口缩放等操作。

![image.png](https://img.linkzz.eu.org/main/images/2023/12/c447c404a437f0a8ceba3e515663daa1.png)

### 3.2 Smart Dock

上面那种模式虽然和 `PC` 操作逻辑很像，但是会有一些问题，比如应用切换其实并不方便，而且有的应用没做小窗适配，导致小窗布局出现问题，我喜欢设置 `Smart Dock` 然后全屏打开应用。

打开 `Smart Dock` 进行应有的赋权，会在下方出现一个 `Dock` 栏，可以显示菜单、通知、音量设置、时间、正在运行的应用等信息，操作便利。

![image.png](https://img.linkzz.eu.org/main/images/2023/12/a9833adcea7e8ec2987c60dc0152e35d.png)

### 3.3 传统安卓桌面

当让你也可以什么都不配置，直接使用朴素的安卓桌面，这个安卓的平板操作是一致的。

### 3.4 Root

ROM 内置了 [KerneISU](https://kernelsu.org/zh_CN/guide/installation.html)，可以方便的获取 Root 权限。

![image.png](https://img.linkzz.eu.org/main/images/2023/12/0db6d639ca64dff8300c1542463df622.png)

## 4. 性能和应用兼容情况

虚拟机配置方面给到了 `6 核 8G` 的配置，宿主机 `CPU` 为 `i5-10400` ，在今天来看也很普通的配置了，在运行效率上比 Windows 上运行模拟器好不少。

### 4.1 Geekbench 6 跑分情况

![image.png](https://img.linkzz.eu.org/main/images/2023/12/38dbc1b4188aa80433ab8ce73f134a8c.png)

`Geekbench 6` CPU 跑分情况，单核 `1293` ， 多核 `4480`，可能是虚拟机的原因，内核调度有点问题，跑分的时候 CPU 跑不满，要是跑满了分数还会高点，当然日常使用肯定是没什么问题了，单核跑分和 `8Gen1` 接近，多核在 `8Gen1` 和 `8Gen2` 之间。

![image.png](https://img.linkzz.eu.org/main/images/2023/12/91562a2d823f0ea40c2ad1344de72400.png)

![image.png](https://img.linkzz.eu.org/main/images/2023/12/2e6fd90957ec4fa50a72406b7fbe9b18.png)

GPU 跑分结果为 `4405` 分，图形接口为 `Vulkan` ，性能的话接近 `小米11T pro` 的 `骁龙 888` 。

![image.png](https://img.linkzz.eu.org/main/images/2023/12/89c7c4e14a9306195f0432b706f7290b.png)

![image.png](https://img.linkzz.eu.org/main/images/2023/12/abeb50cd73291c06ed6af5cbc99c8190.png)

> 以上跑分也就图一乐，实际上 `x86` 平台和 `arm` 平台的比较是不严谨的，双方运行的环境差异太大，这里的跑分只是大致参考一下性能，总之这样的性能流畅运行大部分日常应用是没问题的，下文的 `GIF` 是一些应用运行的效果，可以看出还是很流畅的。

### 4.2 兼容性

兼容性方面由于我测试的不全面，不能完全说全面兼容，但是不管是一些 `32bit` 的应用，还是最新的 64 位应用都能运行，所以上文说的将它当作 `HDPC` 是可行的。

在目前的测试中只有 bilibili 普通版打开 Crash，报错日志：

```log
12-15 14:50:47.629 13749 29123 E AndroidRuntime: kotlin.UninitializedPropertyAccessException: lateinit property crash has not been initialized
12-15 14:50:47.629 13749 29123 E AndroidRuntime:        at com.bilibili.gripper.container.crashreport.BLCrashInitTask.c(BL:1)
12-15 14:50:47.629 13749 29123 E AndroidRuntime:        at com.bilibili.gripper.container.crashreport.a.f(BL:8)
12-15 14:50:47.629 13749 29123 E AndroidRuntime:        at com.bilibili.gripper.container.crashreport.a.a(BL:1)
12-15 14:50:47.629 13749 29123 E AndroidRuntime:        at com.bilibili.lib.gripper.api.TaskCompat.c(BL:9)
12-15 14:50:47.629 13749 29123 E AndroidRuntime:        at com.bilibili.lib.gripper.api.TaskCompat.b(Unknown Source:0)
12-15 14:50:47.629 13749 29123 E AndroidRuntime:        at com.bilibili.gripper.container.crashreport.a.b(BL:1)
12-15 14:50:47.629 13749 29123 E AndroidRuntime:        at com.bilibili.gripper.container.crashreport.BLCrashInitTask$$CompatProducer$$execute$$Lambda.invokeProducer(BL:2)
12-15 14:50:47.629 13749 29123 E AndroidRuntime:        at com.bilibili.lib.gripper.api.internal.ProducerLambda.invokeSuspend(BL:11)
12-15 14:50:47.629 13749 29123 E AndroidRuntime:        at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(BL:4)
12-15 14:50:47.629 13749 29123 E AndroidRuntime:        at kotlinx.coroutines.w0.run(BL:22)
12-15 14:50:47.629 13749 29123 E AndroidRuntime:        at kotlinx.coroutines.scheduling.CoroutineScheduler.m(BL:1)
12-15 14:50:47.629 13749 29123 E AndroidRuntime:        at kotlinx.coroutines.scheduling.CoroutineScheduler$c.d(BL:4)
12-15 14:50:47.629 13749 29123 E AndroidRuntime:        at kotlinx.coroutines.scheduling.CoroutineScheduler$c.n(BL:4)
12-15 14:50:47.629 13749 29123 E AndroidRuntime:        at kotlinx.coroutines.scheduling.CoroutineScheduler$c.run(BL:1)
12-15 14:50:47.629 13749 29123 E AndroidRuntime:        Suppressed: kotlinx.coroutines.DiagnosticCoroutineContextException: [p2{Cancelling}@d87693a, java.util.concurrent.Executors$DelegatedScheduledExecutorService@193a9eb]
12-15 14:50:47.631   992  2975 W ActivityTaskManager:   Force finishing activity com.bilibili.app.in/tv.danmaku.bili.MainActivityV2
```

### 4.3 运行情况

下面看一段 GIF 看下运行情况：

![blissos-screen.gif](https://img.linkzz.eu.org/main/images/2023/12/08f75822d4bc771d70e4c324a91896a7.gif)

## 5. 总结

以上看到了最后运行了一个游戏，最终的运行效果没有放出来，是的，这个游戏能进界面，但是会在进行对战的时候闪退，试了一下游戏兼容性，腾讯的MOBA如LOLM也是在对战时闪退，一些游戏能正常运行，如“凡人修仙传手游”可以流畅运行，所以用来游戏挂机要看游戏是否兼容。

除此之外完全可以当作能 `7 * 24` 小时待机的云手机使用，体验和实机也差不多，适合喜欢折腾的小伙伴，或者进行 app 开发测试等场景，当然如果你有什么有趣的玩法，也欢迎留言一起讨论！
