# 04｜Pipeline 与事件传播

## 1. Pipeline、Handler、Context

- Pipeline：当前 Channel 的处理链。
- Handler：实际处理事件的方法所在对象。
- Context：这个 Handler 在这条 Pipeline 中的节点与操作入口。

Pipeline 内部用 HandlerContext 双向链表组织 Handler。入站和出站沿不同方向查找匹配处理器，使用同一条链。

双向结构便于从当前 Context 向前或向后传播，同时维护添加、删除等操作。并不是必须使用两条独立的单向链。

## 2. 入站与出站

```text
链表顺序：HEAD → 分帧器 → 解码器 → 编码器 → 业务 Handler → TAIL

入站：网络 → 分帧器 → 解码器 → 业务 Handler
出站：业务 Handler 发起写出 → 编码器 → 网络
```

传播时会跳过不处理当前方向或事件的方法。

- 入站例子：连接激活、读到数据、连接失活、用户事件。
- 出站例子：write、flush、close 等操作。
- 一个 ChannelDuplexHandler 能同时参与两个方向。

## 3. 入站如何继续传播

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    System.out.println(msg);
    ctx.fireChannelRead(msg);
}
```

普通入站 Handler 如果既不传播也不输出转换结果，后续 Handler 就收不到这条消息。

在编解码器的 decode 方法中，通常把结果放入 out，由父类负责传播，不要再重复 fire 同一结果。

## 4. super 与 fire

```java
ctx.fireChannelActive();
```

表示从当前节点继续向后传播激活事件。适配器默认的 `super.channelActive(ctx)` 也会继续传播，所以通常二选一，不要两种都调用而造成重复传播。

它与过滤器链“当前节点处理后交给下一节点”的思路相似，但 Netty 还处理事件方向、执行器切换与引用计数等问题。

## 5. ctx.write 与 channel.write

```java
ctx.writeAndFlush(reply);
ctx.channel().writeAndFlush(reply);
```

- Context 发起写出：从当前 Context 向前寻找出站处理器。
- Channel 发起写出：通常从 Pipeline 尾部发起，反向经过匹配的出站处理器。

因此，编码器应位于业务 Handler 的前面，才能被该业务 Context 的写出路径经过。

出站 Handler 继续传递时要保留 promise：

```java
@Override
public void write(ChannelHandlerContext ctx,
                  Object msg,
                  ChannelPromise promise) {
    ctx.write(msg, promise);
}
```

## 6. fireChannelRead 是同步还是异步

每次找到下一个目标 Context 后，Netty 大致做这样的判断：

```text
当前线程能否直接执行目标 Context？
    ├─ 能 → 直接调用目标方法
    └─ 不能 → 向目标执行器提交任务
```

所以它不是固定的“全同步”或“全异步”。

当前实验的典型链路：

```text
worker：网络日志 → 分帧 → 字节解码成对象 → 提交业务任务
                                            ↓
businessGroup：                         ChatHandler
```

worker 的 fireChannelRead 可以在业务任务完成前返回。业务日志和外层“传播调用返回”日志没有固定的跨线程先后顺序。

## 7. 调用返回不等于全链路完成

```java
try {
    ctx.fireChannelRead(msg);
    System.out.println("传播调用已返回");
} finally {
    // 在本 try 结束或异常退出时执行
}
```

- 没有切线程的同步方法，返回前已经执行到对应返回点。
- 已提交到其他执行器的任务，此时可能仍在排队或运行。
- 分帧器即使同步返回，也可能保存着不完整的数据等待以后处理。
- finally 管理当前代码块的退出，不是整个异步处理链的完成回调。

## 8. 怎样理解日志的粒度

分帧前的 ByteBuf 日志反映一次入站读取事件，不等于一个 TCP 报文，也不保证是一条完整业务消息。

一条业务消息可能对应多次原始字节日志；一次原始字节事件也可能产生多条分帧后的消息。

读取日志时使用不移动读指针的方法，例如 ByteBufUtil.hexDump；不要为了打印而无意消耗下游需要的数据。

