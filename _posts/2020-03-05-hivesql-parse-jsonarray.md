---
layout:     post
title:      "hivesql解析json数组并拆分成多行"
subtitle:   "hivesql解析json数组并拆分成多行"
date:       2020-03-05
author:     "Woods"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Hive
---

##### 原始数据：

		[{"name":"woods","app_id":"abc123"},{"name":"tiger","app_id":"def456"}]

数据存在表`dev.woods_test`中



##### 需求与方法：


1. 解析json，一行拆分成两行

```sql
select a_json
from
(
select split(regexp_replace(regexp_extract (json_col ,'(\\[)(.*?)(\\])',2),'\\},\\{','\\}|\\{'),'\\|') as a_list
from dev.woods_test
) a
lateral view explode(a_list) a_list_tab as a_json
```

结果：

```
{"name":"woods","app_id":"abc123"}
{"name":"tiger","app_id":"def456"}
```



2. 解析上述结果json,取出name和app_id值

```sql
select 
get_json_object(a_json,'$.name') as name,
get_json_object(a_json,'$.app_id') as app_id
from
(
select split(regexp_replace(regexp_extract (json_col ,'(\\[)(.*?)(\\])',2),'\\},\\{','\\}|\\{'),'\\|') as a_list
from dev.woods_test
) a
lateral view explode(a_list) a_list_tab as a_json
```

结果：

```
woods,abc123
tiger,def456
```







---
欢迎访问个人博客:[Woods Blog](https://wsjwoods.github.io/)

