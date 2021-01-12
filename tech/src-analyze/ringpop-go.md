---
title: Ringpop-go
description: 
published: true
date: 2021-01-12T13:06:53.419Z
tags: distributed system, ringpop
editor: markdown
dateCreated: 2021-01-12T13:06:53.419Z
---

# Ringpop-go源码解析
> Ringpop-go<https://github.com/temporalio/ringpop-go>是Uber内部自己开源的分布式应用程序协调库，主要提供的功能：1. 在所有成员之上构建了一个一致性哈希环consistent hash ring 2. 依据此环提供了请求转发的路由功能 。如果了解过Cassandra等分布式系统的底层实现的话，就会发现这两个特征是如此熟悉，这是一种基本的shard方案。


