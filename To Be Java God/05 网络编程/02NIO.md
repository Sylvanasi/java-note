# NIO 核心知识

> 一句话理解：NIO 用少量线程等待一批连接上的“可接受、可读、可写”事件，使线程不必为每条空闲连接单独阻塞。

![[附件/知识总览/02NIO.svg]]

## 1. NIO 真正解决的问题

BIO 中，一个线程常常阻塞在某个连接上：

典型 BIO 设计中，线程 A、B、C 分别等待各自连接的 read；大量空闲连接可能占用大量等待线程。

如果连接很多但多数时间没有数据，大量线程都在等待。

NIO 把等待方式改为：

事件线程通过 Selector 等待一批已登记连接的就绪状态，处理后再进入下一轮选择，无需为每条空闲连接驻留线程。

NIO 的优势不是“所有操作都完全不阻塞”，而是应用线程不必为每条空闲连接各驻留一个阻塞读取线程。

## 2. 核心对象与就绪事件

| 对象 | 作用 | 可以把它理解成 |
|---|---|---|
| `Channel` | 网络或文件等 I/O 通道，具体读写能力由类型决定 | 一条连接或一个设备 |
| `Buffer` | 暂存待写或刚读出的字节 | 程序搬运数据的容器 |
| `Selector` | 等待并报告多个 Channel 的就绪事件 | 统一的事件登记与通知中心 |
| `SelectionKey` | Channel 与 Selector 之间的登记记录 | “这条 Channel 关注哪些事件” |

一个 Channel 注册到 Selector 后会得到 `SelectionKey`：

一个 Channel 在一个 Selector 上的注册关系对应一个 SelectionKey，保存关注集合、就绪集合及可附加状态。

常见事件：

```java
SelectionKey.OP_ACCEPT   // 有连接可以接受
SelectionKey.OP_CONNECT  // 可尝试完成连接，仍需 finishConnect 检查结果
SelectionKey.OP_READ     // 有数据可读，或连接已结束
SelectionKey.OP_WRITE    // 发送缓冲区有空间，可以继续写
```

## 3. 服务端启动流程

打开监听 Channel、绑定地址、设为非阻塞，创建 Selector 并注册 OP_ACCEPT；循环选择、遍历就绪 Key、处理事件，再移除已处理项。

可以写成：

```java
ServerSocketChannel server = ServerSocketChannel.open();
server.bind(new InetSocketAddress(9092));
server.configureBlocking(false);

Selector selector = Selector.open();
server.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    selector.select();

    Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
    while (iterator.hasNext()) {
        SelectionKey key = iterator.next();

        iterator.remove();
        if (!key.isValid()) {
            continue;
        }

        if (key.isAcceptable()) {
            accept(server, selector);
        } else {
            if (key.isValid() && key.isReadable()) {
                read(key);
            }
            if (key.isValid() && key.isWritable()) {
                write(key);
            }
        }
    }
}
```

上例是局部事件分派示意，accept/read/write 为应用方法，异常隔离、退出与资源管理另需实现。读写可能同时就绪，故不能总将二者写成互斥分支；读取后还要检查 Key 是否仍有效。

selectedKeys 是保留就绪 Key 的集合，不是每轮自动清空的新列表。遍历时移除已经处理的 key，避免下一轮重复处理旧事件。

`Selector.select()` 在没有就绪事件时会等待；有事件、被唤醒或到达超时时间后返回。select 返回本次就绪操作集合被更新的 Key 数量，不是业务消息数，也不一定等于 selectedKeys 当前大小。无参 select 没有超时参数；select(timeout) 才涉及指定超时，且选择操作也可能因其他原因返回。

从集合中移除一个 `SelectionKey`，只表示“本轮事件已经处理过”；要让 Channel 不再被 Selector 关注，需要调用 `key.cancel()` 或关闭 Channel。

## 4. 接受连接：OP_ACCEPT

`OP_ACCEPT` 表示监听信道有新连接可以接受：

```java
if (key.isAcceptable()) {
    ServerSocketChannel server =
            (ServerSocketChannel) key.channel();

    SocketChannel client = server.accept();
    if (client == null) {
        continue;
    }

    client.configureBlocking(false);
    client.register(selector, SelectionKey.OP_READ);
}
```

