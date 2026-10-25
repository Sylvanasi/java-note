# 03｜EventLoop 与线程模型

## 1. EventLoop 做什么

当前 NIO 传输下，EventLoop 循环处理：

1. 多个 Channel 的网络就绪事件。
2. 提交给它的普通任务。
3. 到期的定时任务。

没有工作时可以在 Selector 等待，因此“非阻塞网络编程”不代表线程一直忙轮询。

## 2. boss 与 worker

```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup(1);
```

- boss 管理监听 Channel 的接受连接事件。
- worker 中的 EventLoop 管理建立后的连接。
- 一个连接注册后通常固定关联一个 EventLoop；同一个 EventLoop 能管理多条连接。

## 3. 为什么不能在 EventLoop 中做耗时阻塞操作

假设连接 A、B 由同一个 worker EventLoop 管理：

```text
A 的 Handler 同步查数据库耗时 3 秒
    ↓
EventLoop 被这个方法占住
    ↓
B 的网络事件、任务和该 EventLoop 上的定时任务也可能延迟
```

常见阻塞来源包括同步 JDBC、同步远程调用、长时间 sleep、等待锁、在当前 EventLoop 上等待需要它自己推进的 Future。

## 4. 指定业务执行器

```java
DefaultEventExecutorGroup businessGroup =
        new DefaultEventExecutorGroup(2);

ch.pipeline().addLast(
        businessGroup, "chatHandler", new ChatHandler());
```

这表示该 Handler 的事件回调由分配给它的业务执行器执行。Channel 的网络注册关系没有因此转移，连接仍由原来的 worker EventLoop 管理。

业务线程也会繁忙，也会排队；两个业务线程不保证任意两个连接永远能同时执行。默认有序执行器会保持每个对应 Context 的任务顺序。

## 5. 后续 Handler 能否回到 worker

```java
pipeline.addLast(new HandlerA());
pipeline.addLast(businessGroup, "business", new HandlerB());
pipeline.addLast(new HandlerC());
```

若 B 处理完才继续传播：

```text
worker：执行 A → 提交 B 的任务 → 处理其他工作
业务线程：执行 B → 提交 C 的任务
worker：执行 C
```

C 没有显式执行器时，默认使用 Channel 的 EventLoop。Netty 在传播时根据目标 Context 的执行器决定是否需要提交任务。

“这条消息的 C 要等 B 传播后才能执行”和“worker 阻塞等 B”是两件不同的事。

## 6. 定时任务

```java
channel.eventLoop().scheduleAtFixedRate(
        () -> channel.writeAndFlush(createMessage()),
        0, 12, TimeUnit.SECONDS);
```

任务由这个 EventLoop 执行，不会因为使用了调度方法就自动变成独立的阻塞业务线程。任务本身仍应短小。

周期为 12 秒是本次业务发送实验参数，不是 TCP 或 Netty 的固定心跳要求。

## 7. main 返回为什么进程还在

当前 EventLoopGroup 默认启动非守护线程：main 返回后，只要这些线程仍存活，JVM 就会继续运行。

- 非守护线程：会使 JVM 保持运行。
- 守护线程：不会阻止 JVM 正常退出；只剩守护线程时，JVM 不等待它们完成。

因此：

```text
main：配置客户端 → connect → 返回
EventLoop：处理连接 → 网络事件 → 心跳定时任务 → 继续等待
```

结束客户端时需要管理 Channel 和线程组的生命周期，不应依靠把线程改为守护线程来代替资源清理。

## 8. 业务很快时是否必须单独分组

简单日志、轻量字段判断、短小计算通常不需要专门为每个 Handler 切线程。线程切换和排队也有成本。是否拆分取决于业务是否阻塞、耗时与隔离要求。

