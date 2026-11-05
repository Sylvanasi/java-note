# 04｜Pipeline 与事件传播

![[附件/知识总览/04-Pipeline与事件传播.svg]]

## 1. Pipeline、Handler 与 Context

| 对象 | 职责 |
|---|---|
| ChannelPipeline | 一个 Channel 的处理链，组织事件处理步骤 |
| ChannelHandler | 实际提供事件处理方法的对象 |
| ChannelHandlerContext | Handler 在这条 Pipeline 中的节点与操作入口 |

DefaultChannelPipeline 内部以 Context 双向链表组织 Handler。入站和出站沿不同方向查找匹配的节点，使用同一条链，而不是两条互不相关的处理链。

一个满足共享条件的 Handler 可以出现在不同 Pipeline 中，但每处安装都有自己的 Context。不能把 Handler 对象与某条链上的节点视为同一概念。

## 2. 入站、出站与处理器方向

以依次添加分帧器、解码器、编码器和业务 Handler 为例，网络数据按入站方向经过分帧、解码和业务处理；编码器若只处理出站操作，会被入站消息跳过。业务发起写操作时按出站方向寻找适用的处理器，由编码器将对象转成字节，再交给传输层。

| 方向 | 典型事件或操作 | 默认链表查找方向 |
|---|---|---|
| 入站 | channelActive、channelRead、channelInactive、用户事件 | 从 HEAD 一侧向 TAIL 一侧 |
| 出站 | write、flush、close 等 | 从 TAIL 一侧向 HEAD 一侧 |

ChannelDuplexHandler 可以参与两个方向。方向表示事件与操作的传播，不表示服务端只能接收、客户端只能发送。

## 3. 入站如何继续传播

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    System.out.println(msg);
    ctx.fireChannelRead(msg);
}
```

ctx.fireChannelRead 从当前 Context 后面的适用入站节点继续传播，不重新调用当前节点。普通 Handler 消费事件后不再传播，下游就不会收到这次事件；消费引用计数消息时还需遵守释放责任。

在编解码器的 decode 方法中，通常把转换结果加入 out，由父类传播，不要再 fire 同一结果造成重复。

```java
@Override
public void channelActive(ChannelHandlerContext ctx) throws Exception {
    super.channelActive(ctx);
}
```

ChannelInboundHandlerAdapter 的默认 channelActive 会继续传播。因此该方法中的 super.channelActive(ctx) 与 ctx.fireChannelActive() 通常二选一。这个结论针对默认适配器实现，不代表任意父类的 super 方法都只是转发。

## 4. 写操作起点与 Promise 透传

| 调用 | 出站查找起点 |
|---|---|
| ctx.writeAndFlush(reply) | 当前 Context 的前面，不包含当前节点 |
| ctx.channel().writeAndFlush(reply) | 从 Pipeline 尾部发起 |

因此，当业务 Handler 使用自己的 ctx 写出时，需要经过的编码器应放在它前面。放在其后面的编码器可能被跳过。排列顺序必须结合调用起点分析，不能只背“出站反向”。

在某个出站 Handler 自身的 write 方法中，继续使用 ctx.write(msg, promise) 可向前透传；改用 channel.write 可能从尾部重新经过自己，引起重复处理或递归。

```java
@Override
public void write(ChannelHandlerContext ctx,
                  Object msg,
                  ChannelPromise promise) {
    ctx.write(msg, promise);
}
```

原 Promise 将这次写操作的结果传给调用方。若自行丢弃消息，也必须按语义完成或失败该 Promise，并处理消息资源，不能让结果一直悬而未决。

## 5. 同步调用、任务提交与返回边界

Netty 找到目标 Context 后，会根据目标执行器与当前线程判断：可以在当前线程执行时直接调用；需要切换执行器时提交任务。

所以 fireChannelRead 不是固定的全同步或全异步。比如 worker 上完成分帧、解码后将业务 Handler 提交给 businessGroup，传播调用可以先返回，业务任务仍在排队或运行。跨线程日志的先后不能仅从源码行顺序推断。

```java
try {
    ctx.fireChannelRead(msg);
    System.out.println("传播调用已返回");
} finally {
    // 这里只表示当前代码块退出
}
```

finally 不是整个异步链的完成通知。即使没有线程切换，分帧器也可能只积累了一段数据并返回，还没有输出完整业务消息。因此“调用返回”“完整消息已产生”“业务已完成”是不同判断。

## 6. 日志粒度与读指针

分帧前的 ByteBuf 日志反映交给该处理器的一次字节事件，不保证对应一个 TCP 报文或一条业务消息。

- 一条业务消息可能分多次到达，形成多次字节日志。
- 一次字节事件可能包含多条业务消息，经分帧后分别交给下游。
- 分帧后适合观察帧边界；解码后适合观察业务字段，两者不能替代底层字节日志。

打印字节时使用不移动 readerIndex 的方式，例如 ByteBufUtil.hexDump。不要为了打印调用 readBytes 等消费方法，导致下游读不到数据。日志本身也要避免长时间阻塞 EventLoop。
