# 03｜EventLoop 与线程模型

![[附件/知识总览/03-EventLoop与线程模型.svg]]

## 1. EventLoop 处理三类工作

以 NIO 传输为例，一个 EventLoop 循环处理已注册 Channel 的网络就绪事件、提交给它的普通任务以及到期的定时任务。

没有工作时可以在 Selector 中等待。网络 I/O 的“非阻塞”不意味着线程必须始终忙轮询，也不意味着 Handler 内的普通 Java 方法会自动变成非阻塞执行。

## 2. boss、worker 与连接的归属

```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup(1);
```

在典型服务端配置中，boss 管理监听 Channel 的接受连接事件，worker 中的 EventLoop 管理接受连接后的网络 I/O。boss、worker 表达职责，二者都可以使用 NioEventLoopGroup。

连接注册后，通常持续关联同一个 EventLoop；一个 EventLoop 可以管理多条连接。这里讨论常规注册生命周期，不表示 Netty 绝对禁止注销及重新注册。

因此，连接 A 与 B 即使有独立 Pipeline，也可能共享同一个执行网络事件的线程。

## 3. 阻塞为何会影响其他连接

如果 A 和 B 由同一个 worker EventLoop 管理，A 的 Handler 同步查数据库耗时 3 秒，这段时间 EventLoop 被占用，B 的网络事件、普通任务以及到期定时任务都可能延迟。

常见来源包括同步 JDBC、同步远程调用、sleep、锁等待及长时间 CPU 计算。等待一个需要当前 EventLoop 自己推进的 Future，还可能造成死锁风险。

轻量日志、字段判断与短小计算通常可以直接执行。是否拆到业务线程，需要结合阻塞性、耗时和隔离需求判断；切线程本身也有排队与调度成本。

## 4. 业务执行器改变回调线程

```java
DefaultEventExecutorGroup businessGroup =
        new DefaultEventExecutorGroup(2);

ch.pipeline().addLast(
        businessGroup, "chatHandler", new ChatHandler());
```

这个配置使该 Handler 的事件回调交给分配的业务执行器执行。Channel 的网络注册关系并未迁移，I/O 仍由原来的 worker EventLoop 管理。

业务执行器也会排队，两个业务线程不保证任意两条连接都能同时执行业务。DefaultEventExecutorGroup 提供有序执行器，维持分配到同一执行器的任务顺序；它不等于为每条消息新开线程。

不要在每次消息到达时创建一个新的 businessGroup。线程组应作为应用资源统一管理，结束时释放。

## 5. 后续 Handler 如何切回 EventLoop

```java
pipeline.addLast(new HandlerA());
pipeline.addLast(businessGroup, "business", new HandlerB());
pipeline.addLast(new HandlerC());
```

以入站事件依次传播为例：

1. worker EventLoop 执行 A，A 向 B 传播时将任务交给 B 的业务执行器，worker 可以继续处理其他工作。
2. 业务执行器运行 B；B 完成这次处理并向后传播事件时，将 C 的处理交回其执行器。
3. C 没有显式指定其他执行器，默认由该 Channel 的 EventLoop 执行。

**消息的后续步骤等待传播，不等于 worker 阻塞等待。** Netty 根据目标 Context 的执行器决定直接调用还是提交任务。

如果 B 自己再次异步提交业务，却在完成前就传播事件，不能据此推断 C 一定在该异步业务完成后执行。处理完成与传播时机需要业务代码明确组织。

## 6. 定时任务与进程生命周期

```java
channel.eventLoop().scheduleAtFixedRate(
        () -> channel.writeAndFlush(createMessage()),
        0, 12, TimeUnit.SECONDS);
```

这个任务仍在该 EventLoop 上执行，不会自动得到独立业务线程；执行时间也可能受事件循环繁忙程度影响。12 秒只是例子中的应用参数，不是 TCP 或 Netty 规定的心跳周期。

默认 NioEventLoopGroup 使用非守护线程；线程已启动且存活时，即使 main 返回，JVM 仍可继续运行。非守护线程维持进程存活，守护线程不阻止 JVM 在只剩守护线程时正常退出；自定义 ThreadFactory 可以改变线程属性。

退出时应按资源所有权管理连接、周期任务以及 worker、boss、业务执行器组，调用 shutdownGracefully 并观察终止结果。仅把线程改为守护线程，不能替代资源清理。
