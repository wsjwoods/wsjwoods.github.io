---
layout:     post
title:      "Presto读取MySQL数据"
subtitle:   "SQL中需要显示类型转换"
date:       2019-05-28
author:     "Woods"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Presto
    - 大数据
---

###### 环境
>Presto 0.216

###### 配置文件
```bash
[root@bigdata-003 catalog]# vi mysql.properties 
connector.name=mysql
connection-url=jdbc:mysql://bigdata-001:3306
connection-user=root
connection-password=root
```

###### SQL使用
```sql
select count(1)
from mpp.dw.rpt_cashequippedsaleback_info 
where t_date=cast('2019-05-22' as date)
```

由于Presto不支持自动类型转换。

SQL中`t_date` 和 `'2019-05-22'` 分别是date类型和varchar类型

如果不使用cast就行类型转换，会报错：
```
SQL 错误 [1]: Query failed (#20190528_083715_00149_gmav9): line 2:55: '=' cannot be applied to date, varchar(10)
```

需要将等号后面的值转成前面的date类型。

###### 注意：
如果cast写在等号前面，如：`cast(t_date as varcher)=2019-05-22'` </br>
presto加载数据会全量抽取数据在过滤，无法谓词下推。

---
欢迎访问个人博客:[Woods Blog](https://wsjwoods.github.io/)！