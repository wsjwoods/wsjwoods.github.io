---
layout:     post
title:      "安装Python3并包含sqlite3"
subtitle:   "python3支持sqlite3"
date:       2019-05-30
author:     "Woods"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Python
    
---



#### 下载sqlite3的包
```bash
wget https://www.sqlite.org/2017/sqlite-autoconf-3170000.tar.gz --no-check-certificate
tar zxvf sqlite-autoconf-3170000.tar.gz
cd sqlite-autoconf-3170000
./configure --prefix=/usr/local/sqlite3 --disable-static --enable-fts5 --enable-json1 CFLAGS="-g -O2 -DSQLITE_ENABLE_FTS3=1 -DSQLITE_ENABLE_FTS4=1 -DSQLITE_ENABLE_RTREE=1"
```
#### 下载Python3
```bash
wget https://www.python.org/ftp/python/3.6.1/Python-3.6.1.tgz
```
下载其他版本请去 https://www.python.org/downloads/

#### 编译Python3并包含sqlite3
1. 在进行解压之前先创建一个解压目录：
```bash
mkdir -p /usr/local/python3
```

2. 接着把刚才下载的Python3.6.1安装包解压在该目录下：
```bash
tar -zxvf Python-3.6.1.tgz -C /usr/local/python3
```

3. 先进入到刚才解压的目录：
```bash
cd /usr/local/python3/Python-3.6.1
```

4. 编译python3

配置一下安装目录/usr/local/python3，这样做的好处是下次想卸载软件直接卸载该目录下的就可以了.

LD_RUN_PATH:指定要在链接和运行时搜索库的目录
```bash
LD_RUN_PATH=/usr/local/sqlite3/lib ./configure --prefix=/usr/local/python3 LDFLAGS="-L/usr/local/sqlite3/lib" CPPFLAGS="-I /usr/local/sqlite3/include"
LD_RUN_PATH=/usr/local/sqlite3/lib make
LD_RUN_PATH=/usr/local/sqlite3/lib sudo make install
```

#### 配置
1. PYTHON_HOME添加进环境变量
```bash
vi /etc/profile
export PYTHON_HOME=/usr/local/python3
export PATH=$PYTHON_HOME/bin:$PATH
```

2. 修改默认python问python3

添加软连接
```bash
ln -s /usr/local/python3/bin/python3 /usr/bin/python
```

3. 修改yum文件，解决python冲突问题

没想到最后python3与python2冲突了，导致yum不能用了,不过修改一下
```bash
vi /usr/bin/yum
vi /usr/libexec/urlgrabber-ext-down
```
这两个文件开头的python为python2.7即可


参考：
>https://www.cnblogs.com/loveapple/p/9937030.html
https://blog.csdn.net/zhangdongren/article/details/82685932

---
欢迎访问个人博客:[Woods Blog](https://wsjwoods.github.io/)