要区分两个 Channel：

- `ServerSocketChannel`：只在服务端存在，用于监听和接受连接。
- `SocketChannel`：每个客户端连接一个，用于双向读写。

Selector 只报告“可以 accept”，不会代替程序调用 `accept()`。

## 5. 读取：OP_READ

`OP_READ` 表示 Channel 可以尝试读取，但不代表一定有一条完整消息。

非阻塞 `SocketChannel.read(buffer)` 的返回语义：

| 返回值 | 含义 |
|---|---|
| `> 0` | 本次读到了若干字节 |
| `0` | 本次没有读入字节；可能暂时无数据，也可能 Buffer 无剩余空间 |
| `-1` | 已达输入结束；常见为对端结束发送，也可能是本地输入关闭 |

```java
int n = client.read(buffer);

if (n == -1) {
    key.cancel();
    client.close();
} else if (n > 0) {
    // 处理本次读到的字节
}
```

`read()` 返回 `-1` 只表示对端不再发送新数据，本方在连接完全关闭前仍可以继续写出响应。

### 不能假设一次 read() 得到完整消息

TCP 是字节流：

| 次数 | 可能读到的字节片段 |
|---|---|
| 第一次 | hel |
| 第二次 | lo、换行、wor |
| 第三次 | ld、换行 |

应用必须自己累积：

将旧累积与新字节合并，按换行或长度字段提取完整消息，保留未完成末尾。

一个常见结构是给每个 Channel 保存独立的累积缓冲区：

每个连接的 ClientState 持有独立 pending 缓冲区，保存尚未完成消息的字节，供下次继续追加。

每条连接都有自己独立的半包状态，不能由一个线程共用一个待处理数组。

可以把连接状态挂到 `SelectionKey` 上：

```java
key.attach(new ClientState());

ClientState state = (ClientState) key.attachment();
```

`ClientState` 通常包含该连接自己的累积缓冲区、待写队列、协议解析进度和关闭状态。

### ByteBuffer 的 capacity、position、limit

| 参数 | 含义 |
|---|---|
| `capacity` | Buffer 的总容量 |
| `position` | 下一次读取或写入的位置 |
| `limit` | 当前允许读写的边界 |

### ByteBuffer 的 flip() 与 compact()

写模式与读模式的 position/limit 不同：

```java
// Buffer 初始为写入状态；有 compact 保留的数据时不要再次 clear
channel.read(buffer);

// 切换为读取 Buffer 中的数据
buffer.flip();

while (buffer.hasRemaining()) {
    // 提取完整消息；遇到半帧时保留其起点并退出
}

// 丢弃已处理数据，保留未处理数据，准备下次写入
buffer.compact();
```

解析半帧前需记住起点，数据不足时恢复 position，才能让 compact 保留整段未完成内容。若缓冲区已满却无完整消息，还需扩容或按协议上限拒绝，不能无限 read 0 空转。

处理完整消息后，如果末尾还有半条消息，不能直接 `clear()` 覆盖，而应使用 `compact()` 保留剩余字节。

`clear()` 只是重置 position 和 limit，让 Buffer 进入可写入状态，不会真正把底层内存清零。

### 本次对象传输实验：EOF 作为请求结束

NioServer 源码注释与输入处理采用“一条连接发送一个对象、EOF结束请求”的简化教学协议：

该教学协议中，客户端序列化一个对象并发送，调用 shutdownOutput 结束发送方向；服务端收到 EOF 后反序列化，写完 UTF-8 响应再关闭连接。

这里把 EOF 当作请求结束标记。它适合一次请求一条连接，但不适合在同一条长连接中反复发送对象。

若要在长连接中连续传输对象，需要改用长度字段、固定长度等明确的应用层分帧规则。

## 6. 写入：OP_WRITE 不是“我想写就注册”

主动发送数据时，直接调用：

```java
int written = client.write(byteBuffer);
```

返回值表示本次实际写入了多少字节，可能小于剩余数据量。

如果一次没有写完：

