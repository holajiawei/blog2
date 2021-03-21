---
title: MySQL InnoDB引擎事务锁机制一图解
description: 
published: true
date: 2021-03-21T08:45:42.227Z
tags: database, mysql
editor: markdown
dateCreated: 2021-03-21T08:45:42.227Z
---

# MySQL InnoDB引擎事务锁机制一图解

## 锁的分类

### 按照锁的用途来分
在行级别进行读写控制，需要以下
- 共享锁 Share S锁
- 互斥锁 Exclusive X锁

除此之外InnoDB还实现了表级别的意图读写锁，分别是IS和IX锁，这两个都是Table来操作，使用流程如下：
- 在获取S锁之前需先获得对应表的IS锁
- 在获得X锁之前需要先获得对应表的IX锁

