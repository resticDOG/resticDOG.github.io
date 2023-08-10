---
title: 【NanoPi R2S旁路网关系列】2. 自定义域名、DDNS、KMS
date: 2023-08-10
description: "NanoPi R2S 旁路网关"

tags:
  - openwrt
  - wireguard
categories:
  - 教程
  - openwrt
  - 网络
lightgallery: true

toc:
  auto: true
---

## 前言

[上一章](/posts/openwrt-wireguard)我们实现了 wireguard
在任意地方安全的访问家庭网络，这章我们将实现自定义域名和 DDNS 动态更新公网 IP
的需求。

## 自定义域名

内网自定义域名有什么好处：

- 防止 IP 变更之后的访问难题

想象一下这样一个场景，一开始我内网部署了一个 jellyfin 服务，ip
为`192.168.5.132`，但某天我的网段变了，我得用`192.168.1.132`这个 IP
去访问，这时好不容易浏览器记住的密码和所有缓存信息都失效，还有有记错 IP
需要重新查看的烦恼。

- 有记忆特征

IPv4 是 4 段数字，即使家里网段固定只需要记忆一段的数字，那也是 0-254 共 255
个数字，数字的记忆对人类来说是不如字符来的方便的，`jellyfin`和`132`请问你是更喜欢记忆哪一个，虽然自建服务也可以通过搭建
dashbord
服务如[heimdall](https://heimdall.site/)，但是这样打开网页的效率着实有点低，你需要打开
heimdall 页面，再去点击相应的链接，同样还要维护服务的变更。

- 签发证书实现 https

有了自定义域名，我只需要修改 nginx 配置，将反代的 IP 变更即可解决以上 2
个问题，同时通过自签名的泛域名证书，我可以通过 https 访问自建服务，https
可以开启一些自建服务的一些功能。

### Dnsmasq 配置

openwrt 的 dns 服务基于 dnsmasq，自定义一个内网域名解析我们只需要修改配置即可:

```bash
vi /etc/config/dhcp
```

`config dnsmasq`配置下添加 list address 选项

```text
config dnsmasq
        option domainneeded '1'
        option localise_queries '1'
        option rebind_protection '1'
        option rebind_localhost '1'
        option local '/lan/'
        option domain 'lan'
        option expandhosts '1'
        option authoritative '1'
        option readethers '1'
        option leasefile '/tmp/dhcp.leases'
        option resolvfile '/tmp/resolv.conf.d/resolv.conf.auto'
        option ednspacket_max '1232'
        option localuse '1'
        list address '/linkzz.hm/192.168.5.120'
        option localservice '0'
```

上面定义了一个 `linkzz.hm` 的域名，指向`192.168.5.120`，这个 IP 就是 nginx
服务的 IP，这个指向是支持泛域名的，就是`*.linkzz.hm`都是指向 nginx
的，所有自建服务都可以用子域名来配置 nginx，无须再对单独的自建服务添加域名解析。

下面我们尝试一下这个配置是否生效。

先重启一下 dnsmasq 服务：

```bash
/etc/init.d/dnsmasq restart
```

查询 DNS:

```bash
nslookup linkzz.hm 192.168.5.99
```

![image.png](https://img.linkzz.eu.org/main/images/2023/08/7e6a8105a83a88eec01a0cf622688ac1.png)

可以看到一级域名和二级域名都正常返回了 nginx 地址

### Nginx 配置

我的 Nginx 部署在 PVE 容器里面，配置一下反代 openwrt 的 80 端口：

```nginx
upstream openwrt-server {
    server 192.168.5.99:80;
}

server {
    listen 80;
    server_name openwrt.linkzz.hm;

    location / {
        proxy_pass http://openwrt-server;
        sendfile off;
        proxy_set_header Host                   $host:$server_port;
        proxy_set_header X-Real-Ip              $remote_addr;
        proxy_set_header X-Forwarded-For        $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto      https;

        client_max_body_size            10m;

        proxy_connect_timeout           90;
        proxy_send_timeout              90;
        proxy_read_timeout              90;
    }
}
```

测试重启 Nginx:

```bash
nginx -t && nginx -s reload
```

访问[http://openwrt.linkzz.hm](http://openwrt.linkzz.hm):

![image.png](https://img.linkzz.eu.org/main/images/2023/08/33e009f4b570a52cfe0e0efafe7b3664.png)

反代成功

## DDNS

家宽虽然有公网 IP，但 IP 并不是固定的，运营商有策略不定期更换公网
IP，每次更换之后如果没有在家，而且没有其他渠道查看公网 IP，这时 wireguard
就失去与家里的连接了，为了避免这种情况，我们需要用一个固定的域名来解析家里的公网
IP，然后依靠 DDNS 服务定时更新解析的 IP 地址。

R2s 固件已经安装了 DDNS 的 Luci，我们直接配置就好了

![image.png](https://img.linkzz.eu.org/main/images/2023/08/bdba7d8865ab1609f8371399bf6a5a28.png)

我的域名解析是 Cloudflare 托管的，我们安装 Cloudflare 的 ddns 脚本包：

```bash
opkg install ddns-scripts-cloudflare
```

### 获取 Cloudflare API Token

在”我的个人资料“ -> "API 令牌"处创建一个 API 令牌

![image.png](https://img.linkzz.eu.org/main/images/2023/08/ff8ed89b48f0a48bc23d9823c5671e0a.png)

模板选择”编辑区域 DNS“

![image.png](https://img.linkzz.eu.org/main/images/2023/08/63b158096cae0cb6894428d11fcf4fe7.png)

区域资源选择要包括的域名，TTL 按需求选择。

![image.png](https://img.linkzz.eu.org/main/images/2023/08/3ac88b0f6cdce1a178952ce3de45591e.png)

创建好之后 Cloudflare 还给出了测试脚本，测试一下正常把 Token 保存起来。

### DDNS 脚本配置

新建一个服务：

![image.png](https://img.linkzz.eu.org/main/images/2023/08/bbf0da79dc2e784c1b0a58b43a93f9ef.png)

填上相关信息：

![image.png](https://img.linkzz.eu.org/main/images/2023/08/7e2733ac1bfc608274c915c1b3bb9386.png)

- 查询主机名：你要使用的域名，**该域名必须存在 dns
  解析**，也就是要预先将现在的公网 IP 和这个域名绑定，如：mc.example.com
- DDNS 服务商：选择`cloudflare.com-v4`
- 域名：二级域名和一级域名要用”@“符号连接，如：mc@example.com
- 用户名: `Bearer`
- 密码：上面申请的 API Token

选择 IP 来源：

![image.png](https://img.linkzz.eu.org/main/images/2023/08/fbd54d958ef8def2207ecf47169ce0d8.png)

- IP 地址来源：选择 URL
- 用于检测的 URL：http://checkip.dyndns.com
- 事件网络：选择`br-lan`

看到 pid 号就代表已经运行成功：

![image.png](https://img.linkzz.eu.org/main/images/2023/08/01f9c50d91304b6239cb20a2398a5058.png)

## KMS 服务

KMS 是一个 Windows 激活的网络工具，平时都是使用一些公网搭建的 KMS
服务，今天我们自建一个内网的服务，不使用公网的服务。

### 安装

在[Github 仓库](https://github.com/cokebar/openwrt-vlmcsd/tree/gh-pages)找到相应平台的安装包，R2s
对应的`cortex-a53`平台：

```bash
wget https://raw.githubusercontent.com/cokebar/openwrt-vlmcsd/gh-pages/vlmcsd_svn1113-1_aarch64_cortex-a53.ipk

opkg install vlmcsd_svn1113-1_aarch64_cortex-a53.ipk
```

在[Github Release 页面](https://github.com/cokebar/luci-app-vlmcsd/releases)下载最新的
ipk 包

```bash
wget https://github.com/cokebar/luci-app-vlmcsd/releases/download/v1.0.2-1/luci-app-vlmcsd_1.0.2-1_all.ipk

opkg install luci-app-vlmcsd_1.0.2-1_all.ipk
```

### 配置

Web 端需要退出再登录或者刷新缓存才能看到 luci
界面，直接使用默认配置，勾选自动激活，启动即可。

![image.png](https://img.linkzz.eu.org/main/images/2023/08/4efc5b20b3210eed306ed11d712f24d3.png)

## 结语

本章实现了自定义域名和 DDNS 的功能，对于我平时对旁路网关的使用需求已经足够了
😌，现在局域网的设备已经完全可以将网关设置为我们的 R2s
正常上网，公网的访问也可以通过 wireguard
进行，当然，生命不休，折腾不止，后面如果有新的折腾方向，我会再来更新，敬请期待！
