---
layout:     post
title:      "配置Hadoop,Hive的存储与压缩"
subtitle:   "Orc,Parquet等存储和压缩"
date:       2017-01-13
author:     "Woods"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Hadoop
    - Hive
---

## hadoop的压缩大体分为三个步骤：
###### 1.map阶段：
压缩文件通过split分片进入到maptask

所以压缩文件必须是支持分片的（text,lzo[index]）

###### 2.shuffle阶段
mapshuffle落地到磁盘时，选用压缩速度快的格式。

###### 3.reduce output阶段
分为两种场景：
一.reduce的输出作为下一个任务的输入，此时压缩文件最好采用支持分片的格式，或者保证output的每个文件不大于block块的大小（可以不支持分片）
二.reduce的输出文件作为归档使用，不在进行后续的计算，可以使用压缩比高的格式（Bzip2）




## 集群配置
core-site.xml
```
<property>
<name>io.compression.codecs</name>
<value>
org.apache.hadoop.io.compress.GzipCodec,
org.apache.hadoop.io.compress.DefaultCodec,
org.apache.hadoop.io.compress.BZip2Codec,
</value>
</property>	
```
mapred-site.xml	
```
<property>
<name>mapreduce.output.fileoutputformat.compress</name>
<value>true</value>
</property>

<property>
<name>mapreduce.output.fileoutputformat.compress.codec</name>
<value>org.apache.hadoop.io.compress.BZip2Codec</value>
</property>	
```

hive设置

```
SET hive.exec.compress.output=true;
SET mapreduce.output.fileoutputformat.compress.codec=org.apache.hadoop.io.compress.BZip2Codec;
```

## hive存储格式
hive file_format:
```
  : SEQUENCEFILE
  | TEXTFILE    -- (Default, depending on hive.default.fileformat configuration)
  | RCFILE      -- (Note: Available in Hive 0.6.0 and later)
  | ORC         -- (Note: Available in Hive 0.11.0 and later)
  | PARQUET     -- (Note: Available in Hive 0.13.0 and later)
```
SEQUENCEFILE：序列化  <key,value>形式，文件大小比文本文件大，一般不使用。

可以通过该网页查看[hive](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+ORC)压缩详细信息


###### orc格式

![orc](https://wsjwoods.github.io/img/in-post/OrcFile1.png)

列式储存，带索引。

创建一个orc格式的hive表
```
create table page_views_orc(
track_time string,
url string,
session_id string,
referer string,
ip string,
end_user_id string,
city_id string
) row format delimited fields terminated by '\t'
stored as orc ;
```

![orc](https://wsjwoods.github.io/img/in-post/OrcFile2.jpg)

orc.compress 默认使用zlib压缩

如果不使用压缩可以 `stored as orc tblproperties ("orc.compress"="NONE")`

###### PARQUET格式

设置parquet的压缩格式
```
set parquet.compression=gzip
```

Hive 0.13 and later
```
CREATE TABLE parquet_test (
 id int,
 str string,
 mp MAP<STRING,STRING>,
 lst ARRAY<STRING>,
 strct STRUCT<A:STRING,B:STRING>) 
PARTITIONED BY (part string)
STORED AS PARQUET;
```







