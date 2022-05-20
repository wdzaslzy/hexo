---
title: YARN组织架构
date: 2022-05-11 11:37:43.194
updated: 2022-05-11 11:37:43.194
url: /archives/yarnarchitecture
categories: 
- yarn详解
tags: 
- hadoop
- yarn
---

# **简介**

<!--more-->

yarn的核心思想是将集群资源进行管理。例如：任务的调度与监控。

为达到该目的，yarn提供了两个组件：ResourceManager 和 NodeManager。



其中，ResourceManager用来管理集群资源，NodeManager是每台机器的框架代理，负责监控每台机器的资源使用情况（cpu、内存、磁盘、网络），并将它监控的机器信息上报给ResourceManager。



当提交一个任务到yarn上时，需要在应用程序里面初始化一个ApplicationMaster与多个Container。ApplicationMaster是特定框架的库，目的是与ResourceManager资源进行协商，它会运行在NodeManager之上，监控整个任务运行情况。Container是一个作业运行的容器。



![img](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/NybEnB2kmZbPnP13/img/5668a817-7832-4fb2-9329-1b27ac2dbb7f.png) 



# **ResourceManager**

ResourceManager具有两个主要组件：Scheduler 和 ApplicationsManager

## **Scheduler**

在YARN中，资源调度器（Scheduler）是ResourceManager中的重要组件，主要负责对整个集群（CPU，内存）的资源进行分配和调度，分配资源以Container的形式分发到各个应用程序中（如MapReduce作业、Flink作业），应用程序与资源所在节点的NodeManager协作利用Container完成具体的任务（如Reduce Task）。



Scheduler是以插件的形式存在的，可以通过用户的需求来选择使用，也可以自己研发后交给YARN来管理。

YARN框架默认提供了3种Scheduler：

- FIFO Scheduler

先进先出，先提交的任务先执行

- Capacity Scheduler

以capacity为中心，把资源划分到若干个队列中，各个队列内根据自己的逻辑分配资源。通过队列来达到业务上的隔离。

- Fair Scheduler

公平原则。尽可能让每个任务得到的资源相同。例如：job1正在运行，job2提交后，如果无足够资源，job1会分配一部分资源给job2使用。

## **ApplicationsManager**

ApplicationsManager负责接收任务并提交，协商第一个容器来执行特定程序的ApplicationMaster（ApplicationMaster对于不同的计算引擎，实现都不同），并在任务提交失败时，进行重试。





# **NodeManager**

NodeManager负责单个机器的资源管理和监控，并定期上报给ResourceManager。同时接受ApplicationMaster的命令来启动Container。

在NodeManager上运行两类应用程序：ApplicationMaster 和 Container。

## **ApplicationMaster**

ApplicationMaster是一个应用程序的入口。它是应用级别的。由应用程序来定义该job需要多少资源，再由它主动向ResourceManager进行申请。一个应用程序提交后，会先分配一个Container启动ApplicationMaster，再由ApplicationMaster来申请需要工作的Container。

## **Container**

Container是YARN中资源的抽象。



# **YARN工作流程**

1. Client 向 YARN 提交应用程序，其中包括 ApplicationMaster 程序及启动 ApplicationMaster 命令
2. ResourceManager 为该 ApplicationMaster 分配第一个 Container，并与对应的 NodeManager 通信，要求它在这个 Container 中启动应用程序的 ApplicationMaster
3. ApplicationMaster 向ResourceManager 注册
4. ApplicationMaster 为 Application 的任务申请并领取资源
5. 获取到资源后，要求对应的 NodeManager 在 Container 中启动任务
6. NodeManager 收到 ApplicationMaster 的请求后，为任务设置好运行环境（包括环境变量、JAR 包等），将任务启动脚本写到一个脚本中，并通过运行该脚本启动任务
7. 各个任务通过 RPC 协议向 ApplicationMaster 汇报自己的状态和进度，以让 ApplicationMaster 随时掌握各个任务的运行状态，从而可以在失败时重启任务
8. 应用程序完成后，ApplicationMaster 向 ResourceManager 注销并关闭自己



实际中，集群可能并没有那么多资源来满足 ApplicationMaster 的资源请求，这时 ApplicationMaster 会采用轮循的方式不断申请资源，直到申请到资源或 Application 结束