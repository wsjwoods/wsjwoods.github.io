---
layout:     post
title:      "HashShuffleManager测试shuffle阶段中间文件数量"
subtitle:   "分别测试hash、sort、hash+consolidateFiles"
date:       2017-10-13
author:     "Woods"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Spark
    
---

环境：
>spark 1.6.0 & 1.5.1

#### 1.HashShuffleManager
spark2.0之后就没有HashShuffleManager了，所以使用1.6版本来测试。

spark1.1之前一直是默认HashShuffleManager。

我们知道，spark中的HashShuffleManager，进行shuffle时，会生成大量的中间结果文件，频繁的创建文件会产生大量句柄，并产生大量IO操作。影响执行效率。

此shuffle过程每个task都会产生numReducers个文件，如果一个executor有M个task,经过shuffle阶段后R个reducers。那么当前executor会产生M*R个文件。如果一个节点运行E个executor，此节点上将会产生E*M*R个文件。

下面我们测试一下文件数到底是不是M*R个。我们以本地测试为例。
```java
val sparkConf = new SparkConf()
   sparkConf.setAppName(this.getClass.getName)
     .setMaster("local[2]")
     .set("spark.shuffle.manager","hash")
   val sc = new SparkContext(sparkConf)
val lines = sc.textFile("...",2)
```
设置spark.shuffle.manager = hash
读取文件的并行度设置为2，也就是maptask的数量等于2。
```java
val rdd = lines.flatMap(_.split("\t")).map((_,1)).reduceByKey(_+_,3)
```
这里reduceByKey的shuffle并行度设置为3.

```java
rdd.collect \\触发job
Thread.sleep(2000000) 	\\程序睡眠，查看shuffle中间文件数量，防止临时文件删除
```

日志信息
```
INFO DiskBlockManager: Created local directory at C:\Users\Administrator\AppData\Local\Temp\blockmgr-13afb58d-9521-4e71-8ebb-0d44f0e8bcb6
```
通过日志查看中间结果是存在了C:\Users\Administrator\AppData\Local\Temp\blockmgr-13afb58d-9521-4e71-8ebb-0d44f0e8bcb6文件夹下面。

通过windows的cmd命令查看该文件夹下有多少个文件。
```bash
C:\Users\Administrator> dir C:\Users\Administrator\AppData\Local\Temp\blockmgr-13afb58d-9521-4e71-8ebb-0d44f0e8bcb6\* /s 
```
![shuffleFile06](https://wsjwoods.github.io/img/in-post/shuffleFile01.png)
可以看到有6个文件。
正好等于 mapTask数量 * reduceTask数量  = 2*3 = 6。

下面修改一个reduceTask的数量看看是否和预期一样。
```java
val rdd = lines.flatMap(_.split("\t")).map((_,1)).reduceByKey(_+_,4)
```
这里修改reduceByKey的shuffle并行度为4.

在看一下文件数量。
![shuffleFile06](https://wsjwoods.github.io/img/in-post/shuffleFile02.png)

8个文件，正好等于 2*4.

#### 2.SortShuffleManager
如果改成 SortShuffleManager
```java
val sparkConf = new SparkConf()
   sparkConf.setAppName(this.getClass.getName)
     .setMaster("local[2]")
     .set("spark.shuffle.manager","sort")
   val sc = new SparkContext(sparkConf)
val lines = sc.textFile("...",2)
```
应该是 core * maptask = 2*2 = 4
测试 
![shuffleFile06](https://wsjwoods.github.io/img/in-post/shuffleFile03.png)
结果显示是 4个文件 = 2*2

改成val lines = sc.textFile("...",3)
应该是 2*3 = 6
测试
![shuffleFile06](https://wsjwoods.github.io/img/in-post/shuffleFile04.png)
结果显示 6个文件 = 2*3


#### 3.HashShuffleManager + consolidateFiles
spark.shuffle.consolidateFiles这个参数在1.6版本之后被移除了
所以使用1.5.1的版本来测试
```java
val sparkConf = new SparkConf()
sparkConf.setAppName(this.getClass.getName)
  .setMaster("local[2]")
  .set("spark.shuffle.manager","hash")   
  .set("spark.shuffle.consolidateFiles","true")  \\1.5及之前有这个参数
val sc = new SparkContext(sparkConf)

val lines = sc.textFile("...",3)

val rdd = lines.flatMap(_.split("\t")).map((_,1)).reduceByKey(_+_,5)
```
读取文件的并行度设置为3，也就是maptask的数量等于3.

这里reduceByKey的shuffle并行度设置为5.

开启了map端输出文件合并功能，理论上中间文件数量应该是core * R = 2*5 = 10
测试
![shuffleFile06](https://wsjwoods.github.io/img/in-post/shuffleFile05.png)
结果显示 10个文件 = 2*5


修改 core数 = 3
setMaster("local[3]")
理论上文件数 = 3*5 = 15

测试
![shuffleFile06](https://wsjwoods.github.io/img/in-post/shuffleFile06.png)
结果显示 15个文件 = 3*5


---
欢迎访问[个人博客](https://wsjwoods.github.io/)！
