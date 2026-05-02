---
title: 内网自部署存储迁移到自部署的 S3
date: 2026-05-02
description: "将自部署服务的媒体文件集中迁移到自建 S3 存储，解决 PVE 容器存储空间不足的问题"
layout: post

tags:
  - 折腾
  - S3
  - Rustfs
  - NAS
  - Docker
  - Affine
  - Gitea
  - Memos
categories:
  - Linux
  - 折腾
lightgallery: true

toc:
  auto: true
---

# 内网自部署存储迁移到自部署的 S3

## 1. 存储的烦恼

我日常会使用一些自部署的服务，如写作的 [Affine](https://affine.pro/)、代码服务 [Gitea](https://github.com/go-gitea/gitea)、备忘录记录服务 [memos](https://github.com/usememos/memos) 记录自己的碎碎念。自部署这些服务的时候通常会涉及一些资源的存储，比如 Affine 中上传的图片资源、Gitea 的 package 制品、memos 中上传的图片等都需要消耗自身的存储空间。而我部署的平台是 PVE 的 Docker LXD 容器，这个容器里面使用 Docker 部署了一堆服务，存储已经扩了一遍又一遍了，然而主机上的存储也见底了。

![储存已经见底了](https://img.linkzz.eu.org/main/images/2026/05/711b35ab86bfdf8ee4fb8bd6637255c7.png)

正好家里还有一台绿联的 NAS，里面有一个闲置的存储池，正好用来建一个 S3 存储给以上这些服务使用。这样一方面可以在同一个地方管理上传的对象，另一方面也可以利用存储池 Raid 做一些备份。

## 2. 部署 Rustfs

要说开源 S3 实现就不得不提大名鼎鼎的 [Minio](http://github.com/minio/minio) 了，然而其因为商业化已经停止了仓库的维护。况且我的 NAS 也仅是 4C8G 的配置，所以资源能省则省，基于 Rust 开发的 [Rustfs](https://github.com/rustfs/rustfs) 就是最好的选择——部署简单，资源占用低，性能还高，简直是居家旅行、自部署常备。

NAS 上的 Docker 不能命令行，只能图形界面操作，就略过部署过程了。

![Rustfs 部署](https://img.linkzz.eu.org/main/images/2026/05/c7d3edadea5e5e398ff52b3cad71cce5.png)

部署好了之后打开 console 端口 9001 即可进行存储桶操作。

![存储桶管理](https://img.linkzz.eu.org/main/images/2026/05/c345cf9a65177b41a965d9b58138639d.png)

### 2.1 使用 AWS CLI 测试访问

创建一个访问密钥，使用 [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) 来测试访问。在 `~/.aws/credentials` 中配置好访问密钥，`~/.aws/config` 中配置 endpoint 信息：

```ini
[default]
region=us-east-1
endpoint_url=http://192.168.5.103:9000
```

region 随便填一个值就可，测试访问：

```bash
➜ : aws s3 ls s3://
2026-04-22 00:58:25 affine-storage
2026-04-22 15:20:44 gitea-storage
2026-04-22 00:42:14 linkzz-bucket
```

## 3. 使用 Rustfs 存储数据

下面介绍使用 Rustfs 储存的服务。

### 3.1 Affine

内网 Affine 的搭建可以看[这篇文章](https://wiki.linkzz.hm/workspace/aac4eaa1-d97e-42d2-8e1a-75e477650f0b/4Aqcj_NhG39sVQQzau7LR)。Affine 的存储引擎默认是本地文件系统，在 admin 页面可以更换为 S3 存储。

![Affine S3 配置](https://img.linkzz.eu.org/main/images/2026/05/023f075953fd7a7a5c57a38884a01ab7.png)

可以分两个存储桶，一个存上传的媒体文件，一个存用户头像。内网使用用户就一个，单独建一个桶来存媒体文件就好了。

![Affine 文件上传](https://img.linkzz.eu.org/main/images/2026/05/a72419efd2dbd25029e74b65e69f7d3c.png)

文件已然美美的上传了。

### 3.2 Gitea

Gitea 的代码不支持 S3 存储，但好在自己写的代码也不会太占空间。但是如果用到 Gitea 的 Package 上传一些 Docker 镜像或者是存储流水线存储的制品，这些可是很占空间的。好在这些都是支持 S3 存储的。这里要赞扬一波 Gitea，公司内部的代码库也是我搭的 Gitea，虽然脱胎于 Gogs，但在保持 Gogs 轻量级体验的同时引入了很多 GitHub 相关的功能，如完全兼容 GitHub Actions 的流水线、同样的 Package 系统（支持 Docker 镜像、Java Nexus Maven 仓库、NPM Registry 等），常见的语言依赖仓库都支持，省了很多自建的烦恼，使用起来也是极其丝滑。一个开源的产品能做到这样真的没得说。

Gitea 需要在配置文件中开启 S3 存储，加入下面的配置项重启即可：

```ini
[storage]
STORAGE_TYPE = minio
MINIO_ENDPOINT = 192.168.5.103:9000
MINIO_ACCESS_KEY_ID = YOUR_AK
MINIO_SECRET_ACCESS_KEY = YOUR_SK
MINIO_BUCKET = gitea-storage
MINIO_LOCATION = us-east-1
```

构建一个 Docker 镜像上传试试看：

![Gitea Docker 镜像上传](https://img.linkzz.eu.org/main/images/2026/05/ac5aeb6992bd134e15adea4ab8e18ce4.png)

![Gitea Package 存储桶](https://img.linkzz.eu.org/main/images/2026/05/7f390540d08d5e38332030dfcc141500.png)

gitea-storage 桶里面也有值了，只是为啥这么小，压缩率这么大的吗？后续再研究一下。

### 3.3 Memos

Memos 也是原生支持 S3 存储，后台配置一下即可：

![Memos S3 配置](https://img.linkzz.eu.org/main/images/2026/05/0ffece1bc2e98d25ae75f4690842f57c.png)

测试一下：

![Memos 上传测试](https://img.linkzz.eu.org/main/images/2026/05/e453fd03447be529563f19673764d75c.png)

看下桶里面有没有数据：

![Memos 桶数据](https://img.linkzz.eu.org/main/images/2026/05/cf3bfd45a8cbc11344d54612336ec606.png)

没问题！搞定！

## 4. 总结

终于是将一些自部署服务的媒体文件都搜集起来了。使用 S3 来集中存储自部署的这些文件，以后部署在甲骨文上的一些服务也可以将存储迁移到 S3 桶里面。后续再做一个定期将服务数据打包上传 S3，再也不怕哪天甲骨文突然被收回了。
