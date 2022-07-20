---
title: HBase容灾之ClusterReplication实践
date: 2022-06-17 10:44:34
categories: [HBase调优]
tags: [hbase,性能调优]
description:本篇是基于ClusterReplication的理论基础，进行实践。理论基础请参考前文：{% post_link 'HBase容灾之ClusterReplication机制' %}
top: 9
---

本篇是基于ClusterReplication的理论基础，进行实践。理论基础请参考前文：{% post_link 'HBase容灾之ClusterReplication机制' %}



## 异步串行复制

### 环境准备

准备两个集群，其中一个为主，一个为备。

- 主备集群版本必须要一致
- 主备集群能够互相通信，且时钟要同步



**在主集群上开启replicat功能**

```xml
<property>
    <name>hbase.replication</name>
    <value>true</value>
    <description>打开replication功能</description>
</property>
```



目前我已准备了两套HBase集群，版本：2.1.0

| 节点        | 主备 |
| ----------- | ---- |
| 172.16.0.10 | 主   |
| 10.0.0.95   | 备   |



**第一步：在主/备集群创建一张测试表**

在主集群创建一张测试表：t_master，建表语句：

```shell
hbase> create 't_master', {NAME => 't', DATA_BLOCK_ENCODING => 'DIFF', COMPRESSION => 'SNAPPY'}
```

用该建表语句，在备集群建立一张一模一样的表。



**第二步：分别在主备集群上put一条数据**

主机群：

```shell
hbase> put 't_master', 'lizy', 't', '28'
```

![](../../images/hbase/20220711151557.png)

备集群：

```shell
hbase> put 't_master', 'ly', 't', '35'
```

![](../../images/hbase/20220711151708.png)



### 开启异步复制

1. 将主集群中待复制的表列簇的REPLICATION_SCOPE设置为1。默认为0。

![](../../images/hbase/20220617112513.jpg)

修改命令：`help 'alter'`查看使用帮助。

```shell
hbase> alter 't_master', NAME => 't', REPLICATION_SCOPE => 1
```

![](../../images/hbase/20220617134134.png)

2. 为主备集群添加复制关系

   cluster_key为备集群的zk地址

```shell
hbase> add_peer '1', CLUSTER_KEY=> "10.0.0.97,10.0.0.98,10.0.0.99:2181:/hbase/t_master", STATE=> "DISABLED", TABLE_CFS => { "t_master" => []}, SERIAL=> true
```



3. 添加复制关系后，可以用list_peers查看

![](../../images/hbase/20220617134539.jpg)



4. 确认没问题后，根据业务需求看是否将历史数据导入备集群

如果需要导入历史数据，则通过HBase内置的ExportSnapshot工具进行导入。导入方式在后面介绍



5. 向主集群写入一条数据

   ```shell
   hbase> put 't_master', 'yz', 't', '18'
   ```

   分别查看主备集群的数据。此时：主集群有两条数据，备集群有一条数据（环境准备时写入的）。

   ![](../../images/hbase/20220711163720.png)



6. 待数据导入完成后（或不需要导入数据时），打开peer

```shell
hbase> enable_peer '1'
```



7. 在主集群上再写入一条数据

   ```shell
   hbase> put 't_master', 'wl', 't', '35'
   ```

   

   







## 同步复制

