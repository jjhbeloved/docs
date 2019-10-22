# Interview

## reference

- [redis 知识解答](https://zhuanlan.zhihu.com/p/41951014)
- [redis 内存](https://yq.aliyun.com/articles/67122)
- [redis 奇怪问题集](https://blog.csdn.net/qq_35923749/article/details/85240071)

## 索引和跳表

[聊聊Mysql索引和redis跳表](https://zhuanlan.zhihu.com/p/61900308)

### question

1. mysql 索引如何实现
2. mysql 索引结构B+树与hash有何区别。分别适用于什么场景
3. 数据库的索引还能有其他实现吗
4. redis跳表是如何实现的
5. 跳表和B+树，LSM树有和区别呢

### resolve

解决的都是同一种问题，**用于解决数据集合的查找问题，即根据指定的key，快速查到它所在的位置（或者对应的value）**

1. 需要支持哪些查找方式，单key/多key/范围查找
2. 插入/删除效率
3. 查找效率（即时间复杂度）
4. 存储大小（空间复杂度）

### answer

1. mysql 索引使用 B+ 树
2. hash 支持(k,v)格式, 能快速根据 k 计算出 hashcode 来查找到 v
3. B+ 树
   1. 数据有序结构
   2. 考虑存储的效率, 不让树高度过高, 每一个节点可以有n个子节点
   3. 叶子节点不是存储单个数据, 而是存储一页(page和操作系统对其)数据
   4. 叶子节点存储的数据冗余了非叶子节点数据, 以便支持范围查询
   5. 叶子节点之间通过指针链接, 来支持翻页, 减少磁盘I/O
4. skip list
   1. 有n个level
   2. 每个level 之间的节点数是 1/2
   3. 通过节点链接来实现链表的二分查找
5. 由于 `redis` 是**单线程**操作的, skip list 的实现使用的 一个 node 上有n个level链表这种数据结构
