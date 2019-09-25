# Collection

## 1. Object

### 1.1 hashcode() & equals()

初学者可以简单理解，hashCode方法实际上**返回的就是对象存储的物理地址**（实际可能并不是）。  

`hashCode()` 返回散列值

`equals()` 是用来判断两个对象**是否等价**

> 当equals()方法被override时，hashCode()也要被override
>
> 等价的两个对象散列值一定相同，但是散列值相同的两个对象不一定等价

方法主要作用于 hash tables(e.g.: HashMap):

- hashcode能**大大降低对象比较次数，提高查找效率**(因为每次去equals当变量很多效率很低)
- 先比对hashcode是否相等, 相等表明(物理地址上)存在值, 再详细判断值是否相等
- 如果不存在, 则可以快速确定值不相等

## 2. Set

### 2.1 原理

Set是去重的, 实际是基于 `Map` 的 key 来存储 并做去重. 比对使用 `(hashcode && equals)`.

当保存的 Object 如果不重写 equals 和 hashcode, 会继承默认的 native hashcode 物理地址. 就会出现虽然 Object 的值是一样的, 但是因为 hashcode 的物理地址不同, 而导致认为Object 是不相等的.

### 2.2 HashSet

内部是使用 `HashMap` 记录. 所有的实现方法都是调用 HashMap 的. 由于使用的是 hash 值作为存储, **无序**

查找时间复杂度`O(1)`

### 2.3 TreeSet

内部使用 `TreeMap` 记录. 基于**双向链表** 维护元素插入的顺序, **有序**

查找时间复杂度`O(log(N))`

## 3. List

### 3.1 原理

比对使用 `equals`, 遍历逐个比对.

### 3.2 ArrayList(非线程安全)

随机读写, 使用`动态数组`进行存储. 内部使用 `Arrays.copyOf` 实现内存拷贝扩缩容.

`length`是**数组长度**, `size`是数组实际**数据个数**.

当 size > length 时, 数组要做扩容. 默认是 length的一半, 如果还不足, 直接扩容至添加数据的个数大小.

contains时间复杂度 O(n)

`modCount`变量 作用是保证不会在修改过程中, 有数据变更(存在则抛出异常). 解决for循环做了 add/remove 等动作

``` java
int DEFAULT_CAPACITY = 10;
add(v); // 直接赋值到对应size, 无需数组内存拷贝
add(index, v); // 性能不好, 要对数组内存做 copy 移位
removeIf(); // 使用了 bitset 来标识删除对象, 减少存储
grow(); // 按照旧的数组大小扩容一倍, 如果不足, 扩容至需要的大小
```

### 3.2 LinkedList(非线程安全)

顺序读写, 使用`双向链表`进行存储, 可以当作 stack, queue 使用

``` java
 Node<E> node(int index); // 使用折半法加快 链表的查询
```

### 3.3 Vector(线程安全)

与 ArrayList 类似

## 4. Queue

[集合 Queue](https://www.jianshu.com/p/35760d7bac0d)

1. LinkedList
2. PriorityQueue
3. LinkedBlockingQueue
4. ArrayBlockingQueue
5. PriorityBlockingQueue

### 4.1 PriorityQueue(非线程安全)

使用 `动态数组` 来存储, 基于 `heap小根堆`结构来保证**有序**.

初始化时使用 `siftDown()` 实现**小根堆** //todo
add queue 时, 使用 `siftUp()` 来维持**小根堆**

``` java
grow(); // size<64 扩容一倍, 否则扩容50%
siftUp(); // 值比父节点小, 上移, todo 时间复杂度
siftDown(); // 值比子节点大, 下沉
add(); // 使用siftUp() 来维持小根堆
remove(); // todo
poll(); // todo
```

### 4.2 ArrayBlockingQueue

使用 `动态数组` 来存储, 基于 `heap小根堆`结构来保证有序.

使用 `ReentranLock` 和 `notEmpty/notFull` 来实现线程安全.

``` java
enqueue(); // 每次成功入队列后, 发送一个 notEmpty.signal() 信号唤醒 await() thread.
offer(); // 如果存储满了使用 notFull.await() 等待唤醒
```

## 5. BitSet

[BitSet的原理以及应用 代码解析](https://benjaminwhx.com/2018/06/05/BitSet%E7%9A%84%E5%8E%9F%E7%90%86%E4%BB%A5%E5%8F%8A%E5%BA%94%E7%94%A8/)

[BitSet运用](https://www.cnblogs.com/liun1994/p/6637198.html)

位图操作使用较少的内存就能处理较大的**数值**. 字符串可以根据 `hashcode` 来做.

为什么选择long这种数据类型，注释的解析是基于性能的原因，现在64位CPU已经非常普及，可以一次把一个64bit长度的long放进**寄存器(cache line)**作计算

``` java
/*
 * BitSet由很多个words组成，每一个words是long类型，一个long是64bit，需要6个地址位（2的6次方）
 */
private final static int ADDRESS_BITS_PER_WORD = 6;
// 1算数左移6位，即1 × 2^6 = 64  即 01000000
private final static int BITS_PER_WORD = 1 << ADDRESS_BITS_PER_WORD;
// 1 x 2^6 - 1 = 63 即 00111111
private final static int BIT_INDEX_MASK = BITS_PER_WORD - 1;

/* 用于向左或向右移动部分word的掩码 */
private static final long WORD_MASK = 0xffffffffffffffffL;

// 这里使用了 long bit 左移后 超过 64 bit 会从第 0 bit 开始实现循环
words[wordIndex] |= (1L << bitIndex); // Restores invariants
```

BitSet 支持的最大存储值为 Integer.MAX_VALUE = 2^31-1 约为20亿. Integer 占用 4 byte = 32 bit. `BitSet.set(int v);`

[50亿数据统计和排序](https://www.itread01.com/content/1544706735.html)
如果要做 50 亿的数据排序和查询, 需要做一个 BitSet[] 数组才能处理.