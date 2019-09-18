# Object

主要数据结构有 **简单动态字符串（SDS）, 双端链表, 字典, 跳跃列表, 整数集合, 压缩列表**

组合成5个对象 **字符串对象, 列表对象, 哈希对象, 集合对象, 有序集合对象**

对象系统还实现了基于`引用计数`技术的**内存回收机制**: 当程序不再使用某个对象的时候， 这个对象所占用的内存就会被自动释放； 另外， Redis 还通过引用计数技术实现了对象共享机制， 这一机制可以在适当的条件下， 通过让多个数据库键共享同一个对象来节约内存

对象带有**访问时间记录信息**， 该信息可以用于计算数据库键的空转时长， 在服务器启用了 maxmemory 功能的情况下， **空转时长较大**的那些键可能会优先被服务器**删除**

## 1. 对象的类型和编码

Redis 中的每个对象都由一个 `redisObject` 结构表示

``` c++
typedef struct redisObject {
    // 类型
    unsigned type:4;
    // 编码
    unsigned encoding:4;
    // 指向底层实现数据结构的指针
    void *ptr;
    // ...
} robj;
```

### type 类型

- REDIS_STRING: 字符串对象
- REDIS_LIST: 列表对象
- REDIS_HASH: 哈希对象
- REDIS_SET: 集合对象
- REDIS_ZSET: 有序集合对象

### encoding 编码

- REDIS_ENCODING_INT: long 类型的整数
- REDIS_ENCODING_EMBSTR	embstr: 编码的简单动态字符串, 使用**一次内存分配**
- REDIS_ENCODING_RAW: 简单动态字符串, 使用**两次内存分配**
- REDIS_ENCODING_HT: 字典
- REDIS_ENCODING_LINKEDLIST: 双端链表
- REDIS_ENCODING_ZIPLIST: 压缩列表
- REDIS_ENCODING_INTSET: 整数集合
- REDIS_ENCODING_SKIPLIST: 跳跃表和字典

### type 和 encoding 的关系

``` shell
REDIS_STRING -> REDIS_ENCODING_INT	使用整数值实现的字符串对象。
REDIS_STRING -> REDIS_ENCODING_EMBSTR	使用 embstr 编码的简单动态字符串实现的字符串对象。
REDIS_STRING -> REDIS_ENCODING_RAW	使用简单动态字符串实现的字符串对象
REDIS_LIST -> REDIS_ENCODING_ZIPLIST	使用压缩列表实现的列表对象
REDIS_LIST -> REDIS_ENCODING_LINKEDLIST	使用双端链表实现的列表对象
REDIS_HASH -> REDIS_ENCODING_ZIPLIST	使用压缩列表实现的哈希对象
REDIS_HASH -> REDIS_ENCODING_HT	使用字典实现的哈希对象
REDIS_SET -> REDIS_ENCODING_INTSET	使用整数集合实现的集合对象。
REDIS_SET -> REDIS_ENCODING_HT	使用字典实现的集合对象。
REDIS_ZSET -> REDIS_ENCODING_ZIPLIST	使用压缩列表实现的有序集合对象。
REDIS_ZSET -> REDIS_ENCODING_SKIPLIST	使用跳跃表和字典实现的有序集合对象
```

在 redis 4.0 后, `REDIS_LIST` 对象**不使用** ziplist 和 linkedlist 结构了, 直接使用结构 `quicklist`

## 2. 字符串对象 int/embstr/raw

如果字符串对象保存的是一个字符串值， 并且这个字符串值的长度**小于等于 44 byte**， 那么字符串对象将使用 `embstr` 编码的方式来保存这个字符串值

如果字符串对象保存的是一个字符串值， 并且这个字符串值的长度**大于 44 byte**， 那么字符串对象将使用一个简单动态字符串（SDS）来保存这个字符串值， 并将对象的编码设置为 `raw`

`raw 编码`会调用**两次内存分配**函数来分别创建 redisObject 结构和 sdshdr 结构， 而 `embstr 编码`则通过调用**一次内存分配**函数来分配一块连续的空间， 空间中依次包含 redisObject 和 sdshdr 两个结构

如果我们要保存一个**浮点数**到**字符串对象**里面， 那么程序会先将这个**浮点数转换成字符串值**， 然后再保存起转换所得的字符串值

