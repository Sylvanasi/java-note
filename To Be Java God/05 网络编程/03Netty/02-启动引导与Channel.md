# 02｜启动引导与 Channel

## 1. Bootstrap 负责组合配置

服务端使用 `ServerBootstrap`，客户端使用 `Bootstrap`。它们描述线程组、Channel 实现、连接参数和初始化处理器，然后发起异步 bind/connect。

服务端示意：

```java
ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.group(bossGroup, workerGroup)
         .channel(NioServerSocketChannel.class)
         .childHandler(new ChannelInitializer<SocketChannel>() {
             @Override
             protected void initChannel(SocketChannel ch) {
                 ch.pipeline().addLast(new ChatHandler());
             }
         });

ChannelFuture bindFuture = bootstrap.bind(9094);
```

片段只展示启动关系；真实 Pipeline 需要先安装匹配协议的分帧器、解码器和编码器。

## 2. 两类 Channel

| 类型 | 作用 | 当前例子 |
|---|---|---|
| 监听 Channel | 绑定端口、接收新连接 | NioServerSocketChannel |
| 连接 Channel | 与一个对端收发数据 | NioSocketChannel |

```text
监听 Channel：0.0.0.0:9094
    ├─ 接受客户端 A → 连接 Channel A → Pipeline A
    └─ 接受客户端 B → 连接 Channel B → Pipeline B
```

连接 Channel 可以拥有监听 Channel 作为 parent，但不能理解成“所有字节装在一个大 Channel 里的小分区”。它们是不同网络对象，拥有各自的状态和处理链。

关闭某个连接 Channel，不代表监听端口关闭。关闭监听 Channel 后，已建立连接的生命周期也需要明确管理。

## 3. handler 与 childHandler

- `ServerBootstrap.handler(...)`：为监听 Channel 配置 Handler。
- `ServerBootstrap.childHandler(...)`：为接受到的新连接配置 Handler/初始化器。
- 客户端 `Bootstrap.handler(...)`：为客户端自己的连接配置处理链。

`childHandler` 的重点是“新接收的连接怎么处理”。它不代表把所有业务固定交给 boss 线程。

## 4. ChannelInitializer 什么时候执行

新连接初始化时，`initChannel(ch)` 为它安装 Pipeline。

```java
protected void initChannel(SocketChannel ch) {
    ch.pipeline().addLast(new ChatMessageDecoder());
}
```

这段写法下：

```text
连接 A 初始化 → Decoder A → A 的多次消息反复使用它
连接 B 初始化 → Decoder B → B 的多次消息反复使用它
```

不是每次调用 decode 都重新创建 Decoder。成员变量会跨调用保留；局部变量每次调用重新建立。

`ByteToMessageDecoder` 带有累积与解析状态，不能把同一个实例直接共享给多个连接。

## 5. future.channel() 取到什么

```java
ChannelFuture bindFuture = bootstrap.bind(9094);
Channel serverChannel = bindFuture.channel();
```

这里拿到监听 Channel。

```java
ChannelFuture connectFuture = bootstrap.connect("127.0.0.1", 9094);
Channel clientChannel = connectFuture.channel();
```

这里拿到客户端的连接 Channel。Future 返回时 Channel 通常已经创建，连接操作仍可能尚未完成。

服务端 `ChatHandler` 中的 `ctx.channel()` 则是当前客户端对应的连接 Channel。

## 6. 一个监听 Channel 对应多少客户端

一个监听 Channel 可以接收多条连接。需要监听多个端口或地址时，可以创建多个监听 Channel，不由 boss 线程数直接决定数量。

客户端程序同一份 main 可以启动多个进程。每个进程建立自己的连接，连接相同服务端端口，本地端口由操作系统分配。

## 7. 易错点

- bind 返回 Future，不等于绑定已成功。
- future.channel() 取到的是当前操作关联的 Channel，不是统一的“最大 Channel”。
- 服务端的 accept 角色不限制收发方向，连接建立后 TCP 双向通信。
- 在初始化器里 new 的 Handler 默认按这段代码的创建位置复用，不要只凭类名猜实例数量。

