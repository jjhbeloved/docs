### １. 概念
**存储区域网络**（Storage Area Network，简称SAN）采用网状通道（Fibre Channel ，简称FC，区别与Fiber Channel光纤通道）技术，通过FC交换机连接存储阵列和服务器主机，建立专用于数据存储的区域网络
### 2. 组成
#### 2.1 主机层
主机使用 **HBA卡**，一般来说都是 **LC接口** 的，由于交换机大多也都是LC 接口类型的，所以我们使用光纤大多 **lc-lc** 类型的
HBA卡要想连接 **光纤交换机** ，中间必须要有 **光纤线**
#### 2.2 交换层
**FC光纤交换机** 和 **SFP模块**

> **小型的SAN网络** 基本通过一个或几个 **光纤交换机** 独立工作或者 **级联** 就可以支撑整个公司的SAN网络环境
> **特大型的SAN网络** 就需要一个 **路由器（router）**，大型TCP/IP 网络需要路由器，大型的SAN网络同样也需要路由器

光纤交换机目前市场主流的是接口速率 **16GB和32GB** ，端口从 **8口到80口**，更多的端口需要可以支持插入背板的光纤交换机，以来扩容端口的需求
#### 2.3 存储层
存储层只是一个概括而已，主要是指连接在光纤交换机上用于 **提供数据存放的设备**，如 **存储设备，磁带库设备，NAS设备** 等
存储设备大多都有2到多个控制器，**控制器** 通过光纤设备连接到光纤交换机，在交换机上配置相应的Zone,从而识别主机，映射到主机，最终完成主机设备存储，带库等相关设备的操作
#### 2.4 监控层

-----
[SAN描述参考](https://www.sohu.com/a/215559593_151779)
[SAN/NAS/DAS描述](https://www.cnblogs.com/bitepeng/p/4142676.html)