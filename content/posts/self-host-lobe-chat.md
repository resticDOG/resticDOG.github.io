---
title: 自签名证书部署内网 https 版 lobe-chat
date: 2025-02-07
description: "自签名证书部署内网 https 版 lobe-chat"
layout: post

tags:
  - AI
  - Docker
categories:
  - Docker
lightgallery: true

toc:
  auto: true
---

## 1. 背景

一开始部署了 [Nextchat](https://github.com/ChatGPTNextWeb/NextChat) 作为我日常的 AI 问答网页应用，原因很简单：

1. 该项目启动早，社区很活跃
2. 网页应用结合 PWA 使用起来也很方便
3. 支持模型供应商全面

但是使用了一段时间之后接触到了更为全面和强大的 [lobe-chat](https://github.com/lobehub/lobe-chat)，lobe-chat 发展迅速，社区更为活跃，社区版支持子部署，且没有功能限制，公司级的开源项目，产品 UI 设计很现代，社区 Roadmap 还计划支持更多的功能，如文生视频，助手自主学习等功能（无恰饭，纯使用感受）。

起初仅通过 docker-compose 部署了纯前端版本的 lobe-chat，这个版本由于是纯前端，没有多端同步的功能，同一份配置需要在家里电脑、公司电脑和手机上重复配置，相当麻烦，所以还是部署数据库版本的完全体 lobe-chat 更为合适。

## 2. 官方脚本部署

### 2.1 脚本部署

说干就干，目前版本是 `v1.49.12`，使用官方 docker-compose 部署脚本部署：

```bash
bash <(curl -fsSL https://lobe.li/setup.sh) -l zh_CN
```

支持 3 种模式部署：

- [本地模式（默认）](https://lobehub.com/zh/docs/self-hosting/server-database/docker-compose#%E6%9C%AC%E5%9C%B0%E6%A8%A1%E5%BC%8F)：仅能在本地访问，不支持局域网 / 公网访问，适用于初次体验；
- [端口模式](https://lobehub.com/zh/docs/self-hosting/server-database/docker-compose#%E7%AB%AF%E5%8F%A3%E6%A8%A1%E5%BC%8F)：支持局域网 / 公网的 `http` 访问，适用于无域名或内部办公场景使用；
- [域名模式](https://lobehub.com/zh/docs/self-hosting/server-database/docker-compose#%E5%9F%9F%E5%90%8D%E6%A8%A1%E5%BC%8F)：支持局域网 / 公网在使用反向代理下的 `http/https` 访问，适用于个人或团队日常使用；

第一种模式纯自身访问，无法从其他设备访问，局限性很大；第二种模式在局域网中访问可行，但是 IP 端口访问的方式在局域网环境中我并不推崇，一是需要记忆特定的 IP 端口，而是 http 的方式无法安装 PWA 应用，仅适用于初次体验，而我自身用于家庭服务器，并且部署了一个自签名证书的 Nginx，完全可以试用该证书部署 https 的服务，而不用担心暴露公网，通过 VPN 即可在任何地方访问家庭服务也很方便，关于自签名证书和我的服务器拓扑 VPN 配置等信息我在 [这篇文章](/docker-trust-ca) 以及 [这篇文章](/openwrt-wireguard) 中有详细介绍，这里不再赘述。

先规划 4 个域名作为服务：

|域名|描述|代理端口|
|--|--|--|
|lobe.linkzz.hm     |   lobe chat 主域名 | 3210 |
|lobe-login.linkzz.hm | lobe chat oauth提供商casdoor管理地址，登录/注册地址 | 8000 |
| lobe-mc.linkzz.hm |    lobe chat S3服务，minio api 访问地址 | 9000 |
| lobe-mc-ui.linkzz.hm | minio s3 console ui 访问地址 | 9001 |

以上在官方脚本部署的时候输入相应的域名即可，其他的密码等按照提示输入修改，之后脚本会修改 `.env` 、`docker-compose.yml` 、`init_data.json` 3 个配置文件，确认无误之后执行 `docker compose up -d` 启动服务

### 2.2 排查问题

执行 `docker compose ps` 查看服务状态，发现所有服务均处于正常 `RUNNING` 状态

![image.png](https://img.linkzz.eu.org/main/images/2025/02/0487740befded03f50d9fdb5b96ebf03.png)

但是执行 `docker compose logs -f` 发现 `minio` 服务一直在报错

```log
lobe-minio     | /bin/sh: line 4: curl: command not found
lobe-minio     | Waiting for MinIO to start…
```

查看 `docker-compose.yml` 文件

```yaml
name: lobe-chat-database
services:
  network-service:
    image: alpine
    container_name: lobe-network
    ports:
      - '${MINIO_PORT}:${MINIO_PORT}' # MinIO API
      - '9001:9001' # MinIO Console
      - '${CASDOOR_PORT}:${CASDOOR_PORT}' # Casdoor
      - '${LOBE_PORT}:3210' # LobeChat
    command: tail -f /dev/null
    networks:
      - lobe-network

  postgresql:
    image: pgvector/pgvector:pg16
    container_name: lobe-postgres
    ports:
      - '5432:5432'
    volumes:
      - './data:/var/lib/postgresql/data'
    environment:
      - 'POSTGRES_DB=${LOBE_DB_NAME}'
      - 'POSTGRES_PASSWORD=${POSTGRES_PASSWORD}'
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U postgres']
      interval: 5s
      timeout: 5s
      retries: 5
    restart: always
    networks:
      - lobe-network

  minio:
    image: minio/minio
    container_name: lobe-minio
    network_mode: 'service:network-service'
    volumes:
      - './s3_data:/etc/minio/data'
    environment:
      - 'MINIO_API_CORS_ALLOW_ORIGIN=*'
    env_file:
      - .env
    restart: always
    entrypoint: >
      /bin/sh -c "
        minio server /etc/minio/data --address ':${MINIO_PORT}' --console-address ':9001' &
        MINIO_PID=\$!
        while ! curl -s http://localhost:${MINIO_PORT}/minio/health/live; do
          echo 'Waiting for MinIO to start...'
          sleep 1
        done
        sleep 5
        mc alias set myminio http://localhost:${MINIO_PORT} ${MINIO_ROOT_USER} ${MINIO_ROOT_PASSWORD}
        echo 'Creating bucket ${MINIO_LOBE_BUCKET}'
        mc mb myminio/${MINIO_LOBE_BUCKET}
        wait \$MINIO_PID
      "

  casdoor:
    image: casbin/casdoor
    container_name: lobe-casdoor
    entrypoint: /bin/sh -c './server --createDatabase=true'
    network_mode: 'service:network-service'
    depends_on:
      postgresql:
        condition: service_healthy
    environment:
      RUNNING_IN_DOCKER: 'true'
      driverName: 'postgres'
      dataSourceName: 'user=postgres password=${POSTGRES_PASSWORD} host=postgresql port=5432 sslmode=disable dbname=casdoor'
      runmode: 'dev'
    volumes:
      - ./init_data.json:/init_data.json
    env_file:
      - .env

  lobe:
    image: lobehub/lobe-chat-database
    container_name: lobe-chat
    network_mode: 'service:network-service'
    depends_on:
      postgresql:
        condition: service_healthy
      network-service:
        condition: service_started
      minio:
        condition: service_started
      casdoor:
        condition: service_started

    environment:
      - 'NEXT_AUTH_SSO_PROVIDERS=casdoor'
      - 'KEY_VAULTS_SECRET=Kix2wcUONd4CX51E/ZPAd36BqM4wzJgKjPtz2sGztqQ='
      - 'NEXT_AUTH_SECRET=NX2kaPE923dt6BL2U8e9oSre5RfoT7hg'
      - 'DATABASE_URL=postgresql://postgres:${POSTGRES_PASSWORD}@postgresql:5432/${LOBE_DB_NAME}'
      - 'S3_BUCKET=${MINIO_LOBE_BUCKET}'
      - 'S3_ENABLE_PATH_STYLE=1'
      - 'S3_ACCESS_KEY=${MINIO_ROOT_USER}'
      - 'S3_ACCESS_KEY_ID=${MINIO_ROOT_USER}'
      - 'S3_SECRET_ACCESS_KEY=${MINIO_ROOT_PASSWORD}'
      - 'LLM_VISION_IMAGE_USE_BASE64=1'
    env_file:
      - .env
    restart: always

volumes:
  data:
    driver: local
  s3_data:
    driver: local

networks:
  lobe-network:
    driver: bridge
```

发现 minio 的服务中会重复检查 minio 的服务正常启动之后再执行创建桶的操作，但是容器确实 `curl` 工具导致服务桶服务创建，但是容器并没有退出，这样即使 lobe-chat 的服务能正常运行后面涉及到存储操作的时候也一定会出现问题，这就来解决一下这个问题。

minio 服务是基于 `minio/minio:latest` 镜像，通过 docker hub 可以查看该版本的镜像 layer，在 docker hub 中搜索 `minio/minio:latest` ，在 tags 中筛选出对应标签即可看到各个架构镜像的 hash, 点击 hash 即可跳转镜像层浏览页，我这里使用的 `linux/amd64` 架构的镜像。

![image.png](https://img.linkzz.eu.org/main/images/2025/02/4408a032740e09ace7ecaffe3a5448ee.png)

```markdown
`1``LABEL maintainer="Red Hat, Inc."``0 B`

`2``LABEL vendor="Red Hat, Inc."``0 B`

`3``LABEL url="https://www.redhat.com"``0 B`

`4``LABEL com.redhat.component="ubi9-micro-container"``0 B`

`5``LABEL name="ubi9/ubi-micro"``0 B`

`6``LABEL version="9.5"``0 B`

`7``LABEL distribution-scope="public"``0 B`

`8``LABEL com.redhat.license_terms="https://www.redhat.com/en/about/red-hat-end-user-license-agreements#UBI"``0 B`

`9``LABEL summary="ubi9 micro image"``0 B`

`10``LABEL description="Very small image which``0 B`

`11``LABEL io.k8s.description="Very small image which``0 B`

`12``LABEL io.k8s.display-name="Red Hat Universal Base``0 B`

`13``LABEL io.openshift.expose-services=""``0 B`

`14``COPY dir:f89de9a231f3a4edbee769aa68d7a28b19bf78f5d7243315a0df3b339f370b11 in /``0 B`

`15``COPY file:b37d593713ee21ad52a4cd1424dc019a24f7966f85df0ac4b86d234302695328 in /etc/yum.repos.d/``0 B`

`16``CMD /bin/sh``0 B`

`17``LABEL "build-date"="2025-01-09T12:48:29" "architecture"="x86_64" "vcs-type"="git" "vcs-ref"="0bf50525dc8ce0b1f0fb83b0d10a826d8c769a89"``6.92 MB`

`18``/bin/sh``389 B`

`19``ARG RELEASE=RELEASE.2025-02-03T21-03-04Z``0 B`

`20``LABEL name=MinIO vendor=MinIO Inc <dev@min.io>``0 B`

`21``ENV MINIO_ACCESS_KEY_FILE=access_key MINIO_SECRET_KEY_FILE=secret_key MINIO_ROOT_USER_FILE=access_key MINIO_ROOT_PASSWORD_FILE=secret_key``0 B`

`22``RUN |1 RELEASE=RELEASE.2025-02-03T21-03-04Z /bin/sh -c``1.56 MB`

`23``COPY /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ # buildkit``125.3 KB`

`24``COPY /go/bin/minio* /usr/bin/ # buildkit``38.92 MB`

`25``COPY /go/bin/mc* /usr/bin/ # buildkit``10.23 MB`

`26``COPY /go/bin/curl* /usr/bin/ # buildkit``2.29 MB`

`27``COPY CREDITS /licenses/CREDITS # buildkit``169.36 KB`

`28``COPY LICENSE /licenses/LICENSE # buildkit``11.6 KB`

`29``COPY dockerscripts/docker-entrypoint.sh /usr/bin/docker-entrypoint.sh # buildkit``500 B`

`30``EXPOSE map[9000/tcp:{}]``0 B`

`31``VOLUME [/data]``0 B`

`32``ENTRYPOINT ["/usr/bin/docker-entrypoint.sh"]``0 B`

`33``CMD ["minio"]``0 B`

```

可是看到该镜像层中明明有 `COPY /go/bin/curl* /usr/bin` 的操作，却在镜像中无法找到该命令，这不应该呀，仔细排查一通才发现，我本地的镜像根本没更新到，原因是官方的脚本在最后部署的时候只说了运行 `docker compose up -d` 而没有更新进行，恰好我的原来已然下载了 minio 的 `latest` 标签镜像，这才在 `docker compose up` 的时候没有去更新该镜像，再来看下本地镜像的镜像层信息，运行 `docker history minio/minio:latest` 如下：

```text
root@Docker ~/apps/lobe-chat$ docker history minio/minio:latest --no-trunc
IMAGE                                                                     CREATED         CREATED BY                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    SIZE      COMMENT
sha256:57f1cc3888f19d7d4480f3a6539d20992e18c5aeeaebced1518bd07c84379e11   12 months ago   CMD ["minio"]                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 0B        buildkit.dockerfile.v0
<missing>                                                                 12 months ago   ENTRYPOINT ["/usr/bin/docker-entrypoint.sh"]                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  0B        buildkit.dockerfile.v0
<missing>                                                                 12 months ago   VOLUME [/data]                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                0B        buildkit.dockerfile.v0
<missing>                                                                 12 months ago   EXPOSE map[9000/tcp:{}]                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       0B        buildkit.dockerfile.v0
<missing>                                                                 12 months ago   COPY dockerscripts/docker-entrypoint.sh /usr/bin/docker-entrypoint.sh # buildkit                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              675B      buildkit.dockerfile.v0
<missing>                                                                 12 months ago   COPY LICENSE /licenses/LICENSE # buildkit                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     34.5kB    buildkit.dockerfile.v0
<missing>                                                                 12 months ago   COPY CREDITS /licenses/CREDITS # buildkit                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     1.77MB    buildkit.dockerfile.v0
<missing>                                                                 12 months ago   COPY /go/bin/mc /usr/bin/mc # buildkit                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        26.9MB    buildkit.dockerfile.v0
<missing>                                                                 12 months ago   COPY /go/bin/minio /usr/bin/minio # buildkit                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  99.9MB    buildkit.dockerfile.v0
<missing>                                                                 12 months ago   COPY /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ # buildkit                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            215kB     buildkit.dockerfile.v0
<missing>                                                                 12 months ago   ENV MINIO_ACCESS_KEY_FILE=access_key MINIO_SECRET_KEY_FILE=secret_key MINIO_ROOT_USER_FILE=access_key MINIO_ROOT_PASSWORD_FILE=secret_key MINIO_KMS_SECRET_KEY_FILE=kms_master_key MINIO_UPDATE_MINISIGN_PUBKEY=RWTx5Zr1tiHQLwG9keckT0c45M3AGeHD6IvimQHpyRywVWGbP1aVSGav MINIO_CONFIG_ENV_FILE=config.env MC_CONFIG_DIR=/tmp/.mc                                                                                                                                                                                                                                              0B        buildkit.dockerfile.v0
<missing>                                                                 12 months ago   LABEL name=MinIO vendor=MinIO Inc <dev@min.io> maintainer=MinIO Inc <dev@min.io> version=RELEASE.2024-02-04T22-36-13Z release=RELEASE.2024-02-04T22-36-13Z summary=MinIO is a High Performance Object Storage, API compatible with Amazon S3 cloud storage service. description=MinIO object storage is fundamentally different. Designed for performance and the S3 API, it is 100% open-source. MinIO is ideal for large, private cloud environments with stringent security requirements and delivers mission-critical availability across a diverse range of workloads.   0B        buildkit.dockerfile.v0
<missing>                                                                 12 months ago   ARG RELEASE                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   0B        buildkit.dockerfile.v0
<missing>                                                                 12 months ago   /bin/sh -c #(nop) LABEL "distribution-scope"="public" "vendor"="Red Hat, Inc." "build-date"="2024-01-18T10:06:04" "architecture"="x86_64" "vcs-type"="git" "vcs-ref"="303dc144996db01765d69e8ea45d2d617d953e42" "io.k8s.description"="Very small image which doesn't install the package manager." "url"="https://access.redhat.com/containers/#/registry.access.redhat.com/ubi9/ubi-micro/images/9.3-13"                                                                                                                                                                     22.7MB
<missing>                                                                 12 months ago   /bin/sh -c #(nop) ADD file:0fc7de14fc44e1d80c5ac6d6062ee58a9464555c26081dbb7050df8f2a8e586d in /root/buildinfo/Dockerfile-ubi9-ubi-micro-9.3-13                                                                                                                                                                                                                                                                                                                                                                                                                               0B
<missing>                                                                 12 months ago   /bin/sh -c #(nop) ADD file:6dca2b6931e721dc2515dbd66acd41501bf4bbd06ad9c6d0d5950a8f4af77a5d in /root/buildinfo/content_manifests/ubi9-micro-container-9.3-13.json                                                                                                                                                                                                                                                                                                                                                                                                             0B
<missing>                                                                 12 months ago   /bin/sh -c #(nop) LABEL release=13                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            0B
<missing>                                                                 12 months ago   /bin/sh -c #(nop) CMD /bin/sh                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 0B
<missing>                                                                 12 months ago   /bin/sh -c #(nop) COPY file:eec73f859c6e7f6c8a9427ecc5249504fe89ae54dc3a1521b442674a90497d32 in /etc/yum.repos.d/ubi.repo                                                                                                                                                                                                                                                                                                                                                                                                                                                     0B
<missing>                                                                 12 months ago   /bin/sh -c #(nop) COPY dir:b4c56c8ee136a3d8550817bb736bcd586be7da120fa51e85d80783b087276962 in /                                                                                                                                                                                                                                                                                                                                                                                                                                                                              0B
<missing>                                                                 12 months ago   /bin/sh -c #(nop) LABEL io.openshift.expose-services=""                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       0B
<missing>                                                                 12 months ago   /bin/sh -c #(nop) LABEL io.k8s.display-name="Ubi9-micro"                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      0B
<missing>                                                                 12 months ago   /bin/sh -c #(nop) LABEL description="Very small image which doesn't install the package manager."                                                                                                                                                                                                                                                                                                                                                                                                                                                                             0B
<missing>                                                                 12 months ago   /bin/sh -c #(nop) LABEL summary="ubi9 micro image"                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            0B
<missing>                                                                 12 months ago   /bin/sh -c #(nop) LABEL com.redhat.license_terms="https://www.redhat.com/en/about/red-hat-end-user-license-agreements#UBI"                                                                                                                                                                                                                                                                                                                                                                                                                                                    0B
<missing>                                                                 12 months ago   /bin/sh -c #(nop) LABEL version="9.3"                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         0B
<missing>                                                                 12 months ago   /bin/sh -c #(nop) LABEL name="ubi9/ubi-micro"                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 0B
<missing>                                                                 12 months ago   /bin/sh -c #(nop) LABEL com.redhat.component="ubi9-micro-container"                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           0B
<missing>                                                                 12 months ago   /bin/sh -c #(nop) LABEL maintainer="Red Hat, Inc."                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            0B
```

果然这个版本中并没有静态的 curl 工具，基础镜像中也没有自带 curl，这才导致了 curl 命令找不到的问题

知道了原因解决就简单多了，执行 `docker compose pull` 更新一下镜像，之后再执行 `docker compose up -d` 运行服务就好了

![image.png](https://img.linkzz.eu.org/main/images/2025/02/76548de5afcc6aa51fd646762ec26b5a.png)

## 3. 自签名证书的问题

按照上面的域名配置好端口转发之后访问 <https://lobe.linkzz.hm> 首页跳转登录的时候报错了

<https://lobe.linkzz.hm/next-auth/error?error=Configuration>

![image.png](https://img.linkzz.eu.org/main/images/2025/02/b55a576187324344f4ccddc27cdea4f8.png)

查看后台日志报错如下：

```log
lobe-chat      | [auth][error] TypeError: fetch failed
lobe-chat      |     at node:internal/deps/undici/undici:13502:13
lobe-chat      |     at process.processTicksAndRejections (node:internal/process/task_queues:105:5)
lobe-chat      |     at async iY (/app/.next/server/chunks/1774.js:368:46924)
lobe-chat      |     at async iQ (/app/.next/server/chunks/1774.js:368:49798)
lobe-chat      |     at async i5 (/app/.next/server/chunks/1774.js:368:52440)
lobe-chat      |     at async i6 (/app/.next/server/chunks/1774.js:368:56596)
lobe-chat      |     at async tr.do (/app/node_modules/.pnpm/next@15.1.6_@babel+core@7.26.7_@opentelemetry+api@1.9.0_@playwright+test@1.50.1_react-dom@19._snrbxk6bhvu4ah46wgxzyq2vee/node_modules/next/dist/compiled/next-server/app-route.runtime.prod.js:18:17582)
lobe-chat      |     at async tr.handle (/app/node_modules/.pnpm/next@15.1.6_@babel+core@7.26.7_@opentelemetry+api@1.9.0_@playwright+test@1.50.1_react-dom@19._snrbxk6bhvu4ah46wgxzyq2vee/node_modules/next/dist/compiled/next-server/app-route.runtime.prod.js:18:22212)
lobe-chat      |     at async doRender (/app/node_modules/.pnpm/next@15.1.6_@babel+core@7.26.7_@opentelemetry+api@1.9.0_@playwright+test@1.50.1_react-dom@19._snrbxk6bhvu4ah46wgxzyq2vee/node_modules/next/dist/server/base-server.js:1452:42)
lobe-chat      |     at async responseGenerator (/app/node_modules/.pnpm/next@15.1.6_@babel+core@7.26.7_@opentelemetry+api@1.9.0_@playwright+test@1.50.1_react-dom@19._snrbxk6bhvu4ah46wgxzyq2vee/node_modules/next/dist/server/base-server.js:1822:28)
lobe-chat      | [NextAuth] Error: {
lobe-chat      |   cause: 'Configuration',
lobe-chat      |   message: 'Wrong configuration, make sure you have the correct environment variables set. Visit https://lobehub.com/docs/self-hosting/advanced/authentication for more details.',
lobe-chat      |   name: 'NextAuth Error'
lobe-chat      | }
```

同样排查了一番之后才发现因为是自签名的 ssl 证书，在容器中访问的时候并没有加载宿主机的证书，**导致 `lobe-chat` 服务访问 `lobe-casdoor` 容器的时候证书错误**。

这里有两个解决方案：

### 3.1 忽略证书错误

由于 `lobe-chat` 是一个 nodejs 程序，我们在 nodejs 的配置中配置忽略证书错误即可正常访问 casdoor 和 minio 的 api，配置如下:
编辑 `.env` 文件，添加一行环境变量：

```bash
NODE_TLS_REJECT_UNAUTHORIZED=0
```

之后重新创建容器：

```bash
docker compose down && docker compose up -d
```

### 3.2 挂载宿主机证书目录

第二种方法前面文章提到过，在信任了该证书的宿主机上可以挂载宿主机的证书目录到容器中，这样容器也可正常访问其他两个服务。

修改 `docker-compose.yml` 文件 `diff`：

```yaml
name: lobe-chat-database                                        name: lobe-chat-database
services:                                                       services:
  network-service:                                                network-service:
    image: alpine                                                   image: alpine
    container_name: lobe-network                                    container_name: lobe-network
    ports:                                                          ports:
      - '${MINIO_PORT}:${MINIO_PORT}' # MinIO API                     - '${MINIO_PORT}:${MINIO_PORT}' # MinIO API
      - '9001:9001' # MinIO Console                                   - '9001:9001' # MinIO Console
      - '${CASDOOR_PORT}:${CASDOOR_PORT}' # Casdoor                   - '${CASDOOR_PORT}:${CASDOOR_PORT}' # Casdoor
      - '${LOBE_PORT}:3210' # LobeChat                                - '${LOBE_PORT}:3210' # LobeChat
    command: tail -f /dev/null                                      command: tail -f /dev/null
    networks:                                                       networks:
      - lobe-network                                                  - lobe-network

  postgresql:                                                     postgresql:
    image: pgvector/pgvector:pg16                                   image: pgvector/pgvector:pg16
    container_name: lobe-postgres                                   container_name: lobe-postgres
    ports:                                                          ports:
      - '5432:5432'                                                   - '5432:5432'
    volumes:                                                        volumes:
      - './data:/var/lib/postgresql/data'                             - './data:/var/lib/postgresql/data'
    environment:                                                    environment:
      - 'POSTGRES_DB=${LOBE_DB_NAME}'                                 - 'POSTGRES_DB=${LOBE_DB_NAME}'
      - 'POSTGRES_PASSWORD=${POSTGRES_PASSWORD}'                      - 'POSTGRES_PASSWORD=${POSTGRES_PASSWORD}'
    healthcheck:                                                    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U postgres']                   test: ['CMD-SHELL', 'pg_isready -U postgres']
      interval: 5s                                                    interval: 5s
      timeout: 5s                                                     timeout: 5s
      retries: 5                                                      retries: 5
    restart: always                                                 restart: always
    networks:                                                       networks:
      - lobe-network                                                  - lobe-network

  minio:                                                          minio:
    image: minio/minio                                              image: minio/minio
    container_name: lobe-minio                                      container_name: lobe-minio
    network_mode: 'service:network-service'                         network_mode: 'service:network-service'
    volumes:                                                        volumes:
      - './s3_data:/etc/minio/data'                                   - './s3_data:/etc/minio/data'
    environment:                                                    environment:
      - 'MINIO_API_CORS_ALLOW_ORIGIN=*'                               - 'MINIO_API_CORS_ALLOW_ORIGIN=*'
    env_file:                                                       env_file:
      - .env                                                          - .env
    restart: always                                                 restart: always
    entrypoint: >                                                   entrypoint: >
      /bin/sh -c "                                                    /bin/sh -c "
        minio server /etc/minio/data --address ':${MINIO_PORT           minio server /etc/minio/data --address ':${MINIO_PORT
        MINIO_PID=\$!                                                   MINIO_PID=\$!
        while ! curl -s http://localhost:${MINIO_PORT}/minio/           while ! curl -s http://localhost:${MINIO_PORT}/minio/
          echo 'Waiting for MinIO to start...'                            echo 'Waiting for MinIO to start...'
          sleep 1                                                         sleep 1
        done                                                            done
        sleep 5                                                         sleep 5
        mc alias set myminio http://localhost:${MINIO_PORT} $           mc alias set myminio http://localhost:${MINIO_PORT} $
        echo 'Creating bucket ${MINIO_LOBE_BUCKET}'                     echo 'Creating bucket ${MINIO_LOBE_BUCKET}'
        mc mb myminio/${MINIO_LOBE_BUCKET}                              mc mb myminio/${MINIO_LOBE_BUCKET}
        wait \$MINIO_PID                                                wait \$MINIO_PID
      "                                                               "

  casdoor:                                                        casdoor:
    image: casbin/casdoor                                           image: casbin/casdoor
    container_name: lobe-casdoor                                    container_name: lobe-casdoor
    entrypoint: /bin/sh -c './server --createDatabase=true'         entrypoint: /bin/sh -c './server --createDatabase=true'
    network_mode: 'service:network-service'                         network_mode: 'service:network-service'
    depends_on:                                                     depends_on:
      postgresql:                                                     postgresql:
        condition: service_healthy                                      condition: service_healthy
    environment:                                                    environment:
      RUNNING_IN_DOCKER: 'true'                                       RUNNING_IN_DOCKER: 'true'
      driverName: 'postgres'                                          driverName: 'postgres'
      dataSourceName: 'user=postgres password=${POSTGRES_PASS         dataSourceName: 'user=postgres password=${POSTGRES_PASS
      runmode: 'dev'                                                  runmode: 'dev'
    volumes:                                                        volumes:
      - ./init_data.json:/init_data.json                              - ./init_data.json:/init_data.json
    env_file:                                                       env_file:
      - .env                                                          - .env

  lobe:                                                           lobe:
    image: lobehub/lobe-chat-database                               image: lobehub/lobe-chat-database
    container_name: lobe-chat                                       container_name: lobe-chat
    network_mode: 'service:network-service'                         network_mode: 'service:network-service'
    depends_on:                                                     depends_on:
      postgresql:                                                     postgresql:
        condition: service_healthy                                      condition: service_healthy
      network-service:                                                network-service:
        condition: service_started                                      condition: service_started
      minio:                                                          minio:
        condition: service_started                                      condition: service_started
      casdoor:                                                        casdoor:
        condition: service_started                                      condition: service_started
    volumes:                                                  <
      - /etc/ssl/certs:/etc/ssl/certs:ro                      <

    environment:                                                    environment:
      - 'NEXT_AUTH_SSO_PROVIDERS=casdoor'                             - 'NEXT_AUTH_SSO_PROVIDERS=casdoor'
      - 'KEY_VAULTS_SECRET=Kix2wcUONd4CX51E/ZPAd36BqM4wzJgKjP         - 'KEY_VAULTS_SECRET=Kix2wcUONd4CX51E/ZPAd36BqM4wzJgKjP
      - 'NEXT_AUTH_SECRET=NX2kaPE923dt6BL2U8e9oSre5RfoT7hg'           - 'NEXT_AUTH_SECRET=NX2kaPE923dt6BL2U8e9oSre5RfoT7hg'
      - 'DATABASE_URL=postgresql://postgres:${POSTGRES_PASSWO         - 'DATABASE_URL=postgresql://postgres:${POSTGRES_PASSWO
      - 'S3_BUCKET=${MINIO_LOBE_BUCKET}'                              - 'S3_BUCKET=${MINIO_LOBE_BUCKET}'
      - 'S3_ENABLE_PATH_STYLE=1'                                      - 'S3_ENABLE_PATH_STYLE=1'
      - 'S3_ACCESS_KEY=${MINIO_ROOT_USER}'                            - 'S3_ACCESS_KEY=${MINIO_ROOT_USER}'
      - 'S3_ACCESS_KEY_ID=${MINIO_ROOT_USER}'                         - 'S3_ACCESS_KEY_ID=${MINIO_ROOT_USER}'
      - 'S3_SECRET_ACCESS_KEY=${MINIO_ROOT_PASSWORD}'                 - 'S3_SECRET_ACCESS_KEY=${MINIO_ROOT_PASSWORD}'
      - 'LLM_VISION_IMAGE_USE_BASE64=1'                               - 'LLM_VISION_IMAGE_USE_BASE64=1'
    env_file:                                                       env_file:
      - .env                                                          - .env
    restart: always                                                 restart: always

volumes:                                                        volumes:
  data:                                                           data:
    driver: local                                                   driver: local
  s3_data:                                                        s3_data:
    driver: local                                                   driver: local

networks:                                                       networks:
  lobe-network:                                                   lobe-network:
    driver: bridge                                                  driver: bridge

```

## 4. 结语

私有的 AI 助手一直是我的刚需，知识库功能也比较吸引人，接下来我会将我的文档导入其中来构造私有的个人 RAG 助手，以便以后总结使用。

