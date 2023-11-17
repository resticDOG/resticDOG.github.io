---
title: pve环境10代i5-10400核显直通
date: 2023-11-17
draft: false
description: "pve环境10代i5-10400核显直通"
tags:
  - pve
  - 虚拟机
  - 直通
categories:
  - pve
lightgallery: true

toc:
  auto: true
---

## 1. 背景

3年前，怀着不折腾不舒服斯基的心情，我按当时最新的硬件（大概）组了一台All in One
主机，这这台主机拥有当时最为先进的14nm+++工艺的CPU i7-10400和性能遥遥领先的
UHD630核显，我当时的想法也很普通，就是集家庭透明网关、NAS、Linux开发机、Windows
娱乐机、以及Linux Docker服务器于一身的超级家庭数据中心，下面看下我当时所选的硬
件：

| 类别 | 项目               | 价格 |
| ---- | ------------------ | ---- |
| CPU  | i5-10400           | --   |
| 主板 | 华擎B-460M钢铁传奇 | --   |
| 内存 | 酷兽8G \* 4 = 32G  | --   |
| SSD  | 三星980 1TB        | --   |
| SSD  | 爱国者 128GB       | --   |
| HDD  | 希捷酷狼4TB \* 2   | --   |
| 机箱 | 先马趣造           | --   |
| 散热 | 九州风神           | --   |

当时我选配的时候正值挖矿潮，显卡是没想法的，只当一个服务器用，这其中Windows娱乐
机的需求一开始没有显卡加速，使用起来确实无法胜任我的需求，我希望的是这台机器能24
小时开机，能浏览网页，能流程播放h265视频。但很显然只依靠CPU模拟的显卡是无法完成
以上工作的，于是就想到了直通核显到Windows客户机，然而当时针对10代的直通教程真是
少之又少，爬了很多帖子之后只能做到直通安装Ubuntu并拥有hdmi输出，Windows则不是
hdmi黑屏就是显卡驱动Code 43，后来矿难入了一张2060s之后直通用来打游戏，核显就只是
用来为jellyfin提供硬件解码加速，直通核显这事就一直搁浅了。而今折腾之心渐起，而今
itel也早已经更新到了14代酷睿，igpu性能也较10代大幅提升了，早先的10代直通恐怕也很
多大佬已经研究透彻了，而且我平时用的一个Windows虚拟机主要使用微软的RDP远程桌面，
没有GPU加速下看视频内容实在是难以忍受，于是开始了新的爬帖之旅。果然，10代直通的
中文内容也多了起来，也有很多人做成了直通并显示hdmi接口内容，于是就有了今天的文
章。

## 2. BIOS准备

这里参考pve官方Wiki内容，需要BIOS设置好直通所需的技术：

### 2.1 启用VT-d

启用CPU虚拟化，启用vt-d（直通必备）

### 2.2 启用CSM

启用CSM并将所有启动项设置为"仅传统"

> 待会创建的虚拟机将使用legacy启动模式直通，若不开启DP接口和hdmi接口将无法输出画
> 面，而且如果你是先用pve的默认显示安装完Windows再安装核显驱动，然后再添加核显的
> pci设备，启动之后核显还是无法驱动的，会报错“代码43”，除非你有一个正确的igpu
> bios文件，有了这个文件的话核显可正常驱动，但是看不到pve的SeaBIOS界面，启动
> Windows核显驱动正常加载之后可以输出Windows画面，本文最后会介绍提取vbios的方
> 法。

### 2.3 启用多图形适配器

有些主板BIOS设置的主GPU是PCIE通道的gpu且多图形适配器功能是关闭的，也就是说你的主
板PCIE X16的插槽插上显卡之后核显会被屏蔽，需要开启多图形适配器功能（我的主板是这
个设置项）并且将主图形适配器设置为“板载”。

## 3. 宿主机(pve)设置

我的pve环境如下：

