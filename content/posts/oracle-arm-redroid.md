---
title: 甲骨文免费Arm主机新玩法-云安卓手机
date: 2023-12-06
description: "甲骨文免费Arm主机新玩法-云安卓手机"
layout: post

tags:
  - 折腾
  - Android
  - Docker
  - Linux
categories:
  - Android
  - Linux
lightgallery: true

toc:
  auto: true
---

## 1. 前言

我的一篇 [文章](../android-x86-pve/) 中有提到过云安卓手机的项目-redroid，该
[项目](https://github.com/remote-android/redroid-doc) 基于容器技术，构建一个安卓
的运行时，同时通过Linux的内核模块，支持调用宿主机的硬件资源，同时其可运行于
`x86` 架构之上，通过转译来运行仅支持 `arm` 架构的安卓应用，用来跑app测试可以一
试，正好甲骨文的arm主机资源没有好好利用，今天就来折腾一下这个玩法。

## 2. 加载内核模块 ashmem_linux、binder_linux

### 2.1 基于 `Ubuntu 20.04` 以上发行版

这两个模块是容器运行必须的内核模块，按照官方文档，在 `Ubuntu 20.04` 以上版本中，
这两个模块已经编译到内核里了，可以直接 `modprobe` 命令加载，所以如果你的
`arm服务器` 正好是`Ubuntu 20.04` 以上版本，按照一下命令即可运行：

```bash
# 安装额外内核模块
apt install linux-modules-extra-`uname -r`
# 加载内核模块
modprobe binder_linux devices="binder,hwbinder,vndbinder"
modprobe ashmem_linux

# 运行容器
docker run -itd --rm --privileged \
    --pull always \
    -v ~/data:/data \
    -p 5555:5555 \
    redroid/redroid:11.0.0-latest \
    androidboot.redroid_gpu_mode=guest
```

### 2.2 Oracle Linux 8

否则如果你像我一样开主机的时候选了 `Oracle Linux 8` 的话，很遗憾，官方没有该系统
的运行文档，以上两个内核模块也并没有编译，奈何我对该发行版不熟，升级了官方内核到
`5.10` 版本来编译redroid提供的两个内核模块的
[源码](https://github.com/remote-android/redroid-modules) ，最终用
`amazonlinux2` 的分支可已正常编译`ashmem`模块，然而另一个 `binder` 模块始终无法
编译成功。

```bash
# Amadevel-uname-r == `uname -r`"
sudo make # build kernel modules
sudo make install # build and install *unsigned* kernel modules
```

### 2.3 Debian 10

于是我又想到了，`Ubuntu` 起源于 `Debian`，那倘若我将系统dd成Debian是否就可以正常
加载这些内核模块了呢，于是我找到了一个手动
[重装的教程](https://lala.im/7905.html) 该教程基于网络安装，是走的官方渠道获取的
系统镜像，不用担心被植入后门，这也是为什么不选择那些 `一键dd脚本` 的原因，缺点是
只能安装 `Debian 10` ，较新的 `Debian 11` 只能装完之后再寻求升级方法了。

比较顺利系统安装成功之后顺利登录系统，看了一下系统内核版本是 `4.19` 版本，还是有
点老，于是从官方库升级了一下最新的稳定版内核 `5.10` ，~~开始编译以上两个模块，这
次很顺利编译成功了（虽然报了几个警告）~~，看了官方内核已经编译了两个内核模块，果
然和 `Ubuntu 20.04` 以上版本一样，2个内核模块都可以加载。按照官方的教程开始运行
容器：

```bash
# 加载内核模块
modprobe ashmem_linux
modprobe binder_linux devices=binder1,binder2,binder3,binder4,binder5,binder6
# 修改权限
chmod 666 /dev/binder*
chmod 666 /dev/ashmem

# 运行容器
docker run -itd --rm --privileged \
    --pull always \
    -v /dev/binder1:/dev/binder \
    -v /dev/binder2:/dev/hwbinder \
    -v /dev/binder3:/dev/vndbinder \
    -v ~/data11:/data \
    -p 5555:5555 \
    --name redroid11 \
    redroid/redroid:11.0.0-latest \
    androidboot.redroid_gpu_mode=guest

# adb 连接容器
adb connect arm-2

# scrcpy 连接
scrcpy -e
```

然而在 `scrcpy` 连接容器的时候却报了个错：

```log
[server] ERROR: Could not create default video encoder for h264
List of video encoders:
    (none)
[server] ERROR: Exception on thread Thread[video,5,main]
java.lang.IllegalArgumentException: Failed to initialize video/avc, error 0xfffffffe (NAME_NOT_FOUND)
	at android.media.MediaCodec.native_setup(Native Method)
	at android.media.MediaCodec.<init>(MediaCodec.java:2000)
	at android.media.MediaCodec.<init>(MediaCodec.java:1978)
	at android.media.MediaCodec.createEncoderByType(MediaCodec.java:1933)
	at com.genymobile.scrcpy.ScreenEncoder.createMediaCodec(ScreenEncoder.java:229)
	at com.genymobile.scrcpy.ScreenEncoder.streamScreen(ScreenEncoder.java:73)
	at com.genymobile.scrcpy.ScreenEncoder.lambda$start$0$com-genymobile-scrcpy-ScreenEncoder(ScreenEncoder.java:294)
	at com.genymobile.scrcpy.ScreenEncoder$$ExternalSyntheticLambda0.run(Unknown Source:4)
	at java.lang.Thread.run(Thread.java:1012)
INFO: Renderer: opengl
INFO: OpenGL version: 4.6 (Compatibility Profile) Mesa 22.3.3
INFO: Trilinear filtering enabled
```

scrcpy报告找不到有效的视频编码器，然而虽然甲骨文的arm主机没有给gpu资源，但是看了
用cpu 编码也是可行的呀，为啥运行不了，最终在这个
[Issue](https://github.com/remote-android/redroid-doc/issues/407) 下面找到了答
案，那即是在编译的内核中需要启用内核 `codec2` 的支持，于是最终还是走到了自己编译
内核这步。

### 2.4 自编译Debian的内核

Debian的内核源码以软件包的方式提供，我们搜索一下最新的源码包：

```bash
➜  ~ apt search ^linux-source
Sorting... Done
Full Text Search... Done
linux-source/oldstable 5.10.197-1 all
  Linux kernel source (meta-package)

linux-source-5.10/oldstable,now 5.10.197-1 all
  Linux kernel source for version 5.10 with Debian patches
```

可以看到最新的 `5.10` 的源码， 安装:

```bash
sudo apt install linux-source-5.10
```

在用户目录中编译：

```bash
# 创建目录
mkdir ~/kernel; cd ~/kernel
# 解压内核源码
tar -xaf /usr/src/linux-source-5.10.tar.xz
```

至于为什么不在 `/usr/src` 目录中用 root 编译，官方是这么说的, 我认为非常合理：

> 传统上，Linux内核源代码放置于 `/usr/src/linux/`，需要root权限才能编译。但是，
> 在不需要时应避免使用管理员权限。`src` 群组的成员也可以使用该文件夹，但是应避免
> 使用 `/usr/src/`。把核心源代码置于个人文件夹时，应把安全放在第一位：在
> `/usr/` 内的文件都应明确其在软件包系统内的作用，试图收集内核使用的信息时，不能
> 在读取 `/usr/src/linux` 时误导程序。

配置内核:

```bash
# 先将原本的内核配置保留，在原基础上修改
cp /boot/config-5.10.0-26-arm64 ~/kernel/linux-source-5.10/.config
vim ~/kernel/linux-source-5.10/.config
```

修改一下内容：

```config
CONFIG_DMABUF_HEAPS=y
CONFIG_DMABUF_HEAPS_SYSTEM=y
CONFIG_ANDROID_BINDERFS=y
CONFIG_ANDROID_BINDER_DEVICES="binder,hwbinder,vndbinder"
```

开始编译：

```bash
make deb-pkg LOCALVERSION=-falcot KDEB_PKGVERSION=$(make kernelversion)-1
```

开始漫长的等待，期间有可能提示缺少某个包之类的，按照提示处理一下即可，处理编译建
议单线程，然后经过2天试错之后终于编译完成。安装内核：

```bash
sudo dpkg -i ../linux-image*.deb
# 重启
sudo reboot
```

重启之后重复 `2.3` 的运行步骤即可成功运行, 运行后的一下截图：

![image.png](https://img.linkzz.eu.org/main/images/2023/12/b23ada35a18404e8952cd3b68216a328.png)

![image.png](https://img.linkzz.eu.org/main/images/2023/12/1cf8250b034f8c58c311a9ac5ce3d6ff.png)

运行效果：

![redroid-oracle-arm.gif](https://img.linkzz.eu.org/main/images/2023/12/b3ac2d75502895045eda2e20d9da7c45.gif)

## 3. 结语

从上一张的GIF运行效果图相比大家也不难看出，这个云安卓的效率也不是很强，虽然能运
行安卓应用，但是帧数很低，这是因为以下几个原因：

1. **网络问题**：

   我的主机位置在韩国春川，家宽为联通，最近两边的网络延迟已经飙升到200了，还时常
   伴随丢包，所以操作有很大延迟，对延迟很敏感的应用肯定不适合运行。

2. **配置问题**：

   我开了2台Arm，所以这台的配置为 `2C12G` 没有硬件加速的情况下cpu压力太大，下面
   是运 行抖音播放视频时的资源占用截图：

![image.png](https://img.linkzz.eu.org/main/images/2023/12/66086810e53e4df2454132feeeb66fb8.png)

3. **容器问题**：

   容器在用纯CPU下仅有15Hz的刷新率，使得在源头上就不可能流畅。

**总结**：适合用来做一些静态应用的测试，或者一些挂机的应用，日常使用还是算了。
