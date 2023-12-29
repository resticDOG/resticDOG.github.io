---
title: 记一次 Moonlight 10054 错误的解决经历
date: 2023-12-29
description: "记一次 Moonlight 10054 错误的解决经历"
layout: post

tags:
  - 串流
  - Sunshine
  - Moonlight
categories:
  - Linux
  - 串流
lightgallery: true

toc:
  auto: true
---

## 1. 起因

最近看到很多 Linux 平铺窗口管理器的视频，总觉得很炫酷，体内的折腾之心在沸腾，于是就打起了体验（~~折腾~~）平铺窗口的想法，看到这里各位看官肯定要说：这和 `Moonlight` 串流有啥关系，标题党！辣鸡！

_哈哈，各位别急，且听我慢慢道来_。

首先，我主力的生产系统是 Windows ，我不想在 Windows 下开发，所以 Windows 平铺窗口就 pass 掉了，只能转向 Linux，正好公司一台 `Exsi` 主机还剩很多资源，并且还有一块 `GTX 1060` 的显卡可以用来直通加速，那不正好创建一个带显卡的虚拟机，运行 Linux 桌面，然后我再远程串流这台虚拟机不就行了吗，那串流体验最好的自然是 Moonlight 了， 诶，你看，这不就关联大了吗！

## 2. 环境

好吧，废话少说，我描述一下我的环境，我要远程串流的是一台 `Exsi` 创建的虚拟机，参数如下：

