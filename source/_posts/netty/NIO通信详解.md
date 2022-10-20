---
title: NIO通信详解
date: 2022-09-06 09:22:00.194
categories: 
- Netty
tags: 
- netty
- 高并发
description: 高性能的Java通信，绝对离不开Java NIO技术，现在主流的技术框架或中间件服务器，都使用了Java NIO技术，譬如Tomcat、Jetty、Netty。
---

## NIO的核心组件

Java NIO由以下三个核心组件组成：

- Channel
- Buffer
- Selector

在开发Java应用程序时，有时候会涉及到文件的读取。读一个文件，需要先建立一个流（stream），单纯的通过这个流是可以读取文件数据的，但它只能顺序读，不能随意改变读取指针的位置。这种编程方式是面向流的IO。而NIO操作则不同，在NIO中引入了Channel和Buffer的概念，读取的时候从channel读取到缓冲区，写入的时候从缓冲区写入到channel，它可以随意的读取缓冲区中任意位置的数据。

