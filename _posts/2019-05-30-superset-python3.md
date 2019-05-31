---
layout:     post
title:      "Superset部署，基于Python3"
subtitle:   "Superset踩坑指南"
date:       2019-05-30
author:     "Woods"
header-img: "img/post-bg-js-version.jpg"
tags:
    - BI
    - 大数据
    
---

### 环境：
>python3</br>
>centos 7.4

### 部署：
官网 [http://superset.apache.org/installation.html](http://superset.apache.org/installation.html#getting-started)

1. 安装python 和 其他依赖
```bash
sudo yum upgrade python-setuptools
sudo yum install gcc gcc-c++ libffi-devel python-devel python-pip python-wheel openssl-devel libsasl2-devel openldap-devel
```

建议在virtualenv中安装superset,Python 3已经发布了virtualenv。但是如果由于某些原因它没有安装在您的环境中，您可以通过您的操作系统的包安装它，否则您可以从pip安装:
```bash
pip install virtualenv

python3 -m venv venv
. venv/bin/activate
```

一旦您激活了virtualenv，您所做的一切都将被限制在virtualenv中。要退出virtualenv，只需输入`deactivate`

2. setuptools pip升级至最新版
```bash
pip install --upgrade setuptools pip
```

3. 安装Superset
```bash
pip install superset
```

4. 注册admin用户
```bash
fabmanager create-admin --app superset
```

如果报错：
```
No module named '_sqlite3'
```
问题原因：

python3 缺少_sqlite3

解决办法:

需要重新编译包含sqlite3的python3,见[安装Python3并包含sqlite3](https://wsjwoods.github.io/2019/05/30/python3-install/)


然后可能又会报错：
```
Was unable to import superset Error: cannot import name '_maybe_box_datetimelike'
```

问题原因:


这是 pandas 库版本太高导致的，需要安装低版本的 pandas 库。

解决办法:

查看当前 pandas 版本
```bash
pip list | grep pandas
pandas  0.24.2
```

安装低版本 pandas
```bash
pip install pandas==0.23.4
```
然后重新运行 `fabmanager create-admin --app superset` 命令创建管理员用户。

5. 初始化数据库
```bash
superset db upgrade
```
初始化数据库时报错：
```
sqlalchemy.exc.InvalidRequestError: 
Can't determine which FROM clause to join from, there are multiple FROMS which can join to this entity. 
Try adding an explicit ON clause to help resolve the ambiguity.
```
问题原因:

这是 SQLAlchemy 库版本太高导致的，需要安装低版本的 SQLAlchemy 库。

解决办法:

查看当前 SQLAlchemy 版本
```bash
pip list | grep -i sqlalchemy
Flask-SQLAlchemy 2.3.2   
SQLAlchemy       1.3.2   
SQLAlchemy-Utils 0.33.11
```
安装低版本 SQLAlchemy
```bash
pip install SQLAlchemy==1.2.18
```
然后重新运行`superset db upgrade`命令初始化数据库。




6. 加载样例数据
```bash
superset load_examples
```

7. 创建默认的角色和权限
```bash
superset init
```

8. 启动web server, 默认端口8088,使用 -p 绑定其他端口
```bash
superset runserver -d
or
superset runserver -d -p 8089
```

### 登录
可以使用 http://localhost:8088登录你的superset.


### 配置superset数据源依赖
 database | pypi package | SQLAlchemy URI prefix 
 :-: | :-: | :-:  
 MySQL | pip install mysqlclient | mysql://
 Postgres | pip install psycopg2 | postgresql+psycopg2://
 Presto | pip install pyhive | presto://
 Hive | pip install pyhive | hive://
 Oracle | pip install cx_Oracle | oracle://
 sqlite |  | sqlite://
 Snowflake | pip install snowflake-sqlalchemy | snowflake://
 Redshift | pip install sqlalchemy-redshift | redshift+psycopg2://
 MSSQL | pip install pymssql | mssql://
 Impala | pip install impyla | impala://
 SparkSQL | pip install pyhive | jdbc+hive://
 Greenplum | pip install psycopg2 | postgresql+psycopg2://
 Athena | pip install "PyAthenaJDBC>1.0.9" | awsathena+jdbc://
 Athena | pip install "PyAthena>1.2.0" | awsathena+rest://
 Vertica | pip install sqlalchemy-vertica-python | vertica+vertica_python://
 ClickHouse | pip install sqlalchemy-clickhouse | clickhouse://
 Kylin | pip install kylinpy | kylin://
 BigQuery | pip install pybigquery | bigquery://
 Teradata | pip install sqlalchemy-teradata | teradata://
 Pinot | pip install pinotdb | pinot+http://controller:5436/ query?server=http://controller:5983/


1. 安装mysql依赖
```bash
pip install mysqlclient  
```

如果报错：
```
Collecting mysqlclient
  Downloading http://mirrors.aliyun.com/pypi/packages/f4/f1/3bb6f64ca7a429729413e6556b7ba5976df06019a5245a43d36032f1061e/mysqlclient-1.4.2.post1.tar.gz (85kB)
    100% |████████████████████████████████| 92kB 5.9MB/s 
    Complete output from command python setup.py egg_info:
    sh: mysql_config: command not found
    Traceback (most recent call last):
      File "<string>", line 1, in <module>
      File "/tmp/pip-build-hqrD6X/mysqlclient/setup.py", line 16, in <module>
        metadata, options = get_config()
      File "setup_posix.py", line 51, in get_config
        libs = mysql_config("libs")
      File "setup_posix.py", line 29, in mysql_config
        raise EnvironmentError("%s not found" % (_mysql_config_path,))
    EnvironmentError: mysql_config not found
    
    ----------------------------------------
Command "python setup.py egg_info" failed with error code 1 in /tmp/pip-build-hqrD6X/mysqlclient/
```


去mysqlclient-Github查看https://github.com/PyMySQL/mysqlclient-python
```
Prerequisites
You may need to install the Python and MySQL development headers and libraries like so:

sudo apt-get install python-dev default-libmysqlclient-dev # Debian / Ubuntu
sudo yum install python-devel mysql-devel # Red Hat / CentOS
brew install mysql-connector-c # macOS (Homebrew) (Currently, it has bug. See below)
On Windows, there are binary wheels you can install without MySQLConnector/C or MSVC.
```

需要先
```bash
yum install python-devel mysql-devel
```

然后在
```bash
pip install mysqlclient
```

如果报错:
```
ERROR: Complete output from command python setup.py egg_info:
    ERROR: /bin/sh: mysql_config: command not found
```
需要安装mariadb-devel
```bash
yum install mariadb-devel
```

2. 安装pyhive
```bash
pip install pyhive
```
之后就可以支持Presto、Hive、SparkSQL等数据源了。

---
欢迎访问个人博客:[Woods Blog](https://wsjwoods.github.io/)




