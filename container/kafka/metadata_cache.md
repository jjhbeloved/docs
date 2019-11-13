# metadata cache

kafka broker 是否是无状态的? 答案: 是的. 因为`offset`是由**consumer**控制

## 什么是metadata cache

> metadata cache: **每台broker**在内存中都维护了集群上所有节点和topic分区的**状态信息**

1. 每台broker都保存**相同的**cache.
2. kafka的metadata cache是采用异步更新请求, 弱一致性. 之所以影响不大
   1. client本地会缓存metadata
   2. 如果metadata过期了, client会retry
   3. 缓存很小
3. 只要集群中有broker或分区数据**发生了变更**就需要更新这些cache, 通过**controller**
   1. 当有新broker启动时，它会在Zookeeper中进行注册，此时监听Zookeeper的controller就会立即感知这台新broker的加入，此时controller会更新它自己的缓存（注意：这是controller自己的缓存，不是本文讨论的metadata cache）把这台broker加入到当前broker列表中
   2. 之后它会发送UpdateMetadata请求给集群中所有的broker(也包括那台新加入的broker)让它们去更新metadata cache。一旦这些broker更新cache完成，它们就知道了这台新broker的存在，同时由于新broker也更新了cache，故现在它也有了集群所有的状态信息

## 可能存在的问题

现在更新cache完全由**controller**来驱动，故controller所在broker的负载会极大地影响这部分操作（实际上，它会影响所有的controller操作）。根据目前的设计，controller所在broker依然作为一个普通broker执行其他的clients请求处理逻辑，所以如果controller broker一旦忙于各种clients请求(比如生产消息或消费消息)，那么这种更新操作的请求就会积压起来(backlog)，造成了更新操作的延缓甚至是被取消。

究其根本原因在于当前controller对待数据类请求和控制类请求并**无任何优先级化处理**——controller一视同仁地对待这些请求，而实际上我们更希望controller能否赋予控制类请求更高的优先级

### 名词解释

- **broker**: kafka服务器
- **controller**: 控制器。**选举broker作为controller**，管理和协调kafka集群，和zookeeper通信
- **coordinator**: 协调者。用于实现成员管理、消费分配方案制定(rebalance)以及提交位移等，**每个group选举一个broker作为consumer的协调者**, 实际上就是一个**broker**
- **partition**: 分区，消息实际存储的物理位置。保存在磁盘中的有序队列，维护offset
- **ISR(is-sync replica)**: 同步副本集合。如果follower延迟过大，会被踢出集合，追赶上数据之后，重新向leader申请，加入ISR集合。并不是所有的follower都可以成为leader，ISR集合中的follower可以竞选leader。通过replica.lag.time.max.ms(默认10s)设置follower同步时间，通过RetchRequest(offset)同步leader信息
