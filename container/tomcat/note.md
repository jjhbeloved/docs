# Introduction

![tomcat 结构类图](./imgs/tomcat_struct.png)

tomcat 也被称作 servlet 容器

## 基本结构

Lifecycle 构建了整个 tomcat 的生命周期管理:

- connector

## init　& start

[tomcat 初始化/启动](https://blog.csdn.net/yangsnow_rain_wind/article/details/80049506)

[Tomcat组成与工作原理](https://juejin.im/post/58eb5fdda0bb9f00692a78fc)

![process](./imgs/tomcat_init_process.jpeg)

![init](./imgs/init.jpg)

init_start:
![seq](./imgs/init_start_seq.jpg)

context_parse:
![context_parse](./imgs/parse_context.jpg)

`Bootstrap` 和 `Catalina`: 通过 init -> load -> start ...

- `init` 需要设置 classloader parent 和 classloader
- `load` 使用 **digester** 解析配置文件信息, 顺序是 server -> srv.nameResources -> srv.listeners -> srv.services -> svc.listeners -> svc.executors -> svc.connectors -> connector.listeners -> engine -> host -> context/cluster
  - 同时 init container/executor/connecotr
  - 在 **digester** 解析 container 的同时, 加入 `XxxConfig(ContextConfig)` listener, 在 container start/stop/reload 时解析对应的配置文件

`StandardServer` 使用一个 awaitSocket 来 hang. 选择对应的 `service` 和 `connector` ...

`StandardService` 每个 标准service 对应一个 enginer, 多个 executors, 以及多个 connector

`Connector` 在初始化时, 要指定一个 **protocol handler** 类型, 绑定一个 ProtocolHandler, 每一个 endpoint 绑定一个 port, 并且绑定一个 adapter

> 可以同时存在多个 connector, 但是每个 connector 的 listen port 必须是独立的.
>
> init 只会初始化到 engine container, 后面的 container 没有重写 startIniternal

`Endpoint` 下是一个执行线程池(executors), 并且有**多个(默认是1) Acceptor 线程等待 accept**(使用 LimitLatch 做控制). 可以选择 init/start 时 bind

`Mapper` 在 connector 启动时, 会将 connector 对应的 service 下的 engine 对应的所有 host, host 下的所有 context, context 下的所有 wrapper 都注册到列表中

1. 当有一个 `accept` 请求时, 封装成 wrap, 从 executors 中取一个线程去执行. 将整个 socket 传入, 执行 `handler.process(Http1Processor)` 业务流程. 执行完成后, 如果是长连接LONG, 重新加入 socket 到 waitingRequests.
2. 有一个独立的 `AsynTimeout` 线程, `loop` waitingRequests 队列. 循环 Step 1
3. `Adapter` 实现将 **processor** 和 **connector** 进行连结. 使用 `mapper` 获取对应的 Context/Wrapper, request/response parse and check.
4. 通过 connector 获取到 service 下的 engine container, 并获取 pipeline 和 valves 调用链执行.

<details>
  <summary>使用 tomcat 自定义 web server 流程</summary>

``` java
/**
* 使用 tomcat 自定义 servlet
*/
    @Test
    public void connector() throws LifecycleException, InterruptedException {
        System.setProperty(Globals.CATALINA_HOME_PROP, "/tmp/webapps");
        System.setProperty(Globals.CATALINA_BASE_PROP, "/tmp/webapps");
// 这里类似 web.xml 的配置
/**
    <servlet>
        <servlet-name>xiaoxiaoWrapper</servlet-name>
        <servlet-class>HelloWorld</servlet-class>
    </servlet>
    <servlet-mapping>
        <servlet-name>xiaoxiaoWrapper</servlet-name>
        <url-pattern>/xiaoxiao</url-pattern>
    </servlet-mapping>
*/
        Context context = new StandardContext();
        context.setName("xiaoxiao context");
        context.setPath("/abc");
        context.addLifecycleListener(new Tomcat.FixContextListener());
        context.setDocBase(null);
//        context.addLifecycleListener(new ContextConfig());

        Wrapper wrapper = context.createWrapper();
        wrapper.setName("xiaoxiaoWrapper");
        wrapper.setServlet(new HelloWorld());
        wrapper.setParent(context);
        context.addChild(wrapper);
        context.addServletMapping("/xiaoxiao", "xiaoxiaoWrapper");

        Host host = new StandardHost();
        host.setName("localhost");
        host.addChild(context);
        host.addLifecycleListener(new HostConfig());
        context.setParent(host);

        Engine engine = new StandardEngine();
        engine.setName("aiur");
        engine.setDefaultHost("localhost");
        engine.addChild(host);
        host.setParent(engine);


        Service service = new StandardService();
        service.setName("trade");
        service.addLifecycleListener(event -> System.out.println("service type: " + event.getType() + ", data: " + event.getData() + ", life: " + event.getLifecycle().getStateName()));
        service.setContainer(engine);

        Connector connector = new Connector("HTTP/1.1");
        connector.setPort(8080);
        connector.addLifecycleListener(event -> System.out.println("connector type: " + event.getType() + ", data: " + event.getData() + ", life: " + event.getLifecycle().getStateName()));
        connector.setService(service);
        service.addConnector(connector);


        Server server = new StandardServer();
        server.addService(service);
        service.setServer(server);

        server.init();
        server.start();
        server.await();
    }
```

</details>

## container

1. 每个 container 都含有一个 start-stop  executors, 作用是处理 start/stop event
2. 只有 **enginer container** 保持有一个 `ContainerBackgroundProcessor` daemon thread 做后台定时调度
3. 每个 container 都内置有一个 StandardPipeline
4. 一个 container 内部可以包含多个 业务`Valve` 和一个 StandardXxxValve, 每个 valve 都指向下一个 valve, StandardXxxValve 指向下级 container, 通过 `pipeline` 来控制.

## context container

对 wrapper container 进行管理, 每个 wrapper container 是对 servlet 的封装.所以

所以 **context container** 又别称为 **servlet container**

## LifeCycle & Listener

![listener](./imgs/listener.png)

Container 都继承自 LifeCycle, 每个 lifecycle 都绑定一组 event listener, 因此所有的对此 container 部分 event 感兴趣都可以register 一个 listener.

通过 LifeCycle 可以管理整个 container 的生命周期, 基于 container 是父子继承关系的策略, 只需要管理 root container 即可

## 模式

1. 门面模式 facade, 对不需要暴露的信息, 封装一个门面进行屏蔽
2. 观察者模式 listener, 基于 event 事件驱动
3. 责任链模式 pipeline + valve, 将事件分成多段处理

## 编码

使用 CharsetMapperDefault.properties 配置文件来控制编码

## io 实现

`AbstractEndpoint` 内部包含一个 Executors 线程池, 使用 `TaskQueue` 管理线程队列.

- `TaskQueue` 继承LinkedBlockingQueue, 实现了 ArrayBlockingQueue 的范围控制, 同时又允许在无法正常加入线程池时(线程池满, 线程池会维护 coreSize), force 强制加入.

使用 LimitLatch 限制接入的 connection count.(因为采用的是 NIO 多路复用 select, 所以 connect 需要主动限制)

### jio

jio 使用 acceptor 接受 n 个 connections, 每个connection 交给一个 executors 线程池, 线程池的大小, 决定了并发处理的上线. 没有获取到线程的 connection 都会 wait 直到过期.

### nio

### processor

线程级:

1. 当每次有一个 socket 请求到来时, 会使用 `SocketWrapper<Socket>` 封装到 `SocketProcessor`中, 交给线程池去处理(默认线程池线程数是 200 个)
2. SocketProcessor 会根据 socketServer 是否是 SSL 决定是否需要 SSL 校验. 将数据交给 `handler(Http11ConnectionHandler)` **数据对象级processor**处理.
3. 如果是**超时的长连接**, 返回 `state.LONG`, 将 socket 放至到 `waitingRequests` 并发队列中, 等待下一个请求.
4. 每个 socket 对应的都是一次性的线程, 执行完线程就回收给线程池.
5. 长连接(还有数据没读完) 和 launch升级的/keep-alive socket, 通过 waitingRequests 和 新建一个线程来实现 socket的复用.(**这里连接已经建立, 只是没有 request 数据, 这时候会依靠 async 机制间隔性的sleep 1 s**)

数据对象级:

1. 创建数据 processor, 并且注册到 registry中 **todo**???
   1. 主要做 chunked, identity, void, gzip, buffered filter 初始化
2. 根据状态调用 `org.apache.coyote.Processor` 下的 `process`
   1. 准备 socket
   2. parse requestLine
   3. parse header
   4. preparse request(这里只是给 add request filter), 而 body chuncked 是用filter 做解析, 真正的解析在第一次调用 getParameter 时完成. 这里会判断 body input stream buffer 是否 eof. 会一直读到 CRLF 结束
      1. identity 普通 body
      2. chunked 分片传输 body
   5. `coyoteAdapter.service(request, response)`
   6. 查看 keepalive 是否已经超时, 需要重新链接
3. 根据调用结果 决定是否 release, 还是 poller
   1. poller 会循环使用, 长连接, 还有数据没有读完, 不能清空 processor 记录的数据offset信息
   2. release 会 recycle(重置一些数值), 但是这个 processor 是可以回收利用的. 加入到 RecycleQueue 队列中.

[Tomcat http Body 解析源码分析](https://www.jianshu.com/p/d8a2bc7d3c21)

### AsyncTimeout Thread

1. 如果 `waitingRequests` 没有数据, 说明没有新的 client connect 或者 client request, 这时候会 sleep 1.
2. 并且会监听 running/pause 状态
3. 如果有信号, 则会启动一个线程去执行 `SocketProcessor`
4. **todo** 这里很奇怪 threads should not be created by the webapp classloader, 会设置 classloader

### stop/pause

因为在没有请求到来的时候ServerSocket会一直阻塞，使用 `unlockAccept` 会给serverSocket发送一个请求来unlock阻塞. 以唤醒 socketServer 来监听 running/pause 状态, 以决定是 pause listen 还是 stop.

- pause & running 的时候, acceptor 会陷入 5s 的 sleep loop中.

关闭 serverSocket, 并且 clear progressor

## 定时调度器

## xxx