未写完的 ByteBuffer 保存在连接状态或队列中，并增加 OP_WRITE 关注；后续可写时从原 position 续写，队列为空后移除写关注。

示意代码：

```java
ByteBuffer out = ...;

int n = client.write(out);
if (out.hasRemaining()) {
    pendingOutput = out;
    key.interestOps(key.interestOps() | SelectionKey.OP_WRITE);
}
```

可写事件中：

```java
client.write(pendingOutput);

if (!pendingOutput.hasRemaining()) {
    pendingOutput = null;
    key.interestOps(key.interestOps() & ~SelectionKey.OP_WRITE);
}
```

### 为什么不能一直关注 OP_WRITE

发送缓冲区大多数时间都有空间，如果一直关注 `OP_WRITE`，Selector 可能频繁报告可写，造成空转。

合理策略是：

仅在有未写完数据时关注 OP_WRITE；全部写完后取消关注，而非取消连接注册。

`OP_WRITE` 表示“底层还能接受更多数据”，不表示“业务必须马上发送”。

### 写缓冲与背压

业务生产过快、对端接收过慢时，本地发送路径可能只接受部分字节或暂时接受 0 字节，待写队列持续增长，需要反馈到真正的数据源。

如果无限缓存待写数据，会消耗越来越多内存。应用需要：

- 给每条连接的待写队列设置上限。
- 找到实际生产来源并暂停或减量；请求响应可按情况停读该连接，但群发中停读慢接收者本身不能阻止其他发送者继续生产。
- 设置超时，必要时断开长期不消费数据的客户端。
- 把业务处理与网络线程分开，避免慢业务拖住整个事件循环。

## 7. Selector 事件循环与线程模型

典型事件循环：

每轮先 select 等待，再遍历 selectedKeys，处理适用事件并移除已处理项；剩余状态保存在各连接对象中，下一轮继续。

一个线程可以管理多条连接，但一次只能执行一个任务。如果某个 Handler 中查询数据库 3 秒，这个 Selector 线程中的其他连接也会等待。

因此：

- 网络事件处理应尽量短。
- 阻塞 I/O、数据库查询和远程 HTTP 调用应交给业务线程或其他异步机制。
- 多个 Selector 线程可以分担连接，但每条 Channel 的状态通常只由一个线程修改，减少并发冲突。

## 8. NIO 与 BIO 对比

| 维度 | BIO | NIO |
|---|---|---|
| 等待方式 | 线程阻塞在单个连接 | Selector 等待一组连接的就绪事件 |
| 连接数量大但空闲 | 通常需要许多等待线程 | 少量线程可以管理大量连接 |
| `read()` | 可能阻塞到有数据、结束或异常 | 非阻塞 Channel 当前没数据时返回 `0` |
| `write()` | 发送路径不足时可能阻塞 | 只写一部分，剩余数据后续继续写 |
| 半包处理 | 每连接通常一个线程可直接累积 | 应用仍必须按连接累积，并发状态更多 |
| 主要难点 | 线程数量和阻塞边界 | 事件状态、半包、续写、背压和线程安全 |

### 二进制位与 interestOps

某个事件通常是一个二进制位：

| 事件 | 二进制位示意 |
|---|---|
| OP_READ | 0001 |
| OP_WRITE | 0100 |

增加关注事件：

```java
interestOps = interestOps | OP_WRITE;
```

移除关注事件：

```java
interestOps = interestOps & ~OP_WRITE;
```

判断是否关注：

```java
boolean writeInterested =
        (interestOps & SelectionKey.OP_WRITE) != 0;
```

按位或用于“打开某个开关”，按位与配合取反用于“关闭某个开关”。

## 9. 常见错误

