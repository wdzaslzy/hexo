---
title: HBase容灾之ClusterReplication实践
date: 2022-06-17 10:44:34
categories: [HBase调优]
tags: [hbase,性能调优]
description:本篇是基于ClusterReplication的理论基础，进行实践。理论基础请参考前文：{% post_link 'HBase容灾之ClusterReplication机制' %}
---

本篇是基于ClusterReplication的理论基础，进行实践。理论基础请参考前文：{% post_link 'HBase容灾之ClusterReplication机制' %}



## 异步串行复制

### 环境准备

准备两个集群，其中一个为主，一个为备。

| 节点        | 主备 |
| ----------- | ---- |
| 172.16.0.10 | 主   |
| 10.0.0.95   | 备   |

主集群有一张表：tsdb-pre-release，建表语句：create  'tsdb-pre-release', {NAME => 't', DATA_BLOCK_ENCODING => 'DIFF', COMPRESSION => 'SNAPPY'}

用该建表语句，在备集群建立一张一模一样的表。

![](../images/hbase/20220617111453.png)



将主集群中待复制的表列簇的REPLICATION_SCOPE设置为1。默认为0。

![](../images/hbase/20220617112513.jpg)

修改命令：`help 'alter'`查看使用帮助。

```shell
hbase> alter 'tsdb-pre-release', NAME => 't', REPLICATION_SCOPE => 1
```

![](../images/hbase/20220617134134.png)



### 开启异步复制

1. 为主备集群添加复制关系

```shell
hbase> add_peer '1', CLUSTER_KEY=> "10.0.0.97,10.0.0.98,10.0.0.99:2181:/tsdb-pre-release", STATE=> "DISABLED", TABLE_CFS => { "tsdb-pre-release" => []}, SERIAL=> true
```



2. 添加复制关系后，可以用list_peers查看

![](../images/hbase/20220617134539.jpg)



3. 确认没问题后，根据业务需求看是否将历史数据导入备集群

   如果需要导入历史数据，则通过HBase内置的ExportSnapshot工具进行导入。导入方式在后面介绍



4. 待数据导入完成后（或不需要导入数据时），打开peer

```shell
hbase> enable_peer '1'
```



5. 在备集群上查看是否有数据写入







## 同步复制

