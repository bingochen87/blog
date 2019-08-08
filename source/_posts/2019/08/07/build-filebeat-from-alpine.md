---
title: 使用 Alpine 做基础镜像构建 Filebeat 7.1.0
tags: ["Filebeat", "Filebeat 7.1.0", "Docker", "alpine"]
categories: ["技术", "Docker"]
summary: 使用 Alpine 做基础镜像构建 Filebeat 7.1.0，以减少 Filebeat 的镜像大小
reward: true
---

## 前言

在处理 `map3-edge` 的 `filebeat` 基础镜像的时候，用的是 elastic 官方的 [elastic/filebeat:7.1.0](https://hub.docker.com/r/elastic/filebeat/)。镜像体积比较大，总大小为 288MB。实在有必要压缩一下，毕竟可以减少网络流量及存储空间。

## 优化

基本的优化处理就是用 `alpine` 做基础镜像去构建，elastic 官方的 `elastic/filebeat:7.1.0` 是基于 centos:7 做构建的，centos:7 这个镜像的大小就已经是 202MB。搜索了一下，网上的 filebeat alpine 都是在7.0以下的版本，并没有7.0以上的。elastic 官方也在 [Provide Alpine based images](https://github.com/elastic/beats-docker/issues/7) 这个 issues 里面明确表示不会提供 alpine 版本的 beat。那就只能是自己动手了。

优化处理，就是基于官方的 Dockerfile 将所有基于 centos 的处理，都替换成 alpine的。官方 [elastic/beats-docker](https://github.com/elastic/beats-docker) Dockerfile 示例：
```dockerfile
# This Dockerfile was generated from templates/Dockerfile.j2
FROM centos:7

RUN yum update -y && yum install -y curl && yum clean all

RUN curl -Lso - https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.1.0-linux-x86_64.tar.gz | \
      tar zxf - -C /tmp && \
    mv /tmp/filebeat-7.1.0-linux-x86_64 /usr/share/filebeat


ENV ELASTIC_CONTAINER true
ENV PATH=/usr/share/filebeat:$PATH

COPY config /usr/share/filebeat

# Add entrypoint wrapper script
ADD docker-entrypoint /usr/local/bin

# Provide a non-root user.
RUN groupadd --gid 1000 filebeat && \
    useradd -M --uid 1000 --gid 1000 --home /usr/share/filebeat filebeat

WORKDIR /usr/share/filebeat
RUN mkdir data logs && \
    chown -R root:filebeat . && \
    find /usr/share/filebeat -type d -exec chmod 0750 {} \; && \
    find /usr/share/filebeat -type f -exec chmod 0640 {} \; && \
    chmod 0750 /usr/share/filebeat/filebeat && \chmod 0770 modules.d && \
    chmod 0770 data logs
USER 1000


LABEL org.label-schema.schema-version="1.0" \
  org.label-schema.vendor="Elastic" \
  org.label-schema.name="filebeat" \
  org.label-schema.version="7.1.0" \
  org.label-schema.url="https://www.elastic.co/products/beats/filebeat" \
  org.label-schema.vcs-url="https://github.com/elastic/beats-docker" \
license="Elastic License"
ENTRYPOINT ["/usr/local/bin/docker-entrypoint"]
CMD ["-e"]
```

将所有 centos 的操作都替换成 alpine 后:
```dockerfile
# This Dockerfile was generated from templates/Dockerfile.j2
FROM alpine

RUN apk add --update --no-cache curl bash \
    && rm -rf /var/cache/apk/*
    
RUN curl -Lso - https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.1.0-linux-x86_64.tar.gz | \
    tar zxf - -C /tmp && \
    mv /tmp/filebeat-7.1.0-linux-x86_64 /usr/share/filebeat

RUN wget -q -O /etc/apk/keys/sgerrand.rsa.pub https://alpine-pkgs.sgerrand.com/sgerrand.rsa.pub \ 
      && wget https://github.com/sgerrand/alpine-pkg-glibc/releases/download/2.29-r0/glibc-2.29-r0.apk \ 
      && apk add glibc-2.29-r0.apk
      
ENV ELASTIC_CONTAINER true
ENV PATH=/usr/share/filebeat:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

COPY config /usr/share/filebeat

# Add entrypoint wrapper script
ADD docker-entrypoint /usr/local/bin
RUN chmod 755 /usr/local/bin/docker-entrypoint

# Provide a non-root user.
RUN addgroup -g 1000 -S filebeat && \
    adduser -u 1000 -G filebeat -h /usr/share/filebeat -S -D filebeat
WORKDIR /usr/share/filebeat
RUN mkdir data logs && \
    chown -R root:filebeat . && \
    find /usr/share/filebeat -type d -exec chmod 0750 {} \; && \
    find /usr/share/filebeat -type f -exec chmod 0640 {} \; && \
    chmod 0750 /usr/share/filebeat/filebeat && \chmod 0770 modules.d && \
    chmod 0770 data logs
USER filebeat

LABEL org.label-schema.schema-version="1.0" \
  org.label-schema.vendor="Elastic" \
  org.label-schema.name="filebeat" \
  org.label-schema.version="7.1.0" \
  org.label-schema.url="https://www.elastic.co/products/beats/filebeat" \
  org.label-schema.vcs-url="https://github.com/elastic/beats-docker" \
license="Elastic License"
ENTRYPOINT ["/usr/local/bin/docker-entrypoint"]
CMD ["-e"]
```

## 优化效果
![优化效果](https://bingozai.s3.ap-east-1.amazonaws.com/blog/2019/08/07/filebeat.png)

优化效果，非常明显，直接就减少了 184MB的大小。

## 遇到的坑
* 官方的示例里面没有处理 `/usr/local/bin/docker-entrypoint` 的运行权限问题，如果是没有加 `RUN chmod 755 /usr/local/bin/docker-entrypoint`，就会报 'standard_init_linux.go:175: exec user process caused "no such file or directory' 这个错。
* 因为从7.x开始，应该是用了 glibc 编译 filebeat，所以直接将 filebeat 下载到 alipne 里面运行，会报 `/usr/local/bin/docker-entrypoint: line 13: /usr/share/filebeat/filebeat: No such file or directory`。修复的办法就是在 Dockerfile 里面加入：
    ```dockerfile
RUN wget -q -O /etc/apk/keys/sgerrand.rsa.pub https://alpine-pkgs.sgerrand.com/sgerrand.rsa.pub \ 
    && wget https://github.com/sgerrand/alpine-pkg-glibc/releases/download/2.29-r0/glibc-2.29-r0.apk \ 
    && apk add glibc-2.29-r0.apk
    ```

