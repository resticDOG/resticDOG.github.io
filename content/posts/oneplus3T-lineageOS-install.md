---
title: 一加 3T 刷 LineageOS 18.1记
date: 2023-11-30
description: "一加 3T 刷 LineageOS 18.1记"
layout: post

tags:
  - 折腾
  - Android
categories:
  - Android
lightgallery: true

toc:
  auto: true
---

现在用的手机是一加8 Pro，旧手机一加3T放在公司作为备用机和必要时候的远程打卡机，最近在在使用scrcpy连接手机发现音频的传输需要系统在Android 10以上，然而这个手机早已失去了官方的支持，最终最新版系统停留在安卓9.0，幸好这手机在国外很受欢迎，有 [LineageOS](https://download.lineageos.org/devices/oneplus3/builds) 的官方支持，最新的系统也有基于Android 11 的 `LineageOS 18.1`，今天就来记录一下安装过程。

> 本文不是教程文章，只是自己的折腾记录，如果你要按照本文的方式来操作，请确保你了解必要的手机刷机的知识如 adb、fastboot、解锁 bootloader 等。

![image.png](https://img.linkzz.eu.org/main/images/2023/11/a33f29c483d4231a3d9135e2431961ba.png)

## 1. 解锁 Bootloader, 刷入第三方 recovery

手机刷机第一步，由于我的一加3T很早以前就已经解锁BL并刷入了第三方TWRP recovery，这里就写一写步骤就好了

### 1.1 安装 oem usb 驱动

Mac和Linux系统无须安装usb驱动，但在Windows上则必须安装，否则 adb 和fastboot无法连接设备。

由于设备古老，在中文互联网上已经很难找到官方驱动了，还好在外网有专门的[网站](https://oneplususbdrivers.com/)下载，虽然标的是官方，但具体是否还待验证，用起来是没有什么问题。

![image.png](https://img.linkzz.eu.org/main/images/2023/11/fcdffc63063c8d58a1bb5efe24b39fec.png)

下载之后是 `setup.exe` 文件，点击安装之后用手机连接Windows，设备管理器里没有Android设备的感叹号就好了。

### 1.2 解锁 Bootloader

安装 adb 工具

```bash
scoop install adb
```

打开USB调试模式，连接手机并允许usb调试。

```bash
adb reboot bootloader
```

或者关机状态下按住 ”音量+“ + "电源" 键进入 fastboot 模式。

```bash
fastboot devices
```

键入一下命令解锁 Bootloader

```bash
fastboot oem unlock
```

没有报错就解锁成功了。

### 1.3 刷入 Recovery

下载 LineageOS 的 [recovery](https://download.lineageos.org/devices/oneplus3) 镜像

进入 fastboot 模式

```bash
adb reboot bootloader
```

验证一下设备是否连接上

```bash
fastboot devices
```

若设备没有列出来请检查驱动是否安装好

没问题的化键入一下命令刷写 `recovery`

```bash
fastboot flash recovery recovery.img
```

### 1.4 刷入 LineageOS 18

下载最新的 [LineageOS 18](https://download.lineageos.org/devices/oneplus3/builds) 镜像，我这里下载的是 `lineage-18.1-20231105-nightly-oneplus3-signed.zip` 版本。

![image.png](https://img.linkzz.eu.org/main/images/2023/11/153a02517a6c77c49b081b6f9398a713.png)

关机模式下按住 ”音量-“ + "电源" 键进入 recovery 模式。

> 注意：一下操作将清除手机所有数据，请确保做好备份！请确保做好备份！请确保做好备份！

选择 ”Factory reset > Format data/factory reset“

将清除 `/data` 和 `/cache` 分区

完成之后选择：”Apply update -> Apply from ADB“ 进入 `adb sideload` ，之后执行一下命令刷入镜像。

```bash
adb sideload Downloads\lineage-18.1-20231105-nightly-oneplus3-signed.zip
```

![image.png](https://img.linkzz.eu.org/main/images/2023/11/63619dc21f7722bca51f6a1e684d63ae.png)

观察 recovery 日志有没有报错，

然而，升级过程中还是出现了问题，升级提示：`Modem firmware from OxygenOS 9.0.2 or newer sock ROMs is prerequisite to be ompatible with this build`

我的手机系统用的是基于 `氢OS` 的 `安卓 8.0` 系统，按提示看来需要先刷 `氧OS 9.0.2` 的底包才行，那好吧，XDA找一下，还是国外资源靠谱，不想某盘，时间一长必然失效，随意找了一个基于官方 `OxygenOS 9.0.6` 的[改版系统](https://xdaforums.com/t/rom-theone3tos-oxygenos-aroma-open-beta-30-stable-9-0-6-11-22-2019.3730932/)，刷入开机试了一下，没用过氧OS，确实很接近原生Android，刷了之后发现recovery也恢复为官方recovery了，无妨，再按以上操作刷入recovery，之后再一次刷入lineage 18，大功告成！

![image.png](https://img.linkzz.eu.org/main/images/2023/11/413862ff65d275ee0dd4c57e776f54cb.png)

系统很简洁

### 1.5 刷入Gapp，Magisk

接下来刷入谷歌服务框架，和 [Magisk](https://github.com/topjohnwu/Magisk) 获取root权限。

- Gapp

下载[MindTheGapps](https://github.com/MindTheGapps/11.0.0-arm64/releases/tag/MindTheGapps-11.0.0-arm64-20230922_081122) 安装包，启动 recovery 模式刷入：

```bash
adb sideload Downloads\MindTheGapps-11.0.0-arm64-20230922_081122.zip
```

手机上提示无效的签名选”Yes"强制安装即可。

- Magisk

root设备增加手机的可玩性，下载最新 [Magisk](https://github.com/topjohnwu/Magisk/releases/latest) app, 找到刚才刷入的 LineageOS 18.1的镜像包，拷贝出 `boot.img`，传输到手机:

```bash
adb push .\Downloads\boot.img /sdcard/boot.img
```

打开 `Magisk` 选择安装。

![image.png](https://img.linkzz.eu.org/main/images/2023/11/a0ac2619f6101b4ac709322e967b29b7.png)

选择并修补一个文件

![image.png](https://img.linkzz.eu.org/main/images/2023/11/dd2e1b1aa1b48e6d838d4dadf0c317a1.png)

选择我们传入的 `boot.img`, 点击 “开始” ![image.png](https://img.linkzz.eu.org/main/images/2023/11/f3de1d5c08f8fbc0ee6ec51d4937d779.png)

成功之后我们可以先试下这个文件是否可以启动。

![image.png](https://img.linkzz.eu.org/main/images/2023/11/461dd7523f4a39402fa65fe6863b8c3a.png)

```bash
adb pull /sdcard/Download/magisk_patched-26400_PVOCH.img .\Downloads\magisk_boot.img
adb reboot fastboot
fastboot boot .\Downloads\magisk_boot.img
```

成功启动之后打开Magisk, 提示修复运行环境表示这个boot镜像已经patch成功，下面用永久的刷入它。

![image.png](https://img.linkzz.eu.org/main/images/2023/11/74c862b2803215d197cd5cefa4b578ef.png)

```bash
adb reboot fastboot
fastboot.exe flash boot Downloads\magisk_boot.img
```

启动之后打开 `Magisk` app 按提示修复运行环境即可。

## 2. 系统体验

丝滑，体感是比官方 `氧OS 9.0` 还要流畅，感觉老机器焕发了新春，只是毕竟硬件性能还是太老了，应用的打开和运行和当下，甚至我现役的 `OnePlus 8 pro` 还是有很大差距的，不管怎么说，当一个备用机那是绰绰有余了。
