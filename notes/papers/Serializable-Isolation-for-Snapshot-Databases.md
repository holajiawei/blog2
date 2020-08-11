---
title: SSI机制原理论文《Serializable Isolation for Snapshot Databases
description: 
published: true
date: 2020-08-11T03:26:33.385Z
tags: database, transaction
editor: markdown
---

# SSI机制原理论文《Serializable Isolation for Snapshot Databases》

> 本篇论文提出了一种检测算法用于解决常见的实现快照隔离的数据库经常会发生的Write Skew问题，并在Oracle Berkeley DB上做了实现，并做了一系列性能测试，相对于传统的S2PL实现串行化隔离的机制有了大幅度的性能提升
{.is-success}

## SI快照隔离带来的问题

SI快照隔离解决了读的一致性问题，并且为了避免Lost Update问题，采取了First-Commiter-Wins的规则，将并发产生冲突的后来者事务进行抛弃。但是SI带来了Write Skew问题，对于需要维持一些约束的数据集进行写操作的事务在并发时容易产生破坏一致性的问题

如果要想解决这类问题需要将事务串行化，但是串行化的传统基于S2PL的实现，性能较差，尤其是在读比较重的业务中尤为突出

### 我们提出的的算法

- 称之为Serializable Snapshot Isolatiion 
- 并发控制算法无论上游业务怎么写，可以保证每个执行都能被serializable
- 算法不会延误读操作，也不会让读操作延误写操作
- 在一些确定的条件下，整体的性能接近于SI所提供的，并远远好于S2PL实现提供的
- 这个算法可以被已提供SI实现的数据库轻易实现

## 算法核心思路

我们就是要在运行时发现不满足串行化执行的事务并把它丢弃掉，我们并不像serialization graph testing那样的方式工作，并不会在commit时期时检测，而是在运行时发现问题就会尽早结束。而且并不需要去做相关的环的追踪实现，我们的工作方式类似于乐观锁，只有在发现两个连续冲突路径时才会终止，这是我们发现的SI隔离产生异常时的特征。因为我们的检测是保守的，可以保证所有非序列化的操作，但也有可能会终止一些不必终止的事务

## 论文的架构

- S2
  - Snapshot Isolation的介绍
  - Write Skew的介绍
  - 幻读
- S3
  - 介绍SSI算法
- S4
  - 如何实现SSI
- S5
  - 性能评价
- S6
  - 总结

## S2

### Snapshot Isolation

- 无阻塞读
- 只能看到事务开始执行时间之前的已提交的修改和自身做的修改
- First-Commiter-Wins:两个并发事务不可能同时提交并修改相同的数据项

### Write Skew

一个无法成环的事务历史可以被serializable

### Phantom

通过增加意图锁防止幻读



## S3

算法实现:我们对于一个事务T，标记T.inConflict 和T.outConflict，分别表示是否有指向T的rw依赖的事务，由别的事务指向T代表in，由指向别的事务代表out。这里的指向的内涵就是Tx -> Ty, 那么Tx包含read，而Ty包含write

结合S2部分说的理论，那么每当有无法序列化的操作出现时，就会有一个事务T同时有T.inConflict和T.outConflict,但是这也意味着存在误报的可能，因为这两个rw依赖edge并不出现在同一个环中，除此之外，对于某些透视T来说（即只读事务来说），它有可能变成受害者

在设置inConflict和outConflict时会检查是否存在另外一边，如果有，那么就可以中止事务，但是对于稍后才会发生写的事务来说我们很难进行设置，所以我们提出了一种新的锁的模式SIREAD，如果某项数据被读那么设置SIREAD锁，如果同时存在SIREAD锁和WRITE锁那就意味着该项数据存在rw-dependency，那么就要设置分别持有锁的inConflict和outConflict，但是这里有个麻烦之处是需要保持SIREAD锁直到所有与T相并发的事务完成

伪代码解释如下:

begin(T) 开启事务

~~~shell
set T.inConflict = T.outConflict = false
~~~

read(T，x) 在某个事务中阅读x数据

~~~shell
get lock(key=x, ownner=T, mode=SIREAD)
if there is a WRITE lock(wl) on x  // 如果存在写锁的话，那么说明会存在rw依赖
	set wl.owner.inConflict = true
	set T.outConflict = true

existing SI code for read(T, x)

for each version (xNew) of x that is newer than what T read:
	if xNew.creator is committed and xNew.creator.outConflict: // 如果存在比T早提交的事务并且该事务的存在rw依赖
		abort(T)
		return UNSAFE_ERROR 
	set xNew.creator.inConflict = true // 既然存在新版本的话，那么修改写入xNew的事务存在rw依赖
	set T.outConflict = true // 这个事务本身也是读的
~~~

write(T, x, xNew): 在事务T中将x的值赋为xNew

~~~ shell
get lock(key=x, locker=T, mode=write)
if there is a SIREAD lock(rl) on x with rl.owner is running or commit(rl.owner) > begin(T):
	if rl.owner is commited and rl.owner.inConflict: // 代表已经出现rw依赖
		abort(T)
		return UNSAFE_ERROR
  set rl.owner.outConflict = true
  set T.inConflict = true

~~~

Commit(T)

~~~
if T.inConflict and T.outConflict : // 提交的时候在检查下
		abort(T)
		return UNSAFE_ERROR

existing SI code for commit(T)

# release WRITE locks held by T
# but do not release SIREAD locks
~~~

对于SI系统来说，还有一个实现关键是存储引擎必须能够获取事务T的相关信息，比如inConflict、outConflict以及获得的任何SIREAD锁,这些信息必须存储到任何覆盖事务T的其他事务时间，就是说我们只能当所有在T结束之前开始的这些事务完成之后才能清除

结论如下:

1. 在SI下无法被序列化执行的事务执行图中必然会存在两条连续的rw依赖
2. 我们的算法能发现所有rw依赖
3. 当连续两个rw依赖被发现时，至少有一个事务会被中止

因为算法的保守性，所以可能导致一些正常的事务被终止，虽然我们可以通过增加对每个冲突事务的引用而不是简单的通过标志位来记录，但是考虑到事务可能有多个rw操作所以可能会导致记录的数据结构非常复杂