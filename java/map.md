# Map

## 1. HashMap(非线程安全)

1. java, 数组(hashcode & capacity - 1)tab[] + 链表(eqauls)Node + 红黑树 存储
   1. jdk 1.7 使用头插法, 当扩容时, 会出现 a<->b循环依赖.
   2. jdk 1.8 使用尾插法, 
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

## 3. ConccurentHashMap

## 4. SortedMap

## 5. HashTable(线程安全)

同步方法, 和 HashMap 一样使用 数组 + 链表, 固定hash 长度为 0x7FFFFFFF

rehash 是重新计算新的 数组长度, 使用 (e.hash & 0x7FFFFFFF) % newCapacity 求出 hash 在新数组中的位置

不允许插入 null key

## 6. TreeMap

红黑树