# idempotent

[Kafka 事务性之幂等性实现](http://matt33.com/2018/10/24/kafka-idempotent/)

## producer 幂等范围

1. 单个producer **同一个session** 之内保证消息 不丢不重复. (因为幂等是靠 producerId + sequenceNumber保证的)
2. **只能**保证**单个partition**的幂等性. (如果要跨 partition, 需要事务介入)

## producer 使用幂等

1. 设置 `ack = all`
2. 设置 `props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true")`

## 要解决的问题

在一些情况下, 可能会导致数据重复, 比如：网络请求延迟等导致的重试操作, 在发送请求重试时 Server 端并不知道这条请求是否已经处理（没有记录之前的状态信息）, 所以就会有可能导致数据请求的重复发送, 这是 Kafka 自身的机制（**异常时请求重试机制**）导致的数据重复

## 原理

> **at least once + 幂等 = exactly once**

幂等, 要解决以下问题:

1. 如何鉴别重复? **唯一键/唯一id** 来判断, 这时候系统一般是需要**缓存已经处理的唯一键记录**, 这样才能更有效率地判断一条数据是不是重复；
2. 唯一键选择的粒度? 核心的解决思路依然是 **分而治之**, 数据密集型系统为了实现分布式都是有分区概念的, 而**分区**之间是有相应的隔离
3. 选择的粒度是否有问题? 一个partition由多个client写入, 如果做 partition 级别, 各 client 之间很难相互感知. 使用 `client+partition` 粒度, 各个client之间完全独立(每个client分配一个独立的PID).

实现的两个重要机制:

1. `PID` (producer id), 识别每个 producer client
2. `sequence number`, client 发送的**每条消息**都会带相应的 sequence number, server 端就是根据这个值来**判断数据是否重复**

### PID

每个 Producer 在初始化时都会被分配一个唯一的 PID, 这个 PID 对**应用透明**的, 完全没有暴露给用户. 对于一个给定的 PID, sequence number 将会**从0开始自增**, 每个 Topic-Partition 都会有一个独立的 sequence number. Producer 在发送数据时, 将会给**每条 msg** 标识一个 **sequence number**, server 也就是通过这个来验证数据**是否重复**. 这里的 PID 是**全局唯一**的, Producer 故障后重新启动后会被分配一个新的 PID, 这也是幂等性无法做到跨会话的一个原因

Producer PID 申请:

调用了 TransactionCoordinator

Server PID 管理:

- 在**本地的 PID 段**用完了或者处于新建状态时, 申请 **PID 段**（默认情况下, 每次申请 `1000` 个 PID）
- TransactionCoordinator 对象通过 generateProducerId() 方法获取下一个可以使用的 PID

PID 端申请是向 ZooKeeper 申请, zk 中有一个 `/latest_producer_id_block` 节点, 每个 Broker 向 zk 申请一个 **PID 段**后, 都会把自己申请的 PID 段信息**写入到这个节点**, 这样当其他 Broker 再申请 PID 段时, 会首先读写这个节点的信息, 然后根据 block_end 选择一个 PID 段, 最后再把信息写会到 zk 的这个节点, 这个节点信息格式如下所示：

``` json
{"version":1,"broker":35,"block_start":"4000","block_end":"4999"}
```

## 处理流程

![idemoptent process](./img/kafka-idemoptent.png)

KafkaProducer 在初始化时会初始化一个 TransactionManager 实例, 它的作用有以下几个部分:

- 记录本地的**事务状态**（事务性时必须）
- 记录一些**状态信息**以保证幂等性, 比如: 每个 topic-partition 对应的下一个 sequence numbers 和 last acked batch（最近一个已经确认的 batch）的最大的 sequence number 等
- 记录 ProducerIdAndEpoch 信息（**PID** 信息）

### producer client process

1. 应用通过 KafkaProducer 的 `send()` 方法将数据添加到 RecordAccumulator 中, 添加时会判断是否需要新建一个 ProducerBatch, 这时这个 ProducerBatch 还是**没有** PID 和 sequence number 信息的
2. `Sender.run()`会先根据 TransactionManager 的 `shouldResetProducerStateAfterResolvingSequences()`方法判断当前的 PID **是否需要重置**, 重置的原因: 如果有 topic-partition 的 batch **重试多次失败**最后因为超时而**被移除**, 这时 sequence number 将**不连续**, 因为 sequence number 有部分已经分配出去, 这时系统依赖自身的机制无法继续进行下去（因为幂等性是要保证不丢不重的）, 相当于程序遇到了一个 fatal 异常. **PID 会进行重置**, TransactionManager 相关的**缓存信息被清空**（Producer 不会重启）, 只是保存状态信息的 TransactionManager 做了 `clear+new` 操作, 遇到这个问题时是**无法保证 exactly once**（有数据已经发送失败了, 并且超过了重试次数）
3. `Sender.sendProducerData()`发送数据, 在`RecordAccumulator.drain()`有不同:
   1. 当前这个 `topic-partition` 的数据出现过**超时**, **不能发送**, 如果是新的 batch 数据直接跳过(没有 seq  number 信息)
   2. 常规的判断: 判断这个 topic-partition 是否可以继续发送（如果出现前面2中的情况是不允许发送的）, 判断 PID 是否有效, 如果这个 batch 是重试的 batch, 那么需要判断这个 batch 之前是否还有 batch 没有发送完成, 如果有, 这里会先跳过这个 Topic-Partition 的发送, 直到前面的 batch 发送完成. 最坏情况下, 这个 Topic-Partition 的 in-flight request 将会减少到1（这个涉及也是考虑到 server 端的一个设置, 文章下面会详细分析）
   3. 如果 batch 还没有这个相应的 PID 和 sequence number 信息, 会在这里进行相应的设置
4. `Sender.sendProduceRequests()`
5. `Sender.reenqueue()` 会对可以恢复的**乱序进行重排**

### producer server process

1. 检查`TXN.id` 有 Write 权限
2. 设置了**幂等性**, 检查是否对 ClusterResource 有 `IdempotentWrite` 权限
3. 验证对 topic 是否有 Write 权限以及 Topic 是否存在
4. 检查是否有 PID 信息, 没有的话走正常的写入流程
5. `log.analyzeAndValidateProducerState()`方法先根据 batch 的 `sequence number` 信息检查这个 batch **是否重复**（server 端会**缓存 PID 对应这个 Topic-Partition 的最近5个 batch 信息**）, 如果**有重复**, 这里当做写入成功返回（**不更新** log 对象中相应的**状态信息**, 比如这个 replica 的 the end offset 等）
6. 有了 PID 信息, 并且**不重复** batch 时, 在更新 producer 信息时, **会缓存最近5个batch信息(`MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION=5`)**, 会做以下校验:
    1. 检查该 PID 是否已经缓存中存在（主要是在 ProducerStateManager 对象中检查）
    2. 如果不存在, 那么判断 sequence number 是否 从0 开始, 是的话, 在缓存中记录 PID 的 meta（PID, epoch,  sequence number）, 并执行写入操作, 否则返回 UnknownProducerIdException（PID 在 server 端已经过期或者这个 PID 写的数据都已经过期了, 但是 Client 还在接着上次的 sequence number 发送数据）
    3. 如果该 PID 存在, 先**检查 PID epoch** 与 server 端记录的是否相同
    4. 如果不同并且 sequence number 不从 0 开始, 那么返回 OutOfOrderSequenceException 异常
    5. 如果不同并且 sequence number 从 0 开始, 那么正常写入
    6. 如果**相同**, 那么根据缓存中记录的最近一次 sequence number（currentLastSeq）**检查是否为连续**（会区分为 0、Int.MaxValue 等情况）, 不连续的情况下返回 OutOfOrderSequenceException 异常

## 思考

### 为什么要求 MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION 小于等于5

Server 端的 ProducerStateManager 实例会**缓存每个 PID** 在每个 **Topic-Partition** 上发送的**最近 5 个batch 数据**（这个 5 是写死的, 至于为什么是 5, 可能跟经验有关, 当不设置幂等性时, 当这个设置为 5 时, 性能相对来说较高, 社区是有一个相关测试文档, 忘记在哪了）, 如果超过 5, ProducerStateManager 就会将最旧的 batch 数据**清除**

如果有6个值, 只能缓存2,3,4,5,6, 1 发送失败retry, batch 不重复, 但是 sequence number 无序, 返回 OutOfOrderSequenceException 异常. 直到超过最大重试次数或者超时, 这样不但会影响 Producer 性能, 还可能给 Server 带来压力.

> 可以细化 OutOfOrderSequenceException 异常解析, 如果 sequence number 小于 next sequence, 一定是重复数据.

### 当 MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION 配置等于1时, 是否保证有序

同时只允许一个请求正在发送, 只有当前的请求发送完成（成功 ack 后）, 才能继续下一条请求的发送, 类似**单线程**处理这种模式, **效率非常差**, 但是可以解决乱序的问题

当出现重试时, max-in-flight-request 可以**动态减少到 1**, 在正常情况下还是按 5.

`client`当请求出现**重试**时, batch 会**重新添加**到队列中, 这时候是根据 sequence number 添加到队列的**合适位置**（有些 batch 如果还没有 sequence number, 那么就保持其相对位置不变）, 也就是**队列中排**在这个 batch 前面的 batch, 其 sequence number 都比这个 batch 的 sequence number 小.

总结:

- **Server 端**验证 batch 的 sequence number 值, **不连续**时, **直接返回**异常
- **Client 端**请求重试时, batch 在 **reenqueue** 时会根据 sequence number 值放到合适的位置（有序保证之一）
- **Sender 线程**发送时, 在遍历 queue 中的 batch 时, 会检查这个 batch 是否是重试的 batch, 如果是的话, 只有这个 batch 是最旧的那个需要重试的 batch, 才允许发送, 否则本次发送跳过这个 Topic-Partition 数据的发送等待下次发送
