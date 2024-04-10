---
title: 简单记录一下 Docker 容器中信任一个自签名证书的案例
date: 2024-04-10
description: 简单记录一下 Docker 容器中信任一个自签名证书的案例
layout: post

tags:
  - Docker
  - selfhost

categories:
  - Linux
lightgallery: true

toc:
  auto: true
---

## 1. 背景

起因是我的 [ttrss](https://tt-rss.org/) 容器需要订阅一个自部署的微信公众号订阅服务 [wewe-rss](https://github.com/cooderl/wewe-rss) ，在 web 页面添加订阅一个订阅源 `http://192.168.5.128:8109/feeds/MP_WXS_3925660753.atom`，添加之后 ttrss 报错

```text
无法从指定的网址下载:cURL error 7: Failed to connect to 192.168.5.128 port 80 after 0 ms: Couldn't connect to server (see https://curl.haxx.se/libcurl/c/libcurl-errors.html) for http://192.168.5.128/feeds/MP_WXS_3925660753.atom
```

但是仔细看报错日志，我的服务端口明明是 `8109` 但是 ttrss 却访问了 80，暂时不知是不是 ttrss 的 BUG，所以我便换了一个 nginx 代理的本地服务来反代 wewe-rss 的服务，由于我 nginx 配置了一个自签名的证书，客户端信任这个证书之后可以开启一些自部署服务只能 `https` 才能开启的功能，比如密码管理器 `vaultwarden` 。扯远了，回到正题，现在我已经配置好了反代的服务，新的订阅服务链接为：`https://wewe.linkzz.hm/feeds/MP_WXS_3925660753.atom`，但是直接在 `ttrss` 中配置还是会报错证书无法信任，所以这就来解决这个问题。

## 2. 信任证书

> 以下的操作是基于 `Ubuntu 2204` 版本，其他发行版没测试过。

### 2.1 确认证书类型

我是使用的 [mkcert](https://github.com/FiloSottile/mkcert) 生成的 CA 证书，证书包含 `rootCA.crt` 和 `rootCA.pem` 文件，crt 是二进制文件，只包含证书内容，在 windows 环境下可以直接安装，而 pem 是 base64 编码的文本文件，它包含了证书内容、证书链等信息，在 Ubuntu 中信任证书需要 pem 文件。

### 2.2 宿主机信任证书

因为可能不止一个容器需要信任证书，所以选择先在宿主机信任证书，然后映射宿主机的证书文件夹到容器，这样宿主机和容器都能达到信任证书的效果。

- 拷贝证书文件

```shel
cp your_pem.pem /usr/local/share/ca-certificates/rootCA.crt
```

注意后缀名需要重命名为 crt

- 更新证书

```shell
# 如果没安装 ca-certificates 需要先安装 ca-certificates
# apt install ca-certificates
update-ca-certificates
```

如果提示 `command not found` 安装 `ca-certificates` 软件包即可

```text
Updating certificates in /etc/ssl/certs…
rehash: warning: skipping ca-certificates.crt,it does not contain exactly one certificate or CRL
1 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d…
done.
```

上面日志没有报错提示添加了1个证书，表示添加信任证书成功。

运行 curl 验证一下：

```shell
curl -v https://wewe.linkzz.hm/feeds/MP_WXS_3925660753.atom
```

```text
*   Trying 192.168.5.120:443…
* TCP_NODELAY set
* Connected to wewe.linkzz.hm (192.168.5.120) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/certs/ca-certificates.crt
  CApath: /etc/ssl/certs
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES128-GCM-SHA256
* ALPN, server accepted to use http/1.1
* Server certificate:
*  subject: O=mkcert development certificate; OU=LINKZZ-LAPTOP\linkzz@linkzz-laptop (linkzz)
*  start date: Dec 18 01:59:08 2023 GMT
*  expire date: Mar 18 01:59:08 2026 GMT
*  subjectAltName: host "wewe.linkzz.hm" matched cert's "*.linkzz.hm"
*  issuer: O=mkcert development CA; OU=LINKZZ-LAPTOP\linkzz@linkzz-laptop (linkzz); CN=mkcert LINKZZ-LAPTOP\linkzz@linkzz-laptop (linkzz)
*  SSL certificate verify ok.
> GET /feeds/MP_WXS_3925660753.atom HTTP/1.1
> Host: wewe.linkzz.hm
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: nginx/1.18.0 (Ubuntu)
< Date: Wed, 10 Apr 2024 09:14:46 GMT
< Content-Type: application/atom+xml; charset=utf-8
< Content-Length: 3944
< Connection: keep-alive
< X-Powered-By: Express
< Access-Control-Allow-Origin: *
< Access-Control-Expose-Headers: authorization
< ETag: W/"f68-vmfb7ZvVPKMACdW27xekYBF0xLA"
<
<?xml version="1.0" encoding="utf-8"?>
```

可以看到已经正常响应了。

### 2.3 映射到容器证书目录

下面更新 ttrss 的容器映射，将宿主机的 `/etc/ssl/certs` 文件夹映射到容器相同目录 `/etc/ssl/certs` 即可，我的是 compose 运行的，更新 docker compsoe 文件：

```yaml
version: "3"
services:
  service.rss:
    image: wangqiru/ttrss:latest
    container_name: ttrss
    ports:
      - 181:80
    environment:
      - SELF_URL_PATH=https://ttrss.linkzz.hm:443/ # please change to your own domain
      - DB_PASS=ttrss_bac # use the same password defined in `database.postgres`
      - PUID=1000
      - PGID=1000
    volumes:
      - feed-icons:/var/www/feed-icons/
      - /etc/ssl/certs:/etc/ssl/certs
    networks:
      - public_access
      - service_only
      - database_only
    stdin_open: true
    tty: true
    restart: always

  service.mercury: # set Mercury Parser API endpoint to `service.mercury:3000` on TTRSS plugin setting page
    image: wangqiru/mercury-parser-api:latest
    container_name: mercury
    networks:
      - public_access
      - service_only
    restart: always

  service.opencc: # set OpenCC API endpoint to `service.opencc:3000` on TTRSS plugin setting page
    image: wangqiru/opencc-api-server:latest
    container_name: opencc
    environment:
      - NODE_ENV=production
    networks:
      - service_only
    restart: always

  database.postgres:
    image: postgres:13-alpine
    container_name: postgres
    environment:
      - POSTGRES_PASSWORD=ttrss_bac # feel free to change the password
    volumes:
      - ~/postgres/data/:/var/lib/postgresql/data # persist postgres data to ~/postgres/data/ on the host
    networks:
      - database_only
    restart: always

  # utility.watchtower:
  #   container_name: watchtower
  #   image: containrrr/watchtower:latest
  #   volumes:
  #     - /var/run/docker.sock:/var/run/docker.sock
  #   environment:
  #     - WATCHTOWER_CLEANUP=true
  #     - WATCHTOWER_POLL_INTERVAL=86400
  #   restart: always

volumes:
  feed-icons:

networks:
  public_access: # Provide the access for ttrss UI
  service_only: # Provide the communication network between services only
    internal: true
  database_only: # Provide the communication between ttrss and database only
    internal: true
```

重启容器即可生效。

正常订阅的截图：

![image.png](https://img.linkzz.eu.org/main/images/2024/04/534db7c24d693353bd904d5fdc9baabc.png)

## 3. 总结

自签名证书对于有很多自部署在 nas 或者内部服务器，不方便公用服务申请证书的服务是很方便的，今天更新了容器中添加自信任证书的方法，总结就是先宿主机信任自签名证书，然后将 `curl` 等命令需要用到的证书文件夹 `/etc/ssl/certs/` 文件夹直接映射到容器中国即可。
