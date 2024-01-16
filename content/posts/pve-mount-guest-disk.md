---
title: pve 宿主机挂载虚拟机磁盘镜像
date: 2024-01-16
draft: false
description: "pve 宿主机挂载虚拟机磁盘镜像"
tags:
  - pve
  - 虚拟机
categories:
  - pve
  - Linux
lightgallery: true

toc:
  auto: true
---

## 1. 背景

有的时候我们需要修改虚拟机的文件，但是此时虚拟机却因为某些原因无法启动了，比如说虚拟机黑苹果修改了 `EFI` 导致启动不了，这时我们有什么办法呢，有人说我们添加一个可以启动的 `EFI` 启动设备再来修改原来 `ESP` 分区 (即原 `EFI` 文件系统) 不就好了，诚然，这是一个办法，但我们今天要介绍的是另一个办法，直接在宿主机挂载虚拟机的磁盘分区，就拿黑苹果 `EFI` 分区为例。

## 2. Raw

`raw` 格式的磁盘镜像文件可以使用 `losetup` 虚拟成一个块设备。再使用 `kpartx` 读取分区表英创建设备映射，从而可以从设备挂载到宿主机：

### 2.1 挂载

- 安装 kpartx

```shell
apt install kpartx
```

下面以虚拟机编号为 108 的 disk1 为例子

- 虚拟块设备

```shell
losetup /dev/loop0 /dev/mapper/pve-vm--108--disk--1
```

- 读取设备的分区表并创建设备分区映射

```shell
kpartx -av /dev/loop0
```

- 查看分区映射

```shell
➜  ~ lsblk
NAME                         MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0                          7:0    0     1G  0 loop
└─loop0p1                    253:23   0  1024M  0 part
sda                            8:0    1  14.6G  0 disk
├─sda1                         8:1    1  11.8G  0 part
├─sda2                         8:2    1    16M  0 part
├─sda3                         8:3    1   2.7G  0 part
├─sda4                         8:4    1    16M  0 part
├─sda5                         8:5    1     2M  0 part
├─sda6                         8:6    1   512B  0 part
├─sda7                         8:7    1   512B  0 part
├─sda8                         8:8    1    16M  0 part
├─sda9                         8:9    1   512B  0 part
├─sda10                        8:10   1   512B  0 part
├─sda11                        8:11   1     8M  0 part
└─sda12                        8:12   1    32M  0 part
nvme1n1                      259:0    0 238.5G  0 disk
├─nvme1n1p1                  259:1    0 238.5G  0 part
└─nvme1n1p9                  259:2    0     8M  0 part
```

可以看到 `loop0` 下有一个分区

- 挂载分区

```shell
mkdir pve-efi
mount /dev/mapper/loop0p1 pve-efi
```

- 查看分区文件

```shell
➜  ~ ls -al pve-efi
total 11
drwxr-xr-x  6 root root  512 Jan  1  1970 .
drwx------ 18 root root 4096 Jan 16 14:09 ..
-rwxr-xr-x  1 root root 4096 Dec 29 07:12 ._EFI
drwxr-xr-x  4 root root  512 Dec 29 07:12 EFI
drwxr-xr-x  2 root root  512 Jan 13 09:48 .fseventsd
drwxr-xr-x  3 root root  512 Dec 29 07:12 .Spotlight-V100
drwxr-xr-x  3 root root  512 Dec 29 07:12 .Trashes
```

### 2.2 卸载

按照挂载的顺序反向卸载

```shell
# 文件系统
umount pve-efi
# 设备分区
kpartx -d /dev/loop0
# 块设备虚拟
losetup -d /dev/loop0
```

## 3. Qcow2

### 3.1 挂载

`Qcow2` 镜像需要通过 nbd 模块挂载

- 检查 nbd 模块是否加载

```shell
lsmod | grep nbd
```

若没有输出结果则加载 nbd

```shell
modprobe nbd max_part=8
```

- 挂载镜像，下面以编号 103 的虚拟机磁盘 2 文件为例

```shell
qemu-nbd --connect=/dev/nbd0 /var/lib/vz-b/images/113/vm-113-disk-1.qcow2
```

- 查看分区

```shell
➜  ~ fdisk -l /dev/nbd0
Disk /dev/nbd0: 32 GiB, 34359738368 bytes, 67108864 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: DACC591B-A767-EA4F-8028-9757172F0505

Device          Start      End  Sectors  Size Type
/dev/nbd0p1  11472896 67108815 55635920 26.5G Linux filesystem
/dev/nbd0p2     20480    53247    32768   16M ChromeOS kernel
/dev/nbd0p3   5894144 11472895  5578752  2.7G ChromeOS root fs
/dev/nbd0p4     53248    86015    32768   16M ChromeOS kernel
/dev/nbd0p5    315392  5894143  5578752  2.7G ChromeOS root fs
/dev/nbd0p6     16448    16448        1  512B ChromeOS kernel
/dev/nbd0p7     16449    16449        1  512B ChromeOS root fs
/dev/nbd0p8     86016   118783    32768   16M Linux filesystem
/dev/nbd0p9     16450    16450        1  512B ChromeOS reserved
/dev/nbd0p10    16451    16451        1  512B ChromeOS reserved
/dev/nbd0p11       64    16447    16384    8M unknown
/dev/nbd0p12   249856   315391    65536   32M EFI System
```

可以看到我挂载的是一个 `FydeOS` 虚拟机的镜像文件，是基于 `Chromium OS` 的操作系统，所以看到了文件系统有 `ChromeOS kernel` 等，下面挂载下 `linux filesystem` 分区。

- 挂载分区

```shell
mkdir chromeos-file
mount /dev/nbd0p1 chromeos-file
```

- 查看分区文件

```shell
➜  ~ cd chromeos-file
➜  chromeos-file ll
total 275992
drwxr-xr-x   9 root root       4096 Nov 26 15:04 .
drwx------  18 root root       4096 Jan 16 16:19 ..
drwxr-xr-x. 17 root root       4096 Nov 14 10:59 dev_image
drwxrwxr-x.  2 root root       4096 Nov 26 15:04 encrypted
-rw-------.  1 root root 8357232640 Nov 26 16:26 encrypted.block
-rw-------.  1 root root         48 Nov 26 15:04 encrypted.key
drwxr-xr-x.  3 root root       4096 Nov 26 15:04 etc
drwxr-xr-x.  6 root root       4096 Nov 26 15:04 home
drwx------   2 root root      16384 Nov 26 14:55 lost+found
drwxr-xr-x. 13 root root       4096 Nov 26 15:04 unencrypted
drwx--x--x.  5 root root       4096 Nov 26 15:04 var_overlay
```

这样就挂载好了一个分区，可以进行编辑了，编辑好了之后卸载回到客户机即可看到改动了。

### 3.2 卸载

卸载还是反向操作

```shell
umount chromeos-file
qemu-nbd --disconnect /dev/nbd0
```

- 卸载 nbd 模块 (可选)

```shell
modprobe -r nbd
```

## 4. 结语

至此就完成了 pve 常用的 2 种格式的镜像文件的挂载操作，该用法的需求是我在折腾黑苹果 `EFI` 开不了机的时候爬帖子学到的，其他格式的文件挂载可以查看 `foxi` 的 [这篇文章](https://foxi.buduanwang.vip/linux/552.html/)
