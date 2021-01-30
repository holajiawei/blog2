---
title: 下一代Google File System(GFS)-Colossus
description: 
published: true
date: 2021-01-30T14:44:46.578Z
tags: gfs, google, colossus
editor: markdown
dateCreated: 2021-01-30T14:44:46.578Z
---

# 下一代Google File System(GFS)-Colossus
> GFS 那篇论文是分布式系统课程的经典案例，Hadoop HDFS可以算作是基于此论文的一种开源实现。GFS是支撑Google早期发展的核心系统。

第一次听到Colossus是在Jeff Dean那份关于分布式系统的经典PPT<https://static.googleusercontent.com/media/research.google.com/zh-CN//pubs/archive/44877.pdf>上看到的

![](https://javatuchuang.oss-cn-shanghai.aliyuncs.com/img/20210130220043.png)

然而全网关于此块的资料非常有限，目前搜集到整理如下:
- https://medium.com/@jerub/the-production-environment-at-google-8a1aaece3767
 	- 一名谷歌SRE工程师描写的目前谷歌生产环境里具体各个组件如何协同组织的
- https://levy.at/blog/22
  - 一位留学CMU大佬的博客文章，里面提到了他们存储课程的 Lecture Slides里面简单介绍了Colossus的架构，比较简单
- https://www.youtube.com/watch?v=q4WC_6SzBz4
  - 这个视频虽然主讲虚拟机的但是内部也简单介绍了一下Colossus以及其使用场景
  - 最重要的简单介绍了其架构设计
  - ![](https://javatuchuang.oss-cn-shanghai.aliyuncs.com/img/20210130221946.png)
  	- curators 是可以水平扩展的，主要承载来自colossus client各类操作，比如创建一个文件
    - metadata database是构建于可扩展NoSQL系统(类似BigTable)之上
    - custodians 便是用于做后置任务，比如维护文件持久性、磁盘剩余空间平衡、RAID重建等工作
	- ![](https://javatuchuang.oss-cn-shanghai.aliyuncs.com/img/20210130223842.png)