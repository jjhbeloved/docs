# Process

**进程通信**是一种`手段`，而**进程同步**是一种`目的`

## 进程调度算法

### 交互系统

#### 时间片轮转

> 将所有就绪进程按 FCFS 的原则排成一个队列，每次调度时，把 CPU 时间分配给队首进程，该进程可以执行一个时间片。当时间片用完时，由计时器发出时钟中断，调度程序便停止该进程的执行，并将它送往就绪队列的末尾，同时继续把 CPU 时间分配给队首的进程。

时间片轮转算法的效率和时间片的大小有很大关系：

1. 因为**进程切换**都要**保存进程的信息并且载入新进程的信息**，如果**时间片太小，会导致进程切换得太频繁**，在进程切换上就会花过多时间
2. 而如果时间片过长，那么实时性就不能得到保证

#### 优先级调度

为每个进程分配一个优先级，按优先级进行调度

为了防止低优先级的进程永远等不到调度，可以随着时间的推移增加等待进程的优先级

#### 多级反馈队列

时间片轮转调度算法和优先级调度算法的结合

设置多个队列, 每个队列分配的时间片不同, 一个任务在多次切换时间片后, 会进入 分配时间片更长的队列.

## 进程同步

1. **临界区**, 为了互斥访问临界资源，每个进程在进入临界区之前，需要先进行检查
2. **同步与互斥**, 同步是目的, 互斥是手段
3. **信号量**, 如果信号量的取值只能为 0 或者 1，那么就成为了 互斥量（Mutex） ，0 表示临界区已经加锁，1 表示临界区解锁
4. **管程**, 使用信号量机制实现的生产者消费者问题需要客户端代码做很多控制，而管程把控制的代码独立出来，不仅不容易出错，也使得客户端代码调用更容易。