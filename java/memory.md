# Memory

## 0. 内存模型

![内存模型](./imgs/java_memory_model.png)

1. read：把一个变量的值从主内存传输到工作内存中
2. load：在 read 之后执行，把 read 得到的值放入工作内存的变量副本中
3. use：把工作内存中一个变量的值传递给执行引擎
4. assign：把一个从执行引擎接收到的值赋给工作内存的变量
5. store：把工作内存的一个变量的值传送到主内存中
6. write：在 store 之后执行，把 store 得到的值放入主内存的变量中
7. lock：作用于主内存的变量
8. unlock

### 0.1 原子性/可见性/有序性

1. 保证以上 8 个命令都具备原子性
2. volatile/sync/final 保证可见性
3. volatile 通过 barrier 禁止指令重拍, sync 通过加锁只允许一个线程执行

### 0.2 happen-before 原则

1. 单一线程有序
2. 管程锁定规则, monitor lock rule, unlock 操作先行发生于后一个 lock 之前
3. 对一个volatile write 现行发生于后一个对这个变量的 read
4. 线程启动规则, start()方法现行发生于此线程的每一个动作.
5. thread join rule, join 之前 thread 已经结束.
6. thread interrupt rule, 可以通过 interrupted 检测到中断
7. finalizer rule, <init>() 先于 finalize()
8. transitivity, 传递行

### 0.3 线程安全

1. 不可变, final, String
2. 互斥同步, cas, sync, reentrantLock
3. 无阻塞, 栈封闭(无逃逸), ThreadLocal 线程本地存储, 可重入代码

## 1. off-heap memory 堆外内存

当对象很大, 占用了堆内存, 却长期不释放, 会导致频繁的 full gc 且 stw频繁.

为了解决 stw 问题, jvm 会把部分对象实例 分配在**heap以外**的内存区域, 这块内存受到 os 管理, 称为堆外内存.

因为这部分区域直接受操作系统的管理，别的进程和设备（例如GPU）可以直接通过操作系统对其进行访问，减少了从 JVM 中复制 **堆内内存** 到 **堆外内存** 的过程。(变成了**共享内存**...)

``` java
public abstract class ByteBuffer extends Buffer implements Comparable {}
/**
* 堆内内存
* 数据的分配存储都在jvm heap上
* 当需要和io设备打交道的时候, 会将jvm堆上所维护的byte[]拷贝至堆外内存, 然后堆外内存直接和io设备交互
* 虽然 jvm 管理的内存也在 os 控制范围内, 但是操作不方便, 因为 byte[] 占用的内存不一定连续, 为了防止 gc 需要 pin 钉住整个堆, 为了不出现这个问题, 开辟一个 directBuffer, 通过这个 buffer 将数据拷贝至 io 设备
* 原理-> https://www.cnkirito.moe/nio-buffer-recycle/
*/
public class HeapByteBuffer extends ByteBuffer {}
/**
* 堆外内存(mmap)
* 数据分配直接存储在内存, 无需 jvm heap 拷贝, 称为 零拷贝（zero-copy）
*/
public abstract class MappedByteBuffer implements ByteBuffer {}
public interface DirectBuffer {}
public class DirectByteBuffer extends MappedByteBuffer implements DirectBuffer {}
```

JVM 使用一个 ByteBuffer 实例对象**记录**堆外内存的**信息**(内存起始地址, 内存大小等等), 当这个实例对象被 GC 后, 实际上 堆外内存就没有引用了, 也就被free了.

1. 使用 HeapByteBuffer 读写都会经过 DirectByteBuffer，写入数据的流转方式其实是：`HeapByteBuffer -> DirectByteBuffer -> PageCache -> Disk`，读取数据的流转方式正好相反。
2. 使用 `HeapByteBuffer` 读写会**额外申请一块**跟**线程绑定**的 `DirectByteBuffer`。这意味着，线程越多，临时 DirectByteBuffer 就越会占用越多的空间。

### 1.1 优点

1. 开辟空间大, 取决于操作系统
2. 减少 stw
3. 直接受 os 控制, 其他线程访问**减少**了 `jvm heap -> page cache` 拷贝的时间
4. 适用**分配次数少, 读写操作频繁**的场景

### 1.2 缺点

1. 内存不可控, 容易 oom, 难排查
2. 数据结构不直观, 序列化/反序列化 很耗时

## 2. in/out heap 最佳实践

1. 当需要申请 **大块的内存** 时，堆内内存会受到限制，只能分配**堆外内存**
2. 堆外内存适用于 **生命周期中等或较长** 的对象。( 如果是生命周期较短的对象，在 YGC 的时候就被回收了，就不存在大内存且生命周期较长的对象在 FGC 对应用造成的性能影响 )
3. **直接的文件拷贝操作**，或者 **I/O 操作**。直接使用堆外内存就能少去内存从用户内存拷贝到系统内存的消耗
4. 同时，还可以使用 **池+堆外内存** 的组合方式，来对 **生命周期较短**，但涉及到 `I/O 操作` 的对象进行 **堆外内存的再使用**( Netty中就使用了该方式 )。在比赛中，尽量不要出现在频繁 new byte[] ，创建内存区域再回收也是一笔不小的开销，使用 `ThreadLocal<ByteBuffer>` 和 `ThreadLocal<byte[]>` 往往会给你带来意外的惊喜~
5. 创建堆外内存的消耗要**大**于创建堆内内存的**消耗**，所以当分配了堆外内存之后，**尽可能复用**它

- 调用 `malloc()` 时，是在PCB表结构中的堆中申请空间，若申请空间失败，即超过给定的堆最大空间时，将会调用brk()系统调用，将堆空间向未使用的区域扩展，brk()之后新增的堆空间不会自动清除，需使用相应的系统调用来清除； 
- 调用 `mmap()` 系统调用使得进程之间通过映射同一个普通文件实现共享内存。普通文件被映射到进程地址空间后，进程可以像访问普通内存一样对文件进行访问，不必再调用read()，write（）等操作

## 3. 字节数组拷贝

``` java
// 将 src 字节数组从 srcPos 位置开始拷贝 len 个 byte 到 以 dstPos 开始的 dst 字节数组中
// 这是基于内存的直接拷贝, 不会重新开辟 byte[] 空间, 效率很不错.
// 有这样的方案, 我们可以使用一个 buffer 来不断的覆盖写.
System.arraycopy(src []byte, srcPos int, dst []byte, dstPos int, len int);
```

``` java
// 这个集成了 byte[] 数组的操作, 方便实现一段 buffer.
ByteBuffer bf = new ByteBuffer();

// 获取当前最近push的数据 position 位置, 也可以设置值来修改位置
bf.position();
// 获取允许 get 数据的 position 上限
bf.limit();
// 获取当前 limit - postition 差值, 也就是允许读取的数据大小
bf.remaining();
// 将 limit = position; position = 0; 根据新插入的数据, 调整允许读取数据的大小
bf.flip();
// 将 position=0; limit=remaing(); offset=position; 虽然内存是共用的, 调整了 offset, 相当于一段独立的内存操作
bf.slice();
// 将position数据初始化
bf.rewind();
// 将所有数据初始化
bf.clear();
```
