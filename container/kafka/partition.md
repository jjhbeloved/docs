# partition

一个`Partition`内的offset是**全局有序*的,一个Partition分成*多个Segment*, 每个`Segment`的offset也都是*有序*的
Segment与Segment之间的offset也是*有序*的, 所有这些Segment组成的一个Partition就是全局有序的

## producer 选择 partition

如果key为`null`，那么生产者会使用默认的分配器，该分配器使用**轮询（round-robin）算法**来将消息均衡到所有分区。

如果key`不为null` 而且使用的是默认的分配器，那么生产者会对key进行哈希并根据结果将消息分配到特定的分区。

## parition leader

每个topic 由多个 partition 构成, 每个 partition 都有n个 replica, 而 n个 replica 中有一个是 leader. 而 leader/follower 是controller**通过zk选举出来**的.

### replica

- **主副本**(leader replica）: 每个分区都有唯一的主副本，**所有的生产者和消费者请求都由主副本处理**，这样才能保证一致性
- **跟随者副本**(follower replica): 分区的其他副本为跟随者副本，跟随者副本不处理生产者或消费者请求，它们只是**向主副本同步消息**，当主副本所在的broker宕机后，跟随者副本会选举出新的主副本

每个分区除了有主副本之外，还有一个**优选主副本（preferred leader）**，它是**topic初始创建时的主副本**。在主题初始创建时，Kafka使用一定的算法来分散所有主题的主副本与跟随者副本，因此备份主副本通常能保证集群的流量是均衡分布的。如果备份主副本是in-sync状态的，那么在主副本发生故障后，它会自动成为新的主副本

> Leader也会定时地将HW**广播**给所有的followers. 广播消息可以附加在从follower过来的**fetch请求的结果**中.
> 同时, **每个副本**(不管是leader还是follower)也会定时地将**HW持久化**到**自己的磁盘**上.

### replica分配

为了起到备份的效果，简单设想下，如果让我们来分配replica，我们会怎么分配:

1. replica与所备份的节点**不能在一台机器**上，否则就起不到备份的效果
2. replica尽量**均匀的分布**在集群机器上，如果replica全部都在某几台机器上，那么一旦这台机器挂了，会丢失多个partition的备份

## 扩容

[KafKa动态扩容群集-(Topic.partitions迁移)](http://blog.yancy.cc/2017/07/11/Bigdata-hadoop/Kafka/KafKa%E6%89%A9%E5%AE%B9%E7%BE%A4%E9%9B%86(Topic.partitions%E8%BF%81%E7%A7%BB)/)
[Kafka高可用——replica分配方式](https://www.jianshu.com/p/082baf0ebce5)
