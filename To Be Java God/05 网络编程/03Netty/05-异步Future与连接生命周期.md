# 05｜异步 Future 与连接生命周期

![[附件/知识总览/05-异步Future与连接生命周期.svg]]

## 1. ChannelFuture 表示一次操作的结果

bind、connect、write、close 等 Channel 操作异步进行，返回的 ChannelFuture 用于查询状态、等待完成或登记完成监听器。

```java
ChannelFuture connectFuture = bootstrap.connect("127.0.0.1", 9094);
```

调用返回时不保证连接已经建立。Future 是结果凭证，不是执行操作的线程，也不是这个连接所有后续业务的完成凭证。

| 状态方法 | 含义 |
|---|---|
| isDone() | 操作完成，可能成功、失败或取消 |
| isSuccess() | 操作成功完成 |
| isCancelled() | 操作被取消 |
| cause() | 失败原因；未完成或成功时通常为 null，不能只凭 null 判断成功 |

## 2. sync、await 与 addListener

| 方式 | 调用线程 | 结果处理 |
|---|---|---|
| sync() | 等待目标 Future 完成，可能阻塞 | 操作失败会重新抛出相应异常，可被中断 |
| await() | 等待目标 Future 完成，可能阻塞 | 等待返回后仍需检查成功、失败或取消 |
| addListener(...) | 登记完成通知，不阻塞等待操作完成 | 在监听器中处理结果；回调可能很快执行 |

```java
bootstrap.bind(9094).sync();
serverChannel.closeFuture().sync();
```

前者等待绑定完成，后者等待监听 Channel 的关闭通知。后者本身不发起关闭。

```java
connectFuture.addListener(f -> {
    if (f.isSuccess()) {
        System.out.println("连接成功");
    } else if (f.isCancelled()) {
        System.out.println("连接已取消");
    } else {
        System.out.println("连接失败：" + f.cause());
    }
});
```

Future 已完成后再添加监听器也能获得通知。不要假定回调一定在 main 执行，或一定在登记方法返回以后执行；执行方式与关联执行器、调用线程有关。

监听器应短小。不要在 EventLoop 中阻塞等待依赖它推进的操作，也不要在完成回调中直接执行耗时 JDBC 或同步远程调用。

## 3. close 发起关闭，closeFuture 观察关闭

| 调用 | 含义 |
|---|---|
| channel.close() | 发起一次关闭请求，返回该操作的结果 |
| channel.closeFuture() | 获取该 Channel 关闭时完成的通知 Future |

```java
channel.closeFuture().addListener(f -> {
    System.out.println("Channel 已关闭");
});
channel.close();
```

关闭通知不专属于某次 close 调用。主动关闭、远端结束、I/O 错误及其他关闭路径都可能最终使 Channel 关闭。默认实现的 closeFuture 用于关闭完成通知，通常无需通过 isSuccess 再分支判断；它不能证明之前 connect 成功。

关闭监听可以在 connect 返回、取得关联 Channel 后登记，不必等连接成功回调。若连接失败，单连接客户端仍需处理线程组等资源；共享线程组管理多个连接时，不能因为一个连接关闭就停掉整个组。

Channel 关闭不代表所有业务已处理、对端已保存消息，或 TCP 的所有内核状态已消失。这些需要分别确认。

## 4. Channel 与 Handler 的生命周期事件

典型成功建立且最终关闭的连接，按阶段理解：

| 阶段 | 事件 | 含义 |
|---|---|---|
| 注册 | channelRegistered | 注册到 EventLoop |
| 活跃 | channelActive | 对 TCP 连接通常表示连接可通信；具体含义取决于 Channel 类型 |
| 接收 | channelRead | 收到一项入站数据，经过解码后可为业务对象 |
| 读取轮次结束 | channelReadComplete | 当前读取轮次结束，不保证完整业务消息已经产生 |
| 失活 | channelInactive | Channel 不再活跃 |
| 注销 | channelUnregistered | 从 EventLoop 注销 |

这是典型生命周期地图，不保证连接失败、重复注册等路径都经过相同事件序列。

handlerAdded / handlerRemoved 表示 Handler 被加入或移出 Pipeline，是处理器自身生命周期。exceptionCaught 表示异常传播，应结合异常类型与业务策略记录、恢复或关闭，不把所有异常都等同于连接已经关闭。

## 5. 正常关闭与心跳超时的区别

对端正常结束发送时，本端可能从读到 EOF 等 I/O 信号及时获知。默认常见 TCP 使用方式下会进一步关闭 Channel，并触发相关事件，不必先等待心跳阈值。

启用半关闭时，收到对端输入结束不一定立即关闭整个 Channel，应按半关闭事件与策略处理。网络黑洞、断网等未及时带来明确 I/O 信号的情况，才需要通过超时等机制识别长期无响应。

空闲检测只说明一段时间内没有期望的读写事件，不直接证明对端死亡。服务端某条连接的 channelInactive，也不意味着监听端口的 Channel 已关闭。

## 6. 线程组优雅关闭与资源所有权

shutdownGracefully 发起异步有序停止，返回的 Future 可用于观察线程组终止。调用返回不表示全部线程已退出；具体等待与停止规则由 quietPeriod、timeout 等配置和实现决定。

应用应先明确停止接入、停止产生新任务、处理必要业务收尾、关闭连接、取消周期任务及关闭线程组的顺序，并观察相应结果。共享资源由其所有者统一结束，不在每条连接的关闭回调中随意关闭。

```java
// 单连接客户端独占 workerGroup 时的局部示例
channel.closeFuture().addListener(f -> {
    workerGroup.shutdownGracefully();
});
```

此片段只展示已取得 Channel 后的资源联动；启动失败路径同样需要清理。不要将其直接照搬到共享线程组的多连接客户端。

优雅关闭不自动保证业务事务完成、对端 ACK 或消息持久化。默认非守护线程仍存活时，main 返回后 JVM 可继续运行；修改为守护线程不能代替资源清理。
