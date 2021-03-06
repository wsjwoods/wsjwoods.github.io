---
layout:     post
title:      "Docker部署"
subtitle:   "Docker部署"
date:       2019-06-04
author:     "Woods"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Docker

---


官网
https://docs.docker.com/install/linux/docker-ce/centos/#install-docker-ce-1

##### 1.卸载旧版本
```
$ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```              
##### 2.安装依赖
```
$ sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
```

添加docker源地址
```
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

##### 3.安装DOCKER CE
安装最新版的DOCKER CE
```
$ sudo yum install docker-ce docker-ce-cli containerd.io
```
启动docker
```
$ sudo systemctl start docker
```
运行hello-world测试docker是否安装完成
```
$ sudo docker run hello-world
```

```
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
1b930d010525: Pull complete 
Digest: sha256:0e11c388b664df8a27a901dce21eb89f11d8292f7fca1b3e3c4321bf7897bffe
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/

```
出现 ==Hello from Docker!== 表示成功。



---
欢迎访问个人博客:[Woods Blog](https://wsjwoods.github.io/)