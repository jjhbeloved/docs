# 基础

## 数据流向

[图解Netty之Pipeline、channel、Context之间的数据流向](https://blog.csdn.net/u012562943/article/details/53256561)

read 只会查找 in 的 handler, write 只会查找 out 的 handler

虽然 in 从  header -> tailer, out 从 tailer -> header. 但是在 pipeline.add 进的就是 in/out 的顺序

## timer 机制