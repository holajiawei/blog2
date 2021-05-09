---
title: MySQL SQL优化器之Ranger
description: 
published: true
date: 2021-05-09T02:44:43.976Z
tags: mysql
editor: markdown
dateCreated: 2021-05-09T02:44:43.976Z
---

# MySQL查询优化-Range
Range优化过的请求只会使用单一索引来获取查询结果的子集，可能是使用索引的一部分;与之相对应的使用多个索引(注意区分索引的多个部分)的查询优化是Index_Merge（顾名思义，就是使用多个Range请求的结果集合并后获得）

- Range查询