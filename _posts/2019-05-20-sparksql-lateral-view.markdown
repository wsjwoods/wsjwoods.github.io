---
layout:     post
title:      "sparksql 数据按逗号拆分成多行"
subtitle:   "lateral view explode()实现拆分多行"
date:       2019-05-20
author:     "Woods"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Spark
    - Sql

---
    

比如：

原表(表名：table1)

 id | num 
 :-: | :-: |
 1 | 001,002,003 |
 2 | 001,002 |


转换成

 id | num 
 :-: | :-: 
 1 | 001 |
 1 | 002 |
 1 | 003 |
 2 | 001 |
 2 | 002 |


使用`lateral view explode()`语法

使用方法是:
```
select id,num_per
from table1 
lateral view explode(split(num, ',')) tmpTable as num_per
where xx=xx
```
注意：

###### 1. where条件需写到他的后面。
###### 2. 这里的==tmpTable==指代虚表视图的名称(不可缺少），可以随意命名。
###### 3. as num_per 表示将拆的列命名，在select中使用。











