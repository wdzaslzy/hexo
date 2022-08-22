---
title: HBase调优之五：分析opentsdb导致hbase整点GC很高问题
date: 2022-03-29 14:00:25.0
updated: 2022-03-29 14:54:48.889
url: /archives/opentsdb2
categories: 
- HBase调优
tags: 
- hbase
- 性能调优
description: 对读写缓存进行了分离，使用了堆外内存，读缓存不再往内存中写。整体gc有了一定的提升，但还是会有整点尖峰情况。查看HBase的读写请求，基本趋于稳定，所以整点的尖峰，猜测和调整HBase参数基本无关。
top: 9
---

# HBase gc优化第一版

<!--more-->

对读写缓存进行了分离，使用了堆外内存，读缓存不再往内存中写。整体gc有了一定的提升，但还是会有整点尖峰情况。

查看HBase的读写请求，基本趋于稳定，所以整点的尖峰，猜测和调整HBase参数基本无关。



下面是读写分离之后，datanode-7上gc的情况。

![1.png](../../images/1-f198c9659bed40159149627984847db7.png)

线上OpenTsdb get/delete 请求情况

![2.png](../../images/2-a27dd438450f4d76ada867912401f2e2.png)

![3.png](../../images/3-9188067aa7f0453484dcd6c06c5fbda5.png)

可以看到，它们都是整点的时候有一个小尖峰。

OpenTsdb整点在做compaction，会发出大量的get和delete请求。在整点时对HBase会有一定的冲击。



OpenTsdb官方提供了另一种写方式：append方式。

该方式的意图是：请求进来之后，不再创建新的列，而是追加在一个列上。这样也就不会再去做compaction操作。



# Release环境调整后的现象

![4.png](../../images/4-caf35a0c685544d68f9f4cce995865f2.png)

线下HBase GC调整前后对比

调整前：10号~11号

![5.png](../../images/5-98f52e5323354e9394775b2cbbd62ae3.png)

调整后：11号下午6点到12号

![6.png](../../images/6-8115ae8444a04fb4abefb5a97fa5cd0a.png)

gc明显平稳了很多，但是也有一些小的波动，原因是：HBase被多个环境引用，比如pre-release环境。pre-release环境也会发出一些compaction。



新老数据查询不受影响。

![7.png](../../images/7-187e9c55684f42be8210bbcbb0c26654.png)



![8.png](../../images/8-d68a5532a5ce4ada8a0d98fd0702222c.png)



参考文章：https://www.cnblogs.com/bigdatasafe/p/10524023.html



# 关闭Compaction和append

![9.png](../../images/9-9aa9bebd7a15472f830717025f72134d.png)

存储在HBase中，底层的StoreFile是多行，查询时，会增加IO，并且，多行存储，rowkey会重复，增加磁盘空间。

开启append模式后，查询情况：

![10.png](../../images/10-03f52e8de113407e89642c4f9f0b07f4.png)



# 减缓压缩速度

![11.png](../../images/11-625af44575b44b24b6f8238e820e3dcc.png)



![12.png](../../images/12-b19570af8c3241f19bb6fc9946e9d96f.png)



![13.png](../../images/13-4c99b7b42a744bad9eb38c199b8acb6d.png)