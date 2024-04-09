---
title: Docker 环境下的 qBittorrent 容器迁移至 lxc 保留配置
date: 2024-04-09
description: "Docker 环境下的 qBittorrent 容器迁移至 lxc 保留配置"
layout: post

tags:
  - pve
  - Docker
categories:
  - pve
  - Linux
lightgallery: true

toc:
  auto: true
---

## 1. 背景

我的 `qBittorrent` 下载器是部署在 `Docker` 容器中的，而这个 Docker 的宿主机是一个 pve 的容器，他有一个独立的 IP,这个 IP 的网关指向了一个透明网关以实现大陆网络优化（国内特色），关于我的网络拓扑可以查看我以前的这篇 [文章](openwrt-wireguard) 但是下载器的流量经过 `clash` 之后需要单独对 `BT` 流量命中规则，我用的是 clash-meta 的核心，支持逻辑规则，于是我是这样写的：

```yaml
# bt下载全直连
- AND,((SRC-IP-CIDR,192.168.5.128/32),(NOT,((DST-PORT,443)))),DIRECT
```

以上规则解释就是：来自 `192.168.5.128`IP 的目标端口非 `443` 的连接全直连，比较简单粗暴一刀切，但是这个规则还是会误杀一部分的流量，但至少可以避免大部分的 `BT` 流量了。

虽然问题是暂时解决了，但这个方案并不是最优，存在以下坑：

1. 如果 `Docker` 的宿主机 `IP` 换了规则得更新
2. 目标端口 80 的流量也应该继续下面的规则匹配 (当然这是因为懒没加到规则里面)
3. 会误杀掉一部分的流量

## 2. 迁移

为了解决上面提到的问题，也为了把 `qbittorrent` 版本升级一下，我决定单独给他一个创建一个 `lxd` 容器，这样可以将网络直接指向上级网关，不必走透明网关。

### 2.1 创建容器

通过 pve 的内置 pct 命令创建容器：

```shell
pct create 117 \
	ugreen:vztmpl/archlinux-base_20230608-1_amd64.tar.zst \
	--cores 6 \
	--memory 512 \
	--hostname qbit \
	--net0 name=eth0,bridge=vmbr0 \
	--features mount="nfs;cifs" \
	--swap 512 \
	--rootfs volume=local-lvm:8
```

### 2.2 设置容器

这样就创建好了一个基于 `archlinux` 的 ct 容器，接下来启动进入容器简单设置一下：

启动容器：

```shell
pct start 117
```

进入容器：

```shell
pct enter 117
```

修改网络开启 dhcp 自动获取 IP

```shell
vi /etc/systemd/network/eth0.network
```

```text
[Match]
Name = eth0

[Network]
Description = Interface eth0 autoconfigured by PVE
DHCP = yes
IPv6AcceptRA = false
```

将 DHCP 改为 yes, 重启网络服务

```shell
systemctl restart systemd-networkd
```

以上内容均可以通过 pve 的 webui 界面完成。

### 2.3 迁移配置 qbittorrent-nox

在 headless 无头环境下运行 qbittorrent 我们需要下载 qbittorrent-nox，`nox -> no X server`，没有 X 服务，外国人还是会起名。

archlinux 下直接通过包管理器安装即可，版本也是当下最新的 `4.6.4` 版本

```shell
pacman -S qbittorrent-nox
```

启动 qbittorrent-nox

```shell
systemctl enable qbittorrent-nox && systemctl start qbittorrent-nox
```

将 Docker 容器中的 qbittorrent 配置文件迁移到 `~/.config/qBittorrent/qBittorrent.conf` 中，我的是 `v4.4.3.1` 迁移到 `v4.6.4`，迁移比较平滑，我所有的配置都继承了下来，如果是更低版本就不好说了，可以选择性的迁移。

### 2.4 下载数据迁移

原来的下载数据种子文件保留在 BT_backup 文件夹中，在 Docker 容器和新的配置文件下载路径一致的情况下可以平滑迁移，qbittorrent 不会报文件丢失，当然如果不需要迁移的话直接忽略掉就好了。

将 BT_backup 文件夹内容复制到新的 qbittorrent 数据文件夹下相应的位置即可。

## 3. 结语

此次迁移过程还是很顺利的，毕竟版本的变化也没那么大，由此我们可以学习下优质开源项目的向下兼容，破坏性的变化引入应该是很谨慎的。

看下迁移后的截图，新的 UI 界面图标太丑了🤣

![image.png](https://img.linkzz.eu.org/main/images/2024/04/aad38a0e6e4c9cdcf7956938c5f520cb.png)
