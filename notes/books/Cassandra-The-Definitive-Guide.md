---
title: Cassandra: The Definitive Guide, 2nd Edition 浓缩精华
description: 
published: true
date: 2020-08-11T03:22:24.973Z
tags: database, cassandra
editor: markdown
---

# Cassandra: The Definitive Guide, 2nd Edition 浓缩精华

> 貌似第二版(出版于2016年)国内只有英文影印版，还没有中文版。我自己是在SafariBooksOnline看的，实在是受制于SBO的累赘笔记功能，所以直接整理成读书笔记了
{.is-success}


##  第一章Beyond Relational Databases

第一章算是个历史课，讲述了伴随着时代的发展，关系型数据库击败IBM IMS走上神坛的故事，后面随着互联网数据的高速发展和膨胀，传统关系型数据库可能并不适合了，然后引出了非关系数据库NoSQL，然后列举了一些非关系数据库的长处。【书写成于2015、2016年吧，现在的时代NewSQL的概念又开始活跃起来了，当然是针对传统SQL关系型数据库的弊病比如事务支持弱、运维成本高等问题重拳出击啦。无论怎样，热烈欢迎新技术的流行，只要有存在的价值，就有学习的意义。我能学的动！:smile:】

## 第二章 Introducing Cassandra

### 特性

用一种电梯推介法<https://en.wikipedia.org/wiki/Elevator_pitch>来简单介绍了Cassandra的优势特性，然后又分开细谈了一下，主要有以下特性:

- Distributed and Decentralized
- Elastic Scalability
- High Availability and Fault Tolerance
- Tuneable Consistency
- Row-Oriented
- High Performance

### 适用场景

- 需要大规模部署
- 应用程序/数据不断增大
- 大量的写入、分析、统计场景
- 多数据中心/基于地理位置的部署模式

## 第三章 Installing Cassandra

没什么意思，略过

## 第四章 The Cassandra Query Language

### 数据模型 

本章在正式介绍Cassandra的查询语言CQL之前先介绍了Cassandra的数据模型，与传统关系型数据库的数据模型还是有很大差异的。

Cassandra的每一Row数据都是由几组Name/Value的键值对来组成的，我们把每一组Name/Value键值对称之为**column**这也是其Schema-Free设计的基础，不像其他传统关系数据库每一Row都要有fixed的大小(固定的列数据)

Cassandra的每一Row的定位需要有一个唯一确认的标识符来标识，我们把它称之为**row key**或者**primary key**。

