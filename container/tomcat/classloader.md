# classloader

重写 classloader 核心点是: **类库路径**(repositories) 和 **加载顺序**

1. 加载顺序决定了是否符合双亲委派原则.
2. 类库路径决定了当前 classloader 允许加载的类
3. 当前 classloader 无法加载 class 的时候, 是否使用 parent.classloader 或者 thread.contextClassloader 进行加载.

## init

通过 common/shared/server 不同的目录创建不同的 classloader(都是使用UrlClassLoader来创建)

## webclassloader

在 context start时, 会将 context 所在的目录下的 **/WEB-INF/classes/** 和 **/WEB-INF/lib** 作为 repositories.

重写了 findClass 和 loadClass. 一般会使用 `loadClass` 去加载一个类.

``` java
// find class 逻辑
// 1. 先查找本地是否存在 class name
findClassInternal(name);
// 2. 不存在, 再使用双亲委派原则

// load class 逻辑
// 1. 先查找本地缓存 class
clazz = findLoadedClass0(name);
// 2. 再查找系统缓存 class
clazz = findLoadedClass(name);
// 3. 使用 bootstrap classloader 加载 class
clazz = j2seClassLoader.loadClass(name);
// 4. 如果有使用 delegateLoad 则使用 parent 加载 class(双亲委派)
clazz = Class.forName(name, false, parent);
// 5. 使用 find class 逻辑
clazz = findClass(name);
// 6. 如果有没有使用 delegateLoad 则使用 parent 加载 class(双亲委派)
clazz = Class.forName(name, false, parent);
```
