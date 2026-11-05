# 01｜Netty 总览与核心组件

![[附件/知识总览/01-Netty总览与核心组件.png]]

## 1. Netty 解决什么问题

Netty 是异步、事件驱动的网络应用框架。以 NIO 传输实现为例，它把连接注册、事件循环、缓冲区、异步写出和处理链组织起来，让业务代码围绕事件与消息编写。

| 模型 | 等待与处理方式 | 应用主要负责什么 |
|---|---|---|
| BIO | 线程可能阻塞在连接的读写操作上 | 连接线程、流、协议及资源关闭 |
| 原生 NIO | 通过 Selector 等待多条连接的就绪事件 | 注册与遍历、缓冲、分帧、部分写入与续写 |
| Netty NIO 传输 | EventLoop 驱动连接，Pipeline 组织处理过程 | 协议、编解码、业务及应用级资源策略 |

**Netty 不等同于 Java NIO。** `NioEventLoopGroup`、`NioServerSocketChannel` 和 `NioSocketChannel` 属于其 NIO 实现，框架也支持其他传输实现。

Netty 简化网络编程，但不会替应用定义消息格式、业务确认、异常策略及连接存活规则。

## 2. 服务端处理主线

### 2.1 启动与接入连接

1. `ServerBootstrap` 组合线程组、Channel 类型、网络参数及处理器配置，调用 `bind()` 绑定监听地址。
2. 监听 Channel 交给 boss 的 EventLoop 管理，负责接受连接。
3. 接受得到的连接 Channel 注册给 worker 的某个 EventLoop，处理这条连接的 I/O。
4. `ChannelInitializer` 为连接安装 Handler，组成它自己的 Pipeline。

监听端口的 Channel 与已连接的 Channel 是不同对象。boss、worker 是职责名称，不是两种不同的 EventLoopGroup 接口类型；线程数由配置与实现决定。

### 2.2 收到请求与发送回复

入站数据先由分帧器识别消息边界，再由解码器转成业务对象，最后交给业务 Handler。

业务生成回复后，写请求经过适用的出站 Handler，编码成网络字节，再由传输层写出。编码器本身就是一种出站 Handler，不能理解成与出站处理互不相关的另一条链。

这里描述的是典型业务流程，不代表所有协议都需要独立分帧器，也不是 Pipeline 的物理排列图。入站、出站的实际传播与调用起点、Handler 类型及排列位置有关。

## 3. 核心组件及原生 NIO 对照

### 3.1 组件职责

| 组件 | 职责 |
|---|---|
| `Bootstrap` / `ServerBootstrap` | 组合配置，分别用于客户端等场景的连接引导及服务端绑定引导 |
| `EventLoopGroup` | 管理一组事件循环，为 Channel 分配 EventLoop |
| `EventLoop` | 处理已注册 Channel 的 I/O 事件、普通任务与定时任务 |
| `Channel` | 监听端口或连接等网络实体的抽象 |
| `ChannelInitializer` | 初始化 Channel，通常用于安装处理链 |
| `ChannelPipeline` | 按顺序组织一个 Channel 的 Handler |
| `ChannelHandler` | 处理连接事件、消息及出站操作 |
| `ChannelHandlerContext` | Handler 在 Pipeline 中的上下文，可访问 Channel 并传播事件或发起操作 |
| `ByteBuf` | 存放和操作网络字节，支持引用计数等机制 |
| `ChannelFuture` | 观察异步 Channel 操作的状态与结果 |
| `ChannelPromise` | 在 Future 基础上允许实现方完成结果；出站透传时应继续携带原 Promise |

**Future 是结果凭证，不是执行 I/O 的线程。** 调用返回与操作完成是两个时点，可通过监听器观察结果；在 EventLoop 中阻塞等待会影响它管理的其他连接。

### 3.2 原生 NIO 的工作由谁承接

| 原生 NIO 工作 | Netty 中的组织方式 |
|---|---|
| 打开服务端 Channel、绑定端口 | `ServerBootstrap` + `NioServerSocketChannel` |
| 监听 `OP_ACCEPT` 并接收连接 | boss EventLoop 管理监听 Channel |
| 将连接注册到 Selector | 将连接 Channel 注册给 worker EventLoop |
| select、处理就绪事件、执行任务 | EventLoop 的循环 |
| 保存连接状态与缓冲 | Channel 属性、Handler 状态与 ByteBuf 等；共享 Handler 需注意状态隔离 |
| 读取字节、拼接完整消息 | 传输层读取配合 ByteBuf、分帧器和解码器 |
| 部分写入后继续发送 | 传输层管理待写数据，必要时通过写就绪事件续写 |

## 4. 三个组织维度

| 名称 | 组织什么 | 典型用途 |
|---|---|---|
| `EventLoopGroup` | 事件循环 | 调度并处理多个 Channel 的事件与任务 |
| `ChannelGroup` | 连接等 Channel 的集合 | 批量发送、关闭及集合管理 |
| `ChannelPipeline` | 单个 Channel 的处理步骤 | 组织编解码、业务及事件处理 |

三者不是上下级关系。连接放进 ChannelGroup，不会因此共享 Pipeline，也不会因此改变所注册的 EventLoop。

## 5. 必须分清的边界

- **连接与线程：** 一个 EventLoop 可管理多个 Channel，不能认为每条连接永久独占一个线程。
- **Pipeline 与执行线程：** 每个 Channel 有自己的 Pipeline；某些 Handler 可配置到其他执行器，不应笼统认为全链都在同一线程运行。
- **共享 Pipeline 与共享 Handler：** 不同连接不共用同一个 Pipeline；满足共享条件的 Handler 实例可以被多个 Pipeline 使用，需确保线程安全。
- **写出与业务成功：** 本地写操作成功，不代表接收方业务已处理成功。应用层可靠性需要按需求设计消息 ID、ACK、重试和去重。

本章建立组件地图；线程模型、事件传播、内存管理与写缓冲等细节由对应专题展开。
