# File

## 文件描述符号

[文件DIRECT 和 SYNC 区别](https://blog.csdn.net/AXW2013/article/details/70242228)

[内存对齐问题](https://www.jianshu.com/p/49f7e6f56568)

## DirectIO

> Linux允许应用程序在执行磁盘IO时绕过**缓冲区高速缓存**，从**用户空间直接将数据传递到文件或磁盘设备**，称为直接IO（direct IO）或者裸IO（`raw IO`）
>
> - 用于传递数据的**缓冲区(buffer)**，其**内存边界**必须**对齐(aligned block)**为**块大小(block size)的整数倍**
> - 数据传输的**开始点**，即文件和设备的**偏移量offset**，必须是**块大小(block size)的整数倍**
> - 待传递**数据的长度**必须是**块大小(block size)的整数倍**
>
> 不遵守上述任一限制均将导致`EINVAL`错误

directBuffer 需要 **AlignedBlock对齐块**

## object storage key

`date-ts-mac-offset-size` 可以定位到文件位置, 以及文件可以append的offset

## 文件io

[文件IO操作的一些最佳实践](https://www.cnkirito.moe/file-io-best-practise/)
