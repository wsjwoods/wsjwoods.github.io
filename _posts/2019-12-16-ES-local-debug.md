---
layout:     post
title:      "云主机自建ES6.x,本地外网Spark连接"
subtitle:   "云主机自建ES6.x,本地外网Spark连接调试"
date:       2019-12-16
author:     "Woods"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Spark
    - ES
---

##### 环境：

		1. 云主机自建elsticsearch6.5.4 (内网ip:192.168.0.3 外网ip:WAN_IP)
		2. Spark2.4



##### 开发：

官方开发文档：[https://www.elastic.co/guide/en/elasticsearch/hadoop/6.5/spark.html](https://www.elastic.co/guide/en/elasticsearch/hadoop/6.5/spark.html)

引入pom

```xml
   <dependency>
      <groupId>org.elasticsearch</groupId>
      <artifactId>elasticsearch-spark-20_2.11</artifactId>
      <version>6.5.4</version>
    </dependency>
```

```scala
val sparkConf = new SparkConf()
    sparkConf
       .set("es.nodes","WAN_IP")
       .set("es.port","9200")
       .set("es.index.auto.create", "true")
       .set("es.index.read.missing.as.empty","true")
      .setMaster("local[2]")
    val spark = SparkSession.builder()
      .config(sparkConf)
      .appName(this.getClass.getSimpleName)
//      .enableHiveSupport()
      .getOrCreate()

import org.elasticsearch.spark.sql._
import spark.implicits._

    //  create DataFrame
    import spark.implicits._
    val seq = Seq(("a",1),("b",2),("c",4))
    val people = spark.sparkContext.makeRDD(seq).toDF("name","age")
    people.show
	
	// 写
    people.saveToEs("spark/people")

    // 读
    spark.esDF("spark/people").show
```



此时windows的idea里执行代码会报错：

```java
[ERROR] 2019-12-17 10:05:54,025 method:org.elasticsearch.hadoop.rest.NetworkClient.execute(NetworkClient.java:147)
Node [192.168.0.3:9200] failed (The host did not accept the connection within timeout of 10000 ms); selected next node [WAN_IP:9200]
[ERROR] 2019-12-17 10:06:04,119 method:org.elasticsearch.hadoop.rest.NetworkClient.execute(NetworkClient.java:147)
Node [192.168.0.3:9200] failed (The host did not accept the connection within timeout of 10000 ms); no other nodes left - aborting...
Exception in thread "main" org.elasticsearch.hadoop.rest.EsHadoopNoNodesLeftException: Connection error (check network and/or proxy settings)- all nodes failed; tried [[192.168.0.3:9200]] 
	at org.elasticsearch.hadoop.rest.NetworkClient.execute(NetworkClient.java:152)
	at org.elasticsearch.hadoop.rest.RestClient.execute(RestClient.java:398)
	at org.elasticsearch.hadoop.rest.RestClient.executeNotFoundAllowed(RestClient.java:406)
	at org.elasticsearch.hadoop.rest.RestClient.exists(RestClient.java:503)
	at org.elasticsearch.hadoop.rest.RestClient.indexExists(RestClient.java:498)
	at org.elasticsearch.hadoop.rest.RestRepository.indexExists(RestRepository.java:336)
	at org.elasticsearch.hadoop.rest.RestService.findPartitions(RestService.java:228)
```



提示连接不上`192.168.0.3:9200` ,发现这个ip是es内网节点ip，但是我们本地只能连接他的外网ip。

想要使用ES_Spark包只通过外网IP连接ES集群，需要开启参数`es.nodes.wan.only=true`

代码修改为：

```scala
val sparkConf = new SparkConf()
    sparkConf
       .set("es.nodes","JD")
       .set("es.port","9200")
       .set("es.index.auto.create", "true")
       .set("es.index.read.missing.as.empty","true")
      .set("es.http.timeout","10000ms")
      .set("es.nodes.wan.only","true")   // 只启用节点外网，可以远程外网连接es集群
      .setMaster("local[2]")
    val spark = SparkSession.builder()
      .config(sparkConf)
      .appName(this.getClass.getSimpleName)
      .getOrCreate()
```



此时执行完美运行。

结果如下

```scala
+----+---+
|name|age|
+----+---+
|   a|  1|
|   b|  2|
|   c|  4|
+----+---+

+---+-----+
|age| name|
+---+-----+
|  2|    b|
|  4|    c|
| 28|woods|
|  1|    a|
+---+-----+

```





---
欢迎访问个人博客:[Woods Blog](https://wsjwoods.github.io/)

