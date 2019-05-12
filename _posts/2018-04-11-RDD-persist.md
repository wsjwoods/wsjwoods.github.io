---
layout:     post
title:      "RDD缓存及序列化缓存"
subtitle:   "JavaSerializer和KryoSerializer对比"
date:       2018-04-11
author:     "Woods"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Spark

=======
    

#### 1. MEMORY_ONLY and not Serializer
![memory_only](https://wsjwoods.github.io/img/in-post/memory_only.png)

使用MEMORY_ONLY 方式缓存，RDD占用内存的大小为22.6M。

#### 2.MEMORY_ONLY_SER and JavaSerializer
![memory_only_javaser](https://wsjwoods.github.io/img/in-post/memory_only_javaser.png)
使用MEMORY_ONLY_SER 并且序列化方式为默认的JavaSerializer方式缓存，RDD占用内存的大小为5.7M。

#### 3.MEMORY_ONLY_SER and KryoSerializer and not registerKryoClasses
![memory_only_kryoser_not_reg](https://wsjwoods.github.io/img/in-post/memory_only_kryoser_not_reg.png)
使用MEMORY_ONLY_SER 并且序列化方式为未注册的KryoSerializer 方式缓存，RDD占用内存的大小为9.7M。

可见未注册的KryoSerializer 占用内存比JavaSerializer还要大一些

#### 4.MEMORY_ONLY_SER and KryoSerializer and registerKryoClasses
![memory_only_kryoser_reg](https://wsjwoods.github.io/img/in-post/memory_only_kryoser_reg.png)

使用MEMORY_ONLY_SER 并且序列化方式为注册的KryoSerializer 方式缓存，RDD占用内存的大小为3.8M。

通过以上测试可以发现，注册的KryoSerializer 序列化方式所占内存是最少的，也是推荐官方来使用的。










