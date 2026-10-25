# 06｜ByteBuf 与内存管理

## 1. ByteBuf 是什么

ByteBuf 是 Netty 的字节缓冲区，不是 Java NIO 的 ByteBuffer，也不是 byte[]。它们都能承载字节，但操作方式和生命周期机制不同。

ByteBuf 使用独立的读指针 readerIndex 和写指针 writerIndex：

```text
[已经读过的区域][当前可读区域][尚未写入的区域]
              ↑           ↑
          readerIndex  writerIndex
```

`readableBytes()` 是 writerIndex - readerIndex。写入后通常可直接按读指针读取，不照搬 ByteBuffer 的 flip 模式。

## 2. 这些方法是否消费字节

| 方法 | 作用 | 移动读指针 |
|---|---|---|
| readByte() | 取一个字节 | 是，移动 1 |
| readInt() | 取 4 字节整数 | 是，移动 4 |
| readBytes(byte[]) | 读取对应字节并填入已有数组 | 是 |
| skipBytes(n) | 跳过 n 字节 | 是 |
| getByte(index) / getInt(index) | 按绝对位置查看 | 否 |
| toString(UTF_8) | 把当前可读区域转为 String | 否 |
| ByteBufUtil.hexDump(buf) | 查看可读区域的十六进制 | 否 |

重要区别：生成字符串不代表原字节已被消费。

## 3. readByte 没有 EOF 哨兵

`InputStream.read()` 返回 int，能用 -1 表示 EOF。`ByteBuf.readByte()` 返回 byte：

- -1 可以是实际数据 0xFF。
- 没有可读数据时继续 readByte，会抛出越界异常。
- 用 isReadable/readableBytes 判断缓冲区是否还有可读数据；消息是否完整则按协议判断。

## 4. readBytes 的两个重载

```java
byte[] bytes = new byte[in.readableBytes()];
in.readBytes(bytes);
```

把字节读进已有数组，读指针随之移动。

```java
ByteBuf another = in.readBytes(n);
```

这个重载返回新 ByteBuf，需要明确其释放责任。忽略返回值不等于自动释放，容易造成泄漏。

## 5. 引用计数解决什么

同一块缓冲内存可能被多处代码使用。引用计数用于判断什么时候可以回收或归还内存池。

```text
retain()  → 引用计数 +1
release() → 引用计数 -1
降到 0    → 释放或归还资源
```

ByteBuf 初始引用计数通常为 1。读取已经释放的数据可能触发 IllegalReferenceCountException。

这与 Socket.close、TCP FIN、应用连接数没有直接等价关系。

## 6. retain 返回什么

```java
ByteBuf other = buf.retain();
System.out.println(other == buf); // true
```

返回当前 ByteBuf 自身，不复制内容。普通赋值 `other = buf` 不会自动增加引用计数。

retain 延长资源生命周期，不自动提供线程安全，不冻结读写指针，也不保存一份独立快照。

## 7. 谁负责释放

普通 ChannelInboundHandlerAdapter 中：

- 仅观察并直接向下游转发：把原来那份的释放责任交给下游。
- 自己消费并终止传播：负责释放输入。
- 自己额外 retain 一份：自己负责配对 release 这一份。

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    try {
        // 在这里消费 msg，不再向下传递
    } finally {
        ReferenceCountUtil.release(msg);
    }
}
```

SimpleChannelInboundHandler 默认会自动释放它匹配并处理的输入。若输入是 ByteBuf，又要继续传递同一份数据，需要保留相应引用，不能在自动释放后让下游使用失效缓冲。

普通 String 和本实验的 ChatMessage 没有 ByteBuf 那种引用计数，不需要对它们手动 retain/release。

## 8. 本次日志实验的真实含义

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

为了在向下游传播后继续观察原数据，日志 Handler 额外保留一份。预先记录范围，是因为下游可能移动同一个缓冲对象的读指针。

完整消息且下游释放原引用时，典型计数为：

```text
收到 1 → 我方 retain 得到 2 → 下游释放剩 1 → 我方 release 到 0
```

如果分帧器还保留着未完成数据，传播调用返回时可能仍是 2；我方 release 后剩 1，由分帧器继续持有。这不表示 finally 没执行，也不表示一定泄漏。

不能把某一次计数序列写成所有 Pipeline 都必须遵循的固定规则。

## 9. ByteBuf 与异步业务的关系

如果解码器已经把字节变成独立 String 或普通消息对象，后面的业务线程不再依赖原始 ByteBuf，可以按字节处理链的责任释放它。

如果确实要把同一块 ByteBuf 额外交给异步任务，必须明确引用交接、失败路径释放以及并发访问方式；retain 只解决生命周期中的一部分问题。

## 参考

- [Netty 引用计数说明](https://netty.io/wiki/reference-counted-objects.html)
- [ByteBuf API](https://netty.io/4.1/api/io/netty/buffer/ByteBuf.html)
