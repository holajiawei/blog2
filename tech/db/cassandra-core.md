---
title: Cassandra 架构、实现原理
description: 
published: true
date: 2020-08-10T05:56:40.467Z
tags: 
editor: markdown
---

# Cassandra 架构、实现原理

Cassandra是一个高可用、高扩展、分布式去中心化、无单点故障、高写入的NoSQL类型数据库。其特征的实现依赖众多已有的分布式、存储、网络通信等基本技术的实现，是一个值得学习的分布式数据系统案例。

> 官网文档还真不好找，看的datastax的文档，https://docs.datastax.com/en/ddac/doc/index.html

## 去中心化

去中心化的实现需要至少三个方面的问题的解决：

- P2P通信协议
- 错误发现和规避
- 自我定位和管理路由

Cassandra使用Gossip协议来让集群中各个节点彼此感知，它们之间相互交换自己状态的信息和之前跟他们gossip过的节点信息，协议执行的单位周期是1秒钟，也就是每一秒，集群中的任一节点都会使用此协议来进行通信

Gossip协议是一种广泛应用于大规模去中心化、假设网络经常发生分区错误(连接中断)的网络系统的通信协议，也会经常应用于分布式数据库的复制过程中，这种协议定义了一种解决节点间彼此确认沟通对象的解决方案(就好比是两人要交换数据，假设对方视而不见或者耳聋眼瞎，怎么确认对方是可以交换数据的人)

> 该协议的实现主要依靠**org.apache.cassandra.gms.Gossiper**类实现，当服务启动后，会使用gossiper来接受endpoint的状态信息，同时gossiper能够维护一份集群中节点死或生状态的列表

Gossip协议的运作模式如下：

- 每一秒，gossiper会随机挑选集群中的一个(最多3个)节点并初始化一个session连接，每一次完整的gossip通信都需要三个步骤（非常类似TCP通信协议的三次握手）
- 本地gossiper向前面选择好的对方发送**GossipDigestSynMessage**
- 对方收到SYN消息后，会返回一个**GossipDigestAckMessage**
- 本地收到对方ACK后，会再返回一个**GossipDigestAck2Message**

Cassandra可以设置一类称之为seed node种子节点的节点，这类节点没有特殊作用，只是在集群启动之初，方便新加入的节点开始gossip过程

当gossiper发现某个节点"死亡"时，它就宣布这个节点死亡并在日志记录下来。Cassandra判断节点死亡的算法称之为**Phi Accrual Failure Detection**。这个算法有两个基本原则：第一条原则是实现故障检测应该与监控分离，这样会更加灵活的改进算法 ；第二条原则就是充分怀疑。相较于传统的基于心跳包heartbeat的这种比较简单且粗暴检测机制会让服务处于两个极端的判断-可用或者不可用，而采用怀疑的方式就有更加准确的服务的现状。

