# Quorum

## 1. 介绍

在分布式系统中有个CAP理论，对于**P（分区容忍性）**而言，是实际存在 从而**无法避免**的。因为，分布系统中的处理不是在本机，而是网络中的许多机器相互通信，故网络分区、网络通信故障问题无法避免。因此，只能尽量地在C 和 A 之间寻求平衡。对于数据存储而言，为了**提高可用性（Availability）**，采用了**副本备份**，比如对于HDFS，默认每块数据存三份。某数据块所在的机器宕机了，就去该数据块副本所在的机器上读取（从这可以看出，数据分布方式是按“数据块”为单位分布的）

当需要修改数据时，就需要更新所有的副本数据，这样才能保证数据的一致性（Consistency）。因此，就需要在**C(Consistency) 和 A(Availability) 之间权衡**。

Quorum机制，就是这样的一种权衡机制，一种将**“读写转化”的模型**

Quorum机制是**抽屉原理**的一个应用。定义如下：**假设有N个副本，更新操作wi 在W个副本中更新成功之后，才认为此次更新操作wi 成功**。称成功提交的更新操作对应的数据为：“成功提交的数据”。**对于读操作而言，至少需要读R个副本才能读到此次更新的数据**。其中，W+R>N ，即W和R有重叠。一般，`W+R=N+1`.(**W, R 是根据集群规模自动计算的**)

### 1.1 极端WARO机制

**WARO(Write All Read one)**是一种简单的副本控制协议，当Client请求向某副本写数据时(更新数据)，只有当所有的副本都更新成功之后，这次写操作才算成功，否则视为失败。**WARO牺牲了更新服务的可用性，最大程度地增强了读服务的可用性**

从这里可以看出两点：

1. 写操作很脆弱，因为只要有一个副本更新失败，此次写操作就视为失败了。
2. 读操作很简单，因为，所有的副本更新成功，才视为更新成功，从而保证所有的副本一致。这样，只需要读任何一个副本上的数据即可

假设有N个副本，N-1个都宕机了，剩下的那个副本仍能提供读服务；但是只要有一个副本宕机了，写服务就不会成功

## 2. 机制分析

### 2.1 Quorum机制无法保证强一致性

1. 如何读取最新的数据? 在已经知道最近成功提交的数据版本号的前提下，最多读R个副本就可以读到最新的数据了。
2. 如何确定 最高版本号 的数据是一个成功提交的数据? 继续读其他的副本，直到读到的 最高版本号副本 出现了W次。

### 2.2 基于Quorum机制选择primary

中心节点(服务器)**读取R个副本**，选择R个副本中版本号**最高的副本**作为新的`primary` .新选出的primary不能立即提供服务，还需要与**至少与W个副本完成同步**后，才能提供服务, 为了保证Quorum机制的规则：**W+R>N**

至于如何处理同步过程中冲突的数据，则需要视情况而定。比如: `(V2，V2，V1，V1，V1），R=3`

1. 如果读取的3个副本是(V1，V1，V1)则高版本的 V2需要丢弃。
2. 如果读取的3个副本是（V2，V1，V1），则低版本的V1需要同步到V2

## 3. Quorum机制应用实例

### 3.1 HDFS

activie namenode/ standby nanmenode的高可用. 依赖 `Journal Nodes Cluster`.

activie namenode通过`EditLog`来同步数据. 在往本地write的同时, 也会**并行**地向 JournalNodeCluster之中的**每一个JournalNode**发送写请求, 只要大多数(majority) 的 JournalNode 节点返回成功就认为向 JournalNode 集群写入 EditLog 成功。如果有 `2N+1` 台 JournalNode，那么根据大多数的原则，最多可以容忍有 `N` 台 JournalNode 节点挂掉。

> Quorum机制: **每次写入JournalNode的机器数目达到大多数(W)时，就认为本次写操作成功了**

当Active NameNode宕机时，StandBy NameNode 向JournalNode**同步EditLog**，从而保证了HA

> Active NameNode 向 JournalNode 集群提交 EditLog 是**同步**的
> 但 Standby NameNode 采用的是**定时异步**从 JournalNode 集群上同步 EditLog 的方式，那么 Standby NameNode 内存中文件系统镜像有很大的可能是落后于 Active NameNode 的，
> 所以 Standby NameNode 在转换为 Active NameNode 的时候需要把**落后的 EditLog 补上来**

----

## reference

[分布式系统理论之Quorum机制](https://www.cnblogs.com/hapjin/p/5626889.html)