# 05｜异步 Future 与连接生命周期

## 1. ChannelFuture 表示操作结果

bind、connect、write、close 等 Channel 操作异步进行，Future 用于查询状态、等待完成或登记回调。

```java
ChannelFuture connect = bootstrap.connect("127.0.0.1", 9094);
```

调用返回时不保证连接已成功。`isDone()` 表示完成，完成可以是成功、失败或取消；`isSuccess()` 才表示成功。

## 2. sync 等待的是什么

```java
bootstrap.bind(9094).sync();
```

等待绑定操作完成；失败时传播相应失败，而不是只等一个任意结果就默认为成功。

```java
serverChannel.closeFuture().sync();
```

等待监听 Channel 关闭。它本身不发起关闭。

是否阻塞、等的是什么，取决于在哪个 Future 上调用等待方法。不要在 EventLoop 中阻塞等待需要当前 EventLoop 自己推进的操作。

## 3. addListener 登记回调

```java
connect.addListener(f -> {
    if (f.isSuccess()) {
        System.out.println("连接成功");
    } else {
        System.out.println("连接失败：" + f.cause());
    }
});
```

登记监听器无需占住主线程等结果。Future 已完成后再添加监听器，也能得到通知。

回调应短小，不要把耗时阻塞任务直接放进去。回调执行线程取决于对应 Future 的执行器，不能一概认为它在 main 中执行。

## 4. close 与 closeFuture

| 调用 | 含义 |
|---|---|
| channel.close() | 主动发起关闭，返回本次关闭操作的 Future |
| channel.closeFuture() | 获取这个 Channel 关闭时完成的通知 Future |

```java
channel.closeFuture().addListener(f -> {
    System.out.println("Channel 已关闭");
});

channel.close();
```

关闭通知可以由主动关闭、对端关闭、异常或其他关闭路径触发。closeFuture 不是某一次 close 调用专属的结果对象。

closeFuture 正常用于成功完成的关闭通知，这里的 `if (f.isSuccess())` 可省略；保留也不影响本次用法。不要用它判断之前的 connect 是否成功。

Channel 已关闭与“所有业务处理完成”“对端已保存消息”“TCP 的所有内核状态已经消失”不是同一层结论。

## 5. 关闭监听必须写在连接成功回调里吗

不必须。connect 返回后已有关联 Channel，可以先登记关闭监听：

```java
ChannelFuture connect = bootstrap.connect("127.0.0.1", 9094);

connect.addListener(f -> {
    if (!f.isSuccess()) {
        System.out.println("连接失败：" + f.cause());
        workerGroup.shutdownGracefully();
    }
});

connect.channel().closeFuture().addListener(f -> {
    workerGroup.shutdownGracefully();
});
```

放进成功回调只是组织流程的一种写法。此示例针对当前单连接客户端；共享线程组管理多连接时，要按整个客户端的生命周期决定何时关闭线程组。

## 6. 常见生命周期事件

典型成功连接会经历：

```text
channelRegistered
    ↓
channelActive
    ↓
channelRead / channelReadComplete / 其他事件
    ↓
channelInactive
    ↓
channelUnregistered
```

- registered：注册到 EventLoop。
- active：连接可以通信，具体语义也与 Channel 类型有关。
- read：收到一项入站数据；经过解码后可以是对象。
- readComplete：本轮读处理完成，不代表一条完整业务消息。
- inactive：连接不再活跃。
- unregistered：从 EventLoop 注销。
- handlerAdded/handlerRemoved：Handler 在处理链中的加入和移除，属于 Handler 自身生命周期。
- exceptionCaught：处理异常，可按策略记录并关闭。

## 7. 为什么正常关闭不等心跳超时

对端正常关闭时，本端通常能从 TCP 结束通知或后续 I/O 中及时获知，并完成对应 Channel 的关闭。

心跳/空闲超时用于处理长时间没有期望数据的情况，不要求所有连接关闭都先等到超时阈值。

服务端某个连接的 channelInactive，不等于绑定 9094 的监听 Channel 关闭。

## 8. 优雅关闭线程组

`shutdownGracefully()` 发起有序停止：在相应规则下处理已提交任务并退出线程，返回结果用于进一步等待关闭完成。

它是异步关闭，不是调用这一行后全部线程已经退出，也不保证任意业务事务、对端确认和消息持久化都已完成。重要业务收尾仍需应用明确安排。

main 返回后，如果 Netty 非守护线程仍活着，JVM 会继续运行。要正常结束进程，应关闭适当的连接和线程组。

## 参考

- [ChannelFuture API](https://netty.io/4.1/api/io/netty/channel/ChannelFuture.html)
- [Channel 的 closeFuture](https://netty.io/4.1/api/io/netty/channel/Channel.html#closeFuture--)

