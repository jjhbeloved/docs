# Memory Manager

## 1. 内存回收

[内存模型](https://www.cnblogs.com/kismetv/p/8654978.html)

C 语言并**不具备**自动的内存回收功能， 所以 Redis 在自己的对象系统中构建了一个**引用计数（reference counting）**技术实现的内存回收机制， 通过这一机制， 程序可以通过跟踪对象的引用计数信息， 在适当的时候**自动释放**对象并进行内存回收

共享对象**只支持整数值的字符串对象**, Redis服务器在初始化时，会创建`10000`个字符串对象，值分别是0~9999的整数值；当Redis需要使用值为0~9999的字符串对象时，可以直接使用这些**共享对象**

> **为什么 Redis 不共享包含字符串的对象?**
>
> 1. 共享对象是 **整数值** 验证复杂度是 O(1)
> 2. 共享对象是 **字符串对象** 验证复杂度是 O(n)
> 3. 共享对象是 集合, 字典, 列表 等, 验证复杂度是 O(m*n)
> 尽管共享更复杂的对象可以节约更多的内存， 但受到 CPU 时间的限制， Redis 只对包含整数值的字符串对象进行共享
> 时间和空间的衡量选择

``` c++
typedef struct redisObject {
    // 对象类型
    unsigned type:4;
    // 对象实际使用的数据结构
　 　unsigned encoding:4;
    // 对象最后访问时间
　 　unsigned lru:REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */
    // 对象引用计数
　 　int refcount;
    // 指针指向具体的数据
　　void *ptr;
} robj;
```

redisObject的结构与对象类型、编码、内存回收、共享对象都有关系；一个redisObject对象的大小为16字节 = `4bit+4bit+24bit+4Byte+8Byte=16Byte`

- 在创建一个新对象时， 引用计数的值会被初始化为 1
- 当对象被一个**新程序 client**使用时， 它的引用计数值会被 +1
- 当对象**不再**被一个**程序 client**使用时， 它的引用计数值会被 -1
- 当对象的引用计数值变为 0 时， 对象所占用的内存会被释放

对象的整个生命周期可以划分为 **创建对象**、**操作对象**、**释放对象** 三个阶段

> 当 set k1 9999, 会创建一个 redisObject -> 9999, 再 set k2 9999 时, 由于存在 9999这个 redisObject, 不会再次创建, 而是使用 refCount + 1.
> 当 del k1 k2 后, redisObject -> 9999 refCount->0 没有引用, 过段时间后就会删除

### 内存碎片

如果Redis服务器中的内存碎片已经很大，可以通过安全重启的方式减小内存碎片：因为重启之后，Redis重新从备份文件中读取数据，在内存中进行重排，为每个数据重新选择合适的内存单元，减小内存碎片

### jemalloc

Redis在编译时便会指定内存分配器；内存分配器可以是 libc 、jemalloc或者tcmalloc，默认是`jemalloc`.

jemalloc作为Redis的默认内存分配器，在减小内存碎片方面做的相对比较好。jemalloc在**64位**系统中，将内存空间划分为**小、大、巨大**三个范围；每个范围内又划分了许多小的内存块单位；当Redis存储数据时，会选择大小**最合适**的内存块进行存储

## 2. 对象的空转时长

如果服务器打开了 `maxmemory` 选项， 并且服务器用于回收内存的算法 `maxmemory-policy` 为 volatile-lru 或者 allkeys-lru ， 那么当服务器占用的**内存数超过**了 maxmemory 选项所设置的上限值时， 空转时长较高的那部分键会优先被服务器释放， 从而回收内存

1. **maxmemory**: 限制 redis 使用的最大内存 bytes. 配置与 `LRU`, `LFU` cache 或者是 `hard memory limit`
2. **maxmemory-policy**: 淘汰策略, 当内存超过 `maxmemory` 限制后, 会先调用 `freeMemoryIfNeeded()` 清空不必要内存占用(比如过期的redisObject). 如果内存还是不足, 基于 `maxmemory-policy` 策略淘汰key释放内存.
    - volatile-**lru**(least recently used)
    - allkeys-**lru**
    - volatile-**lfu**(least frequently used)
    - allkeys-**lfu**
    - volatile-**random**, 在过期的 key 中随机删除一个
    - allkeys-**random**, 在**所有** key 中随机删除一个
    - volatile-**ttl**, 从**过期**的 key 中删除**最近一个**
    - **noeviction**, 不做任何算法淘汰计算, 直接抛出异常.
3. **maxmemory-samples**: `LRU`, `LFU` and `minimal TTL` algorithms are not precise algorithms but **approximated algorithms(近似算法)**, For default Redis will check five keys and **pick the one that was used less recently**(随机选取5个样本中的一个最近最少使用的值).
4. **replica-ignore-maxmemory**: 只在 master 生效 maxmemory, replica 忽略这个选项

redisObject.lru 每次被查询, 都会重置为0.
