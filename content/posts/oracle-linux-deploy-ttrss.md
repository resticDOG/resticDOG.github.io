---
title: 在 Oracle Arm Linux 服务器下打造一个足够好用的 TT-RSS
date: 2025-01-02
draft: false
description: "在 Oracle Arm Linux 服务器下打造一个足够好用的 TT-RSS"

toc:
  auto: true
---

## 1. 起因

我的 [ttrss](https://tt-rss.org/) 容器一直部署在我的 HomeLab 服务器中，我的 HomeLab 配置在 [这篇文章](/posts/pve-igd-passthrough) 的背景章节有提到，这里再列一下：

| 类别  | 项目              | 价格  |
| --- | --------------- | --- |
| CPU | i5-10400        | --  |
| 主板  | 华擎 B-460M 钢铁传奇  | --  |
| 内存  | 酷兽 8G * 4 = 32G | --  |
| SSD | 三星 980 1TB      | --  |
| SSD | 爱国者 128GB       | --  |
| HDD | 希捷酷狼 4TB * 2    | --  |
| 机箱  | 先马趣造            | --  |
| 散热  | 九州风神            | --  |
| 电源  | 先马 500W          | --  |

这套配置是 4 年前入手的全新硬件，入手之后一直当作 HomeLab 服务器 24 小时开机使用，但毕竟不是服务器级别的硬件，终于，我的主板还是在 8 月份倒下了，于是我立马着手更换硬件，但是 B460 的芯片组主板早就停产了，加上我的虚拟机系统是 Pve，所以我还是倾向于购买相同的硬件，这样配置都不用改了，于是我打开了拼多多。

### 1.1 PDD 买电子产品还是得小心

拼多多我也用的并不多，以往我的电子产品大多都是京东入手，但是这次京东并没有全新硬件，都是二手，想着也差不多就在拼多多购买了一块**华擎的 B460Pro4**，没办法，钢铁传奇这个主板估计是当时出货太少，导致二手货也没多少，只能入手相同品牌的 Pro4，好在收到主板后修改的配置也不是很多，拿到货修改 bois 配置后通过 pve 安装 u 盘进入恢复模式，将网卡配置改一改就好了，所有虚拟机和 CT 完美运行。可高兴了没几天这块主板就坏了，毫无疑问翻车了，主板寄回去修了之后还是老样子，商家卖了一块有暗病的主板给我，无奈只能继续维修。

### 1.2 服务停摆

所以我的所有 selfhosted 的服务就挂了很久，其中就有我最为依赖的 **TT-RSS**。

长久以来我通过 [TT-RSS](https://tt-rss.org/) 和 [RSShub](https://docs.rsshub.app/) 来订阅我所感兴趣的一些信息，结合 [Fluent-reader](https://github.com/yang991178/fluent-reader) 和 [ntfy](https://ntfy.sh/) 推送，我能以相对短的时间获得社交网站如 Twitter、Weibo 的更新，我享受信息带来的多巴胺释放。

## 2. 还得是甲骨文

就在我万籁俱灰的时候，我看到了 Oracle 发给我的邮件 -《Oracle 政策更新通知》，我这才想起被我冷落了很久的两台 `Oracle arm VPS`，这两台 `2c16G` 的 Arm VPS 可是没有部署多少重量级的服务，用来部署一个小小的 TT-RSS 那还不是绰绰有余。

### 2.1 构建运行镜像

说干就干，首先创建文件夹，创建 `docker-compose.yaml`

```bash 
mkdir tt-rss
cd tt-rss
touch docker-compose.yaml
```

输入以下内容：

```yaml

services:
  db:
    image: postgres:15-alpine
    restart: unless-stopped
    env_file:
      - .env
    environment:
      - POSTGRES_USER=${TTRSS_DB_USER}
      - POSTGRES_PASSWORD=${TTRSS_DB_PASS}
      - POSTGRES_DB=${TTRSS_DB_NAME}
    volumes:
      - db:/var/lib/postgresql/data

  app:
    image: cthulhoo/ttrss-fpm-pgsql-static:latest
    restart: unless-stopped
    env_file:
      - .env
    volumes:
      - app:/var/www/html
      - ./config.d:/opt/tt-rss/config.d:ro
    depends_on:
      - db

  updater:
    image: cthulhoo/ttrss-fpm-pgsql-static:latest
    restart: unless-stopped
    env_file:
      - .env
    volumes:
      - app:/var/www/html
      - ./config.d:/opt/tt-rss/config.d:ro
    depends_on:
      - app
    command: /opt/tt-rss/updater.sh

  web-nginx:
    image: cthulhoo/ttrss-web-nginx:latest
    restart: unless-stopped
    env_file:
      - .env
    ports:
      - ${HTTP_PORT}:80
    volumes:
      - app:/var/www/html:ro
    depends_on:
      - app

volumes:
  db:
  app:
```

你以为接下来就是 `docker compose up -d` 运行容器了吗，并不是，上面 app 和 web-nginx 服务的容器不支持 arm 架构，是无法直接运行的，所以我们需要重新编译一下这两个容器，创建 `docker-compose.override.yaml` 文件，输入以下内容：

```yaml
services:
  app:
    image: cthulhoo/ttrss-fpm-pgsql-static:latest
    depends_on:
      - db
    build:
      dockerfile: .docker/app/Dockerfile
      context: https://git.tt-rss.org/fox/tt-rss.git
      args:
        BUILDKIT_CONTEXT_KEEP_GIT_DIR: 1

  web-nginx:
    image: cthulhoo/ttrss-web-nginx:latest
    build:
      dockerfile: .docker/web-nginx/Dockerfile
      context: https://git.tt-rss.org/fox/tt-rss.git

```

开始构建运行镜像

```bash
docker compose up --build -d
```

查看运行情况

```bash
docker comose ps
```

![image.png](https://img.linkzz.eu.org/main/images/2024/10/9f6b9fb38465aeaf024a7baf3e0fa7df.png)

可以看到所有服务正常运行

### 2.2 加入亿点点魔法

正常这时候就可以使用了（如果不嫌弃丑的话），但是想要更好的体验我们还得加入一点点魔法。

#### 2.2.1 主题

主题使用 [tt-rss-feedly-theme](https://github.com/levito/tt-rss-feedly-theme)，包含很多 feedly 系列的主题，响应式布局，移动端体验还行，当然我使用的最多的还是通过 `fever api` 用专业的客户端阅读。查看 `app` 服务的 `volume` 详情

```bash
docker volume inspect ttrss_app
```

```text
[
    {
        "CreatedAt": "2024-10-10T13:46:40+08:00",
        "Driver": "local",
        "Labels": {
            "com.docker.compose.project": "ttrss",
            "com.docker.compose.version": "2.21.0",
            "com.docker.compose.volume": "app"
        },
        "Mountpoint": "/var/lib/docker/volumes/ttrss_app/_data",
        "Name": "ttrss_app",
        "Options": null,
        "Scope": "local"
    }
]
```

`Mountpoint` 字段即 ttrss 的 workspace 文件挂载目录，进入宿主机挂载目录

```bash 
cd /var/lib/docker/volumes/ttrss_app/_data
```

进入主题目录

```bash
tt-rss/themes.local
```

克隆主题

```bash
git clone https://github.com/levito/tt-rss-feedly-theme/tree/dist
```

注销登录之后重新登录即可在偏好设置中应用主题

![image.png](https://img.linkzz.eu.org/main/images/2025/01/284cb802a4cf0921aabb36159697eac2.png)

#### 2.2.1 Fever

fever 插件是一个 api 插件，其他 rss 客户端可以通过 fever 协议同步 TT-RSS 的订阅源，同步已读未读，同步收藏等，是实现多端同步的基础。

进入插件目录

```bash
cd tt-rss/plugins.local
```

克隆插件

```bash 
git clone https://github.com/DigitalDJ/tinytinyrss-fever-plugin.git fever
```

注意文件夹名字必须是 fever，否则插件加载不出来

在 " 偏好设置 "-> " 插件 " 中启用插件

![image.png](https://img.linkzz.eu.org/main/images/2024/10/cc8da9e8d111222b2063f33f3166908f.png)

偏好设置中勾选 " 启用 API"

![image.png](https://img.linkzz.eu.org/main/images/2024/10/de8818caf74c2b7172be6663fedfbace.png)

刷新之后即可看到 fever 的设置界面

![image.png](https://img.linkzz.eu.org/main/images/2024/10/2cdd3f91791bd805f3071639b40264fc.png)

设置当前用户的 fever 密码，他提示 fever 的 api 使用未加盐的 MD5 用户密码，所以为了安全起见建议使用 HTTPS

之后在客户端配置一下即可，这里客户端使用的 [Fluent Reader](https://github.com/yang991178/fluent-reader)

![image.png](https://img.linkzz.eu.org/main/images/2024/10/4b9e26a5cf00f65dc4bb925eb644f8f6.png)

#### 2.2.2 Mercury

[parser-api](https://github.com/postlight/parser-api) 是一个全文获取 api 工具，某些博客站点输出的 rss 只包含摘要，使用他可以获取大部分博客站点的全文，插件我们使用 [@HenryQW](https://henry.wang/) 的 tt-rss 插件，容器也是他做的，同时这位大佬还是 [Awesome-TTRSS](https://github.com/HenryQW/Awesome-TTRSS) 的作者，我一开始 TT-RSS 的部署便是来自于此仓库。

docker-compose .override.yaml 加入 mercury 服务

```yaml
services:
  app:
    image: cthulhoo/ttrss-fpm-pgsql-static:latest
    depends_on:
      - db
      - mercury
    build:
      dockerfile: .docker/app/Dockerfile
      context: https://git.tt-rss.org/fox/tt-rss.git
      args:
        BUILDKIT_CONTEXT_KEEP_GIT_DIR: 1

  web-nginx:
    image: cthulhoo/ttrss-web-nginx:latest
    build:
      dockerfile: .docker/web-nginx/Dockerfile
      context: https://git.tt-rss.org/fox/tt-rss.git

  mercury:
    image: wangqiru/mercury-parser-api:latest
    container_name: mercury
    restart: always
```

进入插件目录添加插件

```bash

cd /var/lib/docker/volumes/ttrss_app/_data/tt-rss/plugins.local

git clone https://github.com/HenryQW/mercury_fulltext

```

" 偏好设置 "-> " 插件 " 中启用插件，" 供稿 "->" 插件 " 中配置 api 地址

![image.png](https://img.linkzz.eu.org/main/images/2024/10/1cbce03d36a233f1146b31a0f26f3ad6.png)

试用一下：

![mercury-full-text.gif](https://img.linkzz.eu.org/main/images/2024/10/cb48ea3a200e5bc36adc6e28f166f3ce.gif)

#### 2.2.2 OpenCC

这个插件因人而异，是一个简繁转换的插件，我是因为阅读繁体实在有些吃力，所以需要转换一下

这里同样使用 [@HenryQW](https://henry.wang/) 大佬的 OpenCC 插件，但是他的 opencc-api 的镜像不支持 arm 架构，但是别急，我们依然可以自己构建一个 arm 的镜像，只不过只能使用 1.3 的版本

克隆代码

```bash 

git clone https://github.com/HenryQW/OpenCC.henry.wang.git

cd OpenCC.henry.wang

```

修改 `Dockerfile`， `python2` 改为 `python3`

```diff
diff --git a/Dockerfile b/Dockerfile
index bf718ea..25d3a89 100644
--- a/Dockerfile
+++ b/Dockerfile
@@ -7,7 +7,7 @@ WORKDIR /usr/src/app
 COPY ["package.json", "package-lock.json*", "npm-shrinkwrap.json*", "./"]

 RUN  apk update && \
-    apk add --no-cache --virtual build-pkg build-base python2 && \
+    apk add --no-cache --virtual build-pkg build-base python3 && \
     npm install --production --silent && \
     mv node_modules ../ && \
     apk del build-pkg
```

构建镜像

```bash
docker build -t opencc-arm:latest .
```

构建成功之后修改 `docker-compose.override.yaml` 文件

```yaml
services:
  app:
    image: cthulhoo/ttrss-fpm-pgsql-static:latest
    depends_on:
      - db
      - mercury
      - opencc
    build:
      dockerfile: .docker/app/Dockerfile
      context: https://git.tt-rss.org/fox/tt-rss.git
      args:
        BUILDKIT_CONTEXT_KEEP_GIT_DIR: 1

  web-nginx:
    image: cthulhoo/ttrss-web-nginx:latest
    build:
      dockerfile: .docker/web-nginx/Dockerfile
      context: https://git.tt-rss.org/fox/tt-rss.git

  mercury:
    image: wangqiru/mercury-parser-api:latest
    container_name: mercury
    restart: always

  opencc:
    image: opencc-arm:latest
    container_name: opencc
    environment:
      - NODE_ENV=production
    restart: always
```

进入插件目录下载安装插件

```bash

cd /var/lib/docker/volumes/ttrss_app/_data/tt-rss/plugins.local

git clone https://github.com/HenryQW/ttrss_opencc

```

" 偏好设置 " -> " 插件 " 中启用插件，" 供稿 " -> " 插件 " 中配置 api 地址

![image.png](https://img.linkzz.eu.org/main/images/2024/10/58c6c36f6b0b8b5d4fb2e0dfa463eb76.png)

试用一下

![opencc.gif](https://img.linkzz.eu.org/main/images/2024/10/5756819631cf924473771a8be04b2a77.gif)

#### 2.2.2 Ntfy

ntfy 是一个自建的消息推送服务，通过 ntfy 我们可以自定义推送规则来推送重要的信息到我们的手机或者是 PC，但是 Ntfy 并没有官方 TT-RSS 插件，但是没关系，ntfy 的使用场景和 mercury 以及 opencc 很像，我们依照他们的代码自己实现一个就好了，我已经实现并开源了 Ntfy 的 TT-RSS [插件](https://github.com/resticDOG/tt-rss-plugin-ntfy)，你可以直接使用，如果项目对你有帮助还请点一个小小的⭐

##### 安装插件

```bash
cd /var/lib/docker/volumes/ttrss_app/_data/tt-rss/plugins.local

git clone https://github.com/resticDOG/tt-rss-plugin-ntfy.git ntfy
```

##### 启用插件

![image.png](https://img.linkzz.eu.org/main/images/2024/12/cbe241aeee01c425b5d5b7b0eb7dba95.png)

打开 [ntfy web app](https://ntfy.sh/app) ，当然你也可以自建自己的 ntfy 实例，它是完全开源的，新建一个订阅主题，主题名称最好使用随机生成或者具有一定的特殊性，避免无关消息混入其中

![image.png](https://img.linkzz.eu.org/main/images/2024/12/282f78e5f0f1b9deaa8bdb7976638286.png)

接着打开 " 偏好设置 "-> "Ntfy settings (ntfy)"，填写 `ntfy server` 和 `topic` 信息，如果是自建的 server 并且启用了 token 鉴权记得填写 token

![image.png](https://img.linkzz.eu.org/main/images/2025/01/f10c48302ad2460886fe2b5572f7fbca.png)

##### 推送消息

有两个途径配置推送：

1. **对订阅源启用**

编辑需要订阅的订阅源（也就是供稿），在 " 插件 "tab 页，勾选 "Send notification via Ntfy"，已经启用推送的订阅源在设置页面也可看到。
![image.png](https://img.linkzz.eu.org/main/images/2025/01/0005e6a2c4098bb90fa58e20686fdfac.png)

1. **过滤器触发**

ntfy 可以通过过滤器规则配置推送，这样可以实现多样的推送规则，如关键词触发推送，供稿分类推送等，我在这个上面应用的比较多，比如，通过 rsshub 订阅 twitter web3 KOL 博主来获取最新的空投信息，这样保证能相对及时的收到最新的空投信息

![image.png](https://img.linkzz.eu.org/main/images/2025/01/431a1d4070d69e8cc697fdb68ce859bd.png)

如上面的规则，编辑好规则在应用操作中添加 "Invoke plugin: Ntfy: Send Notification" 即可

##### 桌面通知

PC 上可以打开 ntfy.sh 的桌面通知，这样通知即刻到达，上班时刻看一眼，摸鱼时刻打开

![image.png](https://img.linkzz.eu.org/main/images/2025/01/2a4bd7b72fd19c7517a6ce8a100bb216.png)

## 3. 总结

这篇文章本来应该是 2024 年 10 月份发的，奈何实在太懒，写了一半就搁置了，今年的大部分文章都是在上班时间摸鱼写的，中年人的悲伤回家还要带孩子，能抓到一点空闲时间都被 LOL 大乱斗占了，周末也是在陪孩子中度过，当然这些都是借口，以后文章还是得更新起来，今年会第一次写一个年终总结，今年确实也发生了不少事，写下来一是为了记录，也反思一下近 3 年来的一个碌碌无为的状态，展望一下未来。