![](https://javatuchuang.oss-cn-shanghai.aliyuncs.com/img/20190808143442.png)

那么Cassandra的每一Row的基本结构如图所示。

就算是同一table，由于前面所述row中column组成不同，也会有不同的结构，Cassandra会在底层处理查询语句时将我们没有存储某些column的row的值设为**null**

![](https://javatuchuang.oss-cn-shanghai.aliyuncs.com/img/20190808143835.png)

Cassandra还有一类特殊的primary key，称之为***composite key* **用以代表wide rows。其中composite key是由两部分组成的，其中一部分称之**partition key**，用以确定这条row会存储在哪个node上，这个partition key可以由多个column组成;另一部分是clustering column，用以确定该row在该partition中的排序(cassandra的数据存储是有序的)。除此之外还有一个static column的概念，这种column代表同一个partition中都存在的column

![](https://javatuchuang.oss-cn-shanghai.aliyuncs.com/img/20190808145505.png)

除此之外，还有以下几个概念

- **table** row的集合
- **keyspace** table的集合
- **cluster** 可能会跨越多个node的keyspace的集合 

### 查询语言模型

官方文档手册 https://cassandra.apache.org/doc/cql3/CQL-3.0.html

- CQL增删改查一定需要Primary Key
- Primary Key一旦设置就无法改变，毕竟整个分布式实现在此基础上实现

#### Column

- timestamp 每一次写入数据时，就会对每一列的数据产生一个timestamp，cassandra用它来解决冲突，一般来说，后写入的获胜，我们不被允许查看primary的timestamp
- ttl 这个也是针对每个colume层级的，目前还没有针对row层级的ttl，而且如同timestamp一样，我们也不能对primary key做什么，并且只能对column赋值时修改ttl的值

#### CQL TYPE

这个也没啥好说好写的，有一些比较特殊的

- text/varchar类型的编码都是UTF-8

- uuid cql uuid类型是基于Ver4版本的，这种随机数UUID版本需要指定不同node的随机数源, cql提供足够唯一性的实现，但不绝对保证毫无碰撞

  > RFC 4122 建议“在各种主机上生成 UUID Ver4 的分布式应用程序必须愿意依赖所有主机上的随机数源

- timeuuid 这是Ver1版本的(基于网卡MAC地址和时间戳)，所以唯一性保证比uuid这个类型的强

### 二级索引

在不加索引的情况下，Cassandra不允许查询非组成Primary Key的column

创建索引时如果指定索引名称，默认是"<table_name><column_name>_idx"的形式

由于Cassandra的数据存储是跨很多节点的，每个节点只会存储那些在自身节点上分区数据的二级索引的，所以二级索引的查询就可能会涉及多个节点的查询，可能会比较费时，尤其以下几种情况更为严重

- 当二级索引的列数据非常的不一致，接近于唯一的那种的话，索引就会非常大
- 当二级索引的列数据非常统一，那么查出来的数据就会区分度较低，索引内部包含太多行数
- 当二级索引的列数据频繁更新或者删除的时候，超过内部压缩线程处理的量时就会产生错误

有的时候违背设计规范的表或者物化视图Materialized View要比使用二级索引方便很多。从3.4版本起，Cassandra使用一种SASI（SSTable Attached Secondary Index）技术来实现二级索引，并且可以支持一些不等式查询，比如<或者> 或者Like

## 第五章 Data Modeling

首先用ER图展示了一个领域逻辑和使用RDBMS的设计，不过RDBMS和Cassandra的设计还是存在差异的，具体的差异存在于以下几个方面

- 没有JOIN Cassandra倾向使用额外的表来记录join的结果
- 没有外键约束和参照完整性，需要自行在表中额外存储
- 反规范化，不符合RDBMS的第N范式设计
- 查询优先的设计模式 在cassandra的世界中你可能并不需要急着设计数据模型而是优先考虑应用程序如何查询和使用他们然后再设计数据模型来支持它
- **为优化存储而设计** 搜索单个partition往往会提供最高性能，另外cassandra的table的存储是按文件分割的，最好将相关的column集中于同一个table中
- **排序是一个重要的设计决策** 在cassandra中，排序完全不同于rdbms中的排序，rdbms可以在查询时决定返回的顺序以及定义的顺序，而cassandra则完全是在create table时提供的的cluster column就已经决定了它们的返回顺序。

#### 设计小诀窍

- 使用唯一ID来标识实体在微服务设计风格中极为有用
- 如果涉及到范围搜索，请使用这些属性作为clustering column来进行(这些column会进行排序处理的)
- 使用**Materialized View**来解决某些高基数类型的列(数量大且不尽相同)涉及二级索引效率不高的情况
  - 这样会略微降低写性能，因为需要同步保持一致性

#### 评估和改善

- 计算分区大小
  - cassandra的硬限制是20亿个cell每个分区，但是可能没达到的时候就出现问题了
  - Nv = Nr(Nc-Npk-Ns)+Ns ,其中N代表Number数量，v：cell或者value，s代表static，r代表row，c代表column数量
  - 计算物理存储时不要忘记TTL和timestamp的大小，按8Byte字节计算是一般形式
- 切分大分区的方法
  - 最直接的就是添加一列到分区键里面，大多数情况下，将一个已经存在的列移动到分区键是比较高效的办法
  - 或者增加新的列作为分区键，可能需要业务逻辑修改
  - 或者使用桶bucket分法，将月份、日期等作为分区键

## 第六章 The Cassandra Architecture

Cassandra非常适合用于跨越物理位置隔离的系统中。Cassandra 的默认配置是单数据中心DC1单机架RAC1

![](https://javatuchuang.oss-cn-shanghai.aliyuncs.com/img/20190813171354.png)

第六章涉及到相当多的底层实现，主要是以下几个方面，从网络通信、分布式去中心化模型、查询、复制策略、底层数据和持久化结构、缓存、故障恢复、分布式协商一致性方案、数据压缩、anti-entropy、merkle tree、源码的SEDA的设计模式、系统服务、系统KeySpace。所以还是单独拆出去整理了。

连接如下:<https://holajiawei.com/cassandra-arch/>

## 第七章 Configuring Cassandra

本章主要描述了Cassandra相关的一些配置

首先介绍了一个管理Cassandra的工具Cassandra Cluster Manager(CCM),一些Partitioner、Snitch、Node的配置，这个就不在赘述了。查资料都可以查到，不过需要注意的一个配置就是Node分到的token设置，默认num_tokens的数量是256，这个其实不是绝对的，完全可以调整更多的token到高性能机器上或者更少token到低性能机器上。

## 第八章 Clients

讲述了一些Cassandra客户端的使用，没有意思，略过

## 第九章 Reading and Writing Data

Cassandra得益于顺序性写入且不可变的存储结构，能够有效避免写入放大的情况。不过由于Cassandra架构的设计，对于同一Row在不同副本间的更新同步(一致性)是实现ACID的关键一步。

### Writing

得益于Memtable和SSTable，Cassandra的写入速度非常快，同时它的写入也是顺序附加上的append。由于commit log和hinted handoff机制的存在，数据库永远都是可以写入的，并且在同一个column中写入都是原子性

append的特性导致Cassandra的insert和update并没有太多区别，甚至是天然的Upsert的实现，不过在LWT中有一点小不同。

memtable和sstable都是每张table独立维护的，但是commitlog确是所有table公用的，sstable是不可变的，所有的写入都是append上去的，也就是意味着每table的partition都是跨越了多个sstable文件。SSTable也有其他相关的文件结构如下：

- data.db  SSTable存储数据的地方
- index.db primary index存储的地方，存储各个row在data.db中位置
- filter.db 一个存储于内存中的结构用来检查某row数据再去sstable中查询前是否存在于memtable中
- Digest.xxxx xxx可以是crc32、adler32、sha1 ，data文件的校验和
- Summary.db 在内存中存储的部分分区索引
- SI_*.db secondary index 内置的二级索引存储文件位置。多个二级索引可能存在于每一个sstable中

写入的一致性级别有以下几种:

- ANY: 最少一个节点写入即可，甚至包含hint handoff的写入
- ONE\TWO\THREE
- LOCAL_ONE
- QUORUM 多数，replica factor/2 + 1
- LOCAL_QUORUM 本地datacenter满足多数
- EACH_QUORUM
- ALL

![](https://javatuchuang.oss-cn-shanghai.aliyuncs.com/img/20190820142811.png)

如果集群是跨多个数据中心的，那么在选择更高一致性级别时，本地coordinator节点会选择一个remote coordinator来做复制节点选择工作，后续replica node直接与原始的local coordinator来通信

![](https://javatuchuang.oss-cn-shanghai.aliyuncs.com/img/20190820143447.png)

Cassandra的Batch操作具有原子性的特点，但是并不是事务方法，它只是节省了RTT时间和适合批量做操作，但是千万不要觉得批量操作会提高性能，相反，批量操作相对于单个操作时间来说是更花费时间并带来更多GC压力，所以Cassandra使用了一个限制batch处理大小，警告阈值默认为5kb，失败阈值默认为50kb

### Reading

在Cassandra中，读一般要比写慢，但是通过增加更多节点，增加更多内存，可以使更多数据进入Cache中。偶尔需要执行read repair

read的一致性级别与写的一致性级别基本类似，不过为了解决一致性的问题，我们还是推荐R+W > N的公式

![](https://javatuchuang.oss-cn-shanghai.aliyuncs.com/img/20190821110113.png)

![](https://javatuchuang.oss-cn-shanghai.aliyuncs.com/img/20190821110151.png)

digest实际上是返回数据的hash值，coordinator会从最快返回的副本中获取数据，然后计算其hash值再从其他副本节点获取的digest进行比较，如果一致且满足一致性等级要求的话就会返回数据；如果不一致或者无法满足一致性等级需求，=那么就会由coordinator发起读修复工作。

其中SSTable的查询工作又可以分成好几步：先从bloomfilter中查找有没有，再去key cache中找下有没有，如果再没找到就会使用2个不同level的index来寻找，第一个先来寻找partitionkey在那个文件中，第二个partition key文件中具体什么部位存着。

读修复过程由coordinator发起，它会所有的副本节点发起full read请求，然后coordinator会逐个column进行合并挑选，挑选的原则就是谁有最新的timestamp。如果两个timestamp一致的话，它会按照字典序比较两者值，谁值大选谁。同时，coordinator也会识别返回数据的任何副本，并向所有副本发出读修复请求，然后更新合并数据。当然了读修复过程会根据一致性等级要求来执行，如果是弱一致性要求比如ONE这种，可能会在返回数据后后台默默执行；如果是quorum或者ALL这种强一致性要求的情况下，就会在返回数据前修复完成。

查询时Where语句限制条件如下:

- 所有组成partition key的column必须指明
- 如果要限制某一个clustering column，那么之前组成primary key的clustering key都需要做限制

虽然使用allow filtering可以忽略parition key，但是并不推荐这么使用，因为这种查询非常耗时。使用IN语句大概稍微好点不过由于可能值的区域不连续，也容易降低读取性能。

coordinator内置的重试策略默认是99.0PERCENTILE，意味在第99%百分位响应时间内未收到响应的话，就会启动重试

SSTable做删除时会对原有文件做压缩，合并的数据进行排序，在排序的数据上创建新索引，并将新的合并、排序、索引的文件写入新的文件，注意删除也是一种写入操作，所以一致性级别跟一样。

## 第10章 Monitoring

围绕JMX介绍了一些监控工具，比如JConsole、nodetool，没啥好讲的

## 第11章 Maintenance

常见的健康检查手段:

- Nodetool status 确保所有节点存活
- nodetool tpstats 查看节点性能改善负载，下降时需要考虑扩展机器配置或者增加节点
- 查看log记录
- 查看cassandra配置文件
- 查看keyspace配置
- 确定网络互通和NTP工作正常

基本维护

- flush memtable -> sstable
- cleanup 
- repair 这个执行的anti-entropy的修复工作，这个修复工作会占用很多IO和内存，成本比较昂贵，尤其是在大表中，假设单个节点100万分区，Merkle树的每个叶节点代表大约30个分区，即使只有一个分区需要修复，但这三十个分区都需要修复，产生了额外的费用。Cassandra提供了三个不同等级的选项，FULL 、INCREMENTAL 、ANTI-COMPACTION。增量修复将未修复和已修复数据进行了分离。

增量修复会从更小的partition开始执行，这样会节省更多的网络传输。Cassandra增加metadata到每一个SSTable中来记录修复状态，可以使用sstablemetadata工具来查看。

从2.2版本开始，增量更新和并行更新成为默认选型

在修复过程中，由于dynamic snitch会更加主动将节点请求转发给其他未参与修复的节点。并行修复过程会使整个集群负载更高，但修复时间更短。

本章还讲述了添加节点、移除节点、添加数据中心、修复节点故障时的方法、定期备份和恢复。以及常用的一些工具。

- sstableutil 会列出指定table相关的sstable文件
- sstablekeys 指定sstable中包含的partition keys

## 第十二章 Performance Tuning

使用nodetool proxyhistogram可以查看节点作为coordinator时延时

使用nodetool tablehistogram可以查看具体table相关读写延时

使用cassandra-stress可以进行压力测试

使用trace on cql可以做query trace.

## 第十三章 Security

用户身份认证和RBAC

使用SSL加密通信

## 第十四章 Deploying and Integrating

规划Cassandra集群

生产环境推荐使用8C16GB以上机型，磁盘选择HDD或者SSD都可以，不过SSD的性能相对较好，推荐SSD，至于RAID方面推荐RAID0，因为应用层级做了冗余手段，避免NAS或者SAN。网络带宽1Gbps以上，避免节点间再置放负载均衡。