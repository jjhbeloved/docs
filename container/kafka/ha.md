# HA

[Kafka系列](http://www.dengshenyu.com/%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F/2017/11/14/kafka-consumer.html)

## 选举算法

replica + ISR

### replica

一个 kafka 共用 n 个 broker(物理服务), 每个 topic 可以设置 partition(并发度).

> 将所有 n broker和待分配的 i 个 partition 排序.
> 将第 i 个 partition 分配到第 (i mod n) 个 broker 上.
> 将第 i 个 partition 的第 j 个副本(replica)分配到第 ((i + j) mod n) 个 broker 上.

客户端生产消费消息都是**只**跟 **leader** 交互, leader是对应partition的概念，**每个 partition** 都有 **一个** leader

### ISR(In-Sync Replicas)

> **zookeeper**的Zab协议算法是这个 **Majority Vote** 的类型，raft也是

如果我们有2f+1个replica(包含Leader和Follower), 那在commit之前必须保证有f+1个Replica复制完消息，为了保证正确选出新的Leader，fail的Replica不能超过f个。因为在剩下的任意f+1个Replica里，至少有一个Replica包含有最新的所有消息

优点: 系统的潜力只取决于最快的几个Broker，而非最慢那个

劣势: 同等数量的机器，它所能容忍的fail的follower个数比较少。比如：如果要容忍1个follower挂掉，必须要有3个以上的Replica，如果要容忍2个Follower挂掉，必须要有5个以上的Replica

结论是: 就是这种算法更多用在Zookeeper这种共享集群配置的系统中使用，而很少在需要**存储大量数据**的系统中使用的原因。

> **kafka**所使用的这种ISR概念的算法更像微软的 **PacificA**算法

优点: 在这种模式下，对于f+1个Replica，一个Partition能在保证不丢失已经commit的消息的前提下容忍f个Replica的失败。

缺点: 相应的，leader需要等待最慢的那个replica，但是Kafka作者认为Kafka可以通过Producer选择是否被commit阻塞来改善这一问题，trade-off就是节省下来的Replica和磁盘。

同步副本算法, 维护一个**ISR列表**, 列表中的follower需要与leader的副本**保持信息一致**.

> 如果Leader不在了，新的 leader 必须拥有原来的 leader commit 过的所有消息. ISR列表里的副本都跟上了 leader，所以就是在这里边**选一个**

producer写数据到leader，副本(replica)有**单独的fetch线程**去拉取**同步消息**。拉取的时间间隔可配置

> **High watermark**（高水位线）以下简称HW，表示消息被leader和ISR内的follow都确认commit写入本地log，所以在HW位置以下的消息都可以被消费（不会丢失）。
> **Log end offset**（日志结束位置）以下简称LEO，表示消息的最后位置。LEO>=HW，一般会有没提交的部分

follower replica会有**单独的线程**（ReplicaFetcherThread），去从leader上去拉去消息同步。当follower的`HW`赶上leader的，就会**保持或加入**到isr列表里，就说明此follower满足上述最基本的原则（跟上leader进度）。isr列表存在zookeeper上, 并且由 **leader 进行管理**. 因为 leader 会跟踪与其保持同步的 replica 列表, 并且由 leader 还可以决定移除落后太多的 replicas

CAP理论，在分布式系统下，要在A高可用性和C一致可靠性做个折衷, 如果Leader在一条消息被commit前等待更多的Follower确认，那在它宕机之后就有更多的Follower可以作为新的Leader，但这也会造成**吞吐率的下降**

producer的ack参数选择，取优先考虑可靠性，还是优先考虑高并发:

- 0 纯异步，不等待，写进socket buffer就继续
- 1 leader写进server的本地log，就返回，不等待follower的回应
- -1 相当于all，表示等待follower回应再继续发消息。保证了ISR列表里至少有一个replica，数据就不会丢失，最高的保证级别

### 有损副本选举

> 设置 `unclean.leader.election.enable: true`，会允许落后副本成为新的主副本

当集群中有多个副本, 除了主副本外, 其他副本都发生故障了. 主副本继续写入, 这时候主副本也不可用了. 必须从 一致性 和 可用性 中做一个选择.
