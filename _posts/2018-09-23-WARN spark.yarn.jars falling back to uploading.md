---
layout:     post
title:      "WARN spark.yarn.jars falling back to uploading"
subtitle:   "spark on yarn 提交WARN"
date:       2018-09-23
author:     "Woods"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Spark
---

spark yarn模式的任务在启动时会出现如下警告，表示需要上传本地的spark jar包到hdfs上，这一步肯能非常耗时。
```
WARN Client: Neither spark.yarn.jars nor spark.yarn.archive is set, falling back to uploading libraries under SPARK_HOME
```

解决办法就是提前收到上传spark jar包到HDFS，然后修改spark-default.conf配置文件
```
hdfs dfs -mkdir /home/hadoop/spark_jars
hdfs dfs -put ${SPARK_HOME}/jars/* /home/hadoop/spark_jars/ 
```

然后在spark-default.conf添加一行
```
spark.yarn.jars=hdfs://hadoop001:8020/home/hadoop/spark_jars/*
```

在重新提交任务，就不会在出现WARN,并能够快速启动任务













