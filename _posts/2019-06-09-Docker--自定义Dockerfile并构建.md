---
layout:     post
title:      "Docker--自定义Dockerfile并构建"
subtitle:   "通过Dockerfile给官方centos镜像添加vim及ifconfig功能，并构建成自己的镜像。"
date:       2019-06-09
author:     "Woods"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Docker
    - 容器
    - 大数据
    
---

#### 1. 编写一个centos的Dockerfile
1. 指定登录默认路径为/home
2. 安装vim以支持vim编辑器
3. 安装net-tools以支持ifconfig
4. 暴露80端口
```
vim /home/docker/Dockerfile-centos01
```
```
FROM centos
MAINTAINER wsjwoods<wsjwoods@gmail.com>

ENV MYPATH /home
WORKDIR $MYPATH

RUN yum -y install vim 
RUN yum -y install net-tools

EXPOSE 80
CMD /bin/bash
```

#### 2. bulid构建Dockerfile
```
docker build -f /home/docker/Dockerfile-centos01 -t mycentos:1.2 .

```
等待完成。

查看images
```
docker images
```
```
REPOSITORY                 TAG                 IMAGE ID            CREATED             SIZE
mycentos                   1.2                 016b635f1349        26 seconds ago      447MB

```
出现了刚刚构建的mycentos镜像。

#### 3. 验证镜像
根据刚才构建的镜像创建容器
```
docker run -it mycentos:1.2
```
1. 落脚点是`/home`目录
```
[root@7f99ccdc581d home]#
```
2. vim命令

3. ifconfig命令 

都能正确使用。

自定义Dockerfile构建完成。

---
欢迎访问个人博客:[Woods Blog](https://wsjwoods.github.io/)