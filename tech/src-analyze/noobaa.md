---
title: Noobaa-Core源码分析
description: 
published: true
date: 2021-01-31T16:16:39.691Z
tags: distributed system, nooba
editor: markdown
dateCreated: 2021-01-31T15:59:17.940Z
---

# Noobaa-Core源码分析
> 最近看到一个很有意思的项目，Noobaa算是一个混合云存储的方案 <https://www.noobaa.io/>，支持S3 、S3兼容、自建对象存储等多云、多地域联合统一存储解决方案，内建了压缩、重复数据删除、加密、迁移等多种功能，这让我想起了之前一篇论文的标题"Software-Defined Object Storage“,软件定义对象存储真的太合适不过了

> 这个项目目前只有Core部分和部署用Operator开源了，文档几乎没有，所以最快速度摸清其中实现的方式就只有阅读源码了。
{.is-info}

## 代码整体分析
NoobaaCore的主要编程语言为NodeJS