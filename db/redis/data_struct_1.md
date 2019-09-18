# data struct 1

[redis设计与实现](http://redisbook.com/preview/adlist/implementation.html)

## 1. SDS(simple dynamic string) 简单动态字符串

SDS 还被用作**缓冲区（buffer）**： AOF 模块中的 **AOF 缓冲区**， 以及客户端状态中的**输入缓冲区**， 都是由 SDS 实现的

``` c++
sds struct {
   free int // 未使用 buffer 空间
   len int // 已使用 buffer 空间
   buf []byte // 数据空间
}
```

1. **空间预分配(空间换时间), 减少扩容内存分配次数**, 如果buf小于 1MB, 每次分配的都会分配double数据长度, 否则直接多分配 1MB 数据
2. **惰性空间释放, 减少缩容内存分配次数**
3. 使用 free 和 len 减少获取字符长度时需要遍历整个字符数组, **常数复杂度**获取 len
4. **二进制安全**
5. 兼容 C 字符串函数

## 2. List 链表

当一个**列表键**包含了数量比较**多**的**元素**， 又`或者`列表中包含的**元素**都是**比较长**的字符串时， Redis 就会使用链表作为列表键的底层实现

``` c++
typedef struct listNode {
    // 前置节点
    struct listNode *prev;

    // 后置节点
    struct listNode *next;

    // 节点的值
    void *value;
} listNode;
```

``` c++
typedef struct list {
    // 表头节点
    listNode *head;

    // 表尾节点
    listNode *tail;

    // 链表所包含的节点数量
    unsigned long len;

    // 节点值复制函数
    void *(*dup)(void *ptr);

    // 节点值释放函数
    void (*free)(void *ptr);

    // 节点值对比函数
    int (*match)(void *ptr, void *key);
} list;
```

1. **双端**： 链表节点带有 prev 和 next 指针， 获取某个节点的前置节点和后置节点的复杂度都是 O(1) 。
2. **无环**： 表头节点的 prev 指针和表尾节点的 next 指针都指向 NULL ， 对链表的访问以 NULL 为终点。
3. **带表头指针和表尾指针**： 通过 list 结构的 head 指针和 tail 指针， 程序获取链表的表头节点和表尾节点的复杂度为 O(1) 。
4. 带链表**长度计数器**： 程序使用 list 结构的 len 属性来对 list 持有的链表节点进行计数， 程序获取链表中节点数量的复杂度为 O(1) 。
5. **多态**： 链表节点使用 void* 指针来保存节点值， 并且可以通过 list 结构的 dup 、 free 、 match 三个属性为节点值设置类型特定函数， 所以链表可以用于保存各种不同类型的值

## 3. Diction 字典

当一个**哈希键**包含的**键值**对比较**多**， 又`或者`**键值对**中的**元素**都是**比较长**的字符串时， Redis 就会使用字典作为哈希键的底层实现

![字典表结构](./imgs/redis_dictionay_data_struct.png)


### hash 表

``` c++
typedef struct dictht {
    // 哈希表数组
    dictEntry **table;

    // 哈希表大小
    unsigned long size;

    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size - 1
    unsigned long sizemask;

    // 该哈希表已有节点的数量
    unsigned long used;
} dictht;
```

table 属性是一个数组， 数组中的**每个元素**都是一个指向 dict.h/dictEntry 结构的指针， 每个 dictEntry 结构保存着一个键值对

### hash表节点

``` c++
typedef struct dictEntry {
    // 键
    void *key;
    // 值, 可以是一个指针， 或者是一个 uint64_t 整数， 又或者是一个 int64_t 整数
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;
    // 指向下个哈希表节点，形成链表
    struct dictEntry *next;
} dictEntry;
```

### dict字典

``` c++
typedef struct dict {
    // 类型特定函数
    dictType *type;

    // 私有数据
    void *privdata;

    // 哈希表, ht[0] 平常读写使用, ht[1] rehash时使用
    dictht ht[2];

    // rehash 索引
    // 当 rehash 不在进行时，值为 -1
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */
} dict;
```

### hash 算法

``` shell
# 使用字典设置的哈希函数，计算键 key 的哈希值
hash = dict->type->hashFunction(key);

# 使用哈希表的 sizemask 属性和哈希值，计算出索引值
# 根据情况不同， ht[x] 可以是 ht[0] 或者 ht[1]
index = hash & dict->ht[x].sizemask;
```

当字典被用作数据库的底层实现， 或者哈希键的底层实现时， Redis 使用 [MurmurHash2](http://code.google.com/p/smhasher/) 算法来计算键的哈希值

### 解决键冲突

Redis 的哈希表使用**链地址法（separate chaining）**来解决键冲突

程序总是将新节点添加到链表的表头位置（复杂度为 O(1)）, **头插法**. Java的 hashmap 在 jdk1.7还是1.8 改成了 尾插法, 解决了并发 rehash 死循环的问题(`todo 需要分析下这个原因`)

### rehash

在执行 BGSAVE 命令或 BGREWRITEAOF 命令的过程中， Redis 需要创建当前服务器进程的子进程， 而大多数操作系统都采用**写时复制（copy-on-write）技术**来优化子进程的使用效率， 所以在子进程存在期间， 服务器会提高执行扩展操作所需的负载因子， 从而尽可能地避免在子进程存在期间进行哈希表扩展操作， 这可以避免不必要的内存写入操作， 最大限度地节约内存

1. 为字典的 ht[1] 哈希表分配空间， 这个哈希表的空间大小取决于要执行的操作， 以及 ht[0] 当前包含的键值对数量 （也即是 ht[0].used 属性的值）：
   - 如果执行的是扩展操作， 那么 ht[1] 的大小为第一个大于等于 ht[0].used * 2 的 2^n （2 的 n 次方幂）将保存在 ht[0] 中的所有键值对 rehash 到 ht[1] 上面
   - 如果执行的是收缩操作， 那么 ht[1] 的大小为第一个大于等于 ht[0].used 的 2^n
2. 将保存在 ht[0] 中的所有键值对 rehash 到 ht[1] 上面
3. 当 ht[0] 包含的所有键值对都迁移到了 ht[1] 之后 （ht[0] 变为空表）， 释放 ht[0] ， 将 ht[1] 设置为 ht[0] ， 并在 ht[1] 新创建一个空白哈希表， 为下一次 rehash 做准备

### 渐进式 rehash

渐进式 rehash 的好处在于它采取分而治之的方式， 将 rehash 键值对所需的计算工作均滩到对字典的每个添加、删除、查找和更新操作上， 从而避免了集中式 rehash 而带来的庞大计算量

1. 为 ht[1] 分配空间， 让字典同时持有 ht[0] 和 ht[1] 两个哈希表。
2. 在字典中维持一个索引计数器变量 rehashidx ， 并将它的值设置为 0 ， 表示 rehash 工作正式开始。
3. 在 rehash 进行期间， 每次对字典执行**添加、删除、查找或者更新操作时**， 程序除了执行指定的操作以外， 还会顺带将 ht[0] 哈希表在 rehashidx 索引上的所有键值对 rehash 到 ht[1] ， 当 rehash 工作完成之后， 程序将 rehashidx 属性的值++。
4. 随着字典操作的不断执行， 最终在某个时间点上， ht[0] 的所有键值对都会被 rehash 至 ht[1] ， 这时程序将 rehashidx 属性的值设为 -1 ， 表示 rehash 操作已完成

字典的删除（delete）、查找（find）、更新（update）等操作会在两个哈希表上进行.

新添加到字典的键值对一律会被保存到 ht[1] 里面， 而 ht[0] 则不再进行任何添加操作, 这一措施保证了 ht[0] 包含的键值对数量会只减不增

## 4. skiplist 跳表

跳跃表支持**平均 O(log N)** **最坏 O(N)** 复杂度的节点查找, 还可以通过顺序性操作来批量处理节点。 在大部分情况下， 跳跃表的效率可以和平衡树相媲美， 并且因为跳跃表的实现比平衡树要来得更为简单， 所以有不少程序都使用跳跃表来代替平衡树。

Redis 只在两个地方用到了跳跃表， 一个是实现**有序集合键**， 另一个是在**集群节点中用作内部数据结构**

**有序集合键**的底层实现之一： 如果一个有序集合包含的**元素数量**比较**多**， 又`或者`有序集合中**元素**的成员（member）是**比较长**的字符串时， Redis 就会使用跳跃表来作为有序集合键的底层实现。

### 跳跃表节点

``` c++
typedef struct zskiplistNode {
    // 后退指针, 用于从表尾向表头方向访问节点
    struct zskiplistNode *backward;
    // 分值
    double score;
    // 成员对象
    robj *obj;
    // 层, 一般来说， 层的数量越多， 访问其他节点的速度就越快
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 跨度, 记录两个节点之间的距离
        unsigned int span;
    } level[];

} zskiplistNode;
```

- 分值: 跳跃表中的所有节点都按分值**从小到大**来排序
- 成员对象: 而字符串对象则保存着一个 SDS 值

各个节点保存的成员对象必须是唯一的， 但是多个节点保存的分值却可以是相同的

### 跳跃表

``` c++
typedef struct zskiplist {
    // 表头节点和表尾节点
    struct zskiplistNode *header, *tail;

    // 表中节点的数量
    unsigned long length;

    // 表中层数最大的节点的层数
    int level;
} zskiplist;
```

1. header 和 tail 指针分别指向跳跃表的表头和表尾节点， 通过这两个指针， 程序定位表头节点和表尾节点的复杂度为 O(1)
2. 通过使用 length 属性来记录节点的数量， 程序可以在 O(1) 复杂度内返回跳跃表的长度
3. level 属性则用于在 O(1) 复杂度内获取跳跃表中层高最大的那个节点的层数量， 注意**表头节点的层高**并**不计算**在内
