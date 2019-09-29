# reflect

1. 运行时 判断对象所属类
2. 运行时 构造任意对象
3. 运行时 判断类所具有的成员变量和方法
4. 运行时 调用任意对象方法

运行时, 非编译时

## 优缺点

优点:

1. 可扩展, 适合开发框架

缺点:

1. 性能开销大, 解释执行(但是可以把代码加载进 method area 变成静态代码)
2. 安全性差, 反射调用方法时可以忽略权限检查，因此可能会破坏封装性而导致安全问题
3. 破坏了抽象性