```bash
➜  ~ pveversion --verbose
proxmox-ve: 7.3-1 (running kernel: 5.15.74-1-pve)
pve-manager: 7.3-3 (running version: 7.3-3/c3928077)
pve-kernel-5.15: 7.2-14
pve-kernel-helper: 7.2-14
pve-kernel-5.15.74-1-pve: 5.15.74-1
ceph-fuse: 15.2.17-pve1
corosync: 3.1.7-pve1
criu: 3.15-1+pve-1
glusterfs-client: 9.2-1
ifupdown2: 3.1.0-1+pmx3
ksm-control-daemon: 1.4-1
libjs-extjs: 7.0.0-1
libknet1: 1.24-pve2
libproxmox-acme-perl: 1.4.2
libproxmox-backup-qemu0: 1.3.1-1
libpve-access-control: 7.2-5
libpve-apiclient-perl: 3.2-1
libpve-common-perl: 7.2-8
libpve-guest-common-perl: 4.2-3
libpve-http-server-perl: 4.1-5
libpve-storage-perl: 7.2-12
libspice-server1: 0.14.3-2.1
lvm2: 2.03.11-2.1
lxc-pve: 5.0.0-3
lxcfs: 4.0.12-pve1
novnc-pve: 1.3.0-3
proxmox-backup-client: 2.2.7-1
proxmox-backup-file-restore: 2.2.7-1
proxmox-mini-journalreader: 1.3-1
proxmox-widget-toolkit: 3.5.3
pve-cluster: 7.3-1
pve-container: 4.4-2
pve-docs: 7.3-1
pve-edk2-firmware: 3.20220526-1
pve-firewall: 4.2-7
pve-firmware: 3.5-6
pve-ha-manager: 3.5.1
pve-i18n: 2.8-1
pve-qemu-kvm: 7.1.0-4
pve-xtermjs: 4.16.0-1
qemu-server: 7.3-1
smartmontools: 7.2-pve3
spiceterm: 3.2-2
swtpm: 0.8.0~bpo11+2
vncterm: 1.7-1
zfsutils-linux: 2.1.6-pve1
```

### 3.1 内核参数

添加内核参数，启用iommu：

```bash
vim /etc/default/grub
```

修改 `GRUB_CMDLINE_LINUX_DEFAULT` 项的内容如下：

```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt pcie_acs_override=downstream,multifunction initcall_blacklist=sysfb_init"
```

更新启动参数

```bash
update-grub
```

> 参数说明： `quiet`: 启用安静模式 `intel_iommu=on iommu=pt`: intel平台开启iommu
> 功能 `pcie_acs_override=downstream,multifunction`: 强制启用更高级别的设备隔离
> `initcall_blacklist=sysfb_init`: 这是个重要的参数，由于内核在启动时，初始化函
> 数可能会启动显示设备用于输出系统帧缓冲，导致显卡设备被占用，从而在接下来的直通
> 中出现异常。

### 3.2 加载vfio相关模块

```bash
echo -e "vfio\nvfio_iommu_type1\nvfio_pci\nvfio_virqfd" > /etc/modules
```

### 3.4 绑定相关设备id

查看核显设备id

```bash
lspci -kn -s 0000:00:02
```

```text
00:02.0 0300: 8086:9bc8 (rev 03)
        DeviceName: Onboard - Video
        Subsystem: 1849:9bc8
        Kernel driver in use: i915
        Kernel modules: i915
```

上面的 `8086:9bc8` 即是核显的 `vendor:device` ，需要编辑内核参数，使得vfio驱动接
管该设备：

```bash
echo "options vfio-pci ids=8086:9bc8" > /etc/modprobe.d/vfio.conf
```

### 3.3 屏蔽相关核显驱动

```bash
echo -e "blacklist snd_hda_intel \n \
blacklist snd_hda_codec_hdmi \n \
blacklist i915" > /etc/modprobe.d/blacklist.conf
```

刷新`initramfs`

```bash
update-initramfs -u -k all
```

重启pve宿主机

```
reboot
```

## 4. 虚拟机安装

### 4.1 虚拟机镜像准备

先准备好`Win11`安装镜像和`virtio`驱动镜像,virtio驱动镜像可以
在[这里](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/?C=M;O=D)下
载。

### 4.2 虚拟机创建

