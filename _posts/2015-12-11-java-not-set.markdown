---
layout:     post
title:      "报错：JAVA_HOME is not set"
subtitle:   "sudo 无法提交任务"
date:       2015-12-11
author:     "Woods"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Linux
    - 大数据
    - Hadoop
---
使用sudo -u hdfs 命令执行脚本时报错


![javahome-not-set](/img/in-post/javahome-not-set.png)
执行  `sudo -u hdfs spark-shell`
报错 JAVA_HOME is not set


解决方法：
修改linux系统environment
```vim /etc/environment```
添加一行：
```JAVA_HOME=/opt/jdk```
