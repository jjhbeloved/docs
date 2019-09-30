# Collection

## 1. Object

### 1.1 hashcode() & equals()

初学者可以简单理解，hashCode方法实际上**返回的就是对象存储的物理地址**（实际可能并不是）。  

`hashCode()` 返回散列值, 作用于 map 加快搜索效率

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

ArrayList 实现了 `writeObject()` 和 `readObject()` 来控制只序列化数组中有元素填充那部分内容. 存储数据的 elementData 数组使用 `transient` 声明无需序列化.

``` java
int DEFAULT_CAPACITY = 10;
add(v); // 直接赋值到对应size, 无需数组内存拷贝
add(index, v); // 性能不好, 要对数组内存做 copy 移位
removeIf(); // 使用了 bitset 来标识删除对象, 减少存储
grow(); // 按照旧的数组大小扩容1.5倍, 如果不足, 扩容至需要的大小
```

### 3.2 LinkedList(非线程安全)

顺序读写, 使用`双向链表`进行存储, 可以当作 stack, queue 使用

``` java
 Node<E> node(int index); // 使用折半法加快 链表的查询
```

### 3.3 Vector(线程安全)

与 ArrayList 类似

### 3.4 CopyOnWriteArrayList(线程安全 - 写时复制)

使用 `动态数组` 来存储, 使用 `ReentrantLock` 加锁.

**写操作**在一个复制的数组上进行，**读操作**还是在原始数组中进行，读写分离，互不影响. 最终一致性, 读到的数据存在延时.

``` java
set(); // 使用 Arrays.copy 将原有数据拷贝一份 进行修改
add(); // 使用 Arrays.copy 将原有数据拷贝一份 进行新增
```

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

### 4.2 ArrayBlockingQueue(线程安全)

使用 `动态数组` 来存储, 使用 `lock` 和 `notEmpty/notFull` 来实现线程安全. 使用 `putIndex` 和 `takeIndex` 表示 offer 和 take 的位置.

由于读写都使用一个 lock, 因此实际上不是读写分离的.

``` java
enqueue(); // 每次成功入队列后, 发送一个 notEmpty.signal() 信号唤醒 await() thread.
offer(); // 如果存储满了使用 notFull.await() 等待唤醒
Itr(); // todo
size(); // lock/unlock
```

### 4.3 LinkedBlockingQueue(线程安全)

使用 `单向链表` 来存储, 使用 `takeLock/putLock` 和 `notEmpty/notFull` 来实现线程安全. 使用 `putIndex` 和 `takeIndex` 表示 offer 和 take 的位置.

由于读写各使用一个锁, 读写单独操作head 和 last Node, 因此可以实现读写分离, 性能更好.

``` java
size(); // AtomicInteger.get()
put(); // 接受响应中断, 入队后 notFull 主动 signal
// 遍历链表时, 需要将 头尾锁 lock
remove(); // take/put lock 都会被锁住
contains(); // take/put lock 都会被锁住
drainTo(); // 将n个数据导出(原数据删除), 不会导致分段锁, 性能较好
```

### 4.4 LinkedBlockingDeque(线程安全)

使用 `双向链表` 来存储, 使用 `lock` 和 `notEmpty/notFull` 来实现线程安全.

### 4.5 PriorityBlockingQueue(线程安全)

使用 `动态数组` 来存储, 基于 `heap小根堆`结构来保证**有序**. 使用 `lock` 和 `notEmpty` 来实现线程安全.

``` java
tryGrow(); // 扩容本身要耗费时间, 就不应该占用 lock, 所有一进入要释放, 但是又可能存在并发问题, 所以使用 CAS 来控制, 而并发进入的线程如果没有执行扩容操作, 需要主动 Thread.yield() 放弃 cpu
```

### 4.6 SynchronousQueue(线程安全)

