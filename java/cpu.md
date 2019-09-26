# cpu

[JAVA 拾遗 — CPU Cache 与缓存行](https://www.cnkirito.moe/cache-line/)

[7个示例科普CPU CACHE](https://coolshell.cn/articles/10249.html)

[Java性能之线程上下文切换究极解析](https://zhuanlan.zhihu.com/p/82848203)

## 1. cpu cache

CPU高速缓存是用于减少处理器访问内存所需平均时间的部件。在金字塔式存储体系中它位于自顶向下的第二层，仅次于CPU寄存器。其容量远小于内存，但速度却可以接近处理器的频率。

![cpu level cache](./imgs/cpu_cache.png)

### 1.1 cpu cache line

缓存行 (Cache Line) 便是 `CPU Cache` 中的**最小单位**，CPU Cache 由若干缓存行组成，**一个缓存行**的大小通常是 `64 字节（这取决于 CPU）`，并且它有效地引用**主内存中的一块地址**。一个 Java 的 long 类型是 8 字节，因此在一个缓存行中可以存 8 个 long 类型的变量

![cpu多级缓存行](./imgs/cpu_cache_line.png)

CPU 缓存在**顺序访问连续内存数据**时挥发出了最大的优势

### 1.2 伪共享

**多个线程**同时读写**同一个缓存行**的不**同变量**时导致的 **整个 cpu cache line 失效**

![cpu cache line 伪共享](./imgs/cpu_cache_padding.png)

#### 1.2.1 字节填充

只需要保证不同线程的变量存在于不同的 CacheLine 即可，使用多余的字节来填充可以做点这一点

java6:

``` java
public class PaddingObject{
    // java long object header 占用 8 byte
    public volatile long value = 0L;    // 实际数据
    public long p1, p2, p3, p4, p5, p6; // 填充
}
```

java7:

``` java
abstract class AbstractPaddingObject{
    protected long p1, p2, p3, p4, p5, p6;// 填充
}

public class PaddingObject extends AbstractPaddingObject{
    public volatile long value = 0L;    // 实际数据
}
```

java8:

``` java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.TYPE})
public @interface Contended {
    String value() default "";
}
// 注意需要同时开启 JVM 参数：-XX:-RestrictContended=false
```

> ConcurrentHashMap 中，使用 `@sun.misc.Contended` 对静态内部类 CounterCell 进行修饰. (因为会**频繁并发**的**读和修改**, 针对 cpu cache 不做字节填充, 会退化缓存功效)
> Thread 中, 使用 `@sun.misc.Contended`

#### 1.2.2 遍历同样大小的数组和链表， 哪个比较快

1. **数组** 结构是连续的内存地址，所以数组全部或者部分元素被**连续存在CPU缓存**里面， 平均读取每个元素的时间只要3个CPU时钟周期
2. **链表** 的节点是分散在堆空间里面的，这时候CPU缓存帮不上忙，**只能是去读取内存**，平均读取时间需要100个CPU时钟周期

这样算下来，数组访问的速度比链表快33倍

## 2. epoll 多路复用

`epoll` 解决**线程资源浪费问题**. 无需多线程 I/O 阻塞, 可以使用较少的线程资源 按需分配 cpu(时间片).

场景如下, n个线程都在等待 I/O, CPU的节约可以让线程通过 wait/notify 或者 cas 来解决, 但是在I/O阻塞过程中的线程资源浪费是无法避免的(一台机器能创建的线程数是有限的, 而这些线程只是在等待). 所有的链接都使用 1 个 acceptor/mainReactor 保持链接, 有 n 个 subReactor. mainReactor 需要记录映射到 acceptor 的哪个链接上. 当 accpetor 接受到 event 的时候, 会选择一个 subReactor 来执行(这样就可以打开多个链接只需要一个线程).

需要解决的问题:

1. object.wait() 实实在在的进行了context switch(golang 的切换实际上是协程切换, 可能并没有导致线程上下文切换), [无法避免].
2. 如何实现单线程 epoll 监听 event. 在 acceptor 记录一个 map映射, 在 subReactor 设置线程池. 使用2个 sync/buffer 队列. 一个用于 request, 一个用于 response.
