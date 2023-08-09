---
title: 使用Oracle对象存储作为博客图床（一）
date: Tue Jul 11 16:20:02 CST 2023
draft: false
description: "使用甲骨文免费对象存储作为博客图床"
images:
  - "https://img.linkzz.eu.org/main/images/2023/07/2e8160bd1dd204c75ec540bbce0449b3.png"
resources:
  - name: "featured-image"
    src: "https://img.linkzz.eu.org/main/images/2023/07/2e8160bd1dd204c75ec540bbce0449b3.png"

tags:
  - Oracle Cloud
  - 图床
categories:
  - 教程
lightgallery: true

toc:
  auto: true
---

先看效果：

![oracle_photo_blog.gif](https://img.linkzz.eu.org/main/images/2023/06/6746b05d3db94d4f5e9dd174a0d73831.gif)

作为一个专业白嫖党，只使用 Oracle 的 4 个免费鸡怎么能满足呢，拥有 20GB
免费空间的对象存储当然也要利用起来，正好最近又重拾起了鸽了多年的博客，正好作为博客的图床。

<!-- more -->

## 前置条件

1. 一个 Oracle Cloud 账号（需要信用卡认证，大陆已经很难申请了）
2. 良好的科学环境

## Oracle Object Storage 存储桶准备

1. 进入 Oracle Cloud
   [对象存储](https://console.ap-chuncheon-1.oraclecloud.com/object-storage/buckets)管理界面，创建新的存储桶

   ![image.png](https://img.linkzz.eu.org/main/images/2023/07/2e8160bd1dd204c75ec540bbce0449b3.png)

   默认配置即可，创建完之后编辑可见性，这里可以选择公开或者私有，
   **公开**：所有文件允许读取
   **私有**：可使用预先验证的请求实现部分文件公开（推荐这种方式）

   ![image.png](https://img.linkzz.eu.org/main/images/2023/07/40096c4fdfbf90922518292d20237ac3.png)

2. 创建预先验证的请求

   ![image.png](https://img.linkzz.eu.org/main/images/2023/07/09d4c0a3290c70273f698dc5d29fd99d.png)

   选择具有前缀的对象，前缀填写`images`,下文配置 picgo
   时需要匹配这个前缀，否则会有权限问题，访问类型默认只读即可，到期时间尽量一步到位多配几年。

   ![image.png](https://img.linkzz.eu.org/main/images/2023/07/6f53d7d3c8cc4abc2c1ed8f3819f4b1b.png)

   将 URL 复制保存，这即时存储桶相应前缀目录访问的 BaseURL

   ![image.png](https://img.linkzz.eu.org/main/images/2023/07/d11281cd23ec1d9931e76826b5339693.png)

   上传一个图片试一下能否访问

   ![image.png](https://img.linkzz.eu.org/main/images/2023/07/f803947dcc9cc5d3678bed1740995af6.png)

   可以看到由于存储桶的权限设置，使用 URL 路径访问是行不通的

   ![image.png](https://img.linkzz.eu.org/main/images/2023/07/38a0255029419d9b1b66bc7e2b0ae25a.png)

   需要替换为刚才创建的预见请求的 URL 前缀
   `https://objectstorage.ap-chuncheon-1.oraclecloud.com/p/xxx/n/{namespace}/b/blog/o/images/2022-05-23.png`

3. 存储桶 Endpoint 甲骨文的对象存储 endpoint
   模式为`https://{namespace}.compat.objectstorage.{region}.oraclecloud.com/`

- namespace: 存储桶信息名称空间字段

  ![image.png](https://img.linkzz.eu.org/main/images/2023/07/87bc00ded688e7a6032829eb38465986.png)

- region: 这个是你甲骨文账号的区域，通常在你的 Oci 的 console
  地址里就有体现，如我的春川为`ap-chuncheon-1`

  ![image.png](https://img.linkzz.eu.org/main/images/2023/07/e1ed2c4899256a527738e62af48deea8.png)

4. 申请 Oracle 账户 Ak 和 Sk
   右上角头像，进入“我的概要信息“，”客户密钥“选项”生成密钥“

   ![image.png](https://img.linkzz.eu.org/main/images/2023/07/a78dd370e5b5ccd75fdbfabe1d48be02.png)

   同样保存已生成密钥，该密钥为账户的`SecretAccessKey`

   ![image.png](https://img.linkzz.eu.org/main/images/2023/07/b893e0be916eadc48c61f025d687a83e.png)

   `AccessKeyID`在列表页，同样复制保存

   ![image.png](https://img.linkzz.eu.org/main/images/2023/07/f73f802dbce811f62ee740a004194777.png)

## 安装配置 Picgo 和 s3 插件

准备工作完成，下面开始配置图床工具。这里有两种选择：

1. 具有 GUI 界面：[PicGo](https://github.com/Molunerfinn/PicGo)
2. Cli 命令行操作：[PicGo-Core](https://github.com/PicGo/PicGo-Core)
   上面两种看个人喜好，我比较习惯命令行，所以选择第二种，配置方面都是一样的，区别为是否提供了
   GUI，

### 安装 Node 环境

自行按[nodejs](https://nodejs.org/zh-cn)官网安装

### 安装 Picgo-Core

```bash
npm install picgo -g
```

或者使用 Yarn

```bash
yarn global add picgo
```

验证安装

```bash
picgo -h
```

出现以下结果表明安装成功
![image.png](https://img.linkzz.eu.org/main/images/2023/07/86ebb4100d6fccadc15da9dd4ac54b2f.png)

### 添加 S3 插件

s3 插件理论上支持所有兼容 S3API
的对象存储，具体参看插件[主页](https://github.com/wayjam/picgo-plugin-s3)

```bash
picgo add s3
```

安装成功之后配置文件含有以下信息
![image.png](https://img.linkzz.eu.org/main/images/2023/07/02cc67a290ac0bcfcdf73e54d98fb949.png)

### 配置 s3 插件

```json
{
  "picBed": {
    "uploader": "aws-s3",
    "current": "aws-s3",
    "aws-s3": {
      "accessKeyID": "{ak}",
      "secretAccessKey": "{sk}",
      "bucketName": "{bucket}",
      "region": "{region}",
      "uploadPath": "images/{year}/{month}/{md5}.{extName}",
      "endpoint": "https://{namespace}.compat.objectstorage.{region}.oraclecloud.com/",
      "acl": "public-read",
      "pathStyleAccess": true,
      "urlPrefix": "https://objectstorage.ap-chuncheon-1.oraclecloud.com/p/{publicKey}/n/{namespace}/b/blog/o",
      "disableBucketPrefixToURL": true
    }
  },
  "picgoPlugins": {
    "picgo-plugin-s3": true
  }
}
```

上面配置除了`uploadPath`字段，其他地方的占位符号"{}"应替换为相应值:

- ak: 上文申请的甲骨文账号`AccessKeyID`
- sk: 上文申请的甲骨文账号`SecretAccessKey`
- bucket: 桶名称，我的这里就是`blog`
- region: 上文获取的`region`区域，我这里是`ap-chuncheon-1`
- uploadPath 的占位符不需要修改，且注意前缀 images
  一定要匹配上文`预先验证的请求`处的前缀
- namespace: 上文获取的 namespace
- urlPrefix: 这个字段为`预先验证的请求`处申请的 URL 前缀
- disableBucketPrefixToURL: 这个字段必须为
  true,否则上传链接自动添加桶前缀导致访问失败，具体可看这个[Issue](https://github.com/wayjam/picgo-plugin-s3/issues/30)

### 验证配置

截图工具随意截图到剪贴板

```bash
picgo upload
```

上传剪贴板图片

![image.png](https://img.linkzz.eu.org/main/images/2023/07/38b3dcbecf01b7745306f1ed0baa658b.png)

访问图片链接成功

![image.png](https://img.linkzz.eu.org/main/images/2023/07/401f1d04985e61ad664dbc0a48580bab.png)

到此就实现图床最基本的功能了，搭配任意编辑器都可使用图床了，只是稍显麻烦，如果在编辑器中复制上传，成功后即刻显示图片体验会更好。下面我们使用
Obsidian 搭配对应插件实现上面功能。

## 安装配置 Obsidian

下载安装[Obsidian](https://obsidian.md/)

关闭安全模式搜索”Image auto upload Plugin“插件安装

![image.png](https://img.linkzz.eu.org/main/images/2023/07/c594d34182e984173701c5b791a80771.png)

插件配置

![image.png](https://img.linkzz.eu.org/main/images/2023/07/b773c31195e62390496bb44ba90f1386.png)

默认上传器选择`PicGo-Core`

## 结语

至此使用 Oracle
作为对象存储教程已经完成，但是还是有些缺陷，如图片访问效果不理想，因为甲骨文的对象存储不支持
cdn，导致不同区域访问效果不同。下一章我们使用 CloudFlare
加速我们的图片访问，以及特殊情况下制作 GIF 上传的方法。
