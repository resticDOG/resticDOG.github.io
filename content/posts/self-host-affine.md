---
title: 私有知识库和笔记工具 Affine 搭建, 修复导入 Markdown 图片丢失的问题
date: 2025-11-12
description: "私有知识库和笔记工具 Affine 搭建, 修复导入 Markdown 图片丢失的问题"
layout: post

tags:
    - Wiki
    - Notion
    - Affine
categories:
    - SelfHost
lightgallery: true

toc:
    auto: true
---

# 私有知识库和笔记工具 Affine 搭建, 修复导入 Markdown 图片丢失的问题

## 前言

一直以来我使用 Obsidian 作为我的知识库管理和编辑工具，配合自建的图床和 Nextcloud 的同步，我几乎可以在任何地方随意维护我的知识库，后面因为 Obsidian 的 Vim 模式用起来不顺手，在中文之间移动有卡顿的感觉，所以索性编辑器直接就用了 `Vim` ，但我一直苦恼一个问题，就是我的知识库一直以 Markdown 源文件的方式在 Nextcloud 的同步下，我虽然可以随时打开 Obsidian 查看，但是在移动端查看不方便，以及每次想查询的时候还得打开客户端，我本人是很不喜欢客户端的，我希望能在 web 上就可以查看，当然 Nextcloud 也能在网页端查看渲染的 Markdown 文件，但是效果也差强人意，正好刷 Koala 聊编程的时候看到了 Affine，其主打的 Notion 替代品的概念吸引了我，虽然我也用 Notion， 但用的并不多，而且 Notion 无法迁移我的本地知识库，导入的 Markdown 格式各种问题，抱着试试看的心态部署了一下，虽说官方提供了完整的 `docker-compose`配置文件，但实际配置下来还是遇到了一些问题，在此权当记录一下。

## 安装 affine 服务端

### Docker compose 安装服务端

还是使用 `dokcer compose`部署，下载最新版本 `docker-compose.yaml`文件。

```shellscript
wget -O docker-compose.yml https://github.com/toeverything/affine/releases/latest/download/docker-compose.yml
```

&#x20; 下载配置模板修改配置

```shellscript
wget -O .env https://github.com/toeverything/affine/releases/latest/download/default.env.example
```

下面是我的配置

```
# 数据库挂载目录
DB_DATA_LOCATION=./postgres
# 上传的附件目录
UPLOAD_LOCATION=/mnt/media/affine
# 配置文件目录
CONFIG_LOCATION=./config
# 数据库配置
DB_USERNAME=affine
DB_PASSWORD=****
DB_DATABASE=affine
# select a revision to deploy, available values: stable, beta, canary
# 部署的版本：稳定版
AFFINE_REVISION=stable

# set the host for the server for outgoing links
# AFFINE_SERVER_HTTPS=true
# AFFINE_SERVER_HOST=affine.yourdomain.com
# or
# 这里是我的自定义的内网域名，如果放到公网改成你自己的域名，如果内网IP访问则去掉这行
AFFINE_SERVER_EXTERNAL_URL=https://wiki.linkzz.hm
```

运行

```shellscript
docker compose up -d
```

正常运行之后打开 http://{youip}:3010 进行管理员配置

### 反向代理

内网部署服务众多，我不想记忆每个服务的端口，所以我选择使用内网域名来进行反代，下面是 `Nginx` 反代的配置：

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}
upstream wiki-server {
    server 192.168.5.128:3010;
}
server {
    listen      443 ssl;
    server_name wiki.linkzz.hm;
    ssl_certificate     ssl/linkzz.hm/_wildcard.linkzz.hm.pem;
    ssl_certificate_key  ssl/linkzz.hm/_wildcard.linkzz.hm-key.pem;

    ssl_session_timeout  5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;

    location / {
            proxy_pass                                           http://wiki-server;
            proxy_set_header Host                                $host:$server_port;
            proxy_set_header X-Real-IP                           $remote_addr;
            proxy_set_header X-Forwarded-For                     $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto                   https;
            proxy_set_header Upgrade                             $http_upgrade;
            proxy_set_header Connection                          $connection_upgrade;
            proxy_cache_bypass                                   $http_upgrade;
            proxy_http_version                                   1.1;
    }

    error_page 404 /404.html;
        location = /40x.html {
    }

    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }
}

