# CQL

可以使用 `tracing on;` 开启 sql 解析

## BATCH

A batch applies all DML statements within a **single partition** before the data is available, ensuring `atomicity` and `isolation`. A well-constructed batch targeting a single partition can reduce client-server traffic and more efficiently update a table with a single row mutation.

For **multiple partition** batches, **logging** ensures that all DML statements are applied. Either all or none of the batch operations will succeed, ensuring `atomicity`. Batch isolation occurs only if the batch operation is writing to a single partition.

> **Important**: Only use a multiple partition batch when there is no other viable option, such as asynchronous statements. Multiple partition batches may decrease throughput and increase latency.

### LOGGED | UNLOGGED

If **multiple partitions** are involved, batches are **logged** by **default**. Running a batch with logging enabled ensures that either all or none of the batch operations will succeed, ensuring atomicity. Cassandra **first** writes the **serialized** batch to the **batchlog system table** that consumes the serialized batch as blob data. **After** Cassandra has successfully written and **persisted** (or hinted) the rows in the batch, it **removes** the **batchlog data**. There is a `performance penalty` associated with the batchlog, as it is written to two other nodes. Thresholds for warning about or failure due to batch size can be set.

If you do not want to incur a penalty for logging, run the batch operation without using the batchlog table by using the `UNLOGGED` keyword. Unlogged batching will issue a warning if too many operations or too many partitions are involved. **Single partition** batch operations are `unlogged` by default, and are the only unlogged batch operations recommended.

Although a logged batch enforces atomicity (that is, it guarantees if all DML statements in the batch succeed or none do), Cassandra **does no other transactional enforcement** at the batch level. For example, there is no batch isolation unless the batch operation is writing to a single partition. In multiple partition batch operations, clients are able to read the first updated rows from the batch, while other rows are still being updated on the server. In single partition batch operations, clients cannot read a partial update from any row until the batch is completed.

### USING TIMESTAMPS

Sets the write time for transactions executed in a BATCH.
> **Restriction**: USING TIMESTAMP does not support LWT (lightweight transactions), such as DML statements that have an IF NOT EXISTS clause.
By default, Cassandra applies **the same timestamp** to **all data modified** by the batch; therefore statement order does not matter within a batch, thus a batch statement is not very useful for writing data that must be timestamped in a particular order. Use client-supplied timestamps to achieve a particular order.

### Batching conditional updates

Batch conditional updates introduced as lightweight transactions. However, a batch containing conditional updates can **only** operate **within a single partition**, because the underlying Paxos implementation only works at **partition-level granularity**. If one statement in a batch is a conditional update, the conditional logic must return true, or the entire batch fails. If the batch contains two or more conditional updates, all the conditions must return true, or the entire batch fails.

### Batching counter updates

A batch of counters should use the COUNTER option because, unlike other writes in Cassandra, a counter update is not an `idempotent operation(幂等操作)`.

## UPDATE

### DESCRIPTION

`All UPDATEs` within the same partition key are applied **atomically** and in **isolation**.

> **Note**: Unlike the INSERT command, the UPDATE command **supports counters**. Otherwise, the UPDATE and INSERT operations are identical.

To specify a row, the WHERE clause **must** provide a value for each column of the row's **primary key**.

> **Note**: You **cannot** insert any value **larger than 64K bytes** into a **clustering column**.

### Creating a partition using UPDATE

Since Cassandra processes an UPDATE as an upsert, it is possible to create a new row by updating it in a table. Example: to **create a new partition** in the cyclists table, whose primary key is (id), you can UPDATE the partition with id e7cd5752-bc0d-4157-a80f-7523add8dbcd, even though it **does not exist** yet:

```SQL
UPDATE cycling.cyclists
SET firstname = 'Anna', lastname = 'VAN DER BREGGEN' WHERE id = e7cd5752-bc0d-4157-a80f-7523add8dbcd;
```

### Conditionally updating columns

In Conditionally update columns using `IF` or `IF EXISTS`. The IF EXISTS or IF keywords introduce a **lightweight transaction**.  Using an IF condition incurs a **performance hit** associated with using **Paxos** to support **linearizable consistency**.

Add `IF EXISTS` to the command to ensure that the operation is not performed if the specified row exists:

```SQL
UPDATE cycling.cyclist_id SET age = 28 WHERE lastname = 'WELTEN' and firstname = 'Bram' IF EXISTS;
```

Use `IF` condition to apply tests to one or more column values in the selected row:

```SQL
UPDATE cyclist_id SET id = 15a116fc-b833-4da6-ab9a-4a3775750239 where lastname = 'WELTEN' and firstname = 'Bram' IF age = 18;
```

**Conditional updates** are examples of **lightweight transactions**. They incur a non-negligible **performance cost** and should be used sparingly.

## SELECT
