# 02｜启动引导与 Channel

![[附件/知识总览/02-启动引导与Channel.png]]

## 1. Bootstrap 组合配置并发起启动

服务端使用 `ServerBootstrap`；客户端通常使用 `Bootstrap`。引导对象组合线程组、Channel 实现、参数和处理器，再发起异步 `bind` 或 `connect`。

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

这是说明启动关系的局部片段；线程组创建、协议编解码和优雅关闭按实际需要配置，不是完整可运行程序。

**调用返回不代表绑定成功。** 应通过 Future 的结果或监听器确认；不要在 EventLoop 内用阻塞等待代替异步结果处理。

## 2. 监听 Channel 与连接 Channel

| 对象 | 典型 NIO 实现 | 职责 |
|---|---|---|
| 监听 Channel | `NioServerSocketChannel` | 绑定地址并接受连接 |
| 服务端连接 Channel | `NioSocketChannel` | 与一个客户端收发数据 |
| 客户端连接 Channel | `NioSocketChannel` | 与服务端收发数据 |

一个监听 Channel 可以接受多条连接，每条连接拥有自己的状态和 Pipeline。服务端接受的连接可通过 `parent()` 关联监听 Channel，但它不是监听 Channel 内的一块字节分区。

关闭某条连接，不等于关闭监听端口。关闭监听 Channel 也不能代替已接受连接的统一清理；停止接入与关闭存量连接应分别管理。

监听地址数量不由 boss 线程数直接决定。需要多个监听地址时，可创建并管理多个监听 Channel。

## 3. handler 与 childHandler 的配置对象

| 配置位置 | 配置对象 |
|---|---|
| `ServerBootstrap.handler(...)` | 监听 Channel |
| `ServerBootstrap.childHandler(...)` | 服务端接受到的连接 Channel |
| 客户端 `Bootstrap.handler(...)` | 客户端自己的 Channel |
| `ServerBootstrap.option(...)` | 监听 Channel 的参数 |
| `ServerBootstrap.childOption(...)` | 接受连接的参数 |

`childHandler` 表示“接受连接后怎样初始化和处理”，不表示将业务固定交给 boss。两参数 `group(bossGroup, workerGroup)` 分别配置监听与接受连接所用的事件循环组。

## 4. ChannelInitializer 与 Handler 实例生命周期

`ChannelInitializer` 在 Channel 注册后的初始化阶段配置处理链，并在完成初始化后移除自身。它负责安装 Handler，不负责每条消息的解码。

```java
protected void initChannel(SocketChannel ch) {
    ch.pipeline().addLast(new ChatMessageDecoder());
}
```

在这个创建位置上，每次初始化连接都会创建一个 Decoder：连接 A 使用实例 A，连接 B 使用实例 B；同一条连接收到多条消息时继续使用自己的实例，不是每次 `decode()` 都重新创建。

- 实例成员字段随实例保留；局部变量在方法调用中创建。
- `ByteToMessageDecoder` 具有累积与解析状态，不能把同一个实例加入多个连接的 Pipeline。
- `ChannelInitializer` 本身允许复用；其实现仍需保证安全，不能把各连接的可变状态随意放在共享字段中。
- 是否共享 Handler 实例由创建位置与共享约束决定，不能只凭类名判断。

## 5. future.channel() 与 ctx.channel() 的区别

| 获取位置 | 对应 Channel |
|---|---|
| `serverBootstrap.bind(...).channel()` | 本次绑定关联的监听 Channel |
| `clientBootstrap.connect(...).channel()` | 本次连接关联的客户端 Channel |
| 服务端连接业务 Handler 的 `ctx.channel()` | 当前客户端对应的服务端连接 Channel |

```java
ChannelFuture bindFuture = serverBootstrap.bind(9094);
Channel serverChannel = bindFuture.channel();

ChannelFuture connectFuture = clientBootstrap.connect("127.0.0.1", 9094);
Channel clientChannel = connectFuture.channel();
```

以上变量表示不同的引导对象。正常创建 Channel 后，可以在异步操作完成前取得关联 Channel；取得对象本身并不能证明绑定或连接成功。

## 6. 多客户端与易错边界

同一份客户端 main 可以启动多个进程，各进程建立自己的连接。未显式绑定本地端口时，通常由操作系统分配临时端口；多个连接可以连接同一个服务端端口。

TCP 连接建立后是双向通信。服务端负责接受连接，不代表只能接收；客户端发起连接，也不代表只能发送。

复习时先确认三个问题：操作的是监听还是连接 Channel？Handler 装在哪个对象上？Future 表示的是哪一次异步操作？这比将 Channel 理解成统一的“最大容器”更准确。
