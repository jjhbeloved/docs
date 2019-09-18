# NAS 说明

## 1. NAS 与 SAN 区别

**NAS(Network-Attached Storage)** 通过 **以太网(NFS/CIFS)等网络协议** 使用同一个`FS`文件系统管理
**SAN(Storage Area Network)** 通过 **独立的光纤通道交换机** 每个服务器有各自的`FS`文件管理系统, 但是交换机保证链接信息有序

1. NAS有文件操作和管理系统，而SAN却没有
2. SAN主要是高速信息存储，NAS偏重文件共享。
3. SAN和NAS相比不具有资源共享的特征
4. SAN是只能独享的数据存储池，NAS是共享与独享兼顾的数据存储池。
5. NAS是网络外挂式，而SAN是通道外挂式。
6. SAN高效可扩，NAS简单灵活

## 2. NAS 性能问题

协议最大的缺点是：当NAS控制器有多个连接任务的时候，系统 **无法保证每个连接的服务质量**，换个说法就是，当NAS控制器的负载达到一定的程度的时候，哪个连接在系统获得了响应，系统就调用尽量多的资源为该连接服务，但是在这个时候，系统中其他的连接服务就无法正常的保障了

## 3. 协议

### 3.1 NFS

NFS 的基本原则是“容许 **不同的客户端及服务端** 通过 **一组RPC** 分享相同的文件系统”，它是 **独立于操作系统**，容许不同硬件及操作系统的系统共同进行文件的分享

**NFS** 在文件传送或信息传送过程中 **依赖于RPC协议**, NFS本身就是使用RPC的一个程序。或者说NFS也是一个 RPC SERVER。所以只要用到NFS的地方都要启动RPC服务，不论是 **NFS SERVER** 或者 **NFS CLIENT**

``` text
nfs-utils-* ：包括基本的NFS命令与监控程序
rpcbind-* ：支持安全NFS RPC服务的连接
```

### 3.2 CIFS

xxx

----
[nfs和rpcbind](http://linux.vbird.org/linux_server/0330nfs.php)
