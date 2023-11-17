---
title: 一次防火墙纠错，让我更了解防火墙机制
date: 2023-11-07
draft: false
description: "一次防火墙纠错，让我更了解防火墙机制"
tags:
  - iptables
categories:
  - Linux
lightgallery: true

toc:
  auto: true
---

## 1. 起因

工作中我一直使用wireguard搭建公司 -> 家的桥梁，用于随时随地访问家中内网的各种服
务，如gogs代码仓库、nextcloud同步工作文件，抑或是远程连接windows虚拟机用来摸鱼，
然而尽管拥有公网ip，家里和公司之间只有区区10ms的网络时延，却还是因为运营商对UDPl
的QOS而影响远程服务的传输体验。于是我就搭建了一套phantun服务用来将UDP流量转换为
TCP流量而避过运营商的QOS，建好之后确实体验提升明显，远程的moonlight能稳定跑满30M
上传带宽，然而当时搭建的时候的iptables规则未保存，今天的一次重启让我的phantun死
活连不上，于是就有了今天的文章。

## 2. 排查

### 2.1 报错日志

由于是客户端重启之后连不上的，排除服务端错误的情况，从客户端查起，查看phantun报
错：

```bash
sudo systemctl status phantun.service
```

```text
Nov  8 10:22:11 new-business-dev-001 phantun_client[5517]:  ERROR client > Unable to connect to remote {my-home-ip}:25379
Nov  8 10:22:14 new-business-dev-001 phantun_client[5517]:  ERROR client > Unable to connect to remote {my-home-ip}:25379
```

客户端一直报错连不上服务端，25379端口是`phantun_server`的监听端口，之前连接是好
的，不可能有问题。

### 2.2 抓包分析

遇到网络相关的问题，抓包分析是最为直观的解决方案，我们查看往来服务端的流量信息:

```bash
sudo tcpdump -i any host {my-home-ip}
```

```text
10:28:48.652090 IP 192.168.200.2.61048 > {my-home-ip}.25379: Flags [S], seq 0, win 65535, options [nop,wscale 14], length 0
10:28:49.653239 IP 192.168.200.2.61048 > {my-home-ip}.25379: Flags [S], seq 0, win 65535, options [nop,wscale 14], length 0
10:28:50.654392 IP 192.168.200.2.61048 > {my-home-ip}.25379: Flags [S], seq 0, win 65535, options [nop,wscale 14], length 0
10:28:51.655551 IP 192.168.200.2.61048 > {my-home-ip}.25379: Flags [R], seq 0, win 65535, length 0
```

IP `192.168.200.2` 一直在向服务端发送 SYN 握手包，而服务端是没有响应任何内容的，
这明显可以看出这个包是没办法送达的，因为 `192.168.200.2` 这个地址属于
phantun_client 创建的tun设备的地址，是没有互联网访问能力的，所以我们需要将该接口
的包转发至具有互联网访问能力的接口，通俗的讲就是 lan 口，首先确认内核开启了 IP
转发：

```bash
sudo sysctl -a | grep ip.forward
```

```text
net.ipv4.ip_forward = 1
net.ipv4.ip_forward_update_priority = 1
net.ipv4.ip_forward_use_pmtu = 0
```

看到 `net.ipv4.ip_forward = 1` 的值为1已经开启了IP转发，如若不是需开启ip转发，开
启ipv4的IP转发即可：

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

启用源地址转换：

```bash
sudo iptables -t nat -A POSTROUTING -o ens160 -j MASQUERADE
```

这里 `ens160` 是我可以访问互联网的接口，这里指的是对 `nat` 表的 `POSTROUTING` 链
添加一条规则，对出口接口为 `ens160` 的所有流量执行 `MASQUERADE` 操
作，`MASQUERADE` 是源地址转换（SNAT）的一种实现，通过这样的操作即可实现局域网内
的地址可转换为ens160的地址进行互联网访问，从而使得 `phantun_client` 的流量包能成
功发出。

### 2.3 还是不行？

再看看看日志：

```text
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked v1), capture size 262144 bytes
11:09:26.356501 IP 192.168.200.2.5320 > {my-home-ip}.25379: Flags [S], seq 0, win 65535, options [nop,wscale 14], length 0
11:09:27.357822 IP 192.168.200.2.5320 > {my-home-ip}.25379: Flags [S], seq 0, win 65535, options [nop,wscale 14], length 0
11:09:28.358988 IP 192.168.200.2.5320 > {my-home-ip}.25379: Flags [S], seq 0, win 65535, options [nop,wscale 14], length 0
11:09:29.360160 IP 192.168.200.2.5320 > {my-home-ip}.25379: Flags [R], seq 0, win 65535, length 0
```

额。。。看样子我们并没有成功，日志还是和之前没有区别，这难道是分析有误？我不禁开
始怀疑自己的水平，还是那么菜。然而消沉并没有用，问题还要解决，是时候祭出天书了，
翻遍了我的书籍找到了下面这张图(鸟哥的Linux私房菜)：
![image.png](https://img.linkzz.eu.org/main/images/2023/11/25f7a3eeb14f137bcfebef078dda4a30.png)
可以看到 `wireguard` 的握手包是需要发出之前是需要经过 `FORWARD` 链的，那会不会是
被这条路阻了呢，查看 `iptables` 详情：

```bash
sudo iptables -L -v -n
```

![image.png](https://img.linkzz.eu.org/main/images/2023/11/a880600760263e299c833553e857b63e.png)

果然，`FORWARD` 链默认策略是 `DROP`，正是这个地方阻止了流量包，调整 `FORWARD` 链
默认策略：

```bash
sudo iptables -P FORWARD ACCEPT
```

再来看抓包日志：

```text
13:52:54.939793 IP 192.168.200.2.11968 > {my-home-ip}.25379: Flags [S], seq 0, win 65535, options [nop,wscale 14], length 0
13:52:54.939857 IP 10.1.1.137.11968 > {my-home-ip}.25379: Flags [S], seq 0, win 65535, options [nop,wscale 14], length 0
13:52:54.949558 IP{my-home-ip}.25379 > 10.1.1.137.11968: Flags [S.], seq 0, ack 1, win 65535, options [mss 536,nop,wscale 14], length 0
13:52:54.949585 IP {my-home-ip}.25379 > 192.168.200.2.11968: Flags [S.], seq 0, ack 1, win 65535, options [mss 536,nop,wscale 14], length 0
13:52:54.949644 IP 192.168.200.2.11968 > {my-home-ip}.25379: Flags [.], ack 1, win 65535, length 0
13:52:54.949658 IP 10.1.1.137.11968 > {my-home-ip}.25379: Flags [.], ack 1, win 65535, length 0
```

可以看到握手包正常发出和接收了，再看 `wireguard` 状态：

![image.png](https://img.linkzz.eu.org/main/images/2023/11/25551621cab65e23ef6dd0aeaf590be4.png)

通过 `phantun` 伪装的 `wireguard` 也连接成功，至此问题解决。

### 2.4 保存规则

至此就完结了吗，并没有，我们以上设置的 `iptables` 规则并不是永久的，重启之后之前
设置的规则都会重置，事实上之所有有此次问题正是因为我之前设置好的 `iptables` 规则
没有保存，无法开机自动应用规则导致的此次事件，下面我们就来保存规则：

安装 `iptables-persistent` 包

```bash
sudo apt install iptables-persistent
```

保存规则：

```bash
sudo iptables-save | sudo tee /etc/iptables/rules.v4
```

这样重启之后规则自会自动应用了
