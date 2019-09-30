# Map

## 1. HashMap(非线程安全)

1. java, 数组(hashcode & capacity - 1)tab[] + 链表(eqauls)Node + 红黑树 存储
   1. jdk 1.7 使用头插法, 当扩容时, 会出现 a<->b循环依赖.
   2. jdk 1.8 使用尾插法
2. redis

``` java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // 初始容量 16
static final int MAXIMUM_CAPACITY = 1 << 30; // 最大容量 1024 * 1024 * 1024, int的正数最大可达231-1，而没办法取到231。所以容量也无法达到231。又需要让容量满足2的幂次。所以设置为230
static final float DEFAULT_LOAD_FACTOR = 0.75f; // 当 size 达到总 capacity 75% 开始扩容
static final int TREEIFY_THRESHOLD = 8;
static final int UNTREEIFY_THRESHOLD = 6;
static final int MIN_TREEIFY_CAPACITY = 64;

// JDK1.8 -> 2次扰动处理=1次位运算+1次异或
// key哈希值与key哈希值右移16位，做异或，可以减少冲突量
return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
// 获取容量的 2^n 次最大值, 获取 capacity 的 threshold, 保证是 2 次幂
// int 32 位, 通过 >> 1, >> 2, >> 4, >> 8, >> 16 取与 可以得到 MASK n 个 1
// 从此获取到 length
tableSizeFor(int cap);
/**
* 有旧的值, oldCap 和 oldThreshold 扩容一倍, 超过 MAX 使用 Integer.MAX_VALUE
* 初始化 threshold =  DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY
* 1. 没有头节点, 直接赋值
* 2. 如果是红黑树, 使用 split
* 3. 链表迁移
* 将扩容的数分成 高位 和 低位, 因为实际存储是用指针, 并不会扩容占用过多内存
* hashcode & capacity == 0 意味着扩容后 hash 值不变, 在低位
* hashcode & capacity == 1 意味着扩容后 hash 不变, 在高位
*/
resize();
/**
* 定位到key的 hash 所在 tab[] 的位置, 如果是tab[]上的初始值 Node.next = null
* 否则
* 1. key 值相等, 不做覆盖
* 2. 对象为 红黑树
* 3. key 值不等, 遍历到链表最后一个, addLast. 链表上的值超过 TREEIFY_THRESHOLD-1 转成红黑树
* 在 put 后, 如果超过了 threshold, 需要主动 resize()
*/
putVal();
// 如果 key 值不存 或者 value 为 null 使用fuction来插入值
// 如果 hashcode 存在相等, key 没有相等的, 往链表 head 插入值
computeIfAbsent(key, Function);
```

红黑树:

## 2. LinkedHashMap(非线程安全)

Node 新增 head, after 形成双向链表

``` java
// 将变更的值放到链表的 last
// 新增的值
putVal();
```

可以基于 LinkedHashMap 来实现 LRU Cache. 开启 accessOrder 特性, 在每次修改值时, 会将修改值放置到链表的 last. 在插入新值时, 根据 `removeEldestEntry` 返回值决定是否删除 first node. 可以重写此方法.

## 3. ConccurentHashMap

数组(hashcode & capacity - 1)tab[] + 链表(eqauls)Node + 红黑树 存储, 扩容时多使用一个`Node[] nextTable`.

``` java
// 控制 表初始化和扩容. -1代表初始化, -(1+扩容线程)代表扩容, 正数为表扩容大小
private transient volatile int sizeCtl;
// 再次优化的 hash
spread(); // 即使哈希冲突比较严重，寻址效率也足够高，所以作者并未在哈希值的计算上做过多设计，只是将Key的hashCode值与其高16位作异或并保证最高位为0（从而保证最终结果为正整数）
// arrayBaseOffset方法是一个本地方法，可以获取数组第一个元素的偏移地址
// arrayIndexScale方法也是一个本地方法，可以获取数组的转换因子，也就是数组中元素的增量地址
// 将arrayBaseOffset与arrayIndexScale配合使用，可以定位数组中每个元素在内存中的位置
Class<?> ak = Node[].class;
ABASE = U.arrayBaseOffset(ak);
int scale = U.arrayIndexScale(ak);
ASHIFT = 31 - Integer.numberOfLeadingZeros(scale);
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    // 没有加锁, 也没有直接使用 volatile 让CPU内缓存无效, 而是只更新这个 处理器上的工作内存
    // i<< ASHIFT 是将 i 定位到数组的增量位置上
   return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
// 使用 CounterCell 记录好没一个 tab 对应链表值个数(volatile count)
// 将 count 计算提前到 更新操作中通过 cas 完成
size();
// 如果 tab 不存在, 从初始化表
// 如果 sizeCtrl < 0 协助扩容
// 否则 初始化扩容
tryPresize();
// 实际扩容函数, 如果 nextTable 不存在, 按照原 table 大小扩容一倍.
// 多线程 协助扩容, 每个线程固定步长stride/bound, 从 nextIndex 最大值开始递减分配
// null 或者 已经处理过的 Node 使用 fwd 占位节点来标识已经被处理过了
// 使用 len 最高一位 0/1 来实现自动负载
transfer();
// 占位节点, 如果县城发现类型为 fwd, 则忽略此节点
class ForwardingNode;
// 使用 多线程协助扩容的方案来实现批量 put
putAll();
// 如果 tab[hash] = null 则为链表首节点, 使用 casTabAt 插入 tab
// 如果 tab[] == null 则初始化 tab
// 如果 tab[hash].hash == MOVED 说明正在扩容, 调用 helperTransfer 协助扩容
// 否则插入节点, 使用 sychronized(node) 锁住整个链表头
// 主动给 count + 1
putVal();
// 加1, 如果是 putVal, 每次 addCount 都会伴随这 协助扩容
addCount();
// 若是 sizeCtl<0 意味着有人在初始化或扩容, 当前县城主动 yield 释放 cpu
// 否则 使用 cas 设置 sizeCtl = -1 代表正在初始化, 初始化完成 sizeCtl = n >>> 2
initTable();
```

[transfer()函数分析](https://juejin.im/post/5b00160151882565bd2582e0)

## 4. SortedMap

## 5. HashTable(线程安全)

同步方法, 和 HashMap 一样使用 数组 + 链表, 固定hash 长度为 0x7FFFFFFF

rehash 是重新计算新的 数组长度, 使用 (e.hash & 0x7FFFFFFF) % newCapacity 求出 hash 在新数组中的位置

不允许插入 null key

## 6. TreeMap

红黑树
