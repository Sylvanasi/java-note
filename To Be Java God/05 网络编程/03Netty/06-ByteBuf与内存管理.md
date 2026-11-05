# 06｜ByteBuf 与内存管理

![[附件/知识总览/06-ByteBuf与内存管理.svg]]

## 1. 缓冲区与独立读写索引

ByteBuf 是 Netty 的字节缓冲抽象，不是 byte[] 或 Java NIO ByteBuffer。它提供独立的 readerIndex 与 writerIndex，普通操作下满足：

`0 ≤ readerIndex ≤ writerIndex ≤ capacity`

| 区域 | 范围 | 含义 |
|---|---|---|
| 已读区域 | 0 到 readerIndex 之前 | 已通过相对读操作消费的字节 |
| 可读区域 | readerIndex 到 writerIndex 之前 | 当前尚未消费的数据 |
| 可写区域 | writerIndex 到 capacity 之前 | 当前容量内的可写空间 |

readableBytes 等于 writerIndex - readerIndex。ByteBuf 写入后通常可直接从读索引读取，不照搬 ByteBuffer 的 flip 模式。clear 重置索引，不保证抹除底层数据；读索引前进也不等于释放内存。

## 2. 读取、查看与 EOF 的区别

| 方法 | 行为 | 移动 readerIndex |
|---|---|---|
| readByte() | 读取一个字节 | 是，前进 1 |
| readInt() | 读取四字节整数 | 是，前进 4 |
| readBytes(byte[]) | 复制到已有数组 | 是 |
| skipBytes(n) | 跳过 n 字节 | 是 |
| getByte(index) / getInt(index) | 绝对索引访问 | 否 |
| toString(UTF_8) | 将当前可读区域转换为字符串 | 否 |
| ByteBufUtil.hexDump(buf) | 当前可读区域的十六进制表示 | 否 |

生成字符串不代表消费了原字节。get 方法的索引是绝对位置，解析下一帧时应以当前 readerIndex 为基准，不能始终假设帧从 0 开始。

InputStream.read 返回 int，可用 -1 表示 EOF；ByteBuf.readByte 返回 byte，其中 -1 可能是真实数据 0xFF。没有足够可读字节时相对读操作抛出越界异常，应先检查 isReadable 或 readableBytes。缓冲区当前有数据，不代表协议消息完整；当前无数据，也不代表连接已关闭。

## 3. 复制与共享视图

```java
byte[] bytes = new byte[in.readableBytes()];
in.readBytes(bytes);
```

该重载填充已有数组并移动读索引。普通数组由 Java 对象生命周期管理。

```java
ByteBuf another = in.readBytes(n);
try {
    // 消费新缓冲区
} finally {
    another.release();
}
```

readBytes(n) 返回新分配的 ByteBuf，不能忽略其释放责任。它与只创建共享视图的 readSlice(n) 不同。

| 操作 | 内容是否复制 | 索引与引用 |
|---|---|---|
| copy() / readBytes(n) | 复制 | 新缓冲区，需要自己的释放管理；readBytes 会移动源读索引 |
| slice() / duplicate() | 共享底层内容 | 视图有独立索引，通常不增加共享资源的引用计数 |
| readSlice(n) | 共享底层内容 | 移动源读索引，通常不增加引用计数 |
| retainedSlice / retainedDuplicate / readRetainedSlice | 共享底层内容 | 为共享资源保留额外引用，需配对释放 |

共享视图不是独立快照。需要让视图独立于原持有者继续使用时，应保留对应引用；原引用失效后不能继续依靠普通 slice 访问资源。

## 4. retain 与 release 管理资源生命周期

普通新分配 ByteBuf 的初始引用计数通常为 1。retain 增加一份引用，release 减少一份；降到 0 时回收底层资源或归还池。释放后继续访问可能抛出 IllegalReferenceCountException。

```java
ByteBuf other = buf.retain();
System.out.println(other == buf); // true
```

retain 返回当前对象，不复制内容。普通赋值 other = buf 不增加引用计数。引用计数不等于 Java 变量数量，也不等于连接数量、Socket.close 或 TCP FIN。

retain 只延长生命周期，不自动提供线程安全、不冻结索引，也不保存内容快照。池化与直接内存是不同维度：可以讨论是否复用内存，也可以讨论内存在堆内还是堆外，不能将两者等同。

## 5. Handler 的释放责任

| 场景 | 责任 |
|---|---|
| 普通入站 Handler 观察后原样传播 | 交接原引用，通常由下游最终消费者释放 |
| 自己消费并停止传播 | 释放自己消费的输入 |
| 额外 retain 后继续持有 | 为额外引用安排配对 release，包括失败路径 |
| SimpleChannelInboundHandler 匹配并处理输入 | 默认自动释放；继续传递同一引用计数消息时需保留相应引用 |

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    try {
        // 消费 msg，不再向下游传播
    } finally {
        ReferenceCountUtil.release(msg);
    }
}
```

不要把此消费模板套到原样传播路径，造成下游使用已释放数据。String 或不持有引用计数资源的普通消息对象无需按 ByteBuf 方式 retain/release；但 HttpContent、WebSocketFrame 等持有 ByteBuf 的包装对象仍需管理引用。

已交给 Netty 出站流程的引用计数消息通常由该流程释放；应用若自己丢弃或未成功交接，也要处理资源与操作结果。

## 6. 日志观察与异步交接

最简单的日志方式是在交给下游之前使用不移动索引的 hexDump；若需保存独立观察结果，可先生成字符串或复制所需数据，避免依赖下游之后的可变内容。

下面是同步传播场景下保留额外引用的局部示例：

```java
int start = buf.readerIndex();
int length = buf.readableBytes();
buf.retain();
try {
    ctx.fireChannelRead(buf);
    System.out.println(ByteBufUtil.hexDump(buf, start, length));
} finally {
    buf.release();
}
```

预先记录范围可以避免下游移动读索引影响查看范围，但不保证内容未被下游修改或压缩。若跨执行器并发访问同一缓冲区，此示例本身不提供线程安全；优先在交接前生成日志快照。

在下游同步消费并释放原引用的特定路径中，计数可以依次为 1、retain 后 2、下游释放后 1、额外引用释放后 0。如果分帧器还持有未完成数据，我方释放后仍为 1 也可能完全正常；不能凭一次观测的计数判断 finally 未执行或一定泄漏。

如果解码得到独立 String 或普通业务对象，后续异步业务不再依赖原 ByteBuf。确实需要将 ByteBuf 额外交给异步任务时，必须明确引用交接、任务拒绝和异常路径释放、索引及内容的并发访问约束。retain 只是其中一部分。
