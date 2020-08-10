---
title: 数据库事务一致性中的读偏斜和写偏斜问题
description: 
published: true
date: 2020-08-10T05:53:15.144Z
tags: database, transaction
editor: markdown
---

# 数据库事务一致性中的读偏斜和写偏斜问题

> 在1995年一篇描述SQL隔离级别的论文[A Critique of ANSI SQL Isolation Levels](http://research.microsoft.com/apps/pubs/default.aspx?id=69541)中曾经提出了除我们常见的脏读dirty read、不可重复读non-repeatable read、幻读phantom read之外的更多的异常情况，也就是今天要说的Read Skew读偏斜和Write Skew写偏斜(其实还有一种比较复杂的异常现象:更新丢失 Lost Update，这个单独成篇说)

## 什么是Read Skew和Write Skew

我们首先来假设存在以下一种业务模型：

Table A中记录一个Obj List，Table B中记录具体Obj对象，两者之间存在业务上的一致性约束，既存在着修改Table B中某Obj的同时修改Table A中同Obj的相关属性。

~~~
Table A 中某Obj {
	id:1
	x:"xxx"
	y:"yyy"
}
Table B中某Obj {
	id：1
	m: "MMM"
	x: "xxx"
}
~~~



首先来看read skew的情况，假设同时发生以下两个事务

| 时间t | T1                                           | T2                                        |
| ----- | -------------------------------------------- | ----------------------------------------- |
| 0     | select obj.xxx from table A where obj.id = 1 | Update table A set obj.xxx ="XXX"         |
| 1     | ...                                          | Update table B set obj.xxx = "XXX" commit |
| 2     | ...                                          | commit                                    |
| 3     | Select obj.xxx from table B where obj,id =1  |                                           |

这时候事务T1中在t2时刻中执行的查询语句就会发现：咦，为什么TableB中的obj.xxx与TableA中的obj.xxx不一致呢，这就是在查询过程中产生的数据不再维持原来约束的异常情况，称之为read skew

再来看write skew的情况，假设同时发生以下两个事务

| 时间t | T1                                           | T2                                           |
| ----- | -------------------------------------------- | -------------------------------------------- |
| 0     | select obj.xxx from table A where obj.id = 1 | select obj.xxx from table A where obj.id = 1 |
| 1     | Select obj.xxx from table B where obj,id =1  | Select obj.xxx from table B where obj,id =1  |
| 2     | ...                                          | Update table A set obj.y ="YYY"              |
| 3     |                                              | commit                                       |
| 4     | Update table B set obj.m = "mmm"             |                                              |
| 5     | commit                                       |                                              |

假设表B中obj的m和表A中obj中的y需要保持大小一致，某天脏值检查修复程序(业务侧的程序)同时执行了这个修复动作，但很不幸的是它们的检查顺序并不一致，T1看到了A中的数据为小写，而T2看到了B中数据为大写，它们各自执行了不同的修改，从而破坏了最终的一致性

原本维持一致性约束的AB表由于各自实际处理需求而更新数据，这样破坏了一致性约束，称之为write skew

根据以上的情况，我们可以看到write skew的关键因素有俩:1. 需要保持的一致性约束 2. 做出的更新操作不存在冲突，但读取的数据相同并作为更新操作的依据，这两个因素实际上在一些复杂业务场景里是经常出现的

## 现在数据库的隔离级别对此异常的处理情况

- 我们平时经常使用的MySQL InnoDB是借助MVCC实现的RR的隔离级别，所以前面所述的Read Skew是没有什么问题的，毕竟每个事务看到的都是独立的版本快照，但是Write Skew问题恰恰由于MVCC机制实现的原因，反而变得普遍起来
- PostgreSQL内部通过实现更高级的Serializable Snapshot Isolation Level来避免了Write Skew问题，这是一种复杂的快照机制，基本实现了Serializable隔离级别
- MySQL在启用Serializable隔离级别后，会借助share lock的机制来规避write skew问题，不过这样并发性能就会大打折扣了
- SQL Server 默认基于Lock机制实现的隔离级别可以在RR和Serializable两个隔离级别来规避Write Skew 问题，而基于MVCC机制实现的隔离级别则不能规避
- CockroachDB 跟PostgreSQL一样，实现了SSI来规避Write Skew问题
- yugabyteDB 的事务实现跟Cockrocah类似，都是参考了Google Spanner[Spanner: Google’s Globally-Distributed Database](https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/spanner-osdi2012.pdf)的论文来实现的，只有隔离级别达到serializable时才会避免write skew
- tidb的事务模型则是来源于Google Percolator，来源于这篇论文[Large-scale Incremental Processing Using Distributed Transactions and Notifications](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/36726.pdf),实现了SI快照隔离，不会出现read skew，但会出现write skew

## 关于write skew的其他解释

在看cockroach的文档时，看到这篇[what write skew looks like](https://www.cockroachlabs.com/blog/what-write-skew-looks-like/),根据读写依赖关系解释了为什么快照隔离依然会发生write skew

简单来说的话，两个事务Ta和Tb，我们说Ta比Tb先发生只可能会有以下三种场景之一:

- Ta写了一个值然后Tb去读，这是一种write-read的关系
- Ta写了一个值然后Tb再去写，这是一种write-write的关系
- Ta读了一个值然后Tb再去写，这是一种read-write关系

而对于快照隔离来说，以上三种情况唯一可能被并发执行的只有第三种rw关系，即Ta先读取了值然后Tb再去写，其他俩种对重对于快照隔离来说都违背其设计原则，不会出现

![](https://javatuchuang.oss-cn-shanghai.aliyuncs.com/img/20200312213428.png)

以上为图例·，代表Ta -> Tb ww, Tb-> Td wr, Tb -> Tc rw

当这种情况发生时，是无法完成序列化，并且也不知道最终执行结果的

![](https://javatuchuang.oss-cn-shanghai.aliyuncs.com/img/20200312214034.png)

这种称作Serialization Graph Testing的检测，如果发现即将同时提交的的事务出现了环状结构，那么该事务则不会被提交

然而这种在图中找环的检测是耗时，尤其在图比较巨大时。

所以在Serializable Snapshot Isolation采用的方案并不是检查环，而是检查在快照隔离中每个事务都跟踪他是否在两端都涉及依赖关系或者它本身就是rw依赖关系的两端(即连续俩rw)，如果是，那么它或者它的下一个或者前一个事务会被终止