---
title: Tesla P4 - Nvidia 专业卡 vGPU 解决方案体验
date: 2023-12-13
description: "Tesla P4 - Nvidia 专业卡 vGPU 解决方案体验"
layout: post

tags:
  - pve
  - Linux
  - 虚拟机
categories:
  - pve
  - Linux
  - 虚拟机
lightgallery: true

toc:
  auto: true
---

## 1. 前言

之前体验过 `intel` 的 `vGPU` 解决方案 [Intel GVT-g](https://wiki.archlinux.org/title/Intel_GVT-g) ，我的古早处理器 `i5-10400` 还是 `intel` 很老的gpu架构的 `UHD630` 分配到一个 Win10 虚拟机使用，哪怕是只分配一个 vm 的情况下性能依然不够看，使用 parsec `1080P H264` 串流的情况下帧数无法保证60，这时视频播放就更不用说了，直接GPU占用100%，更不用谈 `2k`、`4k` 等高分辨率串流了。早就了解到 Nvidia 的 vGPU方案支持 `kvm` 平台，而且支持 `Windows` 和 `Linux` 客户操作系统，性能较 intel 核显好得多，于是我弄来了这块小小的 Tesla P4。

![image.png](https://img.linkzz.eu.org/main/images/2023/12/cd216f0fffbeead1be892e87941efe0a.png)

## 2. 硬件

这块 `Tesla P4` 是个单槽半高卡，刚好我的 pve 宿主机剩下一个 x16 的 PCIE 插槽，虽然只有 X4 的速度，但咋对性能没有极致的追求，所以损失一点性能还能接受（主要是穷换不起主板），而且这块显卡最高功耗仅为 `75W` 无需外接供电，实测无负载的时候功耗仅十几瓦，最惊讶的还是他的价格，仅仅只需300块，这简直就是“年轻人的第一台特斯拉”呀，哈哈。缺点就是其是为了数据中心设计的，没有主动散热，所以我们需要外接一个小小的风扇为其降温，就是图中这个，我多花了50从PDD购入。

![image.png](https://img.linkzz.eu.org/main/images/2023/12/46954635d0224200d5df3aca5de0adad.png)

拆机安装

![image.png](https://img.linkzz.eu.org/main/images/2023/12/e77fc46c451ca08cd876167cabfa4304.png)

内部紧凑的空间，一番折腾终于装上了，这台主机的配置可以在[这篇文章](../pve-igd-passthrough/)中找到。

安装的过程中还有2的小插曲：

1. 我的手在拆机的时候碰到散热器挂彩了（所以一定是要祭点什么吗）。
2. 安装之后开机系统识别不了我的 2 块 PCIE 的 M.2 固态，还以为是这块主板的接口有屏蔽关系，最后发现是因为我打开了 GSM，需要关闭 GSM 才能正确识别 M.2 硬盘。

## 3. 软件

一番折腾终于正确安装好了，现在试下他有什么魔力吧。

### 3.1 宿主机设置

> 我的pve版本还是7.x，并没有升级到8.x，所以下面的操作均是在7.x的环境下进行，8.x内核升级到了6.x，按照我的操作不一定能成功！看官自行分辨。

Nvidia官网下载专业卡驱动需要注册申请啥的，比较麻烦，好在有热心大佬维护了一个仓库更新官方驱动，我们使用这个版本，找到最新的[Release](https://github.com/justin-himself/NVIDIA-VGPU-Driver-Archive/releases/) 包，我这里下载了 `535.129.03-537.7` 的版本，下载到宿主机安装:

```bash
wget https://github.com/justin-himself/NVIDIA-VGPU-Driver-Archive/releases/download/16.2/NVIDIA-GRID-Linux-KVM-535.129.03-537.70.zip
# 解压
unzip NVIDIA-GRID-Linux-KVM-535.129.03-537.70.zip
# 安装dkms
apt install dkms
# 安装驱动
Host_Drivers/NVIDIA-Linux-x86_64-535.129.03-vgpu-kvm.run --dkms
# 重启
reboot
```

如没问题重启之后运行 `nvidia-smi`

![image.png](https://img.linkzz.eu.org/main/images/2023/12/a9cd0c7ff1f7eb2bae886275b118fa60.png)

识别到 `Tesla P4` 就安装好了宿主机驱动。

运行 `mdevctl types` 可列出支持的vGPU类型。

![image.png](https://img.linkzz.eu.org/main/images/2023/12/88f17794c397aed7dc72056df7eae95b.png)

### 3.2 系列解读

接下来我们看看这些vGPU的系列有什么区别，看到图中的类型 `nvidia-65` 即 `GRID P4-4Q`

![image.png](https://img.linkzz.eu.org/main/images/2023/12/add06a4dbd2ec36387f0e8e74c80ac69.png)

简单说下类别类目里的一些区别，`GRID` 是 Nvidia 早期 vGPU 技术的名词，`P4` 是显卡型号，后缀的大写字母 `A` 、 `B`、 `Q` 代表不同的侧重点系列，首先，不同的系列需要的授权不同，当然我们使用的是特殊授权，所以这点不重要，对我们来说重要的在以下方面：

| 系列 | 侧重 | 最大分辨率 |
| --- | --- | --- |
| A系列 | 面向虚拟应用程序用户的应用程序流或基于会话的解决方案 | 1280 x 1024 60Hz |
| B系列 | 面向商务专业人士和知识工作者的虚拟桌面 | 5120 x 2880 45Hz |
| Q系列 | 适合需要 Quadro 技术性能和功能的创意和技术专业人员的虚拟工作站 | 7680 x 4320 60Hz |

如图，如果你的vm侧重计算，不在意显示，那选 `A` 系列，如果运行的是 `VDI` 应用，那选 `B` 系列，而我这种注重显示的就选`Q` 系列啦。

而系列字母前的数字则是分配的专有显存的大小，如 `GRID P4-4Q` 代表该vGPU将具有`4GB` 的专有显存。

`可用` 则是可同时启动的实例数目，比如上面的 `GRID P4-4Q` 每台实例具有 `4GB` 的显存，而 `P4` 总共具有 `8GB` 显存，那显然能同时启用的实例即是 `2台` （截图中因为我这台虚拟机已经开机了，所以显示还有1台可用）

### 3.3 Windows客户机

下面我们来看下 Windows 客户机 vGPU 是如何使用的，首先创建虚拟机：

```bash
# 创建6c4G的wendows虚拟机
qm create 116 \
--cores 6 \
--cpu host \
--bios ovmf \
--machine q35 \
--memory $((4 * 1024)) \
--net0 virtio,bridge=vmbr0,firewall=1 \
--name Win-P4-4Q \
--scsihw virtio-scsi-single \
--efidisk0 file=local-vzb:0,format=qcow2 \
--scsi0 file=local-vzb:32,format=qcow2 \
--tpmstate0 file=local-vzb:0 \
--ide2 file=ugreen:iso/Win11_22H2_Chinese_Simplified_x64v2.iso,media=cdrom \
--ide0 file=ugreen:iso/virtio-win-0.1.240.iso,media=cdrom \
--boot order="scsi0;ide2" \
--hostpci0 0000:05:00.0,mdev=nvidia-65,pcie=1 \
--audio0 device=ich9-intel-hda,driver=none
```

#### 3.3.1 Widnows vGPU 驱动

正常安装之后Windows设备管理器中没有正确识别到GPU，显示为“Microsoft基本显示适配器”

![image.png](https://img.linkzz.eu.org/main/images/2023/12/eb0f153c31686c458c3dd2d78f50e84c.png)

Windows 所用的驱动在刚才我们下载的文件下，我们安装 `Guest_Drivers/537.70_grid_win10_win11_server2019_server2022_dch_64bit_international.exe` ，安装完之后即可正常驱动vGPU了。

![image.png](https://img.linkzz.eu.org/main/images/2023/12/a7deb9cfc4c78e618f346abf9b305eda.png)

任务管理器也可以看到 GPU 具有了负载。

![image.png](https://img.linkzz.eu.org/main/images/2023/12/a42334144f6042c566ea93a002f98277.png)

安装完成之后可以打开 `RDP` ，关机将显示设置为“无”，将vGPU设置为主GPU，这样便于程序识别正确的 GPU 。

![image.png](https://img.linkzz.eu.org/main/images/2023/12/7073e1661ae497ec3ecf231da89aef99.png)

#### 3.3.2 vGPU 授权

上文提到过，我们需要对 vGPU 进行授权，获得相关的许可证才能完全的使用 vGPU，否则使用受限，无法完整使用。

我们使用 [fastapi-dls](https://github.com/GreenDamTan/fastapi-dls) 搭建一个授权服务器获取授权，具体做法这里不赘述，想了解详情可以查看项目文档，然后通过授权服务器获取授权，推荐使用 `Powershell` 脚本，方便快捷。

```Powershell
curl.exe --insecure -L -X GET https://<dls-hostname-or-ip>/-/client-token -o "C:\Program Files\NVIDIA Corporation\vGPU Licensing\ClientConfigToken\client_configuration_token_$($(Get-Date).tostring('dd-MM-yy-hh-mm-ss')).tok"
```

安装完重启，查询授权情况：

```Powershell
nvidia-smi -q  | Select-String "License"
```

![image.png](https://img.linkzz.eu.org/main/images/2023/12/50d63590427aaf647868521881a39939.png)

授权成功。

> 这里获取授权之前一定要先 ntp 同步一下系统时钟，我因为没有同步时钟导致和服务器的时间不一致授权一直失败，找了一番原因才找到原因所在。

#### 3.3.3 远程方案

vGPU 是没有物理输出的，但得益于今天各种串流技术的发展，我们的远程连接可以做到和实机相差无几的体验，除了微软自带的RDP方案之外也有不少选择，RDP 虽好，但毕竟不是专门为了 Gaming 串流而生，在视频播放，游戏串流等方面体验较专门为了 Gaming 串流优化的软件如 `Parsec` 、`Sunshine` 等体验还有差距，所以我们就试下后面这两者的串流。

- **[Parsec](https://parsec.app/downloads)**

`Parsec` 的安装我们默认即可，可以不安装 Parsec 的 VDD，因为 vGPU 驱动自带了虚拟显示适配器，我们就用他的就好了，安装好了之后登录账号。在客户端也登录对应的账号就可以看到远程机器了。

![image.png](https://img.linkzz.eu.org/main/images/2023/12/9c1d374788128c9cceb2bb41411f7606.png)

![image.png](https://img.linkzz.eu.org/main/images/2023/12/99e59b9203e3a4363247a899504c14f0.png)

看到这是我在公司连接的效果，场景为浏览器 [ufo test](https://www.testufo.com/) 使用的是 2k 分辨率的串流，网络开销 `7.65Mbps` 换算下来 `1.06Mb/s`， 网络延迟 `18ms`， 宿主机编码延迟 `7.34ms`， 解码延迟 `14.51ms` 左右，整体在 `40ms` 左右的延迟，在公司的使用情况是完全可以当作主力机使用了。

![image.png](https://img.linkzz.eu.org/main/images/2023/12/e6bd3120e7b7cae2b93282fc898fd134.png)

可以看到对应任务的 `GPU` 负载，主要是编码器，可以看到性能开销很小，还有余力进行其他大型的 3D 运算。

- **[Sunshine](https://github.com/LizardByte/Sunshine)**

`Sunshine` 是一个开源的 `moonlight` 协议的服务端实现，在 Nvidia 宣布将不再其驱动中集成 Shield 串流之后 Sunshine 的是使用变得广泛了起来，社区也很活跃，支持 Intel 、Amd 、 Nvidia 三大主流 GPU，具有 `webui` 配置也比较方便。

正常安装好了之后访问 `webui` ，第一次访问需要设置管理密码，好了之后安装打开客户端 [moonlight](https://github.com/moonlight-stream) ，内网环境应该自动发现了设备，我在公司需要主动添加， 输入主机 IP 即可添加。

![image.png](https://img.linkzz.eu.org/main/images/2023/12/9898ddd7b7ae8151c4b0fe8076351d4f.png)

点击图标连接会得到一个 `pin` 码，在 `webui` 中 `Pin` 选项卡下输入该 `pin` 码即可配对成功。

![image.png](https://img.linkzz.eu.org/main/images/2023/12/80cd8968c1eb581be32d1a0eabab040b.png)

接下来点击默认配置的 `Desktop` 程序即可连接桌面

![image.png](https://img.linkzz.eu.org/main/images/2023/12/af13b0a1636c11e23dc0a189ec77cb74.png)

接下来我们看下串流表现

![Dingtalk_20231212165936.jpg](https://img.linkzz.eu.org/main/images/2023/12/343089360d9819af7a9df17f08ae0b21.jpg)

同样场景 `2k` 分辨率的串流，网络延迟 `8ms` 左右，平均编码时间 `0.13ms` ，渲染时间 `0.13ms` ，和 `Parsec` 的指标不同，所以单纯比较数据也是没有意义的，以我实际体验的感觉来看，两者之间是没有很大的差别的，完全可以应付日常使用。

### 3.4 Linux 客户机

接下来体验一下 Linux 客户机，发行版选择 `Arch Linux` ，创建虚拟机：

```bash
qm create 117 \
--cores 6 \
--cpu host \
--bios ovmf \
--machine q35 \
--memory $((4 * 1024)) \
--net0 virtio,bridge=vmbr0,firewall=1 \
--name Arch-P4-4Q \
--scsihw virtio-scsi-single \
--efidisk0 file=local-vzb:0,format=qcow2 \
--scsi0 file=local-vzb:32,format=qcow2 \
--ide2 file=ugreen:iso/archlinux-2023.10.14-x86_64.iso,media=cdrom \
--boot order="scsi0;ide2" \
--hostpci0 0000:05:00.0,mdev=nvidia-65,pcie=1 \
--audio0 device=ich9-intel-hda,driver=none
```

#### 3.4.1 Linux 驱动

正常安装完之后就可以安装显卡驱动了

```bash
# 安装dkms
sudo pacman -S dkms

# 拷贝驱动至虚拟机之后安装
./Guest_Drivers/NVIDIA-Linux-x86_64-535.129.03-grid.run --dkms
```

安装完重启可以查看 `nvidia-smi` 是否正常识别显卡。

#### 3.4.2 Linux 授权

利用刚才我们搭建的授权服务同样可以授权 Linux 的驱动

```bash
# 下载授权文件
sudo curl --insecure -L -X GET https://<dls-hostname-or-ip>/-/client-token -o /etc/nvidia/ClientConfigToken/client_configuration_token_$(date '+%d-%m-%Y-%H-%M-%S').tok

# 重启服务
sudo systemctl restart nvidia-gridd

# 查看授权信息
nvidia-smi -q | grep "License"
```

授权正常将看到一下信息：

![image.png](https://img.linkzz.eu.org/main/images/2023/12/69eea3c965a869046f405dc54682e214.png)

#### 3.4.3 CUDA Toolkit

vGPU 支持 CUDA，正常安装 [CUDA Toolkit](https://developer.nvidia.com/cuda-downloads) 即可，我这里安装了 `12.2` 版本的。

![image.png](https://img.linkzz.eu.org/main/images/2023/12/6a2d0e2238306a4d9ffd2bb1e5070010.png)

#### 3.4.4 桌面环境

Linux 桌面我使用不多，在我看来 Linux server 模式很稳定，但是 Linux GUI 却常有问题，用处真的不大，尤其 `vGPU` 没有物理输出，一旦 GUI 奔溃难以远程将很难调试，这里我只是安装了驱动支持较好的 `x11-vnc` 和 `rustdesk` 用来简单测试一下，桌面环境则是使用的 `KDE`。

- **X11-VNC**

![image.png](https://img.linkzz.eu.org/main/images/2023/12/b3fb2e8ab9b54d653154f7b3999b108b.png)

VNC 的客户端我选择了 `TigerVNC` ，可以看到分辨率可以设置为 `2k` ，但奇怪的是这个客户端无法设置缩放，所以我只能拖动高度条使用，要么就是全屏之后鼠标靠近边缘可以自动滑动画面。或者像下面设置为 `1080P` 使用。

![x11-vnc-p4.gif](https://img.linkzz.eu.org/main/images/2023/12/ad117e2670743000628b1e30106ab448.gif)

- **rustdesk**

[rustdesk](https://github.com/rustdesk/rustdesk) 是一个由 `rust` 编写的开源的远程桌面工具，其目标是做 `TeamViewer` 的替代品，支持 `X` 和 `Wayland` ，支持自建中心服务，`rustdesk` 有维护 `Arch Linux` 的包，直接安装即可。

```bash
sudo pacman -S rustdesk
```

下面是客户端连接的效果：

![rustdesk-p4.gif](https://img.linkzz.eu.org/main/images/2023/12/9e92520c2b5115716b267ecc5f6fd974.gif)

两个工具的远程都还算流畅，但是有个问题暂时没有眉目，就是无法调用显卡硬件编码，所以你看到的 `rustdesk` 的远程是由 `CPU` 进行编码的串流，所以看起来有一丝不流畅，但是显卡的 `3D` 加速是正常的，这个问题待日后再解决吧。

## 4. 总结

这张 `Tesla P4` 的表现可以说是远超我的预期，功耗不高的同时可以解锁到几个 `VM` 同时调用显卡加速，以后还可以用来部署一些 `AI` 模型，仅 300 左右的价格能做到这么多事，可以说物超所值，兴许以后出二手还不会太掉价！哈哈，理财产品。
