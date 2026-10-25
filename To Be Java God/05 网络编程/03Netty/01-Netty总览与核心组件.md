# 01｜Netty 总览与核心组件

## 1. 一句话理解

Netty 是网络应用框架。在当前使用的 NIO 传输实现中，它把连接注册、事件循环、缓冲区、异步写出和处理链组织起来，让业务代码围绕事件和消息编写。

Netty 也支持其他传输实现，因此不能把整个框架严格等同于 Java NIO。当前项目的 `NioEventLoopGroup`、`NioServerSocketChannel`、`NioSocketChannel` 使用 NIO。

## 2. 与 BIO、NIO 的联系

| 模型 | 等待与处理方式 | 应用主要负责什么 |
|---|---|---|
| BIO | 处理线程可能阻塞在某条连接的读写上 | 连接线程、流、协议、资源关闭 |
| 原生 NIO | 少量线程通过 Selector 等待多条连接的就绪事件 | 注册与遍历、缓冲、分帧、部分写入续写 |
| Netty NIO 传输 | EventLoop 驱动多连接，Pipeline 组织处理过程 | 选择协议、编解码、业务与应用级资源策略 |

Netty 代管了大量底层工作，但消息格式、业务确认、异常处理和连接存活规则仍要由应用定义。

## 3. 核心组件

| 组件 | 职责 |
|---|---|
| Bootstrap / ServerBootstrap | 组合客户端/服务端配置并发起启动 |
| EventLoopGroup | 管理一组事件循环 |
| EventLoop | 处理 Channel 的 I/O 事件、普通任务、定时任务 |
| Channel | 一条连接或监听端口的网络抽象 |
| ChannelFuture | 表示异步 Channel 操作的状态与结果 |
| ChannelPipeline | 按顺序组织 Handler 的处理链 |
| ChannelHandler | 处理连接事件、数据与出站操作 |
| ChannelHandlerContext | Handler 在当前 Pipeline 中的位置与上下文 |
| ByteBuf | 存放和操作网络字节，支持引用计数等机制 |

此外，`ChannelInitializer` 用于为新建 Channel 安装处理链；`ChannelPromise` 是可由实现方完成的结果承诺，出站 Handler 透传时应继续携带它。

## 4. 原生 NIO 工作在 Netty 中的对应位置

| 原生 NIO 工作 | Netty 中的组织方式 |
|---|---|
| 打开 ServerSocketChannel、绑定端口 | ServerBootstrap + NioServerSocketChannel |
| 监听 OP_ACCEPT 并接收连接 | boss EventLoop 处理监听 Channel |
| 将连接注册到 Selector | Netty 将连接 Channel 注册给 worker EventLoop |
| select、处理选中的事件、执行任务 | EventLoop 的循环 |
| 每条连接保存自己的缓冲与状态 | Channel、Handler、附件属性等 |
| 读取字节并拼接完整消息 | ByteBuf + 分帧器 + 解码器 |
| 写出没有完成时关注 OP_WRITE 续写 | Netty 传输层维护待发送数据与续写 |

## 5. 服务端主线

```text
ServerBootstrap 组合配置
    ↓ bind
监听 Channel 由 boss 管理
    ↓ 接受连接
客户端连接 Channel 注册到 worker 的 EventLoop
    ↓ 安装并执行 Pipeline
分帧 → 解码 → 业务处理
    ↓ 生成回复
出站处理 → 编码 → 网络写出
```

bossGroup、workerGroup 是职责命名，不是两种不同的 EventLoopGroup 接口类型。线程数由配置决定。

## 6. 三个不能混淆的“组/链”

- EventLoopGroup：事件线程的组织。
- ChannelGroup：连接的集合，便于批量发送和关闭。
- ChannelPipeline：某个 Channel 的处理器链。

它们分别组织线程、连接、处理步骤。

## 7. 易错点

- 一个 EventLoop 能管理多条连接，并不等于“一个连接永久占一个线程”。
- 一条连接的 Pipeline 可以在某些 Handler 处切换到业务线程执行。
- Pipeline 是每个 Channel 自己的；多个 Channel 不会天然共用一个 Pipeline。
- Netty 的异步写入不能替代应用层业务 ACK。