`embstr 编码`的字符串对象实际上是**只读的**： 当我们对 embstr 编码的字符串对象执行任何修改命令时， 程序会先将对象的编码从 embstr 转换成 raw ， 然后再执行修改命令； 因为这个原因， `embstr 编码`的字符串对象在执行**修改命令**之后， 总会变成一个 `raw 编码`的字符串对象

### 2.1 命令

- GET, SET, APPEND, STRLEN, INCRBYFLOAT
- `SETEX key seconds value`, PSETEX 给 key 设置 秒/毫秒 过期时间.
- `SETNX key value`, 只能给**不存在**的 key 设置value
- `GETRANGE key start end`
- `SETRANGE key offset value`
- `GETSET old_value new_value`
- MGET, MSET, MSETNX

BIT ARRAY: [bit 使用](https://www.jianshu.com/p/ea087619adc8)

- `SETBIT key offset value`, 对 key 指定位置设置 位(bit)
- `GETBIT key offset`, 对 key 所储存的字符串值，获取指定偏移量上的位 ( bit )
- `BITCOUNT key [start end]`, 对 key 在指定范围内, 统计位(bit) 为 1 的数量
- `BITOP [AND|OR|XOR|NOT]`

## 3. 列表对象 ziplist/linkedlist

在 redis 4.0 前, 使用 ziplist 结果的条件:

1. 列表对象保存的所有字符串元素的长度都**小于** 64 byte
2. 列表对象保存的元素数量**小于** 512 个

在 redis 4.0 后, `REDIS_LIST` 对象**不使用** ziplist 和 linkedlist 结构了, 直接使用结构 `quicklist`

> 因为是双向链表, 可以当作**队列**使用.

### 3.1 命令

- LPUSH, 将元素推到表头, 可以同时 push 多个, 按队列来看
- RPUSH, 将元素推到表尾, 可以同时 push 多个, 按队列来看
- LPUSHX, RPUSHX 只为已存在的 列表 添加值
- LPOP, 将表头元素推出
- RPOP, 将表尾元素推出
- `BLPOP key [key ...] timeout`, 如果队列中没有元素, 会 block 住 timeout 秒, 执行LPOP
- `BRPOP key [key ...] timeout`, 如果队列中没有元素, 会 block 住 timeout 秒, 执行RPOP
- RPOPLPUSH, 从 source 尾出一个元素, 从 dest 头入一个元素
- BRPOPLPUSH
- `LINSERT key BEFORE|AFTER old_value new_value`, 在指定的值 前|后 插入新值, 如果 列表中有多个 old_value, 只在第一个找到的 old_value 插入. 从头开始查找.
- LINDEX, LLEN
- `LREM key count val`, 删除指定 key 中 n 个 val
- `LTRIM key start end`, 删除制定 key 所有不在[start, end] 范围内的元素
- `LSET key index value`, 替换 index 索引的元素.
- `LRANGE key start end`, 从头遍历, 获取制定范围内 key 下的值, end 为 -1 代表所有

## 4. 哈希对象 hashtable/ziplist

如果是 ziplist 结构, **先添加**到哈希对象中的键值对会被放在压缩列表的**表头**方向， 而**后来添加**到哈希对象中的键值对会被放在压缩列表的**表尾**方向

使用 ziplist 结果的条件:

1. 哈希对象保存的所有字符串元素的长度都**小于** 64 byte
2. 哈希对象保存的元素数量**小于** 512 个

> warning
>
> 这两个条件的上限值是可以修改的， 具体请看配置文件中关于 `hash-max-ziplist-value` 选项和 `hash-max-ziplist-entries` 选项的说明

### 4.1 命令

- HGET, HSET, HSETNX, HEXISTS, HLEN, HGETALL, HINCRBY, HINCRBYFLOAT
- `HDEL key field [filed ...]`
- `HSTRLEN key field`, 获取 key 指定的 field 的 value 长度
- HMGET, HMSET
- `HVALS key`, 获取 key 下所有的 vals

## 5. 集合对象 intset/hashtable

去重 无序 集合

### 5.1 命令

- SADD
- `SDIFF key [key ...]`, 返回 在第一个 key 中不存在的元素集, O(n) 复杂度
- `SINTER key [key ...]`, 返回 所有集合中交集的 元素集, O(n*m) 复杂度
- `SUNION key [key ...]`, 返回 所有集合的并集 元素集, 并且去重. O(n) 复杂度
- SISMEMBER, 在集合中查找给定的元素
- SMEMBERS, 遍历集合所有元素返回
- `SCARD key`, 返回集合数量
- `SRANDMEMBER key count`, 随机的返回 key 内指定的 count 个元素
- `SMOVE source destination member`, 将 source 里的 member 元素 迁移到 destination 中, 原子操作
- `SPOP`, 从集合中**随机**取出一个元素, 返回给客户端后, 删除
- `SREM key member [member ...]`, 从集合中删除指定的 member数据

## 6. 有序集合对象 ziplist/skiplist

当有序集合对象可以同时满足以下两个条件时， 对象使用 ziplist 编码:

1. 有序集合保存的元素数量**小于** 128 个；
2. 有序集合保存的所有元素成员的长度都**小于** 64 字节；

> warning
>
> 这两个条件的上限值是可以修改的， 具体请看配置文件中关于 `zset-max-ziplist-entries` 选项和 `zset-max-ziplist-value` 选项的说明

### 6.1 ziplist 压缩表

ziplist 每个集合元素使用**两个**紧挨在一起的**压缩列表节点**来保存， **第一个**节点保存元素的`成员（member）`， 而**第二个**元素则保存元素的`分值（score）`. 集合元素按分值从小到大进行排序， 分值较小的元素被放置在靠近表头的方向， 而分值较大的元素则被放置在靠近表尾的方向

比普通的 ziplist 多了一个 压缩列表节点 score.

![有序集合下的压缩列表数据结构](./imgs/sorted_set_ziplist_data_struct.png)

### 6.2 skiplist 跳跃表

``` c++
typedef struct zset {
    // 按分值从小到大保存了所有集合元素
    zskiplist *zsl;
    // 有序集合创建了一个从成员到分值的映射
    dict *dict;

} zset;
```

虽然 zset 结构同时使用跳跃表和字典来保存有序集合元素， 但这两种数据结构都会通过指针来**共享相同元素的成员和分值**， 所以同时使用跳跃表和字典来保存集合元素不会产生任何重复成员或者分值， 也不会因此而浪费额外的内存

![有序集合下的跳跃列表数据结构](./imgs/sorted_set_skiplist_data_struct.png)

### 6.3 命令

- `ZADD score member [score member ...]`, 复杂度 O(log(N))
- ZCARD, ZINCRBY, ZINTERSTORE, ZUNIONSTORE, ZSCAN, ZSCORE, ZREM
- `ZCOUNT key min max`, 返回 key 下 score 在 [min, max] 的元素个数, 可以让 min 和 max 为开区间 `(1 (2` 这种格式, 复杂度 O(log(N))
- `ZLEXCOUNT key min max`, 返回 key 下 value 在 ..., 复杂度 O(log(N))
- `ZRANK key member`, 返回 member 在 key 中的级别(**最低为0**), 从 `低 -> 高`
- `ZREVRANK key member`, 返回 member 在 key 中的级别(最低为0), 从 `高 -> 低`
- ZREMRANGEBYRANK
- `ZRANGE key start end [WITHSCORES]`, 按照下标范围返回. O(log(N)+M)
- ZREVRANGE
- ZREMRANGE
- `ZRANGEBYSCORE key min max [limit offset count]`, LIMIT 限制了 M 的, 复杂度 O(log(N) + M)
- ZREVRANGEBYSCORE
- ZREMRANGEBYSCORE
- `ZRANGEBYLEX key min max [LIMIT offset count]`, lexicographical ordering 词典排序, 过滤如果存在交际, 以长字符串为准. `[aaa [g` 这种不会包括 `a`. 复杂度 O(log(N)+M)
- ZREVRANGEBYLEX
- ZREMRANGEBYLEX
- `ZPOPMIN key [count]`, 从key 从 pop score 最小的 count 个数值
- ZPOPMAX
- BZPOPMIN, BZPOPMAX

## 7. KEYS

- `SCAN cursor [MATCH parttern ] [COUNT count]`, 指定 cursor 游标起点, 可以选的匹配 parttern 以及 显示的 count 数量
- `MOVE key db`, 将当前 db 下的 key 转移到目标 db下