> 故障检测主要依靠**org.apache.cassandra.gms.FailureDetector**类实现，采用了Phi Accrual Failure Detection< [*http://www.jaist.ac.jp/~defago/files/pdf/IS_RR_2004_010.pdf*](http://www.jaist.ac.jp/~defago/files/pdf/IS_RR_2004_010.pdf)>论文中算法实现，颇像TCP协议中拥塞控制协议中动态窗口大小的设置，以一种动态的方式来设定不同阶段时对于错误检测的敏感度(类似于拥塞协议中窗口大小)，Cassandra使用此机制通常能在10s内检测到故障节点

Snitch的机制让Cassandra拥有自主判断高效的将读或写请求路由转发给指定的节点来处理。Snitch通过收集网络拓扑信息来决定请求路由到哪里。它知道节点之间关系如何，是同rack还是同dc。

拿Cassandra读请求为例，它会根据一致性等级来联系一定数量的副本，为了最大化读的速度，Cassandra选择一个其中一个副本来查询所有的请求对象，与此同时访问其他副本中对应数据的hash值来判断这个值是不是最新版本的值。Snitch的角色就是帮助Cassandra寻找最好的路由路径，最好的路由路径用来负责查询中的所有数据出入口

默认的Snitch机制SimpleSnitch并不关注网络拓扑(比如multi-dc、multi-rack),所以有跨数据中心的部署形式最好指定别的Snitch。生产环境推荐使用**GossipingPropertyFileSnitch**.关于网络拓扑，Cassandra提供了静态的配置方式来描述网络拓扑配置，也提供了动态的方式。所有的Snitch都有动态的层次来提供功能支持，动态的方式会伴随读写请求来追踪节点之间的通信性能，甚至还包括追踪哪些节点在执行compaction压缩操作，这样能够供cassandra做出更加高性能的路由转发判断。动态Snitch的实现采用了类似Phi 故障检测的机制，它会动态维护一个阈值badness threshold来检查哪些高优先级转发节点在不满足阈值条件后降级为普通节点，并定期检查维持列表更新，并把分数重置，以便让性能较差的节点能够证明它已恢复。

## 分布式

分布式系统除了要解决前面的感知集群运行状态的问题外，最重要的问题就是分布式环境下的三个问题:

- 数据的分区
- 数据的存储/复制
- 状态的一致性

Cassandra对每个partition key计算出一个hash值，每一个在集群中的node根据这个hash值存储一定范围内的数据

Cassandra使用了一种叫做Ring和VirtualNode的方式来管理集群中的数据。首先来说下Ring的作用和原理。

Cassandra设计了一个64位整形 ID值作为Token来标识分区，这也就意味分区的数目最大就是2^64,Token的取值范围是-2^63到2^63-1。然后把所有在集群中的节点围成一个环来分token，也就意味着每个在ring上的节点可能会分到1个或多个token，每当集群中有新节点加入或者旧节点退出的时候，就会自动来平衡token在节点间的分布，并且承担数据转移的责任。这里的节点都是虚拟节点的概念，这就意味着VirtualNode(以下简称VNode)可以跟真实的物理机器在做一层映射，可以更多或者更少的电脑都可以构建出集群

![](https://javatuchuang.oss-cn-shanghai.aliyuncs.com/img/20190815113509.png)

虽然VNode带来了相当大的操作优势，但是带来的问题也很显著，那就是在修复repair周期内的修复次数也增加，从而增加完整的集群修复时间

Partitioner负责分配数据（包括复制的副本）如何存储在节点上,每一Row通过partitioner计算其hash值获得token，然后根据token来分配，默认的partitioner使用的是murmur3partitioner，它会使用token来帮助每个节点分配相同的数据，并在所有节点中均匀的分布来自所有表的数据。因为整个散列范围也被均匀划分了。其实这也跟你选择的节点分配有关系，如果选择是VNode分配，那么会使用随机算法或者其他特定的分配算法来分配。并且所有在同一个datacenter中的分配算法是一样的，这个算法token的取值范围才是前面说述的-2^63 到 2^63-1，而random使用row key的MD5值，那么取值范围就是2^127-1,因为使用了加密类型的哈希算法，所以性能上要稍弱于murmur3

如上图所示Ring with virtual nodes中，使用murmur这种均匀分布的算法话，显示有16个令牌，共有6个物理节点，复制因子是3的情况下，16*3/6 = 8，那么每个物理节点上有8个VNode。

第一份副本是由Partitioner直接决定的，剩余的副本则是根据replicationStrategy决定。有两个实现SimpleStrategey和NetworkTopologyStrategy，前者是从Ring中第一份副本的位置开始，顺时针位置的Node，并不考虑网络拓扑。而后者放置副本时会将副本放置在同一数据中心中，从第一份副本开始，顺时针转动，到达第一个不同机架上的节点便放一个。在决定在每个数据中心中配置多少副本时，两个主要考虑因素是在本地满足读取而不会产生跨数据中心延迟和故障情况。一般有以下两种

	- 每个数据中心放两份：这种可以保证在ONE这个一致性等级下每个复制组内单个节点的故障和本地读
	- 每个数据中心放三份: 这种可以保证强一致性LOCAL_QUORUM下每个复制组任一节点的故障或者弱一致性ONE下多个节点的故障

至于数据一致性方面，Cassandra实现了可调节的一致性级别。这样用户就可以在更细粒度下进行控制，可以针对每次读写请求来指定一致性等级。更高的一致性要求需要有更多的节点来反馈，这样就会给你更多的保证来保证每一份副本是相同的。ONE、TWO、THREE的一致性级别要求了多少个副本节点必须回应，QUORUM一致性级别要求大部分副本节点响应(一部分是replica factor/2+1),ALL则要求所有副本节点反馈。QUORUM和ALL被认为是强一致性，其余则被认为弱一致性。有一个经常用于Cassandra的经验公式，R+W > N就被认为强一致性。其中R、W、N分别代表读副本数、写副本数、复制因子。注意的是复制因子作用于keyspace，而一致性级别是由客户端请求时设置的，着由过往服务端设置是不易呀宝贵的

有的时候我们需要一种类似于线性一致性的场景，比如读完写的场景，判断有没有存在然后再插入，这是一种更强一致性的要求。Cassandra提供了一套lightweight transaction（LWT）的机制来实现此种一致性的需要，LWT的实现依赖Paxos这种分布式协调协议，无需主协调节点的存在，这是对传统分布式事务协议2PC的替代选择。这里Paxos协议的实现原理就不细说了，参考另外一篇文章:<这里该有一篇连接>，简单来说就是大家在做接受提案提交前确保不再接受新的相关这个提案的提案，并且返回大家进行中的最后一个提案，如果半数以上一致，那么充当领导者角色的人会提交提案，大家也各自提交。但是Cassandra并没有简单的直接使用Paxos协议来实现此LWT，而是将Paxos+2PC结合起来实现的，那样就意味着一次成功的CAS或者RBW需要4次round-trip:

	- Prepare/Promise
	- Read/Result
	- Propese/Accept
	- Commit/Ack

这种操作非常昂贵，所以Cassandra的LWT事务机制还是要谨慎使用。并且LWT被限制于同一个partition来使用(如果跨partition，那么每一个partition存储paxos store也需要同步保证一致性，复杂度会进一步增加)



## 无单点故障

客户端通过访问协调器节点来进行读写。

![](https://javatuchuang.oss-cn-shanghai.aliyuncs.com/img/20190816103656.png)

对于写来说，协调器负责联系所有相关节点根据一致性要求和复制因子决定沟通多少副本进行写入，对于读来说，协调器负责沟通所有相关节点进行读取。

另外关于Cassandra的高可用机制是其Hinted Handoff机制的实现。当写入时发现有节点因为网络分区、硬件故障或者其他原因而断掉，协调器节点就会把包含写入信息的hint贴出来"现在b炸了，我要要写入它的信息，等B恢复后，我会写入它"这个特性保证Cassandra永远都是可以写入。但是注意的是hint没有真正写入到指定节点上时，是不会读取到的。在一致性级别是ANY的情况下，Hint被认为写入成功。不过也存在问题就是下线一段时间后，可能累积了大量的hint，当节点重新上线后，可能触发洪水攻击，为解决此问题，cassandra可以配置时间窗口大小或者完全禁用hint handoff.

当然了hint并不能真正完全解决可用性问题，一致性的保证还需要修复repair机制。

repair机制的实现依赖anti-entropy一种特殊类型的gossip协议，它会通过比较所有的副本数据然后发现其中的冲突差异的地方。这个机制并不是只是简单用于修复数据，它本质是还是一个数据同步协议。Cassandra的副本同步会有两个时机来做，一种是读数据时的read-repair，另一种就是anti-entropy repair。前者很好理解，在读过不同副本有不满足一致性级别的数据时，如果有存在副本有过期值，那么就会立即执行read-repair。而anti-entropy修复则需要手动在节点上执行nodetool操作。执行的节点会跟自己的邻居节点进行MerkleTree的比较，如果发现差异的地方则认为存在冲突，就会进行anti-entropy修复。整体的修复过程称之为major compaction(也是通过compaction来实现的)。MerkleTree是在major compaction过程中生成的，并且只在需要在比较树的时间内保留。



## 高性能

高性能离不开高性能结构的设计。这里分两块来说明，其中一块是内存中的数据结构，另一块是用于储存到磁盘上的

![](https://javatuchuang.oss-cn-shanghai.aliyuncs.com/img/20190816104431.png)

当执行写入操作，这个操作会被立即记录到commit log中，commit log是一种故障恢复手段，写入操作只有真正写入到commitlog中时才会被认为写入成功，这种以防万一没写入memtable(内存数据存储)时可以通过此手段恢复数据。不过commitlog的真正使用只会在下一次节点启动时来重放，其他时间不会进行读取

当写入Commit log完成后，再去写入内存持久化结构memtables，从2.1版本开始，memtable的实现由JVM堆转移到了Native堆中，减少JVM的GC时间。当memtable中的数据到达一定阈值后，memtable中的内容就会flush到磁盘中，然后创建一个新的memtable。flush不是一个阻塞的操作，对于同一个table的memtable来说是逐个flush的

commitlog内部每一个table都有一个标示位标示是否flush到sstable中。所有table都会写入同一个commit log中。当memtable flush完成后会将相应的commitlog的标示位置为0。SSTable是一个不可变的数据结构，一旦写入就无法改变，SSTable执行压缩合并操作会产生新文件。虽然SSTable全程sorted string table，但其实数据的存储在磁盘上是不会按照sorted string来存储的。因为SStable的写入是顺序写入不需要在磁盘上寻位，所以比较高效。SStable会定期压缩来重新组织优化成为更好的阅读性能。通过压缩，key会合并并且排序，列会联合起来，tombstone会被遗弃掉，index会重建，会写入一个全新的SSTable文件。同样的压缩策略也有很多种，可以给不同的table来设置，有适合读频繁场景的LeveledCpmpactionStrategy（LCS），适合写场景的SizeTieredCS。

内存中的KeyCache位于JVM Heap中，用于储存分区key来索引实体来加快访问SStable。Row Cache则是保存了整个Row来加速那些频繁访问的rows。Counter Cache则是用来减少计时器的锁争用来提升性能。默认情况下KeyCache和CounterCache是启用的，而RowCache默认是关闭的。并且这些Cache也会定期存入磁盘中方便下次启动时能快速加热数据

同时Cassandra也保留了一种软删除的特性，称之为tombstone墓碑机制。每当触发删除操作时，数据并不会立即删除，而是使用tombstone来标识，直到SSTable运作压缩时删除。有一个叫做 Garbage Collection Grace Seconds的参数，用以控制多久之后来垃圾回收一个tombstone。

Cassandra内部使用了Bloom Filter 来判断数据是否存在，这个数据结构能够以极小的空间来判断文件存不存在，但数据是不够精确地，即不存在的数据bloom filter肯定是正确的，但存在数据的判断bloom filter数据是不准确的。