| 项目 | 值 | 备注 |
| --- | --- | --- |
| 系统 | Arch Linux |  |
| CPU | 8 vCPU | 宿主机 CPU 为 AMD Ryzen Threadripper 1950X |
| RAM | 8GB |  |
| GPU | NVIDIA GeForce GTX 1060 6GB | 显卡直通 |
| 显示器 | 无 |  |
| 远程串流服务 | [Sunshine v0.21.0](https://github.com/LizardByte/Sunshine) |  |
| 显示服务 | X11 |  |
| 窗口管理器 | [awesome](https://awesomewm.org/download/) |  |
| Windows 客户端 | Moonlight-qt |  |

### 2.1 串流设置

按照 [官网headless系统设置](https://docs.lizardbyte.dev/projects/sunshine/en/latest/about/guides/linux/headless_ssh.html) 的指引，我安装好了 `nvidia` 的专有驱动，驱动能成功识别显卡，设置好了一个虚拟显示器并且成功运行了 `Xorg` ，

![image.png](https://img.linkzz.eu.org/main/images/2023/12/6a94fb1e495e0fc2ed5d7f4ca50a6953.png)

## 3. 排查

`x11vnc` 可以访问桌面，并且 `awesome` 工作正常，`rustdesk` 可以访问并且能开启硬件编码。

![image.png](https://img.linkzz.eu.org/main/images/2023/12/dacb17272df216d17a54faba327fc616.png)

唯独 `sunshine` 连接不上，启动日志显示硬件编码工作正常

![image.png](https://img.linkzz.eu.org/main/images/2023/12/89cbdf1362de2e5a05ece963169e603c.png)

Moonlight客户端可以正常添加和配对串流主机，但在启动【Desktop】串流之后总会报错：

```text
启动 RTSP handshake 失败: 错误 10054
检查防火墙和端口转发规则: TCP 48010，UDP 48000，UDP 48010
```

按照以上提示检测端口连通性，因为我和串流主机都在内网环境，我的客户端是 Windows 系统，防火墙已经关掉了，而串流的 Archlinux 是没有装防火墙服务的，所以不存在防火墙挡掉端口的问题，果然，使用 `nc` 测试端口连通：

```bash
➜  ~ nc -v 10.1.1.63 48010
Connection to 10.1.1.63 48010 port [tcp/*] succeeded!
```

UDP 的端口只有在串流的过程中才会打开监听，所以没法直接测试，但是通过手动端口监听测试也证明 UDP 端口也是没有被防火墙挡掉的。

```bash
# 服务端监听 udp 48010
 ~ nc -lu -p 48010

# 客户端连接
➜  ~ nc -uv 10.1.1.63 48010
Connection to 10.1.1.63 48010 port [udp/*] succeeded!
```

查看 sunshine 的详细连接日志发现执行 desktop 之后 sunshine 接收到了一个 rtsp 的 Request，sequence 是1，之后就客户端就退出报错了。

```log
[2023:12:29:17:03:29]: Debug: ------  h264 ------
[2023:12:29:17:03:29]: Debug: PASSED: supported
[2023:12:29:17:03:29]: Debug: REF_FRAMES_RESTRICT: unsupported
[2023:12:29:17:03:29]: Debug: CBR: supported
[2023:12:29:17:03:29]: Debug: DYNAMIC_RANGE: unsupported
[2023:12:29:17:03:29]: Debug: VUI_PARAMETERS: supported
[2023:12:29:17:03:29]: Debug: -------------------
[2023:12:29:17:03:29]: Info: Found H.264 encoder: h264_nvenc [nvenc]
[2023:12:29:17:03:29]: Debug: ------  hevc ------
[2023:12:29:17:03:29]: Debug: PASSED: supported
[2023:12:29:17:03:29]: Debug: REF_FRAMES_RESTRICT: unsupported
[2023:12:29:17:03:29]: Debug: CBR: supported
[2023:12:29:17:03:29]: Debug: DYNAMIC_RANGE: unsupported
[2023:12:29:17:03:29]: Debug: VUI_PARAMETERS: supported
[2023:12:29:17:03:29]: Debug: -------------------
[2023:12:29:17:03:29]: Info: Found HEVC encoder: hevc_nvenc [nvenc]
[2023:12:29:17:03:29]: Debug: /CN=NVIDIA GameStream Client -- verified
[2023:12:29:17:03:29]: Debug: TUNNEL :: HTTPS
[2023:12:29:17:03:29]: Debug: METHOD :: GET
[2023:12:29:17:03:29]: Debug: DESTINATION :: /serverinfo
[2023:12:29:17:03:29]: Debug: User-Agent -- Mozilla/5.0
[2023:12:29:17:03:29]: Debug: Accept-Language -- zh-CN,en,*
[2023:12:29:17:03:29]: Debug: Accept-Encoding -- gzip, deflate
[2023:12:29:17:03:29]: Debug: Connection -- Keep-Alive
[2023:12:29:17:03:29]: Debug: Host -- 10.1.1.63:47984
[2023:12:29:17:03:29]: Debug:  [--]
[2023:12:29:17:03:29]: Debug: uuid -- d63f79a098fb43e199209d93b001b333
[2023:12:29:17:03:29]: Debug: uniqueid -- 0123456789ABCDEF
[2023:12:29:17:03:29]: Debug:  [--]
[2023:12:29:17:03:29]: Debug: handle_read(): Handle read of size: 93 bytes
[2023:12:29:17:03:29]: Debug: handle_payload(): Handle read of size: 0 bytes
[2023:12:29:17:03:29]: Debug: type [REQUEST]
[2023:12:29:17:03:29]: Debug: sequence number [1]
[2023:12:29:17:03:29]: Debug: protocol :: RTSP/1.0
[2023:12:29:17:03:29]: Debug: payload ::
[2023:12:29:17:03:29]: Debug: command :: OPTIONS
[2023:12:29:17:03:29]: Debug: target :: rtsp://10.1.1.63:48010
[2023:12:29:17:03:29]: Debug: CSeq :: 1
[2023:12:29:17:03:29]: Debug: X-GS-ClientVersion :: 14
[2023:12:29:17:03:29]: Debug: Host :: 10.1.1.63
[2023:12:29:17:03:29]: Debug: ---Begin MessageBuffer---
OPTIONS
---End MessageBuffer---
[2023:12:29:17:03:29]: Debug: ---Begin Response---
RTSP/1.0 200 OK
CSeq: 1



---End Response---
[2023:12:29:17:03:32]: Debug: TUNNEL :: NONE
```

这就是问题的所在，说明客户端很可能没收到这个Sequence为1的应答，导致双方连接不上，并不是因为硬件编码的问题，也不是防火墙的问题，更不是 `X11` 显示系统的问题。

那是什么问题呢，我看着 `RTSP` 陷入了沉思，联想到各大网络直播多是使用的 `RTSP` 协议推流，该不会是 IT 禁用了该协议吧，**果然，公司上班时间是无法观看直播网站的，直播推流的流也是拉取不到的！**

事已至此，一切都真相大白了，**公司用的是深信服的网络行为管理，老板为了省钱IT是外包的，一周来一次，平时接触不到，没想到这个策略的设置如此简单粗暴，内网的RTSP协议都给禁掉了，那开发如果用到该协议推流，我估计光是排查问题就得浪费半天时间吧，哎，结合到最近各大互联网公司频发的运维事故我只想说：降本增效害死人！**

## 4. 解决问题

既然知道问题发生的原因了，解决就好办了，既然直连的 `RTSP` 协议不行，我通过 VPN 开个隧道总可以了吧。果然，通过 wireguard 建立隧道之后一切问题解决，串流成功，不得不感叹 Moolight 和 Sunshine 项目做的真是完美，低延迟，高性能，画质还不差，体验秒杀前面的 VNC 和 rustdesk，接下来继续我的平铺窗口管理器之旅了，Awesome !

![image.png](https://img.linkzz.eu.org/main/images/2023/12/f3eddc820e2eb71018cff4b5991799db.png)
