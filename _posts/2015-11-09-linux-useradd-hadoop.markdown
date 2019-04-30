---
layout:     post
title:      "linux新建用户应用于hadoop权限"
subtitle:   "给其他开发人员创建linux账号并只能操作hdfs指定目录下的文件"
date:       2015-11-09
author:     "Woods"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Linux
    - 大数据
    - Hadoop
---


创建用户组
`[root@vm ~]# groupadd usergroup1`
创建用户并指定组
`[root@vm ~]# useradd -d /usr/user1/ -m user1 -g usergroup1`
给用户设置密码
`[root@vm ~]# passwd user1`
`123456`


创建hdfs
```
hadoop fs -mkdir /user/user1
hadoop fs -chown -R user1:usergroup1 /user/user1
hadoop fs -ls /user
---
Found 2 items
drwx------   - hdfs   supergroup          0 2018-10-23 14:55 /user/hdfs
drwxr-xr-x   - user1 usergroup1           0 2018-10-23 14:51 /user/user1
```

user1用户只有操作/user/user1/ 目录的权限
