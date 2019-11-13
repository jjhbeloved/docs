# Interview

[Kafka原理总结](http://www.thinkyixia.com/2017/10/25/kafka-2/)

``` bash
offset（8 Bytes）
消息体的大小（4 Bytes）
crc32（4 Bytes）
magic（1 Byte）
attributes（1 Byte）
key length（4 Bytes）
key（K Bytes）
payload(N Bytes)等等字段, 可以确定一条消息的大小, 即读取到哪里截止
```

## kafka 的作用和优点

作用: 异步处理, 应用解耦, 流量削峰, 消息通讯

优点:

1. 高吞吐, 使用 **顺序写 + 随机读**
   1. **写**. 在 producer 可以选择 `async`和 `sync`(buffer batch, 但是有丟数据风险)以及`compact`, 在broker写入 **pagecache** 定时刷盘(减少了频繁io, 机器crash还是会丢数据)
      1. 为了性能考虑,每个follower当消息被写到内存时就发送ack(而不是要完全地刷写到磁盘上才ack).
      2. 对于每条提交的消息,我们能够保证的是这条消息一定是存储在多个副本(所有ISR)的内存中.并不保证任何副本已经把这条提交的消息持久化到磁盘中.这是基于响应时间和持久性两者平衡的.
   2. **读**. 如果在 **pagecache** 能读到数据直接返回, 如果缺页中断, 读取 disk, 但是使用了 `zero-copy` 技术, 直接将 **pagecache** 数据 `sendfile` 给socket, 减少 系统态->应用态 之间的buffer拷贝.
   3. 采用 一个 topic 多个 parition 来增加 parallelism
   4. todo 还有文件存储位置, 文件格式 也加快了读写
2. 持久(日志保留 log retention), 所有的数据都保存在 disk 上
3. 可靠, `isr`保证数据副本有效. **分区内消息有序, 同一个key保存在同一个partition**.
4. 可扩展
5. 负载均衡
   1. producer根据用户指定的算法, 将消息发送到指定的partition
   2. 存在多个partiiton, 每个partition有自己的replica, 每个replica分布在**不同(必须)**的Broker节点上
   3. 多个partition需要选取出lead partition, lead partition负责读写, 并由zookeeper负责fail over

## 可接收到的最大信息

kafka服务器可以接收到的消息的最大大小是1000000 byte

## zookeeper 在 kafka 中的作用

1. **管理**broker与consumer的动态加入与离开
2. **触发负载均衡**, 当broker或consumer加入或离开时会触发负载均衡算法, 使得一个consumer group内的多个consumer的订阅负载平衡
3. **维护消费关系**及每个partition的消费信息(isr)

## 如何提升远程用户的吞吐量

调整socket的buffer, 减少网络延迟带来的损耗

## kafka 数据重复问题

kafka **保证** `least-once`(至少一次), **不保证** `exactly-once`(仅有一次), 这样在 producer/consumer 都得自己保证尽量不重复数据. 可以通过在消息上带 uuid 等方式解决.

## topic

### topic 的分区是否可以增加/减少

可以增加, 对于基于key计算的主题而言, 建议在一开始就设置好分区数量, 避免以后对其进行调整

不可以减少, 收益太低, 技术成本高. 如何保证顺序性, 事务性, 时间戳 等等.

可以减少 replica, 这个很好理解

### kafka内置 topic

内置的topic都是以 **__ 下划线开头**, `__consumer_offsets` 保存 `group+topic+partition` 的 offset, `__transaction_offsets` 保存 `transaction.id` 的 offset

## kafka 内部原理

需要注意的是, kafka需要为每个partition分配一些内存来缓存消息数据, 如果partition数量越大, 就要为kafka分配更大的heap space

### kafka中有那些地方需要选举？这些地方的选举策略又有哪些

1. `controller` 选 **broker leader**
   1. 使用 zookeeper EPHEMERAL inode 争抢到的是 leader.
2. `producer` 选 **partition replica leader**
   1. 从**重分配**的AR列表中找到第一个存活的副本, 且这个副本在目前的ISR列表中
   2. 发生**优先副本**（preferred replica partition leader election）的选举时, 直接将优先副本设置为leader即可, AR集合中的第一个副本即为优先副本
3. `group coordinator` 选 **consumer leader**
   1. 消费组内还**没有**leader, 那么**第一个加入**消费组的消费者即为消费组的 leader
   2. 原有 leader 退出, 取保存 consumer 的 hashmap 中的 first
4. 分区选举

#### producer 分区分配

如果消息的key**不为null**, 那么默认的分区器会对**key进行哈希**（采用MurmurHash2算法, 具备高运算性能及低碰撞率）, 最终根据得到的哈希值来计算分区号, 拥有相同key的消息会被写入同一个分区

如果key**为null**, 那么消息将会以**轮询的方式**发往主题内的各个可用分区

#### consumer 分区分配

group coordinator 选出 leader, leader 根据选择的分配算法, 分配出 partitions 上报给 coordinator:

- `RangeAssignor(default)`:
  - 按照消费者总数和分区总数进行整除运算来获得一个跨度, 然后将分区按照跨度进行平均分配, 以保证分区尽可能均匀地分配给所有的消费者.
  - n=分区数/消费者数量, m=分区数%消费者数量, 那么前m个消费者每个分配n+1个分区, 后面的（消费者数量-m）个消费者每个分配n个分区.(**把余数的个数均摊到除数上**)
- `RoundRobinAssignor`:
  - 将消费组内所有消费者以及消费者所订阅的所有topic的partition**按照字典序排序**, 然后通过轮询方式逐个将分区以此分配给每个消费者(**当consumer订阅的每个topic的partition不一致的时候, 可能不均衡**)
- `StickyAssignor`:
  - 分区的分配要尽可能的均匀
  - 分区的分配尽可能的与上次分配的保持相同
  - 当两者发生冲突时, 第一个目标优先于第二个目标
  - **更加的优异**

#### broker 分区分配

当 topic 创立的时候, 需要制定 partitioner, partition个数 和 replica 个数(broker必须大于等于replica的个数). 根据 partitioner 进行topic 分区, 并选出每个 partition replica 的 leader(**使用isr选择, 有点随机**).

为**集群**制定**创建主题**时的分区副本**分配方案**, 即在哪个broker中创建哪些分区的副本:

1. 指定了分区个数
2. 未指定机架信息
3. 指定机架信息

### kafka 日志目录结构

kafka的数据都是以**日志形式**顺序存储的. `/{topic}-{partition_num}/{last_offset}.log |{last_offset}.index |{last_offset}.timeindex | leader-epoch-checkpoint`

`{last_offset}`为**上一个segment**的**最后一个消息的偏移**, `log`文件中保存了**log对应的segment的消息信息**, `index`文件中保存了**每个segment下包含的log entry的offset范围**, 然后根据二分查找法就可以快速定位到具体log文件位置. `timeindex`保存的则是**时间索引**

`leader-epoch-checkpoint` 中保存了**每一任leader**开始写入消息时的offset, 会定时更新, follower被选为leader时会根据这个确定哪些消息可用

### 如何查找 topic 中指定的 offset

1. 通过文件名前缀的数字x定位到 offset在哪个文件.
2. 通过 `offset - x`定位到offset在文件的**相对偏移**.
3. 通过`index文件`中记录的索引找到消息在log文件的**最近位置**
4. 从**最近位置**开始逐条寻找

### 日志保留/清除 log retention/clean

- 日志文件太大了需要删除**最旧的**数据, 使得整体的日志文件大小不超过指定的值, 时间指日志文件中的**最大时间戳**而非文件的最后修改时间
- 清理**过期的segment**, 一个 partition下有多个 segment 文件. 一个 partition 下有多个 segment(每个默认是1G大小), 超过大小的 segment 会做分裂.

### log compaction

由于采用追加式更新数据, 会存在大量重复不使用的数据, 写入速度快, 但是消耗存储. 需要使用 压缩的方案 对重复数据和无效数据 做删除.

- 相同 key 的value只保存一个
- 压缩过的是clean, 未压缩的dirty
- 压缩之后的偏移量**不连续**, 未压缩时连续

### 底层存储的理解（页缓存、内核层、块层、设备层） todo

### 控制器 controller 的作用

集群内 **某个broker** 会成为集群控制器（`cluster controller`）, 它负责管理`broker cluster`, 包括**分配分区到 broker, 选举 partition 的 leader/follower, 监控 broker 故障** 等

集群中的**第一个broker**通过在Zookeeper的/controller路径下创建一个**临时节点**来成为控制器, 当其他broker启动时, 也会试图创建一个临时节点, 但是会收到一个"节点已存在"的**异常**, 这样便知道当前已经存在集群控制器. 当控制器发生故障时, 该临时节点会消失, 这些broker便会**收到通知**, 然后尝试去创建临时节点成为新的控制器

对于一个控制器来说, 如果它发现集群中的一个**broker离开**时, 它会检查该broker是否有分区的**主副本**, 如果有则需要对这些分区选举出新的主副本. 控制器会在**分区的副本列表**中选出一个**新的主副本(使用isr选择, 有点随机)**, 并且发送请求给新的主副本和其他的跟随者, 这样新的主副本便知道需要处理生产者和消费者的请求, 而跟随者则需要向新的主副本同步消息。

如果控制器发现有新的broker（这个broker也有可能是之前宕机的）加入时, 它会通过此broker的ID来检查该broker是否有分区副本存在, 如果有则通知该broker向相应的分区主副本同步消息

每当选举**一个新的控制器时**, 就会产生一个新的**更大的控制器时间戳**（`controller epoch`）, 这样集群中的broker收到控制器的消息时检查此时间戳, 当发现消息来源于老的控制器, 它们便会忽略, **以避免脑裂（split brain）**

### 再均衡的原理 rebalance

consumer leader 和 group coordinator 之间**协定**. 通过 heartbeat 来通知 rebalance, 以下是rebalance步骤:

consumer听到rebalance消息后, 关闭原有的partition consumer, 发送 join group, group coordinator选出一个leader. 再发送 sync group(这里等待coordinator返回分配方案), leader 会在request里给出 partition consumer分配方案, coordinator将方案通过 response 下发给所有的consumer.

java 的 heartbeat 是在poll的时候发送的, 如果业务处理时间太长也会导致 rebalance, 可以单独一个 thread 去发送 heartbeat, 但是要控制好异常的处理(每次收到 rebalance 响应时要重置consumer).

### 幂等实现

> 提供了**单会话 单Partition** Exactly-Once语义

producer client 每个 `topic+partition` 申请分配**一个**`PID(producer id)` 和 `sequence number`, 只有当 同一个 PID 内的 sequence number是连续的, 才是幂等的.

幂等条件:

- 只能保证**同一个 session** 内幂等. (原因是基于session设置的pid)
- 只能保证**单个partition**的幂等性. (原因同上)

PID: 使用 zk 进行获取, 节点`/latest_producer_id_block`记录当前 pid段分配的情况, **每个broker**每次请求 topic+partition 分配一个1000范围的**pid段**缓存在server本地, 使用 `pid epoch`保证最新.

client: 在 send 时给**每个message**加上`pid`和`sequence number`, 只有符合 transaction manager 缓存的 seq 起点, 才会 request.(这里还可以对deque做乱序排序). deque 可以缓存 5 个 batch. // todo 是否会直接丢弃呢?

server: 会缓存每个PID对应的 `topic+partion` **最近的5个batch数据**, 作为 连续判断. 如果**不符合**seq连续和pid, 会**直接丢弃**, 并返回异常信息.

### 事务实现

> 提供了`producer`**跨会话 多Partition** Exactly-Once语义
>
> 使用 `transaction maker + PID + TID` 实现
>
> 不提供`consumer`消费一个已经 commit 的事务的所有 msg 都会被消费
>
> 本质是, 将一组写操作（如果有）对应的消息与一组读操作（如果有）对应的Offset的更新进行同样的标记（即Transaction Marker）来实现事务中涉及的所有读写操作同时对外可见或同时对外不可见

1. kafka **没有**引入全局事务.
2. producer 挂了, 通过**用户主动标识**的 `transaction.id` 的相同的producer进行关联.
3. 使用 `2PC` 引入 `事务协调器(transaction coordinator)`
4. 使用**单独的 topic** 保存`事务日志(transaction log)`
5. 使用 `control message` 保存**事务状态**
6. 使用 `producer epoch` 避免同时active状态的producer具有同一个transactionId(意思是可以一个transactionId有多个producer, 但是只能有一个是active)
7. consumer 基于 read_commited/unread_commited 来决定是否消费事务. 如果消费事务, 需要**客户端缓存事务消息**, **等待control message**告知是abort还是commit.

### kafka不支持读写分离是为什么

- **主写从读**可以让从节点去**分担**主节点的**负载**压力, 预防主节点负载过重而从节点却空闲的情况发生
- **主写从读**也有 2 个很明 显的缺点:
  - **数据一致性**
  - **延时问题**, redis同步过程是`历网络 -> 主节点内存 -> 网络 -> 从节点内存`, kafka同步过程是`网络 -> 主节点内存 -> 主节点磁盘 -> 网络 -> 从节点内存 -> 从节点磁盘`需要磁盘操作非常耗时.

> kafka 由于采用 partition 的方案, 在创建 topic 的时候尽可能的在算法层面让 partition 分配在不同的 broker 上, 这样即便是 主写主读, 也足够均衡. 
> 好处: 代码实现逻辑简单. 将负载粒度细化均摊, 用户可控制负载. 没有延时的影响. 在副本稳定的情况下, 不会出现数据不一致的情况

### kafka在可靠性方面做了哪些改进

基于 replica 提供了可靠保证. 使用了 ack 来保证一致性, 只有isr broker 都commit, 才会变更 `HW`. 在早期, 直接使用 `HW` 做可靠性恢复, 会出现 **数据丢失** 和 **leader/follower 数据不一致**的情况. 引入 `leader epoch` 解决问题.

### kafka中怎么实现死信队列和重试队列

// todo 这里不是很理解.

Kafka连接器可以配置为将无法处理的消息（例如上面提到的反序列化错误）发送到一个**单独的kafka主题**, 即**死信队列**, 有效消息会正常处理, 管道也会继续运行. 可以从死信队列中检查无效消息, 并根据需要忽略或修复并重新处理.

kafka 不支持延迟消息. 自己建立一个 topic 来消费.(msg可以带上 retries 和 nextRetryTime)

retry kafka 只会针对返回是 `RetriableException` 和 `TransactionManager` 执行 retry 请求. 会在 `accumulator.reenqueue()` 加入 deque  执行重试.(??这个就是重试队列?)

### 延时操作的原理

[kafka技术内幕读书笔记（五）：延迟操作](https://blog.csdn.net/qq_36642340/article/details/82562194)

服务端在处理客户端的请求, 针对不同的请求, 可能**不会立即返回响应**结果给客户端(没有消息可以被poll, replica poll比较慢). 在处理这类请求时, 服务端会为这类请求**创建延迟操作对象**放入**延迟缓存队列**中. 延迟缓存的数据结构类似MAP, 延迟操作对象从延迟缓存队列中完成并移除有两种方式:

1. 延迟操作对应的外部事件发生时, 外部事件会尝试完成延迟缓存中的延迟操作
2. 如果外部事件仍然没有完成延迟操作, **超时**时间达到后, 会强制完成延迟的操作

- **延迟加入 DelayedJoin**
  - 组协调器等待当前消费组下所有的消费者都请求加入消费组
  - 协调者不会为每个消费者的加入组请求都创建一个延迟操作, 而是仅当消费组状态从 稳定 转变为 准备再平衡, 才创建一个延迟操作对象
- **延迟心跳 DelayedHeartbeat**
  - 协调者为**每个消费者**都创建一个**延迟心跳**, 并监控每个消费者是否存活
- **延时创建主题 DelayedCreateTopics**, 等待该主题的所有分区副本分配到leader后调用回调函数返回给客户端
- **延迟消费/拉取 DelayedFetch**, 目的是为了让每次拉取消息时可以获取到**指定大小**的数据
  - 超时事件, 等到超时时间之后触发第二次读取日志文件的操作
  - 外部事件
    - follow poll, 当leader新增了log
    - consumer poll, `HW` 增长
- **延迟生产 DelayedProduce**, 协助副本管理器在满足所有副本同步完消息后再向客户端做出响应. 等待 replica 拉取成功, `HW` 增长, 每次增长都会检测是否能完成此次延迟操作.

### kafka中的延迟队列怎么实现, 作用

使用**时间轮**定时器实现：

1. 任务的**添加与移除**, 都是`O(1)`级的复杂度, 同一时间的事件, 在时间轮的一格上, 每一格是一个 timerTaskList, 里面 task 的时间在这一格的范围内. 最低一层的轮 **有序**.
2. 节约资源
3. 只需要有**一个线程**去**推进时间轮**(基于DelayedQueue), 每当时间推进, 时间轮的起点会不断的轮询前进, 高层的数据也会被分解到低层去, 实现类似 **手表** 的轮询. 存储所需的**空间小**, 高延迟的信息被模糊放至到高层. 并且空间会动态扩容/缩容(前提是 延迟的信息不总特别多). **超过**第一层的数据, 存储在**磁盘**上
4. 将**超过定长时间**的延迟消息从CommitLog中剥离出来, 独立存储以保存**更长的时间**, 将WAL中的延迟消息写入到独立的文件中。这些文件按照延迟时间组成一个链表, 这里会有Delay Msg File带来的**随机写**问题, 可以接受.(rocket mq 的事物消息, 可能就使用了这种方式. 为了保存大量文件, 存在**随机读**的问题), 做一个 Delay Msg File 到 Delay Msg Message 的 mapping index, 加快读

作用: 提供延迟操作

#### 延迟队列

> 延迟队列: 消息发送到队列后, 需要延迟多久用户才可见消息.

难点:

- **排序**, 比如用户**先**发了一条延迟**1分钟**的消息, 一秒**后**发了一条延迟**3秒**的消息, 显然延迟**3秒**的消息需要**先**被投递出去。那么服务端在收到消息后需要对消息进行排序后再投递出去. 在MQ中, 为了保证可靠性, 消息是需要顺序落盘的, 且对性能和延迟的要求, 决定了在**服务端**对消息进行**排序**是完全**不可接受**的

- **存储**, 目前MQ的方案中都是基于**WAL**的方式实现的（RocketMQ、Kafka）, 日志文件会被过期**删除**, 一般会保留最近一段时间的数据。支持任意级别的延迟, 那么需要保存最近30天的消息, 存储hold不住.

使用 **timer wheel 时间轮**数据结构来解决排序和存储的问题. 时间轮分层, 每层固定**20个格子(wheelSize)**, 每个格子**固定时间跨度(tickMs)**, `高层的时间跨度=低层总时间跨度=wheelSize*tickMs`, 如果**3层**结构, 能承载 `20 * 20 * 20 = 8000` 个时间跨度. 并且会随着指针 `currentTime` 循环利用.

### 怎么计算Lag

read_uncommitted: `lag = HW/LEO - ConsumerOffset/CurrentPoint`
read_committed: `lag = LSO - ConsumerOffset/CurrentPoin`

LSO 的值等于事务中第一条消息的位置(firstUnstableOffset), 对已完成的事务而言, 它的值同 HW 相同, 所以我们可以得出一个结论: `LSO <= HW <= LEO`

### kafka中怎么做消息审计

### kafka中怎么做消息轨迹

### kafka有哪些指标需要着重关注

1. 文件存储机制设计
2. lag

## 什么是 isr, osr, ar

- isr: In-Sync Replicas 副本同步队列
- osr: Out-Sync Replicas 副本不同步队列
- ar: Assigned Replicas 所有副本

> ar = isr + osr

isr 是由 **leader 维护**的. 当检查到 replica 为**失效副本**时, 从 isr 中移除, 存入 `osr(Outoff-Sync Replicas)`, **新加入的follower**也会先存放在 osr 中

每个partition都有一个isr, 保存所有可以作为主副本的replica列表信息在**zookeeper**上.

### 如何减少isr中的扰动, replica什么时候离开isr

每个 `topic+partition` 都有一个 **__consumer_offset** 记录, **consumer** 改变 `LCO + CP`, **producer** 改变 `LEO + HW`,  包含:

1. **LCO**(last commited offset), consumer group 最新一次 commit 的 offset, 表示这个 group 已经把 LCO 之前的数据都消费成功了
2. **CP**(consumer offset/current position), consumer group 最后一次consume, 但是还没 commit.
3. **LEO**(last end offset), producer 写入到 Kafka 中的最新一条数据的 offset+1 下一条
4. **HW**(high watermark), 取一个partition对应的 isr 中 **最小的LEO** 作为 HW, consumer **最多**只能消费到HW所在的位置的下一条

commit机制会保证**所有的follower ack**后(但是ack的follower数据并不一定flush到disk上, 可能还是保存在 pagecache 中), leader 才commit, 但是由于**messages是batch**的, leader内部会存在队列缓存. 这时候 leader可能会比 follower高. 因此这个阈值不要设置太小.

### 失效副本

follower 在fetch 后会更新该 replica 的 lastCaughtUpTimeMs 标识。Kafka的**副本管理器（ReplicaManager）**启动时会启动一个副本过期检测的定时任务, 而这个定时任务会定时检查**当前时间**与**副本的lastCaughtUpTimeMs**差值是否大于参数`replica.lag.time.max.ms`指定的值, 如果大于, 会从 isr 移除, 当 replica 符合 isr 要求了, 又会加入.

当 leader replica 检测到 isr 中 leader 的 HW **高过** follower 的 HW 超过设定的**阈值** leader 就会从 isr 中把这个 replica 移除

频繁的存在失效副本, 一般是cluster出现故障了:

1. **资源瓶颈**, CPU的使用率、网络流入/流出速率、磁盘的读/写速率、iowait、ioutil等, 也可以适当的关注下文件句柄数、socket句柄数以及内存等方面。
2. **负载不均衡**

isr 内所有的 replica都故障了, 存在数据不一致. 这时候要从 **一致性**(等待最高的replica恢复) 和 **可用性**(立即恢复其中一个replica) 选择. kafka提供了一个配置来选择C或者A.

### 副本在 isr 中停留了很长时间表明什么

### 首选/优先副本不在isr中会发生什么

> 所谓的优先副本是指在Kafka的 AR列表 中的**第一个副本**。理想情况下, 优先副本就是该分区的leader副本, 所以也可以称之为`preferred leader`
> 优先副本的选举是指通过**自动或者手动的方式**促使优先副本选举为leader, 也就是分区平衡, 这样可以促进集群的均衡负载, 也就进一步的降低失效副本生存的几率

如果不在 isr 中, **controller** 将无法将 leadership 转移到首选的副本, 需要重新选举出新的 leader.

## 分区器、序列化器、拦截器

- 分区器
- 序列化器
- 拦截器

> 拦截器 -> 序列化器 -> 分区器

## 生产者客户端的整体结构

### 生产者客户端中使用了几个线程来处理

2个, **主线程(业务线程)**和**sender线程**

- **主线程(业务线程)** 负责创建消息, 然后通过分区器、序列化器、拦截器作用之后缓存到累加器RecordAccumulator中
- **sender线程** 负责将RecordAccumulator中消息通过 `channel + selector` 发送到kafka中, 这里不配置, 默认会做batch 提交. 可能存在数据丢失

## 消费者客户端整体结构

### 消费组中的消费者个数如果超过topic的分区, 那么就会有消费者消费不到数据是否正确

是的.

- 线程数量多于Partition的数量, 有部分线程无法消费该topic下任何一条消息
- 线程数量少于Partition的数量, 有一些线程会消费多个Partition的数据
- 线程数量等于Partition的数量, 则正好一个线程消费一个Partition的数据

### 消费者提交消费位移的值

// todo 不同的 commit 方式, offset 不一样

提交的是当前消费到的最新消息的 `offset+1`

### 重复消费/漏消费

consumer **消费了却没有 commit** 会导致重复消费.

consumer **没有处理完业务/没有处理业务 就commit** 会导致漏消费.

### consumer是非线程安全的, 那么怎么样实现多线程消费

//todo 是不是多线程对一个consumer进行消费consume, 会有线程安全问题

**单线程**创建 consumer, 将 consume 数据推入**内部队列**, **多线程**消费**队列**处理业务(这里可能要考虑消息**顺序性**, 因为多线程处理可能会**导致消息无序**)

## 在使用kafka的过程中遇到过什么困难？怎么解决的

出现consumer 不消费. rebalance 消息通知监听 没处理导致, consumer close了没有reconnect

// todo 这里还可以解析下 commitAsync/commitSync 如果 rebalance 可能会存在丢数据, 需要监听这个notify 来主动commit

## 怎么样才能确保Kafka极大程度上的可靠性

1. ack=all(等待所有的replica反馈poll数据成功)
2. flush disk间隔设置为0(不在pagecache做缓存, 立即flush磁盘)

## 还用过什么同质类的其它产品, 与kafka相比有什么优缺点

rocketmq
