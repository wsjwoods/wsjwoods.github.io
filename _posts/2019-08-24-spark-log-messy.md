---
layout:     post
title:      "Spark打印日志中文乱码问题"
subtitle:   "Spark打印日志中文显示乱码"
date:       2019-08-24
author:     "Woods"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Spark
    
---
###### 1. idea编辑器的编码

​	在idea右下角可以看到

###### 2. pom文件的编码

​    pom文件第一行`<?xml version="1.0" encoding="UTF-8"?>`

###### 3. pom文件的编码

​	spark提交时添加这两个参数：

​	--conf spark.driver.extraJavaOptions=" -Dfile.encoding=utf-8 " \

​	--conf spark.executor.extraJavaOptions=" -Dfile.encoding=utf-8 " \



---
欢迎访问个人博客:[Woods Blog](https://wsjwoods.github.io/)