[同步阻塞队列的几种方案](https://www.cnblogs.com/duanxz/p/3252267.html)

无buffer模式, fair 使用 Queue, no fair 使用 Stack.(默认非公平).

Fifo通常可以支持更大的吞吐量，但Lifo可以更大程度的保持线程的本地化

接使用 `CAS` 实现线程的安全访问

生产的数据, 没有被消费前是不允许再次生产. 如果没有消费者, 生产的数据会立即被丢弃.

spin 个数默认 500

#### 4.6.1 公平模式 TransferQueue

head + tail Node, 线程安全使用 `UNSAFE.compareAndSwapObject` 来保证, 非常**细粒度**的 CAS object

通过输入的值是否是 null 来判断是 take 还是 offer, 使用一个 LockSupport 来实现同步状态切换.

put 的 node 与 tail node 做比较是否是 take/offer 对, 是的话, 从 head node 出队列.

``` java
transfer(); // 先 spin, 然后 LockSupport.park
```

#### 4.6.2 非公平模式 TransferStack

通过输入的值是否是 null 来判断是 take 还是 offer

每次操作都会先入栈, 找到与之匹配的 take/offer 对, 连续2个出栈.

### 4.6 LinkedTransferQueue()

## 5. BitSet

[BitSet的原理以及应用 代码解析](https://benjaminwhx.com/2018/06/05/BitSet%E7%9A%84%E5%8E%9F%E7%90%86%E4%BB%A5%E5%8F%8A%E5%BA%94%E7%94%A8/)

[BitSet运用](https://www.cnblogs.com/liun1994/p/6637198.html)

位图操作使用较少的内存就能处理较大的**数值**. 字符串可以根据 `hashcode` 来做. 

> BitSet能够保证 **如果判定结果为false，那么数据一定是不存在；但是如果结果为true，可能数据存在，也可能不存在(冲突覆盖)**
> BloomFilter 的设计思想和BitSet有较大的相似性，目的也一致，它的核心思想也是使用多个Hash算法在一个“位图”结构上着色，最终提高“存在性”判断的效率

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

// 实际位移 会先根据 类型的占位 做 MASK, 确定范围
// wordIndex 是先除 64 获得数组下表, 再直接获取在 64 bit 的哪几位
words[wordIndex] |= (1L << bitIndex); // Restores invariants
```

BitSet 支持的最大存储值为 Integer.MAX_VALUE = 2^31-1 约为20亿. Integer 占用 4 byte = 32 bit. `BitSet.set(int v);`

[50亿数据统计和排序](https://www.itread01.com/content/1544706735.html)
[Java位向量的实现原理与巧妙应用](https://www.cnblogs.com/liun1994/p/6637198.html)

如果要做 50 亿的数据排序和查询, 需要做一个 BitSet[] 数组才能处理.

### 5.1 实战

有个老的手机短信程序，由于当时的手机CPU，内存都很烂。所以这个短信程序只能记住256条短信，多了就删了。每个短信有个唯一的ID，在0到255之间。当然用户可能自己删短信.现在要求设计实现一个功能: 当收到了一个新短信啥，如果手机短信容量还没”用完”（用完即已经存储256条），请分配给它一个可用的ID。由于手机很破，我要求你的程序**尽量快**，并**少用内存**

审题:

通读一遍题目，可以大概知道题目并不需要我们实现手机短信内容的存储，也就是不用管短信内容以什么形式存、存在哪里等问题。需要关心的地方应该是如何快速找到还没被占用的ID（0 ～ 255），整理一下需求，如下：

1. 手机最多存储256条短信，短信ID范围是[0,255]
2. 用户可以手动删除短信，删除哪些短信是由用户决定的
3. 当收到一条新短信时，只需要分配一个还没被占用的ID即可，不需要是可用ID中最小的ID
4. 题目没说明在手机短信容量已满的情况下，也就是无法找到可用ID时需要怎么办，这里约定在这种情况下程序返回一个错误码即可

设计一个数据结构来存储已被占用的或没被占用的短信ID. 实现一个函数，返回一个可用的ID，当无法找到可用ID时，返回-1, 在实现以上两点的前提下，尽量在程序执行速度和内存占用量上做优化

``` java
public class BitSetDemo {

    private int size;
    private BitSet bitSet;

    public BitSetDemo(int size) {
        this.size = size;
        bitSet = new BitSet(size);
    }

    private int getId() {
        if (usedCapacity() > 0) {
            return bitSet.nextClearBit(0);
        } else {
            throw new RuntimeException("not space.");
        }
    }

    public int addId() {
        int id = getId();
        bitSet.set(id);
        return id;
    }

    public void removeId(int id) {
        bitSet.clear(id);
    }

    public int usedCapacity() {
        return size - bitSet.length();
    }

    public static void main(String[] args) {
        BitSetDemo bit = new BitSetDemo(256);

        System.out.println(bit.usedCapacity());
        int id = bit.addId();
        System.out.println("Id: " + id);
        System.out.println(bit.usedCapacity());
        bit.removeId(id);
        System.out.println(bit.usedCapacity());

        id = bit.addId();
        System.out.println("Id: " + id);
        System.out.println(bit.usedCapacity());

        for (int i = 0; i < 255; i++) {
            bit.addId();
        }
        System.out.println(bit.usedCapacity());
    }
}
```