创建虚拟机，OS类型选Windows，CPU类型选择host，
![image](https://img.linkzz.eu.org/main/images/2023/11/6e0f4819950643ac932c85d047f4f077.png)

BIOS选SeaBIOS，机器类型选i440fx，TPM可加可不加，安装的时候可以跳过检测

![image.png](https://img.linkzz.eu.org/main/images/2023/11/4560286c9cf546c942b5ede6ab7e9707.png)

磁盘总线选择SCSI，大小默认32G，如果空间充足的可适当增加，我这个Win11镜像32G勉强
够用，如果用的储存是ssd可以勾选SSD仿真。

![image.png](https://img.linkzz.eu.org/main/images/2023/11/3e4067c7ded86e3c23645eb71b645897.png)

核心给到6颗VCPU核心，类别选择host，勾选NUMA，可以更好的利用硬件性能。

![image.png](https://img.linkzz.eu.org/main/images/2023/11/79e3ebb0fd963221480f8d5eda81766d.png)

内存大于4G，这里设置为8192（8GB）

![image.png](https://img.linkzz.eu.org/main/images/2023/11/99e18e2e9b5519bf7cba09ba91239c0b.png)

网络模型选择virtio，性能更好，其他默认

![image.png](https://img.linkzz.eu.org/main/images/2023/11/8959fad9e1a70880df7209d988a54718.png)

最后确认，不要勾选立即启动，下一步添加直通设备。

- 选择硬件，将显示设置为无，禁用CPU的虚拟显示适配器，我们将使用直通的核显的DP接
  口来显示和完成Windows的安装。
- 添加PCI设备0000:00:02.0
- 添加virtio-win的iso镜像
- 添加键盘和鼠标usb设备
  ![image.png](https://img.linkzz.eu.org/main/images/2023/11/241c273351e802fddf6b0e0e3ec36101.png)

编辑vm配置文件：

```bash
vim /etc/pve/qemu-server/111.conf
```

修改2个地方：

- 添加一
  行：`args: -set device.hostpci0.addr=02.0 -set device.hostpci0.x-igd-gms=0x2 -set device.hostpci0.x-igd-opregion=on`
- `hostpci0`这一行添加`legacy-igd=1`

最终的conf文件：

```conf
args: -set device.hostpci0.addr=02.0 -set device.hostpci0.x-igd-gms=0x2 -set device.hostpci0.x-igd-opregion=on
agent: 1
boot: order=scsi0;ide2;net0
cores: 6
cpu: host
hostpci0: 0000:00:02.0,legacy-igd=1
ide2: ugreen:iso/Win11_22H2_Chinese_Simplified_x64v2.iso,media=cdrom,size=5723780K
machine: pc-i440fx-7.1
memory: 8192
meta: creation-qemu=7.1.0,ctime=1700200085
name: Win11-iGD
net0: virtio=F6:ED:C6:88:12:F6,bridge=vmbr0,firewall=1
numa: 1
ostype: win11
scsi0: local-lvm:vm-112-disk-0,iothread=1,size=32G,ssd=1
scsihw: virtio-scsi-single
smbios1: uuid=6fd8a123-7bcc-46ba-934e-cb8d5777a9ed
sockets: 1
vga: none
usb0: host=046d:c07e
usb1: host=04d9:0295
vmgenid: 23513025-aa61-4288-88bc-e0daa646e154
```

接下来开启虚拟机，插上核显的 DP 接口，观察是否能看到画面，我的主板的情况是只有
DP 接口能输出画面，HDMI 接口是毫无反应的，后面提取了 vbios romfile 之后两个接口
都可输出画面，后文会提到。

看到输出画面后猛按键盘开始安装过程：

- 第一个问题就是安装过程中需要先加载virtio的驱动才能看到硬盘，直接选择加载驱动安
  装即可。

- 第二，就是安装过程中遇到TMP检测和SecureBoot检测显示不可安装Windows，这时可通过
  按下 `SHIFT + F10` 呼出cmd，输入regedit回车，进
  入`HKEY_LOCAL_MACHINE\SYSTEM\Setup`，创建一个新的项目：`LabConfig`，添加3
  个`DWORD`项，双击项目设置其值为1（十进制）：

```reg
BypassTPMCheck
BypassSecureBootCheck
BypassRAMCheck
```

关闭注册表编辑器之后点后退按钮，再进行下一步即可正常安装。

- 第三，不想联网的可按下 `SHIFT + F10` 呼出cmd，输入一下内容，跳过联网，使用本地
  账户安装。

```cmd
oobe\BypassNRO.cmd
```

之后安装intel核显驱动，我直接安装的是最新版本，可正常驱动：

![image.png](https://img.linkzz.eu.org/main/images/2023/11/2bd8310ec946831684cd205209afeaa1.png)

驱动版本`31.0.101.2125`

![image.png](https://img.linkzz.eu.org/main/images/2023/11/72a93f13ecea1df612fd44309d78cc2e.png)

dxdiag显示Dx加速功能正常（我这里用了sunshine + 虚拟显示器进行远程连接的，所以显
示信息可能和直连显示器的不一样）：

![image.png](https://img.linkzz.eu.org/main/images/2023/11/5bdc34a808992606ec8479a5c129da7a.png)

## 5. 提取核显VBIOS

### 5.1 为什么需要VBIOS

上文中我就提到提取核显vbios的情况，这里有个故事（不想听我bb的可直接跳过），在我
满心欢喜的成功直通了核显之后，我又入了一块 `Tesla P4` 用来折腾Nvidia的vGPU，因为
这张显卡功耗只有区区75w，而且我的主板刚好还剩一个`PCIE 3.0 X16`的插槽未使用，但
是当我插上这张显卡之后我傻眼了，我的2块ssd根本就无法识别了，我的系统和所有vm数据
都是存储在上面的，于是一通排查之后发现在启用CSM的情况下，我的主板无法同时使用ssd
和第二个X16的PCIE插槽，需要关闭CSM才正常，可是我的显卡直通还有用吗，打开虚拟机之
后果然不出我所料，显卡显示代码43，无法正常驱动，于是我想到了10代之后无法进行传统
启动的intel平台也有直通核显成功的案例，这其中的关键就是核显vbios，所以就有了这个
提取vbios的过程。

### 5.2 提取VBIOS

- 下载[UBU](https://mega.nz/folder/k4Z0FAra#hMIhuLoTte8IcwtiDibiAw)
- 主板官网下载最新bios固件，以我的华擎B460M钢铁传奇为例，下载Instant Flash版本
  的，这里我们下载稳定版。
  ![image.png](https://img.linkzz.eu.org/main/images/2023/11/2bf254d93a3a5546038cd48ab88b521c.png)

- 打开`UBU.bat`，选择bios文件
  ![image.png](https://img.linkzz.eu.org/main/images/2023/11/2de25e44e48bc22df29f52913c9c4384.png)

读取到了GOP驱动
![image.png](https://img.linkzz.eu.org/main/images/2023/11/edd963a2629d8502b6e75d002a3cffff.png)

选择 ”2“ 板载显卡GOP驱动

![image.png](https://img.linkzz.eu.org/main/images/2023/11/90e27e4b2a5ab06f1c3b46d7253c6794.png)

选择 ”S“ 导出驱动

在UBU的目录下 ”Extracted“ 目录中可找到导出的Gop驱动
![image.png](https://img.linkzz.eu.org/main/images/2023/11/1313387e259ee7f7776eba71698eb9d4.png)

- 转换为rom文件

下
载[EfiRom](https://github.com/tianocore/edk2-BaseTools-win32/blob/master/EfiRom.exe)工
具，使用EfiRom工具将 `IntelGopDriver.efi` 转为 `rom`

先查看核显的`vendor`和`device`编号，上文已经提到了方法：

```cmd
lspci -kn -s 0000:00:02
```

```text
00:02.0 0300: 8086:9bc8 (rev 03)
        DeviceName: Onboard - Video
        Subsystem: 1849:9bc8
        Kernel driver in use: vfio-pci
        Kernel modules: i915
```

`8086:9bc8`: 8086是vendor编号，9bc8是device编号，皆是十六进制值，下面的命令中需
要添加十六进制前缀`0x`：

```cmd
.\EfiRom.exe -v -e .\IntelGopDriver.efi -o igd.rom -f 0x8086 -i 0x9bc8
```

```text
EfiRom tool start.

Processing EFI file    .\IntelGopDriver.efi

  Got subsystem = 0xB from image

  File size   = 0x11E60

  Output size = 0x12000

EfiRom tool done with return code is 0x0.
```

命令成功退出，将生成 `igd.rom` 上传到 pve 宿主机 `/usr/share/kvm`目录。

### 5.2 验证rom文件是否有效（可选）

正常来说rom文件到此就可以正常应用到虚拟机输出画面了，但是也可在这一步使
用`rom-parser` 工具验证下rom文件的有效性。

克隆 `rom-parser` 库：

```bash
git clone https://github.com/awilliam/rom-parser
```

安装make工具

```bash
apt install make
```

编译rom-parser

```bash
cd rom-parser
make
```

成功编译之后验证rom文件

```bash
./rom-parser ./igd.rom
```

查看输出内容

```text
Valid ROM signature found @0h, PCIR offset 1ch
        PCIR: type 3 (EFI), vendor: 8086, device: 9bc8, class: 000000
        PCIR: revision 3, vendor revision: 0
                EFI: Signature Valid, Subsystem: Boot, Machine: X64
        Last image
```

可以看到rom文件有效

### 5.4 应用romfile

编辑刚才的虚拟机配置文件

```bash
vim /etc/pve/qemu-server/112.conf
```

在`hostpci0`这一行添加 `romfile=igd.rom`

最终配置文件如下：

```conf
agent: 1
args: -set device.hostpci0.addr=02.0 -set device.hostpci0.x-igd-gms=0x2 -set device.hostpci0.x-igd-opregion=on
boot: order=scsi0;ide2;net0;ide0
cores: 6
cpu: host
hostpci0: 0000:00:02.0,legacy-igd=1,romfile=igd.rom
machine: pc-i440fx-7.1
memory: 8192
meta: creation-qemu=7.1.0,ctime=1700200085
name: Win11-iGD
net0: virtio=F6:ED:C6:88:12:F6,bridge=vmbr0,firewall=1
numa: 1
ostype: win11
scsi0: local-lvm:vm-112-disk-0,iothread=1,size=32G,ssd=1
scsihw: virtio-scsi-single
smbios1: uuid=6fd8a123-7bcc-46ba-934e-cb8d5777a9ed
sockets: 1
usb0: host=046d:c07e
usb1: host=04d9:0295
vga: none
vmgenid: 23513025-aa61-4288-88bc-e0daa646e154
```

打开虚拟机，在系统加载核显驱动后DP接口和HDMI接口均正常输出Win11画面。
