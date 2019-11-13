# data consistency

[Data consistency](https://docs.datastax.com/en/cassandra/3.0/cassandra/dml/dmlTransactionsDiffer.html#Atomicity)

## How are consistent read and write operations handled

Consistency refers to how up-to-date and synchronized all replicas of a row of Cassandra data are at any given moment. **Ongoing repair operations in Cassandra ensure that all replicas of a row will eventually be consistent. Repairs work to decrease the variability in replica data, but constant data traffic through a widely distributed system can lead to inconsistency (stale data) at any given time**. Cassandra is a AP system according to the CAP theorem, providing high availability and partition tolerance. Cassandra does have flexibility in its configuration, though, and can perform more like a CP (consistent and partition tolerant) system according to the CAP theorem, depending on the application requirements. Two consistency features are tunable consistency and linearizable consistency.

意思是, cassandra使用repair operations机制, 来保证行数据的一致. 这样可以减少副本的可变性, 但是会导致段时间内的不一致.

### Tunable consistency 可协调一致性

Cassandra **extends** the concept of **eventual consistency** by offering **tunable consistency**. You can vary the consistency for individual **read or write** operations so that the data returned is more or less consistent, as required by the client application. This allows you to make Cassandra act more **like a `CP (consistent and partition tolerant)`** or **`AP (highly available and partition tolerant)`** system according to the CAP theorem

**For read operations**, the read consistency level specifies how many replicas must respond to a read request before returning data to the client application. If a read operation reveals inconsistency among replicas, Cassandra initiates a read repair to update the inconsistent data.

**For write operations**, the write consistency level specified how many replicas must respond to a write request before the write is considered successful. Even at low consistency levels, Cassandra writes to all replicas of the partition key, including replicas in other datacenters. The write consistency level just specifies when the coordinator can report to the client application that the write operation is considered completed. Write operations will use hinted handoffs to ensure the writes are completed when replicas are down or otherwise not responsive to the write request.

**common practice** is to **write** at a consistency `level of QUORUM` and **read** at a consistency `level of QUORUM`

意思是, 默认是**AP系统**, 通过配置, 也可以是一个**CP系统**

### Linearizable consistency 线性一致性

In ACID terms, **linearizable consistency** (or serial consistency) is a **serial (immediate) isolation level** for **lightweight transactions**. Cassandra **does not use** employ traditional mechanisms like `locking or transactional dependencies` when **concurrently updating multiple rows or tables**.

Serial operations for these elements can be implemented in Cassandra with the **Paxos consensus protocol**, which uses a `quorum-based algorithm`. Lightweight transactions can be implemented **without** the need for **a master database** or **two-phase commit process**.

Lightweight transaction **write operations** use `the serial consistency level` for **Paxos consensus** and the regular consistency level for the **write to the table**.

意思是, 不使用 locking 和 事物依赖, 而是使用基于**paxos语义**的线性一致性

### Calculating consistency

**Strong consistency(强一致性)** can be guaranteed when the following condition is true:
> R + W > N
where

- R is the consistency level of read operations
- W is the consistency level of write operations
- N is the number of replicas

**Eventual consistency(最终一致性)** occurs if the following condition is true:
> R + W =< N
where

- R is the consistency level of read operations
- W is the consistency level of write operations
- N is the number of replicas

## How are Cassandra transactions different from RDBMS transactions

Cassandra **does not support** `joins` or `foreign keys`, and consequently does not offer consistency in the ACID sense.

### Atomicity

In Cassandra, a write operation is **atomic at the partition level**, meaning the insertions or updates of two or more rows in the same partition are treated as one write operation. A delete operation is also atomic at the partition level.

Cassandra reports a failure to replicate the write on that node. However, the replicated write that succeeds on the other node is **not automatically rolled back**.

Cassandra uses client-side timestamps to determine the most recent update to a column.

意思是. 原子性是基于**分区级别**的, **同一分区的写**, 都是原子性的. 但是隔离级别要求的不满足写入会失败, 是**不会做回滚操作**的.

### Isolation

Cassandra write and delete operations are performed with **full row-level isolation**. This means that a write to a row within a single partition on a single node is **only visible to the client performing the operation** – the operation is restricted to this scope **until it is complete**. All updates in a batch operation belonging to a given partition key have the same restriction. However, a Batch operation is **not isolated** if it includes changes to **more than one partition**.

意思是, 写操作是基于**操作用户的隔离级别(操作用户可见)**, 但是操作**超过了一个分区(跨分区)**, 那么就**没有隔离级别**了

### Durability

Writes in Cassandra are durable. All writes to a replica node are recorded **both** `in memory` and `in a commit log` on disk before they are acknowledged as a success.

You can manage the local durability to suit your needs for consistency using the commitlog_sync option in the cassandra.yaml file. Set the option to either periodic or batch.

意思是, 持久性在写数据时 同时要写**内存**和**commitLog**都成功了, 才算成功

## How do I accomplish lightweight transactions with linearizable consistency

Linearizable consistency ensures transaction isolation at a level **similar** to the `serializable level` offered by RDBMSs.

This type of transaction is known as `compare and set (CAS)`; replica data is compared and any data found to be out of date is set to the most consistent value. In Cassandra, the process combines **the Paxos protocol with normal read and write operations** to accomplish the compare and set operation.

The Paxos protocol is implemented as a series of phases:

1. **Prepare/Promise**
2. **Read/Results**
3. **Propose/Accept**
4. **Commit/Acknowledge**

For simplicity, this description will use only one `proposer`. A proposer `prepares` by sending a message to a quorum of acceptors that includes a **proposal number**. Each `acceptor` `promises` to accept the proposal if **the proposal number is the highest** they have received. Once the proposer receives a quorum of acceptors who promise, the value for the proposal is `read` from each acceptor and `sent back to` the proposer. The proposer **figures out** which value to **use** and `proposes` the value to a quorum of the acceptors along with the proposal number. Each acceptor `accepts` the proposal with a certain number if and only if the acceptor is **not already promised** to a proposal with a high number. The value is `committed` and `acknowledged` as a Cassandra write operation if all the conditions are met.

Lightweight transactions will **block other lightweight transactions** from occurring, but will **not stop normal read and write operations** from occurring. Lightweight transactions use a timestamping mechanism different than for normal operations and mixing LWTs and normal operations can **result in errors**.

### Reads with linearizable consistency

A SERIAL consistency level allows reading the current (and possibly uncommitted) state of data without proposing a new addition or update. If a SERIAL read finds an uncommitted transaction in progress, Cassandra performs a **read repair** as part of the commit.

## How is the consistency level configured

### Write consistency levels

### Read consistency levels

- **ALL**, Provides **the highest consistency** of all levels and the lowest availability of all levels.
  - Returns the record after **all replicas** have responded.
- **QUORUM**, Used in either single or multiple datacenter clusters to maintain **strong consistency** across the cluster.
  - Returns the record after **a quorum of replicas** from **all datacenters** has responded.
- **LOCAL_QUORUM**, Used in multiple datacenter clusters with **a rack-aware replica placement strategy** ( NetworkTopologyStrategy) and a properly configured snitch. Fails when using SimpleStrategy.
  - Returns the record after **a quorum of replicas** in the **current datacenter** as the coordinator has reported.
- **ONE**, Provides **the highest availability of all the levels** if you can tolerate a comparatively high probability of **stale data** being read. The replicas contacted for reads may not always have the most recent write.
  - Returns a response from **the closest replica**, as determined by the snitch. By default, a read repair runs in the background to make the other replicas consistent.
- **TWO**
  - Returns the most recent data from **two of the closest replicas**.
- **THREE**
  - Returns the most recent data from **three of the closest replicas**.
- **LOCAL_ONE**
  - Returns a response from **the closest replica** in the **local datacenter**.
- **SERIAL**, To read the latest value of a column after a user has **invoked a lightweight transaction** to write to the column, use SERIAL.
  - **Allows reading** the current (and possibly uncommitted) state of data **without proposing a new addition or update**. If a SERIAL read finds an uncommitted transaction in progress, it will commit the transaction as part of the read. Similar to QUORUM.
- **LOCAL_SERIAL**, Used to achieve **linearizable consistency** for lightweight transactions.
  - Same as SERIAL, but confined to the datacenter. Similar to LOCAL_QUORUM.

### How QUORUM is calculated

> quorum = (sum_of_replication_factors / 2) + 1
> sum_of_replication_factors = datacenter1_RF + datacenter2_RF + . . . + datacentern_RF
Exapmles:

- Using a replication factor of 3, a quorum is 2 nodes. The cluster can tolerate 1 replica down.
- Using a replication factor of 6, a quorum is 4. The cluster can tolerate 2 replicas down.
- In a two datacenter cluster where each datacenter has a replication factor of 3, a quorum is 4 nodes. The cluster can tolerate 2 replica nodes down.
- In a five datacenter cluster where two datacenters have a replication factor of 3 and three datacenters have a replication factor of 2, a quorum is 7 nodes.

## How are write requests accomplished

If a replica misses a write, the row is made consistent later using **one of the built-in repair mechanisms**: `hinted handoff, read repair, or anti-entropy node repair`.

意思是, 根据用户选择的一致性级别, 决定了写入副本数多少后即可反馈成功. 后续副本的同步由后台进程执行, 如果这时候副本同步失败, 通过修复机制来解决.

## How are read requests accomplished

Rapid read protection allows Cassandra to still deliver read requests when **the originally selected replica nodes** are either **down or taking too long to respond**. If the table has been configured with the speculative_retry property, the coordinator node for the read request will retry the request with **another replica node** if the original replica node exceeds a configurable timeout value to complete the read request.

意思是, 快书读取保护, 在首次选择的replica node failed后, 会立即retry

### Examples of read consistency levels

- **QUORUM** in a single datacenter
  - 3副本, 返回 2/3 副本状态一致
- **ONE** in a single datacenter
  - 3 副本, 返回最近的即可
- **QUORUM** in two datacenters
  - 6副本, 2个中心任意返回 4副本. 
- **LOCAL_QUORUM** in two datacenters
  - 6副本, 1个中心返回 2副本即可.
- **ONE** in two datacenters
- **LOCAL_ONE** in two datacenters

## cql 优化

### index

// todo
cassandra **不支持**额外创建的组合索引

### ALLOW FILTERING

ALLOW FILTERING是一种**非常消耗计算机资源的查询方式**, 他会从查询出的所有结果中, 逐个过滤

- 如果您的表包含例如100万行，并且其中**95％具有满足查询条件的值**，则查询仍然相对有效，您应该使用`ALLOW FILTERING`
- 如果您的表包含100万行，并且**只有2行包含满足查询条件值**，则查询**效率极低**。Cassandra将无需加载999,998行。如果经常使用查询，则最好在acc_pedal_stroke列上**添加索引**。

不幸的是，Cassandra无法区分上述两种情况，因为它们取决于表格的数据分布。因此，卡桑德拉会警告你并依靠你做出好的选择.

```sql
CREATE TABLE blogs (blogId int,
                    time1 int,
                    time2 int,
                    author text,
                    content text,
                    PRIMARY KEY(blogId, time1, time2));
-- 由于time1不是索引, 会走全表扫描, 需要人为指定
select * from blogs where time1=1;
-- 加上 ALLOW FILTERING 即可全表扫描
select * from blogs where time1=1 ALLOW FILTERING;
-- 走在pk上, 不会走全表
select * from blogs where time1=1 and time2=2 and blogId=1;
CREATE INDEX IF NOT EXISTS idx_blog_time1
ON face_24503.blogs (time1);
-- 给time1 加上index后, 不会走全表扫描
select * from blogs where time1=1;
CREATE INDEX IF NOT EXISTS idx_blog_author
ON face_24503.blogs (author);
-- 即使加入了索引, 还是会走到 ALLOW FILTERING, 因为第二次一定会过滤, 除非加入的是组合索引
select * from blogs where time1=1 and author='';
```

如果给一张表加上**二级索引 index author**, `select * from blog where author='xxx'`, 这时候**不会**走**ALLOW FILTERING**. 但是如果条件语句中有**2个二级索引**, **一定**会走到**ALLOW FILTERING**.(//todo 如果建立的是组合索引, 是否还会有这个问题呢?)

---

[ALLOW FILTERING explained](https://www.datastax.com/dev/blog/allow-filtering-explained-2)
[Cassandra学习六 一些知识点](https://www.cnblogs.com/liufei1983/p/9484581.html)