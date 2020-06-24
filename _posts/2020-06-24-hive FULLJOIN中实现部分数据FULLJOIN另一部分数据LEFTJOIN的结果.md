---
layout:     post
title:      "hive FULLJOIN中实现部分数据FULLJOIN另一部分数据LEFTJOIN的结果"
subtitle:   "hive FULLJOIN中实现部分数据FULLJOIN另一部分数据LEFTJOIN的结果"
date:       2020-06-24
author:     "Woods"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Hive
---

### 需求
一个重点店铺表：dim_shop_point
一个用户对店铺关注表：follow_shop
一个近一年用户对店铺订单表：order_shop_year
全量店铺中有重点店铺和其他店铺
实现重点店铺的近一年订单数据及全量关注人群 + 非重点店铺关注人与近一年用户店铺订单的交集。


### 方案
###### 方案一:
使用订单表order_shop_year ,FULL JOIN上重点店铺的全量关注人群，再LEFT JOIN上非重点店铺的关注人群。
SQL:
```sql
select id,shop_id,is_follow,t1.order_info
from order_shop_year t1
full join 
(
	select id,shop,is_follow
	from follow_shop a 
	inner join dim_shop_point b 
	on a.shop=b.shop
) t2
on t1.id=t2.id and t1.shop=t2.shop
left join 
(
	select id,shop,is_follow
	from follow_shop a 
	left join dim_shop_point b 
	on a.shop=b.shop
	where b.shop is null
) t3
on t1.id=t3.id and t1.shop=t3.shop
```
此时发现follow_shop表和dim_shop_point 表都被读取了两次，而且job数是5。
有没有job数更少，表读取次数更少的方式呢。
###### 方案二:
给关注店铺用户打上标记，重点店铺标记1，非重点店铺标记0。最后订单表FULLJOIN关注人群，
此时有一下情况：
 1. 如果左右都匹配，标记设为1（不管是否是重点店铺）
 2. 如果左表有数，右表为null，表示这个人购买但是没关注，也保留，标记设为1
 3. 如果左表为null，右表有数，表示此人未购买但关注，此时判断此店铺是否为重点店铺，直接保留提前打上的标识数字即可。
最终结果过滤掉标记0的数据就是结果数据。
SQL:
```sql
select id,shop_id,is_follow,t1.order_info
from order_shop_year t1
full join 
(
	select id,shop,is_follow,if(b.shop is null,0,1) as flag
	from follow_shop a 
	left join dim_shop_point b 
	on a.shop=b.shop
) t2
on t1.id=t2.id and t1.shop=t2.shop
where if(t1.id is null,t2.flag,1) = 1
```
这样所有表都是读取了一次，而且job也减少到3个。





---
欢迎访问个人博客:[Woods Blog](https://wsjwoods.github.io/)

