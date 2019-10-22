# classloader

[深度分析Java的ClassLoader机制（源码级别）](https://juejin.im/entry/5a73be026fb9a0634c263ec5)

## 双亲委派

1. Bootstrap ClassLoader(`rt.jar` 等基础类型是由此 类加载器加载)
2. Extension ClassLoader
3. Application ClassLoader

- 如果要维持双亲委派模式, 重写 findClass 和 resolveClass.
- 否则重写 loadClass
- 重写 defineClass

### 类加载流程

加载 -> 链接(校验 -> 准备 -> 解析) -> 初始化

``` java
// 通过以下方法, 可以减少 初始化时间
X x = new X(); // 如果是首次, 隐式的执行到 初始化阶段
Class<?> class = classloader.loadClass(); // 进入 加载阶段, 并不会执行 静态类块 static{}
Class<?> class = Class.forName(); // 进入 解析阶段, 不会执行 构造函数
```

## 实现

1. ClassLoader: 定义了类加载的核心操作。
2. SecureClassLoader: 继承自ClassLoader，负责policy权限等支持。
3. URLClassLoader: 继承自SecureClassLoader，支持从 `jar文件` 和 `package路径中获取class`
   1. 使用 URLClassLoader 需要注意 package 带来的路径影响 [Java含package语句下的编译运行](https://zhuanlan.zhihu.com/p/23501775)
   2. 将package 路径整个打成一个 jar 包, `java cvf xx.jar {package path}`
   3. 如果需要知名 jar 的可执行程序, 需要手动生成 MANIFEST.MF 文件, 或者手动指定 `java cvfe xx.jar xx.xx.MainClass {package path}`

``` java
// Encapsulates the set of parallel capable loader types.
// 封装了并行的可装载的类型的集合
private static class ParallelLoaders {}
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

## 与反射有关的东东

从外部加载的 .class 后, 可以写入脚本直接引用 class, 否则都需要使用反射来实现.

## SecurityManager

> 当运行未知的Java程序的时候，该程序可能有恶意代码（删除系统文件、重启系统等），为了防止运行恶意代码对系统产生影响，需要对运行的代码的权限进行控制，这时候就要启用Java安全管理器

1. 没有配置的权限表示没有
2. 只能配置有什么权限，不能配置禁止做什么
3. 同一种权限可多次配置，取并集
4. 统一资源的多种权限可用逗号分割