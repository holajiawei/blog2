---
title: Noobaa-Core源码分析
description: 
published: true
date: 2021-02-01T13:48:05.390Z
tags: distributed system, nooba
editor: markdown
dateCreated: 2021-01-31T15:59:17.940Z
---

# Noobaa-Core源码分析
> 最近看到一个很有意思的项目，Noobaa算是一个混合云存储的方案 <https://www.noobaa.io/>，支持S3 、S3兼容、自建对象存储等多云、多地域联合统一存储解决方案，内建了压缩、重复数据删除、加密、迁移等多种功能，这让我想起了之前一篇论文的标题"Software-Defined Object Storage“<https://ieeexplore.ieee.org/document/7436653>,软件定义对象存储真的太合适不过了

> 这个项目目前只有Core部分和部署用Operator开源了，文档几乎没有，所以最快速度摸清其中实现的方式就只有阅读源码了。
{.is-info}

## 代码整体分析
NoobaaCore的主要编程语言为NodeJS

- 源码文件目录结构 /src
	- agent
  - api
  - core
  - deploy
  - endpoint
  - hosted_agents
  - lambda_funs
  - native : 使用NodeJS N-API实现的原生交互接口
  	- aws-cpp-sdk: aws-s3 CPP-SDK测试用,未暴露公共接口
    - chunk: C++ 实现的Rabin分块算法
    - third_party: 引入的第三方库
    	- cm256: 一种Erasure Codec实现
      - **isa-l**: 宝藏c实现的低阶存储库，包含CRC校验、ErasureCodec、RAID奇偶异或计算、压缩解压缩实现
      - snappy: 一种压缩算法的实现
      - libutp: 基于UDP实现的用户态事件系统
    - n2n: 基于libuv的接口的简易封装
    