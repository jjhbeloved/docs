# zookeeper

[使用zkclient操作zookeeper的学习过程记录](https://www.cnkirito.moe/zkclient-learning/)

ZooKeeper保证只要大多数服务器可用((N+1)/(2N+1))，整个服务就是可用的

##　guarantees

- **顺序一致性(Sequential Consistency)**: 客户端的更新会按照发送顺序进行操作。
- **原子性(Atomicity)**: 更新操作只会是成功或失败，不存在其他异常结果。
- **单一视图(Single System Image)**: 客户端无论连接到哪个server，看到的都是相同的视图。
- **可靠性(Reliability)**: 当更新操作被执行后，它将一直有效，直到下一次更新操作的执行。
- **及时性(Timeliness)**: client的视图保证在一定时间内能得到更新。

## znode

ZNode由两部分组成：**协调数据**和**状态数据**

- **协调数据**: 业务相关的数据
- **状态结构数据包含**: 数据变更的版本号， ACL(Access Control List)变更，时间戳，以用来缓存验证和协调更新。每次Znode数据发生变化，版本号就会自增。

### znode type

zookeeper 类似于一个文件系统, 数据以文件路径节点的方式存储, 但是节点有4种类型, 由 1,2,1+3,2+3组合而成

1. **Ephemeral**, ephemeral节点是临时性的, 如果创建该节点的session结束了, 该节点就会被自动删除. ephemeral节点不能拥有子节点
2. **Persistent**, persistent节点不和特定的session绑定, 不会随着创建该节点的session的结束而消失, 而是一直存在, 除非该节点被显式删除.
3. **Sequence**, sequence并非节点类型中的一种. sequence节点既可以是ephemeral的, 也可以是persistent的. 创建sequence节点时, ZooKeeper server会在指定的节点名称**后加上一个数字序列**, 该数字序列是递增的. 因此可以多次创建相同的sequence节点, 而得到不同的节点
   1. 该**计数器**是由**父节点控制**, 直接对下级节点生效
   2. **每个父节点**都会负责维护其**子节点创建的先后顺序**，并且如果创建的是顺序节点（SEQUENTIAL）的话，父节点会自动为这个节点分配一个整形数值，以后缀的形式自动追加到节点名中，作为这个节点最终的节点名。

## sessions

client通过TCP连接到单个server，通过该连接发送请求，获取响应，获取监视事件以及发送心跳，当该TCP连接断开后，client会自动连接到其他server

ZK Client通过ZooKeeper提供的client binding 代码（官方提供：C/Java两种Client API）创建一个handle来和ZooKeeper服务建立Session连接。创建一个handle进行连接后，client会处于**CONNECTING**状态，然后client library会进行尝试对ZK服务集群中的server发起连接，成功后client会置为**CONNECTED**状态。当发生不可恢复的错误，例如会话过期，鉴权失败，或者client主动进行关闭，handle会切换为**CLOSED**状态

当client session state从CONNECTED由于disconnected事件变成CONNECTING后，**不建议**创建新的session对象进行连接，因为ZK client library会**自动进行重连**，特别ZK client lib中内置了一些启发式方法来处理“羊群效应”之类的事情。在使用过程中，**仅需要在收到会话到期通知时进行新会话的创建？？？**(这里要看client的实现)

- **会话的超时**管理是由**ZK server负责的**，不是由client负责。当ZK client 创建一个session时传入了一个合法的timeout，ZK集群就会根据该值对client的session进行过期管理。当集群在设定的timeout时间段内没有收到client的请求(心跳)，集群就会认定该session过期了，集群就会将session拥有的ephemeral nodes全部删除，并通知到所有监听被删除nodes的clients(CONNECTED)，如果此时该过期session的client仍然是未连接状态，将不会通知到该client，且该client将一直处于disconnected状态直到该TCP连接重连成功，重连成功后其将会收到SESSION_EXPIRED的通
- **session的保活**是通过**client**来发送请求来实现，当在一段时间内session处于空闲状态，client会发送PING请求来使session保活，PING请求不仅可以让ZK集群直到client是活的，也能让client知道到ZK集群的连接是否是活的
- 总结就是, **keepalive是由client发送的, 至于是否timeout则由服务端来判断**

## watch

客户端可以在znodes上设置监听，ZooKeeper中所有的read操作：getData()，getChildren()和exists()，都提供了参数来设置watch。关于watch的定义：**watch事件是一个一次性的触发器，当watch的Znode发生变更的时候，ZooKeeper会向客户端发送通知**

watch三个特性:

- **一次性触发**, 当监听的Znode发生变化时，**会向client发生一个watch event, 但是客户端注册的watch就失效了, 如果需要继续监听, 需要主动注册监听**。例如：client调用getData(“/znode1”, true)，之后/znode1发生了变化或者删除，client会收到一个watch event，但当/znode1再次发生变化时，client不会在收到watch event，除非client再次调用read操作来设置监听。
- **watch event发送的顺序性**, 发送给client的watch event，在更改操作成功的返回代码到达发起更改的客户端之前，可能无法到达client。监听事件是异步发送给watcher。但ZK提供了顺序性保证：client不会发现其监听的znode发生变化直到它收到watch event。即client会**先收到watch event，然后才会看到Znode的数据**。Watch events的顺序和ZK集群中的对于更新的顺序是严格一致的。
- **监听的分类**, ZooKeeper中存在两种watches：data watches和child watches。getData() and exists() 接口会设置data watches。getChildren() 接口会设置child watches。之所以这么设计是因为，getData() and exists() 接口是用来返回Znode的data，而getChildren() 是返回Znode所有的children列表。setData() 会触发data watch，create() 会触发data watch和父节点的child watch，delete()也同样会触发data watch和父节点的child watch

- Created event：`exists`
- Deleted event： `exists, getData, and getChildren`
- Changed event：`exists, getData`
- Child event：`getChildren`

> warning. 由于watch的一次性触发特性，在获取watch event和发送新的请求来再次进行znode的监听之间是**有延迟**的，所以这中间ZNode可能发生了多次变化，但client不会有watch event的通知.

### 羊群效应

注册监听, 来做分布式锁. 普通的方法是在 parent znode上注册监听, 当parent znode下的字节点发生变更, 就判断watcher者是否是最小的节点, 以此来做抢锁. 这个方法的问题在, 绝大多数情况下, watch到的事件都是无效的(自己并不是最小的). 这个在集群规模大的时候, 引发**羊群效应（Herd Effect）**. 解决方案是使用`exist`, 只注册在比自己**小一位**的那个节点上, 只有这个节点发生变更才会有watch.

## reference

[ZooKeeper的基本介绍](http://walkerdu.com/2019/02/11/zookeeper_basic/)
[zookeeper分布式锁避免羊群效应（Herd Effect）](https://blog.csdn.net/xubo_zhang/article/details/8506163)