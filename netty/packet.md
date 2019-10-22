# packet

## 拆包/粘包问题

netty 使用 ByteBuf 作为一个缓冲区, 如果报文太大, 会被拆分 n 个packet. 需要自己做判断

1. tcp 如何保证数据 拆包后是有序的
2. 如何保证 拆包后的接收方收到的数据是连续的

- `DelimiterBasedFrameDecoder` 是基于**消息边界方式**进行粘包拆包处理的
- `FixedLengthFrameDecoder` 是基于**固定长度消息**进行粘包拆包处理的
- `LengthFieldBasedFrameDecoder` 是基于**消息头指定消息长度**进行粘包拆包处理的
- `LineBasedFrameDecoder` 是基于**行**来进行消息粘包拆包处理的

## option

- RCVBUF_ALLOCATOR: 接收 ByteBuf 缓冲大小
- SO_SNDBUF: 缓冲区大小??

