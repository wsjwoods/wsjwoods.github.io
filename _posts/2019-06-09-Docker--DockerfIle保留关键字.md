---
layout:     post
title:      "Docker--DockerfIle保留关键字"
subtitle:   "常用的Dockerfile保留关键字介绍"
date:       2019-06-09
author:     "Woods"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Docker
    - 容器
    - 大数据
    
---

1. **FROM**   基础镜像，当前新镜像是基于哪个镜像的。
2. **MAINTAINER**   镜像维护者的姓名和邮箱地址
3. **RUN**  容器构建时需要运行的命令
4. **EXPOSE**   当前容器对外暴露出的端口号
5. **WORKDIR**  指定在创建容器后，终端默认登录的进来的工作目录，落脚点，默认是根目录`/`
6. **ENV**  用来在构建镜像过程中设置环境变量
7. **ADD**  将宿主机目录下的文件拷贝进镜像且ADD命令会自动处理URL和解压tar压缩包
8. **COPY** 类似ADD，拷贝文件和目录到镜像中。eg. `COPY src dest` or `COPY ["src","desc"]`
9. **VOLUME**   数据容器卷，用于数据保存和持久化工作。
10. **CMD** 指定一个容器启动时要运行的命令。DockerFile中可以有多个CMD指令，但只有最后一个生效，CMD会被`docker run` 之后的参数替换
11. **ENTRYPOINT**  指定一个容器启动时要运行的命令。目的和CMD一样，但`docker run` 之后的参数会追加而不是替换。
12. **ONBUILD** 当构建一个被继承的DockerFile时运行命令，父镜像在被子继承后父镜像的ONBUILD被触发


---
欢迎访问个人博客:[Woods Blog](https://wsjwoods.github.io/)