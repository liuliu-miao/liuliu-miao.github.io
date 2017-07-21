---
layout: post
title: docker学习笔记(二)Dockerfile
categories: docker
description: docker的安装,helloworld,以及常用命令
keywords: docker install,docker helloworld
---

## docker学习笔记二--Dockerfile
---
参考自[官方文档-Dockerfile](https://docs.docker.com/engine/reference/builder/#usage)部分

### (一)Dockerfile中常用的指令简介
``` bash
FROM #来源于某个基础镜像
FROM ubuntu 16.04


LABEL
ARG SPRING_PROFILE_ACTIVE #获取外部参数 SPRING_PROFILE_ACTIVE为外部参数的名称
RUN  #运行镜像中系统级的命令
RUN apt-get uppdate && apt-get install -y 

RUN apt-get update && apt-get install -y \
    aufs-tools \
    automake \
    build-essential \
    curl \
    dpkg-sig \
    libcap-dev \
    libsqlite3-dev \
    mercurial \
    reprepro \
    ruby1.9.1 \
    ruby1.9.1-dev \
    s3cmd=1.1.* \
 && rm -rf /var/lib/apt/lists/*

RUN ["/bin/bash", "-c", "set -o pipefail && wget -O - https://some.site | wc -l > /number"]

CMD  #运行非系统级别的命令
CMD ["perl", "-de0"]
ENV  #设置系统参数
ADD or COPY 

COPY /home/userName/xxx.jar  /images/  #讲系统中的内容copy到镜像中的指定目录
ENV profile ${SPRING_PROFILE_ACTIVE} #设置参数  ${SPRING_PROFILE_ACTIVE}为外部传入参数
CMD [bash,-c,java -jar /xxx.jar --spring.profiles.active=${profile}]

(/home/userName/xxx.jar为当前系统目录下的内容) 目标地址为镜像目录 /images/
 
EXPOSE  # 不常用,后续补充该字段
ENTRYPOINT
VOLUME
USER
WORKDIR
ONBUILD

```
### (二)一些简单的Dockerfile示例
> 以下为一个简介版本的Dockerfile
``` bash
FROM openjdk:8
ARG SPRING_PROFILE_ACTIVE
 
ENV SPRING_PROFILE_ACTIVE ${SPRING_PROFILE_ACTIVE}

COPY demo-www-1.0-SNAPSHOT.jar  /temp/

CMD [bash,-c,java -jar   /temp/demo-www-1.0-SNAPSHOT.jar --spring.profiles.active=${SPRING_PROFILE_ACTIVE}] 
```
> 项目中使用的镜像
``` bash 
# 基于一个简介版本的可运行jdk环境的docker镜像
FROM reg.longdai.com/base/openjdk:8u131-jdk-alpine

# 作者信息
MAINTAINER mritd <mritd@mritd.me>

# 获取参数
ARG SPRING_PROFILE_ACTIVE
ARG PROJECT_BUILD_FINALNAME

# 设置时区,项目参数,项目名称
ENV TZ 'Asia/Shanghai'
ENV SPRING_PROFILE_ACTIVE ${SPRING_PROFILE_ACTIVE}
ENV PROJECT_BUILD_FINALNAME ${PROJECT_BUILD_FINALNAME}

# 运行命令设置和安装 搭建项目所需要环境
RUN apk upgrade --update \
    && apk add tzdata bash \
    && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && echo "Asia/Shanghai" > /etc/timezone \
    && rm -rf /var/cache/apk/*

# copy 项目中可运行的jar包到镜像中
COPY target/${PROJECT_BUILD_FINALNAME}.jar /${PROJECT_BUILD_FINALNAME}.jar

# 执行启动命令,项目使用spring-boot框架搭建,所以直接使用此命令就好
CMD ["bash","-c","java -jar /${PROJECT_BUILD_FINALNAME}.jar --spring.profiles.active=${SPRING_PROFILE_ACTIVE}"]

# 更复杂的Dockerfile待更新
``` 

### (三)build Dcokerfile
写完Dockerfile后,使用docker build 命令build自己的镜像
``` sh
docker build /path/to/a/Dockerfile  -t testDocker:1.2 .

docker -H https://githu.com/xxx/xxx.git  build -t image_name:1.12 --build-arg SPRING_PROFILE_ACTIVE=dohko   PROJECT_BUILD_FINALNAME=longdai-p2p-core longdai-core

# 为镜像指定tag
docker -H $DOHKO_BUILD_HOST_19 tag $IMAGE_NAME $LATEST_IMAGE_NAME

# 将镜像push到指定仓库
docker -H $DOHKO_BUILD_HOST_19 push $IMAGE_NAME

# 删除本地的docker镜像
docker -H $DOHKO_BUILD_HOST_19  rmi $IMAGE_NAME

# 指定build的文件来源地址
-H https://githu.com/xxx/xxx.git 
# build后的镜像名称为: image_name:1.12 
build -t image_name:1.12 
# build 时带入的参数
--build-arg SPRING_PROFILE_ACTIVE=dohko   PROJECT_BUILD_FINALNAME=longdai-p2p-core
# build的文件中的目录
longdai-core



```
更多详细的命令参考[官方文档Dockerfile reference](https://docs.docker.com/engine/reference/builder/)

