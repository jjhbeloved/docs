# JVM

## 1. 垃圾回收

分为 新生代(eden, s1, s2 - 8:1:1), 老年代(old), 永生代

- `minor/major gc`
- `full gc`(当内存或者是连续内存不足时, 会执行 full gc) 对 heap 内所有内存进行扫描, 扫描期间, 绝大多数java thread 会暂停, stw(stop the world).

### 1.1 算法

**可达性分析**: GC Roots

1. 方法区中的 静态变量引用对象
2. 方法去中的 常量引用对象
3. 本地方法栈 JNI中引用的对象
4. 线程栈 局部变量表引用对象

垃圾回收算法:

1. mark-sweep, 大量碎片
2. mark-compact, 大量数据移动
3. copy, 浪费空间
4. 分代, 年轻代使用 copy, 老年代使用 mark-sweep/mark-compact

![gc回收器](./imgs/gc.jpg)

1. `serial`, 年轻代, 单线程 GC
2. `ParNew`, 年轻代, 多线程 GC
3. `Parallel Scavenge`, 年轻代, 多线程(吞吐优先)
4. `Serial Old` 老年代, 单线程
5. `Parallel Old`, 老年代, 多线程(吞吐优先)
6. `cms concurrent mark sweep`, (响应优先)
   1. 初始标记(暂停)
   2. 并发标记
   3. 重新标记(暂停)
   4. 并发清除
   5. 问题:
      1. 吞吐量低
      2. 碎片多, 可以使用 serial old 来做一次 compact
      3. 无法处理浮动垃圾, 需要预留 30% 内存作为存放浮动垃圾
   6. 响应优先 意味着 更短的垃圾回收频率 和 更小的堆内存
7. `g1`, 将 eden, s, old 划分成大小相等的 region
   1. 初始标记
   2. 并发标记
   3. 最终标记
   4. 筛选回收
   5. 每个 region 可以独立回收, 预测

### 1.2 方法区回收

方法区现在都使用 meta 服务器内存来保存, 但是存在反射和动态代理产生的类信息, 需要回收.

1. class instance 都被回收, 没有在堆中
2. load class classloader 被回收
3. class 没有被引用, 也没有任何地方通过反射访问

### 1.3 内存分配策略

1. 对象优先分配在 eden 上
2. 大对象直接分配到 old(避免无意义的拷贝)
3. 长期存活对象进入 old(每活过一次minior gc加1)
4. s 中同年龄对象总和大于 s空间的一半, 全部进入 old
5. 分配担保, 若是进入老年代空间足够, 进入. 否则查看浮动空间总和是否足够, 进入. 否则触发 FULL GC

FULL GC 条件:

1. System.gc()
2. old 空间不足
3. 空间担保失败
4. 永久代不足

## 2. 运行时数据区

1. 线程
   1. JVM Stack(Local Varaiable Array, Reference To Constant Pool, Operand Stack)
   2. Native Method Stack(JNI)
   3. PC Register(正在执行的虚拟机字节码指令地址)
2. 共享
   1. Heap(分代算法, Young + Old)
   2. Method Area(Class信息, Constant, Static Variable, JIT Class...)
   3. Runtime Constant Pool(String.intern)
   4. Direct Memory(Native DirectByteBuffer)

> **TLAB存在的意义**
>
> 我们知道多线程在分配堆内存时，由于堆是**一整块**内存，出于**线程安全**考虑，分配内存需要对堆**加锁**, 极度消耗资源。`TLAB`就是在Eden上开辟一小块空间，**每个线程**拥有一个自己**独有**的空间，在分配内存时先分配TLAB空间。当TLAB空间被分配完了之后，**再进行**加锁堆内存的分配

## 3. 类加载机制

> 加载 -> 链接(校验 + 准备 + 解析) -> 初始化 -> 使用 ->卸载

1. 加载: 从.class, url, byte 等导入. 将数据结构保存在Method Area, 在内存中生成此类的 Class 对象.
2. 校验: 校验 class 对像是否符合规范
3. 准备: 静态类变量在 Method Area 中分配内存, 赋给初始值
4. 解析: 将 符号引用 转成 直接引用.
5. 初始化: 执行<clinit>() 完成静态变量赋值. <clinit>() 是由编译器自动收集类中所有类变量的赋值动作和静态语句块中的语句合并产生的，编译器收集的顺序由语句在源文件中出现的顺序决定。 **<clinit>()是加载时执行的, 线程安全(虚拟机保证)**

### 3.1 分类

1. bootstrap classloader
2. extension classloader
3. application classloader
4. common classloader, tomcat + web 共享
5. catalina classloader, tomcat 独享
6. shared classloader, web 共享
7. webapp classloader, web 独享
8. jasper classloader, web 独享

### 3.2 双亲委派

> 一个类加载器首先将类加载请求转发到父类加载器，只有当父类加载器无法完成时才尝试自己加载

优点:

1. 层次关系, 让基础类(父类)统一, class 依赖和引用的其他 class 也一并加载
2. 安全, 基础类不会被覆盖
3. 内存节约, 不会重复加载, 加载过后的对象存放在 method area 中作为缓存.

## 4. 异常处理

``` java
class Throwable {}
class Error extends Throwable {}
class Exception extends Throwable {}
```

> Error和Exception的区别：
>
> `Error` 通常是灾难性的致命的错误，是程序无法控制和处理的，当出现这些异常时，Java虚拟机（JVM）一般会选择终止线程
>
> `Exception` 通常情况下是可以被程序处理的，并且在程序中应该尽可能的去处理这些异常。