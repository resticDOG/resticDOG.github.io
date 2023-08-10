---
title: 使用Oracle对象存储作为博客图床（二）
date: 2023-08-09 09:03:11
description: "使用甲骨文免费对象存储作为博客图床"

tags:
  - Oracle Cloud
  - 图床
categories:
  - 教程
lightgallery: true

toc:
  auto: true
---

## 前言

[上一章](/posts/oracle-blog-picture-1)我们已经做到了使用 picgo
上传图片并获得一个可以公开访问的链接，这章介绍如何使用 CloudFlare
代理图片的访问并加上自定义域名，以及录屏转 gif 的方法。

## CloudFlare 代理访问

创建一个 Cloudflare Workers，代码如下：

```js
export default {
  async fetch(request, env) {
    const url = new URL(request.url);

    const path = url.pathname.replace("/blog/", "");

    const originUrl = `${env.OBJECT_STORAGE_BASE_URL}${path}${url.search}`;

    const res = await fetch(originUrl);

    return new Response(res.body);
  },
};
```

以上代码功能是将访问的 url 去掉存储桶名称，通过 workers
获取内容并返回给访问者，充当了一个代理的角色，这样我们可以使用自定义的域名来替代甲骨文对象存储的访问域名，以后换了图床供应商也方便切换，因为博客里的链接是不用改的。

部署完之后需要配置环境变量`OBJECT_STORAGE_BASE_URL`的值为上文的访问地址，一直到`/o/`处，如：

![image.png](https://img.linkzz.eu.org/main/images/2023/08/ce111fadec473463c7db2f96c059d17c.png)

现在你可以使用 works
提供的域名作为图片前缀访问，也可以自定义域名，该功能只限于你自己的域名是由
CloudFlare 托管解析的才可，如果已经使用 CloudFlare 作为域名解析商，在 works
配置的`Custom Domains`配置即可，CloudFlare 会自动生成 Https 证书。

![image.png](https://img.linkzz.eu.org/main/images/2023/08/39e8eba872d9e83535ddbb927ea3ea24.png)

配置好之后将`picgo`的配置文件的`urlPrefix`改为上面的域名：

```json
{
  "aws-sr3": {
    "accessKeyID": "{ak}",
    "secretAccessKey": "{sk}",
    "bucketName": "blog",
    "region": "ap-chuncheon-1",
    "uploadPath": "images/{year}/{month}/{md5}.{extName}",
    "endpoint": "https://{ns}.compat.objectstorage.ap-chuncheon-1.oraclecloud.com/",
    "acl": "public-read",
    "pathStyleAccess": true,
    "urlPrefix": "https://img.linkzz.eu.org",
    "disableBucketPrefixToURL": true
  }
}
```

## GIF 图制作

有时候我们有录屏制作 gif 的需求，不管你在什么平台，总有软件满足制作 gif
的需求，这里我只是介绍我的方法，如果你觉得麻烦，以你自己的方法为准。

### 录屏

windows 商店有一个录屏软件叫`截图工具`，可以区域录屏，一般自带了这个 UWP 程序。

![image.png](https://img.linkzz.eu.org/main/images/2023/08/a8183d9d17dd0989a8fb3e5cd8765d32.png)

打开他选择“视频”之后点击“新建”，画出录屏区域点击“开始”即可对所画区域录制，点击“结束“就可以预览视频，点击保存，之后使用
MP4 转 GIF 工具即可得到 GIF。

## MP4 转 GIF

我使用的是 ffmpeg + Gifski

### ffmpeg

- [ffmpeg](https://www.ffmpeg.org/) -
  大名鼎鼎的图像处理软件不用我多介绍了吧，使用他可逐帧提取视频图片：

```bash
ffmpeg -i video.mp4 farme%04d.png
```

### Gifski

- [Gifski](https://github.com/sindresorhus/Gifski) - Gifski 是一个开源的视频转
  GIF 工具，可以生成高质量 GIF，我们使用的是他的命令行版本。

如果你已经安装了[rust](https://rustup.rs/)，可以使用[cargo](https://doc.rust-lang.org/cargo/getting-started/installation.html)安装

```
cargo install gifski
```

或者你可以从 github
的[release](https://github.com/ImageOptim/gifski/releases)页面选择相应平台的安装包安装

使用 ffmpeg 生成的套图生成 gif:

```bash
gifski -o output.gif frame*.png
```

这样就可以将你的 gif 上传到图床，通过自己的域名愉快的访问啦！

## 结语

至此 Oracle 对象存储就算是利用起来了，其免费 20G
的额度对我这种小流量个人博主来说够用了(~~~其实只是想白嫖~~~)，当然目前来看访问速度作为图床来看还是太慢了，不过也不要钱，还要啥自行车
😂