| 错误 | 后果 | 修正 |
|---|---|---|
| 仍用阻塞 Channel 注册 Selector | 注册时抛异常 | Channel 必须先设为非阻塞 |
| 一次 `read()` 就当完整消息 | 半包、截断、乱码 | 按连接累积并按协议分帧 |
| 把 `read()` 返回 `0` 当连接关闭 | 错误断开连接 | 只有 `-1` 表示读取方向结束 |
| 处理完 selected key 不清理 | 旧事件被反复处理 | `iterator.remove()` 或最后 `clear()` |
| 一直关注 `OP_WRITE` | Selector 空转、CPU 高 | 只有未写完时才注册 |
| 一次没写完就丢弃剩余 ByteBuffer | 数据被截断 | 保留 remaining 数据，后续续写 |
| 所有连接共用一个 pending 数组 | 不同客户端的字节混在一起 | 每条连接独立保存状态 |
| 在 Selector 线程执行慢业务 | 其他连接全部延迟 | 把阻塞任务移到业务线程池 |
| 无限缓存待写消息 | 内存持续增长 | 队列限流并形成背压 |
| 修改 interestOps 时覆盖其他关注位 | 原本要关注的事件被误关 | 用位运算修改指定关注位，明确线程归属 |

## 10. 与 TCP、BIO、Netty 的关系

| 层次或模型 | 职责 |
|---|---|
| TCP | 可靠有序字节流、连接状态与窗口 |
| BIO | 以阻塞读写使用 I/O |
| NIO | Channel、Buffer、Selector 管理就绪事件 |
| Netty | 封装事件循环、处理链、编解码及资源管理 |

NIO 没有消除 TCP 的分帧问题、慢客户端问题和写缓冲问题。它改变的是等待 I/O 的方式，应用协议和状态管理仍需自己处理。

## 11. 自测与答案

1. NIO 为什么能用少量线程管理大量连接？
2. `ServerSocketChannel` 与客户端连接得到的 `SocketChannel` 有什么区别？
3. `OP_ACCEPT` 就绪后，Selector 会自动执行 `accept()` 吗？
4. 非阻塞 `read()` 返回 `0` 和 `-1` 分别表示什么？
5. 为什么一次 `read()` 可能只读到半条消息？
6. `selectedKeys()` 为什么需要清理？
7. 一次 `write()` 没有写完时，剩余数据放在哪里？
8. 为什么不能一直关注 `OP_WRITE`？
9. `interestOps | OP_WRITE` 与 `interestOps & ~OP_WRITE` 分别做什么？
10. 在 Selector 线程中执行慢数据库查询会有什么后果？

### 11.1 对照答案

1. NIO 把多条 Channel 注册到 Selector；线程通过 `select()` 等待就绪事件，只处理当前有事件的连接，因此空闲连接不需要各自占用一个阻塞线程。
2. `ServerSocketChannel` 通常只有一个，负责监听端口和接受连接；`SocketChannel` 每个客户端连接一个，负责该连接的读写。
3. 不会。Selector 只报告 `OP_ACCEPT` 就绪；程序仍要调用 `accept()` 来取得新的 `SocketChannel`。
4. read 返回 0 表示没有读入字节，应同时检查是否暂时无数据及 Buffer 是否还有空间；不能凭 0 推断断线。-1 表示输入结束，常见为对端结束发送，也可来自本地关闭输入。收到 `-1` 后，本方在完全关闭前仍可以写出响应。
5. TCP 是字节流，`read()` 返回的是当前可用字节数，可以少于一条完整消息。消息边界必须由应用协议和累积逻辑确定。
6. selectedKeys 是选择操作维护的就绪 Key 集合，不会每轮自动清空。不清理会让已经处理过的事件留在集合中，下一轮可能被重复处理。
7. 未写完的 `ByteBuffer` 要保存在该连接自己的待写队列中，并注册 `OP_WRITE`；收到可写事件后，从原 position 继续写。
8. Socket 通常长期处于可写状态。一直关注 `OP_WRITE` 会反复触发可写事件并造成空转，因此只在存在未写完数据时注册。
9. `interestOps | OP_WRITE` 打开写事件开关；`interestOps & ~OP_WRITE` 只关闭写事件开关，保留其他事件。
10. Selector 事件处理通常是串行的。一个线程被慢数据库查询阻塞时，同一个 Selector 管理的其他连接也无法及时处理，所以阻塞任务应交给业务线程或其他异步机制。


本文代码为教学片段，已对照 Java NIO API 与 NioServer 的协议说明核对；未重跑多连接、半包、部分写入和断线实验。非阻塞网络 I/O 不保证任意方法及业务逻辑都永不阻塞。
