# WAL

1. 先不考虑并发问题
2. 设计出接口
3. 设计出数据结构

## 1. 建议文件锁 flock

[linux文件锁flock](https://www.cnblogs.com/kex1n/p/7100107.html)

flock，建议性锁，**不具备强制性**。一个进程使用flock将文件锁住，另一个进程可以直接操作正在被锁的文件，修改文件中的数据，原因在于flock只是用于检测文件是否被加锁，针对文件已经被加锁，另一个进程写入数据的情况，内核不会阻止这个进程的写入操作，也就是建议性锁的内核处理策略。

flock主要三种操作类型:

1. `LOCK_SH`，共享锁，多个进程可以使用同一把锁，常被用作读共享锁；
2. `LOCK_EX`，排他锁，同时只允许一个进程使用，常被用作写锁；
3. `LOCK_UN`，释放锁；

进程使用flock尝试锁文件时，如果文件已经被其他进程锁住，进程会被阻塞直到锁被释放掉，或者在调用flock的时候，采用 `LOCK_NB` 参数，在尝试锁住该文件的时候，发现已经被其他服务锁住，会返回错误，errno错误码为 `EWOULDBLOCK`。即提供两种工作模式：**阻塞** 与 **非阻塞** 类型。

## 2. 预留文件空间 fallocate

[用fallocate进行"文件预留"或"文件打洞"](https://blog.csdn.net/weixin_36145588/article/details/78822837)
[Linux下的fallocate操作](https://blog.csdn.net/ZR_Lang/article/details/51032815)

du 和 ls 的区别:

1. du 查看 磁盘占用大小
2. ls 查看 文件大小
3. 文件系统 一个 block 4k, 比如 13kb的文件, 占用 block=13/4=3.25个, 由于 磁盘的一个block 只能被一个文件占用. 因此实际占用的磁盘空间是 4k*4=16kb
4. du 显示的值往往要比 ls 大.
5. 但是 空洞文件, 实际上只会占用开头和结尾的 2个字节(中间为0的数据不会落盘), 因此 du 显示占用磁盘空间为 4kb. 而 ls 显示的大小是 文件属性, 这个值可以人为修改.

### 空洞文件

文件的 **位移量offset** 可以大于文件的**当前长度**, 这种情况下, 对文件的 **下一次写** 将时间 **扩容** 该文件, 在文件中形成 **全部字节为0** 的空洞(一个文件的**两头有数据**而**中间为空**，以'\0'填充). 空洞**是否占用硬盘空间**是由文件系统（file system）决定的. 大部分文件系统是**不占用**的.

文件的 `metadata/inode` 会记录逻辑大小, 而物理只会有基本的 inode 大小 4k.

### 优点

1. 可以让文件尽可能的占用 **连续的磁盘扇区**, 减少后续写入和读取文件时的磁盘寻道开销
2. **迅速占用磁盘空间**, 防止使用过程中所需空间不足
3. 后面再追加数据的话, 不会需要改变文件大小，所以后面将 **不涉及metadata的修改**

## 3. 减少同步IO fdatasync

[linux 同步IO: sync、fsync与fdatasync](https://blog.csdn.net/cywosp/article/details/8767327)
[buffer cache/ page cache](https://zhuanlan.zhihu.com/p/35277219)

1. sync, 将 **所有** 修改过的块缓冲区排入写队列，不等待结果立即返回
2. fsync, 只对 **单一文件** 其作用
   1. 确保将修改过的块立即写到磁盘
   2. 确保更新文件的属性
3. fdatasync, 只对 **单一文件** 其作用, **只做数据块同步**

对于提供事务支持的数据库，在事务提交时，都要确保事务日志（包含该事务所有的修改操作以及一个提交记录）**完全写到硬盘上*，才认定事务提交成功并返回给应用层

Berkeley DB是怎样处理日志文件:

1. 每个log文件固定为10MB大小，从1开始编号，名称格式为“log.%010d"
2. 每次log文件创建时，先写文件的最后1个page，将log文件扩展为10MB大小
3. 向log文件中追加记录时，由于文件的尺寸不发生变化，使用fdatasync可以大大优化写log的效率
4. 如果一个log文件写满了，则新建一个log文件，也只有一次同步metadata的开销

## 4. 大小端

Java 在 x86 默认使用 小端存储, 在 x64 默认使用大端存储

这里存储时用小端存储, 读取的时候用小端读取

``` text
unsigned int value = 0x12345678

内存地址 小端模式存放内容 大端模式存放内容
0x4000 0x78     0x12
0x4001 0x56     0x34
0x4002 0x34     0x56
0x4003 0x12     0x78
```

## 5. interface

``` java
```

``` java
```

## 6. 原理

### open create

初始化, 创建一个文件, 使用 fallocate 空洞文件, 预留出连续块. 并且给操作文件 加上逻辑文件锁 flock.

先给文件写入 head 数据, crc, meta, snap. 写完 head 后, 对齐 page, 调用 fdatasync 完成数据块同步(因为之前使用了 fallocate 不需要再去同步文件属性了.)

### open

根据传入的 index 信息, 找到所有大于索引的文件. 使用 read write 模式打开文件句柄 fd. 并且 使用 flock 加上逻辑文件锁(加不上立即报错)

如果打开的是 write 模式, 可以**预**创建一个 tmp 的空洞文件, 当需要 cut 出一份新文件时, 可以直接使用.(只需要改变文件的 open mode 即可)

``` text
初始化时会创建 reader, 不会创建 writer.
是 wal 日志, 必须要读到最后一个 frame 才允许写
当读到最后一个 frame 后没有值了, 再去创建一个 encoder(而不是提前就去创建 writer).
这时候 wal 已经只要 write 不用 read 了, 所以关闭 reader
```

### read