server {
   listen 80;
   server_name wiki.linkzz.hm;
   rewrite ^(.*)$ https://${server_name}$1 permanent;
}
```

## 解决导入时图片跨域问题

我部署的版本是 `0.25.0` 这个版本导入 Markdown 文件时其他格式都能很好保留，唯独图片存在跨域的问题，看了下控制台，Markdown 中的外链图片会通过官方的 Cloudflare worker 处理，这个worker 默认只处理官方host 的 Referer, 即 <https://app.affine.pro/> ，而我们部署的版本浏览器会自动加上这个域名的Referer，所以图片导入是无法使用的。

### 部署自己的图片处理 worker

好在官方已经开源了这个 worker

[GitHub - toeverything/affine-workers](https://github.com/toeverything/affine-workers)

我们可以自己部署自己的版本来解决这个问题，Fork 一下官方源码，修改以下地方

```diff
modified packages/utils/src/headers.ts
@@ -5,6 +5,7 @@ const ALLOW_ORIGIN: OriginRule[] = [
   'https://app.affine.pro',
   'https://insider.affine.pro',
   'https://affine.fail',
+  'https://wiki.linkzz.hm',
   'https://try-blocksuite.vercel.app',
   /https?:\/\/localhost(:\d+)/,
   /https:\/\/.*?-toeverything\.vercel\.app$/,
modified packages/worker/src/index.ts
@@ -2,7 +2,7 @@ import { DomainRouterBuilder, domainRoutersHandler, type Env } from '@affine/uti

 import { AFFiNEWorker } from './affine.js';

-const WORKER_DOMAIN = 'affine-worker.toeverything.workers.dev';
# 这个地址需要改为你的worker 访问地址
+const WORKER_DOMAIN = 'affine-worker.***.workers.dev';

 const affine = AFFiNEWorker();

modified packages/worker/wrangler.toml
@@ -4,5 +4,19 @@ compatibility_date = "2023-11-21"
 workers_dev = true
 minify = true

-[limits]
-cpu_ms = 500
+# [limits]
+# cpu_ms = 500
+[observability]
+enabled = false
+head_sampling_rate = 1
+
+[observability.logs]
+enabled = true
+head_sampling_rate = 1
+persist = true
+invocation_logs = true
+
+[observability.traces]
+enabled = false
+persist = true
+head_sampling_rate = 1
```

observability 相关的配置是记录worker日志的，可以去掉

使用 `wrangler`或者在 Cloudflare 控制台关联仓库部署一下，下面是 wrangler 的部署步骤：

安装 [wrangler](https://developers.cloudflare.com/workers/wrangler/)

```shellscript
npm install -g wrangler
```

安装 [pnpm](https://pnpm.io/installation)

```shellscript
 npm install -g pnpm
```

安装依赖

```shellscript
pnpm install
```

登录授权 wrangler

```shellscript
wrangler login
```

部署 worker

```shellscript
pnpm run deploy
```

###

### 应用自己的worker

目前使用的这个版本 worker 的地址是硬编码在前端的，无法使用配置项替换，改官方源码还要编译构建，只能自己改下镜像替换一下制品，目前 `0.25.0`使用这个方法是可行的。

创建 Dockerfile 文件， 下面的地址需要替换为你部署好的 worker 访问地址。

```docker
FROM ghcr.io/toeverything/affine:${AFFINE_REVISION:-stable}

RUN sed -i 's|https://affine-worker.toeverything.workers.dev|https://affine-worker.***.workers.dev|g' /app/static/js/index.*.js
```

然后修改 `docker-compose.yml` 文件

```yaml
name: affine
services:
    affine:
        # 使用 Local Dockerfile 构建镜像
        #image: ghcr.io/toeverything/affine:${AFFINE_REVISION:-stable}
        build: .
        container_name: affine_server
```

最后重启之后运行容器

```shellscript
docker compose down && docker compose up -d --build
```

没问题的话就可以试试效果了

![image.png](https://img.linkzz.eu.org/main/images/2025/11/281d097b9eeace3cf204b83f0b3305a9.png)

以上是我的博客的一篇文章导入的效果，可以看到他成功下载了我图床里面的图片并导入到了内置的存储之中。

## 结语

目前使用下来，Affine 还是比较丝滑的，目前这个还不是完整版的 Affine，因为AI 功能尚未配置，不过用作个人知识库倒是足够了，我先深度使用一段时间来看看，后续等 AI 功能稳定了再考虑配置。
