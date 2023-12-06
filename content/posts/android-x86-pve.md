---
title: pve安装Android-x86并配置显卡加速
date: 2023-11-23
draft: false
description: "pve安装Android-x86并配置显卡加速"
tags:
  - pve
  - 虚拟机
  - Android
categories:
  - pve
  - Android
lightgallery: true

toc:
  auto: true
---

先看下效果：

![android-x86-demo.gif](https://img.linkzz.eu.org/main/images/2023/11/c8a6924e9c9fa1972a6f19da3367218c.gif)

可以看到运行很流畅，由于gif压缩小了尺寸，Aida64的信息界面看不清，显卡
是`virtGL`，支持`OpenGL`，3D加速由宿主机`Intel UHD 630`提供，最后的游戏演示可看
出很流程。

## 1. 前言

### 1.1 为什么需要Android

我日常使用的是一个直通了RTX2060s（矿卡）的Win11虚拟机，众所周知，在虚拟机里使用
模拟器性能巨差，我试过夜神模拟器，启动慢不说，游戏电竞帧数，连抖音都卡顿明显。所
以一个能在Hypevision中运行的安卓系统就是一个很好的选择。

### 1.2 选择哪一种

现阶段能作为选择的有3种方案：

- 1. **ChromeOS**:

     首当其冲的就是基于[Chromium OS](https://www.chromium.org/chromium-os/)的
     ChromeOS及其他发行版如国内的[FydeOS](https://fydeos.io/)，这系统号称老电影
     福音，具有完整的桌面环境并兼容Android应用，可惜对于显卡的兼容性不是很好，而
     虚拟机能使用3D加速的显卡，除了显卡直通外还有一种就是最近一两年积极维护
     的[VirtualGL](https://wiki.archlinux.org/title/VirtualGL)，所以ChromeOS在
     VirtualGL下运行是否可行还得画个问号，后面再折腾。

- 2. **容器化方案 -`Waydroid` or `Redroid`**:

     第二种方案就是借助容器化`cgroups`，`namespace` 等内核技术的容器化方案，这种
     方案启动快速，可使用宿主机硬件资源，也能使用宿主机GPU进行3D加速, 但是对于
     Nvidia的显卡兼容性很差。运用pve创建一个lxc容器来运行docker redroid容器也能
     运行，后续折腾一下这个方案。

- 3. **Android-x86:**

     [Android-x86](https://www.android-x86.org/) 在AOSP的基础上增加x86平台的支
     持，可通过native-bridge转义arm指令从而运行arm应用，同样称为老电脑福音，GPU
     兼容性较好，兼容[VirtualGL](https://wiki.archlinux.org/title/VirtualGL) ，
     今天我们就来在pve中尝试一下这个系统有什么妙处。

## 2. 安装Android-x86

### 2.1 镜像下载

打开[Android-x86](https://www.android-x86.org/) 官网，官网提供了2个下载分流，按
需选择，我这里选
择[SourceForge](https://sourceforge.net/projects/android-x86/files/)

![image.png](https://img.linkzz.eu.org/main/images/2023/11/5d0d0aa1d930d4aba35b64d7cb377d8e.png)

共有一下版本：

![image.png](https://img.linkzz.eu.org/main/images/2023/11/7eb5ee18842f0715ae49fe07e057502f.png)

最新版本Release 9.0 基于Android 9.0 Pie，也是下载次数最多的版本，我们选择此版
本。

![image.png](https://img.linkzz.eu.org/main/images/2023/11/68eaaa00161a22fc93115275dfa22175.png)

选择 64 位ISO版本

### 2.2 创建虚拟机

创建虚拟机

![image.png](https://img.linkzz.eu.org/main/images/2023/11/35290b067c3025c055887339e4a180de.png)

选择刚才下载的镜像

![image.png](https://img.linkzz.eu.org/main/images/2023/11/e6c5d3e05a731fbe5a48d77808ccc497.png)

选择 `q35` 机型和 `OVMF` 安装，显卡选择`VirGL GPU`

> `VirGL GPU` 需要宿主机有有效的render节点设备，也就是说你的显卡需要正确的驱动了
> 才能运行，通过`ls /dev/dri` 查看是否有 `render*` 字样。

![image.png](https://img.linkzz.eu.org/main/images/2023/11/ad8920561f6fc190fe3bddb0495dc2c9.png)

存储给到32G即可，如果需要安装大型游戏可以给多一点。

![image.png](https://img.linkzz.eu.org/main/images/2023/11/25c2e923bae364d53066cc0a2f5421c9.png)

安卓比较依赖`CPU`资源，如果运行的是`arm`应用更需要`CPU`的转译，所以这里给到
`10 vCPU` 。

![image.png](https://img.linkzz.eu.org/main/images/2023/11/50cae9a59aec34c61356736995d4d9fd.png)

内存给到 `8G`

![image.png](https://img.linkzz.eu.org/main/images/2023/11/667d8aa918fb3f3fe7037b5220dcfd6e.png)

Android-x86做了虚拟机的兼容，所以这里网卡选择效能最高的 `VirtIO` 。

![image.png](https://img.linkzz.eu.org/main/images/2023/11/21550c21b93f94ce492693d9648e4b13.png)

点击完成之后选择“硬件” -> "添加音频设备", 后端驱动器选择 “无”

![image.png](https://img.linkzz.eu.org/main/images/2023/11/4e23de919b3ff7507f89b56b7f444790.png)

最终配置文件如下：

```text
audio0: device=ich9-intel-hda,driver=none
bios: ovmf
boot: order=scsi0;ide2;net0
cores: 10
cpu: host
efidisk0: local-vzb:114/vm-114-disk-0.qcow2,efitype=4m,pre-enrolled-keys=1,size=528K
ide2: ugreen:iso/android-x86_64-9.0-r2.iso,media=cdrom,size=921M
machine: q35
memory: 8192
meta: creation-qemu=7.1.0,ctime=1700718616
name: Android-x86-efi
net0: virtio=F6:B1:0A:BA:FD:5E,bridge=vmbr0,firewall=1
numa: 0
ostype: l26
scsi0: local-lvm:vm-114-disk-0,iothread=1,size=32G
scsihw: virtio-scsi-single
smbios1: uuid=29493e94-6604-4e93-bee8-ab92aa93f6b4
sockets: 1
vga: virtio-gl
vmgenid: f2d8f76e-fad9-49cf-b957-2bf9bb1d11d1
```

### 2.3 安装

启动安装，通过 `noVNC` 连接安装程序， 也可以选第一项 Live 模式测试一下兼容性

![image.png](https://img.linkzz.eu.org/main/images/2023/11/530f2fea4e44605d57ff3bcbdfea7331.png)

我们直接选择高级选项自动安装到指定硬盘

![image.png](https://img.linkzz.eu.org/main/images/2023/11/d7bc89bf89ebd3a10376864d37d6a079.png)

安装程序自动发现了我们的虚拟硬盘

![image.png](https://img.linkzz.eu.org/main/images/2023/11/273f0d2fc71a7108f98b81eb27bdd7e7.png)

选择 `Yes` 数据自动写入硬盘(图床抽风没传上)

安装成功，运行 `Android-x86` (图床抽风没传上)

## 3. 设置Android-x86

### 3.1 初始化

第一次开机开始初始化设置

![image](https://img.linkzz.eu.org/main/images/2023/11/f8aeca8e1128fed9b4eeb127b0bf1024.png)

语言选择中文

![image.png](https://img.linkzz.eu.org/main/images/2023/11/b97d01e9be39cd5eb3e5d03bcff4b6b2.png)

Wifi网络先跳过

![image.png](https://img.linkzz.eu.org/main/images/2023/11/2b4a17560364965189a0c0c27372b07e.png)

时间默认即可

![image.png](https://img.linkzz.eu.org/main/images/2023/11/4c1c2cc99729a90b1a19d977cb5246cf.png)

接受谷歌服务的协议

![image](https://img.linkzz.eu.org/main/images/2023/11/d118073a633fef0bcbb180115558b9ef.png)

保护功能是不开的，所以这里也不用选了

![image.png](https://img.linkzz.eu.org/main/images/2023/11/6d3aeb7122932e61a9387277be7e1ac0.png)

选择默认主屏幕模式：

- 任务栏 - 基于浮窗的任务栏模式，类似桌面模式，但是使用体验很差。
- Quickstep - 传统安卓桌面，鼠标模拟点击操作，还有抽屉式应用，用过安卓的会比较熟
  悉这个模式，按需选择。
  > 这里选了始终之后如果要切换默认模式在设置 -> 应用和通知 -> 默认应用里面改回来
  > 即可

![image.png](https://img.linkzz.eu.org/main/images/2023/11/2f1d74da66a65b2a38f184fbc5c298f9.png)

看桌面环境是带了 `GApp` 的，不用安装谷歌服务了，且默认 `adbd` 是开启的。

![image.png](https://img.linkzz.eu.org/main/images/2023/11/0f1af6069f686f6a583c5ead5cd43250.png)

### 3.2 设置静态ip

设置 -> 网络和互联网 -> WLAN 点击 `VirtWifi` 连接，安卓会通过DHCP获取ip，一般获
得的ip会显示无法访问互联网，这是因为我们的网络无法访问谷歌404网站，接下来我们修
改这个网络的属性，改为固定ip，我还要修改一下魔法网关。

![image.png](https://img.linkzz.eu.org/main/images/2023/11/df90ae316d598b0178498e57cb6aa1a0.png)

点击网络详情 -> 点击编辑

![image.png](https://img.linkzz.eu.org/main/images/2023/11/73c42d14f86e53ddcd213ec72e1a4f5c.png)

高级选项 -> IP设置选择 “静态”

![image.png](https://img.linkzz.eu.org/main/images/2023/11/62b968c72f395dc5390209ef5ff3e113.png)

按实际情况填好IP和网关、DNS之后保存。

> 保存好之后并不是立即生效的，可关开WLAN重新连接网络之后即可，如果你的网络环境可
> 直接访问Google的话会弹出账号登录同步通讯录等谷歌服务，登录之后即可通过
> `Google Play` 下载应用。

### 3.3 Scrcpy连接Android

[scrcpy](https://github.com/Genymobile/scrcpy) 是安卓远程连接的优秀开源项目，基
于adb连接，可在PC端通过鼠标操作安卓，具有低时延，高帧率，可定制连接质量，可传输
音频（需要Android 11以上版本）等一系列优点，具体可查看
其[文档](https://github.com/Genymobile/scrcpy) scrcpy支持
`Linux`、`Mac`、`Windows` 三大平台，这里以 `Windows 11` 平台为例

- 安装 `Scrcpy`

`Scrcpy` 依赖 `adb` 工具，先安装 `adb`

```bash
winget install --id=Google.PlatformTools  -e
```

再安装 `Scrcpy`

```bash
winget install --id=Genymobile.scrcpy  -e
```

adb 连接 Android-x86

```bash
adb connect 192.168.5.5
```

![image.png](https://img.linkzz.eu.org/main/images/2023/11/5347e25be6a358c89f68bf0a73da286e.png)

第一个设备是我的手机，忽略。

scrcpy 连接

```bash
scrcpy -e
```

![image.png](https://img.linkzz.eu.org/main/images/2023/11/5559fcebbc632ba6ede8743d2baf00b6.png)

-e 参数指定通过tcpip协议连接，如果你只有一个设备连接到adb可不用这个参数。

> Scrcpy 2.0 开始原生支持音频传输，但是需要 Android 11 以
> 上。[sndcpy](https://github.com/rom1v/sndcpy) 需要Android 10以上，所以音频传输
> 是无解的，因为Android 10才加入了音频分享的API，具体可查看这
> 个[Issue](https://github.com/rom1v/sndcpy/issues/25)

![image.png](https://img.linkzz.eu.org/main/images/2023/11/33970cc11aea32f0fe9f127cd8758238.png)

### 3.4 开启arm兼容模式

默认只能运行兼容x86架构的app，运行arm架构的app需要开启arm兼容模式，开启arm转译。
从[foxi](https://foxi.buduanwang.vip/linux/1996.html/) 大佬的网盘下载库文件。

```bash
wget https://foxi.buduanwang.vip/pan/foxi/download/andriod/houdini9_y.zip
```

解压之后上传到 Androi-x86

```bash
unzip.exe houdini9_y.zip
adb -e push .\houdini9_y.sfs /sdcard/Download
```

开启nativebridge

```bash
adb -e shell
su
cd sdcard/Download/
enable_nativebridge
```

以上没有什么错误输出就好了，然后再在设置界面打开兼容模式

![image.png](https://img.linkzz.eu.org/main/images/2023/11/eb5090d61acc1b624b380746806a01e2.png)

安装一下抖音运行测试一下：

正常运行！

![image.png](https://img.linkzz.eu.org/main/images/2023/11/ac4432af0570d42e7f884476cbfe60f2.png)

## 4. 性能测试

### 4.1 查看硬件参数

我们通过AIda64安卓版查看参数

```bash
adb.exe -e install Downloads\aida64-v196.apk
```

查看系统参数

![image.png](https://img.linkzz.eu.org/main/images/2023/11/e6e7e109d1a9ed79fb2ff828a907d247.png)

处理器识别有问题，不过cpu运行是正常的

![image.png](https://img.linkzz.eu.org/main/images/2023/11/561980a13851fea44f62e55d2a553a76.png)

显卡VirGl，支持 OpenGL ES 3.0

![image.png](https://img.linkzz.eu.org/main/images/2023/11/af542d097a5eee6d870cd3493052afe9.png)

### 4.2 Geekbench 6 查看CPU效能

通过Geekbench运行测试看下CPU性能如何

![image.png](https://img.linkzz.eu.org/main/images/2023/11/a51cf0fb5076236da33861b4999df74c.png)

单核性能接近`Pixel 7 Pro`，多核甚至优于小米13，哈哈，当然这里就是图一乐，不同指
令集的CPU肯定不能这么比。

![image.png](https://img.linkzz.eu.org/main/images/2023/11/1a040e9d25bb86bb3db5572c41d33faf.png)

![image.png](https://img.linkzz.eu.org/main/images/2023/11/e80c61fc0fee687b4a89d8424a29ccd8.png)

### 4.3 验证GPU加速是否成功

为了查看虚拟机是否用上了宿主机的GPU资源，我们查看一下GPU的负载情况：

- Intel GPU 宿主机安装 `apt install intel-gpu-tools`

```bash
intel_gpu_top
```

![image.png](https://img.linkzz.eu.org/main/images/2023/11/97090fac0da56e1a200c9454af1916ae.png)

随意操作 Android-x86 看到intel的GPU 3D渲染功能正常运行

## 5. 结语

`Android-x86` 在虚拟机上的表现可以说超出了我的预期，大部分的app均可流畅运行，且
性能开销尚可，只是9.0 版本的Android不支持音频的传输，所以用来当作云手机的效果会
大打折扣，只能用于一些挂机app或者游戏的运行，另外有一个类似的Android-x86 发行版
[BlissOS](https://blissos.org/) 有基于Android 12的版本，下期可以尝试这个版本是否
可解决音频传输的问题，让我拥有一个x86的安卓云手机。
