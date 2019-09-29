# J.U.C

[Java并发编程之锁机制之引导篇](https://juejin.im/post/5bbf040df265da0ac446ccab)
[多线程开发关键技术](http://www.jasongj.com/java/concurrenthashmap/)

## 1. AbstractQueuedSynchronizer(AQS)

AQS自身没有实现任何同步接口，它仅仅是**定义了若干同步状态获取和释放的方法**来供自定义同步组件使用，同步器既可以支持**独占式**地获取同步状态，也可以支持**共享式**地获取同步状态，这样就可以方便实现不同类型的同步组件

AQS的设计是基于**模板方法模式**: 内置有一个 **同步队列**, 使用 head/tail 链表指针.

1. 使用尾插法 `compareAndSetTail(Node expect,Nodeupdate)` 插入新节点(保证线程安全).
2. **头节点**是获取同步状态成功的节点，头节点的线程会在释放同步状态时，将会**唤醒**其下一个节点, 由于只有一个线程能够成功获取到同步状态，因此设置头节点的方法并不需要CAS来进行保证

**同步队列**:

``` java
// 修改同步状态方法
getState();
setState(int newState);
compareAndSetState(int expect, int update);
// 子类重写方法
boolean isHeldExclusively() // 当前线程是否独占锁
boolean tryAcquire(int arg) // 独占式尝试获取同步状态，通过CAS操作设置同步状态，如果成功返回true，反之返回false
boolean tryRelease(int arg) // 独占式释放同步状态。
int tryAcquireShared(int arg) // 共享式的获取同步状态，返回大于等于0的值，表示获取成功，反之失败。
boolean tryReleaseShared(int arg) // 共享式释放同步状态
// 获取同步状态与释放同步状态方法, pred 节点完成后, 主动唤醒 next 节点
// 独占式获取同步状态，如果当前线程获取同步状态成功，则返回
// 否则, 将 node 插入对尾进入同步队列等待, for:: 判断当前节点的 pred 是否是 head, 只有 node.pred 是 head 才可以获取到同步状态
// 如果不是同一个线程, 争抢到意味着 node.pred 已经结束了, 可以主动释放
// node.pred 的状态需要设置为 signal 意味着可以释放, 这时候 node 可以进入 park 状态. 跳过所有 canceld 状态
// 如果 node 被 interrupt, 判断 node.pred 是否是 SIGNAL, 如果不是 调用 unparksuccesor 移除当前节点, 并唤醒下一个节点
void acquire(int arg);
// 独占式只有一个信号量, 共享式可以有多个
void acquireInterruptibly(int arg) // 与 void acquire(int arg)基本逻辑相同，但是该方法响应中断,如果当前没有获取到同步状态，那么就会进入等待队列，如果当前线程被中断（Thread().interrupt()），那么该方法将会抛出InterruptedException。并返回
boolean tryAcquireNanos(int arg, long nanosTimeout) // 在acquireInterruptibly(int arg)的基础上，增加了超时限制，如果当前线程没有获取到同步状态，那么将返回fase，反之返回true
boolean release(int arg) // 独占式的释放同步状态
// 将 node 加入同步队列尾巴. 通过 for:: 和 compareAndSetTail 来保证 尾插是线程安全的
private Node addWaiter(Node mode);
```

`同步队列:等待队列 = 1:N`

**等待队列(FIFO)**: `当我们将等待队列中的线程节点加入到同步队列之后，才会唤醒线程。`

``` java
// 1. 将当前线程加入 等待队列
// 1.1 移除 等待队列中所有 非 condition 的 node
// 1.2 将 node 加入等待队列 尾巴
// 2. 释放同步状态(锁), 并且从 同步队列中释放 线程节点, 唤醒下一个 同步队列节点
// 3. 判断是否已经从 同步队列中出, 是则 park
// 4. 线程被唤醒, 从 等待队列出后, 在同步队列中竞争 同步状态
await();
```

### 1.1 Lock

``` java
lock(); // lock 时无法获取锁, 会处于休眠状态.
lockInterruptibly(); // lock 时会响应中断
tryLock(); // 获取不到 lock 时会立即返回
```

锁的核心实现是依靠 AQS.

LockSupport 的 unpark 是一种信号量, 以来 AQS, 先 `unpark` 后 `park` 也是可以的.

### 1.2 ReentrantLock(重入锁)

1. 非公平, 抢占式
2. 公平, 队列式

#### 1.2.1 ReentrantReadWriteLock(重入读写锁)

`WriteLock`与`ReadLock`这两个锁，其实是通过AQS中的**同一个同步队列**来对线程的进行控制的

在ReentrantReadWriteLock中的同步队列，其实是将同步状态分为了两个部分，其中**高16位**表示`读`状态，**低16位**表示`写`状态, 只支持 65536 = 2 ^ 16 个锁错做.

支持锁降级, 是在拥有 writeLock 的时候, lock readLock, 然后再 unlock writeLock

### 1.3 CyclicBarrier

可以重置, 重复使用.

使用 lock + condition 实现. 内部计数器 count 记录执行了 await() 的线程次数. 当最后一个线程执行 await 时, 会执行 condition.signAll() 唤醒所有 await() 的线程.

### 2.1 CountDownLatch

不允许重置

内部使用 AQS 来实现同步, 多线程调用 countDown() 后直到 0, await 释放.

### 3.1 ForkJoinPool

1. 分割任务
2. 合并结果

ForkJoinPool 实现了工作窃取算法来提高 CPU 的利用率。**每个线程**都维护了**一个双端队列**，用来存储需要执行的任务。工作窃取算法允许空闲的线程从其它线程的双端队列中窃取一个任务来执行。**窃取的任务必须是最晚的任务**，避免和队列所属线程发生竞争

1. ForkJoinTask/RecursiveAction/RecursiveTask, 提供 fork() 和 join()
2. ForkJoinPool 执行 Task, 分配任务到工作线程维护的双端队列

