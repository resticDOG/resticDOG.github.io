---
title: Oracle Linux 8 升级内核
date: 2023-11-24
description: "Oracle Linux 8 升级内核"
layout: post

tags:
  - Oracle Cloud
categories:
  - Linux
lightgallery: true

toc:
  auto: true
---

甲骨文的免费Arm主机开始申请的时候没注意选了 `Oracle Linux 8` 系统而没选自己更熟
悉的`Ubuntu` ，导致各种折腾的时候发现资料有点少，今天折腾一个云安卓 redroid 系统
的时候发现需要编译一些内核模块，而默认的 `5.4` 内核一直编译失败，于是想到升级内
核试试，一路搜索找不到很符合的文章，于是自己摸索了一下，记录于此。

## 1. 查看已经安装的内核包

```bash
➜  client yum list installed | grep kernel
kernel-headers.aarch64                    4.18.0-477.13.1.el8_8                         @ol8_baseos_latest
kernel-tools.aarch64                      4.18.0-477.13.1.el8_8                         @ol8_baseos_latest
kernel-tools-libs.aarch64                 4.18.0-477.13.1.el8_8                         @ol8_baseos_latest
kernel-uek.aarch64                        5.4.17-2136.307.3.1.el8uek                    @ol8_baseos_latest
kernel-uek.aarch64                        5.4.17-2136.309.4.el8uek                      @ol8_baseos_latest
kernel-uek.aarch64                        5.4.17-2136.320.7.1.el8uek                    @ol8_baseos_latest
kernel-uek-devel.aarch64                  5.4.17-2136.307.3.1.el8uek                    @ol8_baseos_latest
kernel-uek-devel.aarch64                  5.4.17-2136.309.4.el8uek                      @ol8_baseos_latest
kernel-uek-devel.aarch64                  5.4.17-2136.320.7.1.el8uek                    @ol8_baseos_latest
```

可以看到我们的内核都是baseos仓库安装的内核，该仓库内核版本比较老，查看
[官网仓库](https://yum.oracle.com/oracle-linux-8.html) 列表，内核仓库有了更新的
包。

![image.png](https://img.linkzz.eu.org/main/images/2023/11/65b914750a8e4af2223222846ec0bc9e.png)

内核版本为 `5.15.0`

![image.png](https://img.linkzz.eu.org/main/images/2023/11/0f874fa31fde2c31254bded5b203d3d1.png)

## 2. 安装内核

先看下已有的 yum 库

```bash
➜  sudo yum repolist
repo id                                        repo name
docker-ce-nightly                              Docker CE Nightly - aarch64
docker-ce-stable                               Docker CE Stable - aarch64
docker-ce-test                                 Docker CE Test - aarch64
epel                                           Extra Packages for Enterprise Linux 8 - aarch64
nginx-stable                                   nginx stable repo
ol8_MySQL80                                    MySQL 8.0 for Oracle Linux 8 (aarch64)
ol8_MySQL80_connectors_community               MySQL 8.0 Connectors Community for Oracle Linux 8 (aarch64)
ol8_MySQL80_tools_community                    MySQL 8.0 Tools Community for Oracle Linux 8 (aarch64)
ol8_addons                                     Oracle Linux 8 Addons (aarch64)
ol8_appstream                                  Oracle Linux 8 Application Stream (aarch64)
ol8_baseos_latest                              Oracle Linux 8 BaseOS Latest (aarch64)
ol8_developer_EPEL                             Oracle Linux 8 EPEL Packages for Development (aarch64)
ol8_ksplice                                    Ksplice for Oracle Linux 8 (aarch64)
ol8_oci_included                               Oracle Software for OCI users on Oracle Linux 8 (aarch64)
```

没有包含Release 7的这个内核仓库，现在添加一下：

```bash
sudo yum-config-manager --add-repo http://yum.oracle.com/repo/OracleLinux/OL8/UEKR7/aarch64
```

检查一下是否添加成功

```bash
➜  ~ sudo yum repolist
repo id                                           repo name
docker-ce-nightly                                 Docker CE Nightly - aarch64
docker-ce-stable                                  Docker CE Stable - aarch64
docker-ce-test                                    Docker CE Test - aarch64
epel                                              Extra Packages for Enterprise Linux 8 - aarch64
nginx-stable                                      nginx stable repo
ol8_MySQL80                                       MySQL 8.0 for Oracle Linux 8 (aarch64)
ol8_MySQL80_connectors_community                  MySQL 8.0 Connectors Community for Oracle Linux 8 (aarch64)
ol8_MySQL80_tools_community                       MySQL 8.0 Tools Community for Oracle Linux 8 (aarch64)
ol8_addons                                        Oracle Linux 8 Addons (aarch64)
ol8_appstream                                     Oracle Linux 8 Application Stream (aarch64)
ol8_baseos_latest                                 Oracle Linux 8 BaseOS Latest (aarch64)
ol8_developer_EPEL                                Oracle Linux 8 EPEL Packages for Development (aarch64)
ol8_ksplice                                       Ksplice for Oracle Linux 8 (aarch64)
ol8_oci_included                                  Oracle Software for OCI users on Oracle Linux 8 (aarch64)
yum.oracle.com_repo_OracleLinux_OL8_UEKR7_aarch64 created by dnf config-manager from http://yum.oracle.com/repo/OracleLinux/OL8/UEKR7/aarch64
```

最后再更新一下:

```bash
sudo yum update
```

可已查看到将要更新的包了，同时 yum 会自动移除旧的内核，这里是同时升级仓库中所有
已安装的包版本，如果只想升级内核可以使用 `sudo yum update kernel\*`

最后重启一下

```
reboot
```

## 3. 验证内核安装

```
➜  ~ uname -r
5.15.0-200.131.27.el8uek.aarch64
```

新的内核已经安装成功！
