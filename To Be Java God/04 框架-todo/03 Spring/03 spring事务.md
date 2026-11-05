# Spring 事务

![[附件/知识总览/03-spring事务.svg]]

本文聚焦同步调用、命令式事务管理及普通返回值；响应式事务或特殊返回类型需按各自规则理解。

## 1. 事务管理与 AOP 模型

数据库负责实际执行事务的提交与回滚；Spring 负责在业务方法边界管理数据库事务。Spring 事务并不是另一套独立的 MySQL 事务。

`@Transactional` 声明事务规则，默认通过 Spring AOP 代理和事务拦截器实现。调用经过代理时，代理在目标方法执行前后管理事务：

调用从 Controller 进入 Service 代理，事务拦截器按规则开启或加入事务，再执行目标方法；正常返回时尝试完成事务，异常退出时按回滚规则处理。

简化伪代码（省略传播行为、回滚标记及提交失败等细节）：

```java
开启事务();
Object result;
try {
    result = 调用目标方法();
} catch (Throwable e) {
    if (符合回滚规则(e)) {
        回滚事务();
    } else {
        提交事务();
    }
    throw e;
}
提交事务(); // 正常返回路径，仍可能因 rollback-only 等失败
return result;
```

调用需要经过代理才能应用该方法的事务增强。同类内部直接调用另一个方法，不会重新经过该方法的代理；如果外层已开启事务，内部数据库操作仍可参与已有事务。

## 2. 异常有没有经过事务拦截器

只有异常传出目标方法，事务拦截器才能按异常规则作判断。异常被内部捕获并正常返回时，拦截器看不到该异常。

```java
@Transactional
public void a() {
    mapper.insert(message);
    try {
        b();
    } catch (IllegalStateException e) {
        // 已处理，不再抛出
    }
}

private void b() {
    throw new IllegalStateException("业务失败");
}
```

假设 B 是普通方法，数据库操作成功，且没有其他回滚标记，则 A 正常返回，事务提交。B 自己捕获异常也是同样结果。

如果 `mapper.insert(message)` 在上述 try 块之前抛出运行时异常，后面的 catch 捕获不到它；异常直接退出 A，传到外层事务拦截器，默认回滚。

如果异常传出事务方法，外面的 Controller 或全局异常处理器再捕获，事务拦截器已先处理回滚，不会因外层捕获而恢复提交。

> “捕获了异常就一定提交”并不成立：事务可能已被其他参与者标记为只能回滚，或数据库操作已使事务无法提交。

## 3. 默认与自定义回滚规则

普通默认配置下，RuntimeException 和 Error 触发回滚，受检异常默认不触发；这不等于受检异常被吞掉。全局回滚策略、方法规则或事务资源状态可能改变结果，应检查实际配置。

```java
@Transactional(rollbackFor = Exception.class)
```

匹配 Exception 及其所有子类，包括受检异常、RuntimeException。Error 不是 Exception 的子类，但默认仍触发回滚。

支持多个类型：

```java
@Transactional(rollbackFor = {
    IOException.class,
    SQLException.class
})
```

新增规则不会取消原有 RuntimeException、Error 的默认回滚行为。异常被内部捕获时，仅配置 rollbackFor 不会自动触发回滚。

`noRollbackFor` 指定不触发回滚的异常类型及其子类：

```java
@Transactional(
    rollbackFor = Exception.class,
    noRollbackFor = IOException.class
)
public void a() throws FileNotFoundException {
    mapper.insert(message);
    throw new FileNotFoundException("文件不存在");
}
```

FileNotFoundException 同时匹配两个规则，但 IOException 更具体，因此提交，异常仍向外传播。这里使用的是 class 类型规则；越接近实际异常类型的规则优先。避免为同一异常配置相互冲突的规则。

不回滚不表示异常被吞掉，也不表示接口成功。

## 4. 捕获异常后主动要求回滚

需要在内部处理异常，同时回滚当前事务时：

```java
try {
    throw new IllegalStateException("业务失败");
} catch (IllegalStateException e) {
    // 记录日志等异常处理
    TransactionAspectSupport.currentTransactionStatus()
            .setRollbackOnly();
}
```

需导入 `org.springframework.transaction.interceptor.TransactionAspectSupport`，并在有效的 Spring 事务调用内使用。

- setRollbackOnly() 标记当前事务只能回滚，不是立即回滚。
- 它不会终止方法，后续代码仍会执行。
- 对普通 REQUIRED 事务范围，它不是只回滚 catch 中的一条 SQL；NESTED 保存点范围需另行判断，不能笼统视为整个物理事务。
- 被标记范围不能因方法正常返回而恢复提交。当前范围主动设置本地回滚与内层参与者标记共享事务，在是否出现 UnexpectedRollbackException 上有区别；不要一概认为 setRollbackOnly 必然抛出该异常。接口仍应表达真实业务结果。




## 5. 传播行为总览与适用前提

**Propagation（propagation）**：传播。这里指方法被调用时，如何处理当前事务。以下以调用经过 Spring 代理、使用同一事务管理器的常规场景为前提。

| 配置名称                | 小写单词与含义               | 记忆方式          | 当前有事务         | 当前没有事务 |
| ------------------- | --------------------- | ------------- | ------------- | ------ |
| **`REQUIRED`**（默认）  | **required**：必需的      | 我必须有事务        | 加入当前事务        | 新建事务   |
| **`REQUIRES_NEW`**  | **requires new**：需要新的 | 我必须用新事务       | 挂起当前事务，新建独立事务 | 新建事务   |
| **`SUPPORTS`**      | **supports**：支持       | 有就用，没有也行      | 加入当前事务        | 在事务外执行 |
| **`MANDATORY`**     | **mandatory**：强制的     | 必须带着事务来       | 加入当前事务        | 抛异常    |
| **`NOT_SUPPORTED`** | **not supported**：不支持 | 我不在事务里执行      | 挂起当前事务，在事务外执行 | 在事务外执行 |
| **`NEVER`**         | **never**：绝不          | 绝不能带着事务来      | 抛异常           | 在事务外执行 |
| **`NESTED`**        | **nested**：嵌套的        | 在大事务里设一个局部回退点 | 使用保存点建立嵌套范围   | 新建事务   |

- **加入**：共用已有事务，内部方法返回时不会单独提交。
- **挂起**：暂时放下外层事务，内部调用结束后恢复；不是提交或回滚。实际挂起能力取决于事务管理器。
- **保存点**：允许局部回滚，但不能独立提交；NESTED 需要事务管理器支持。
- **事务外执行**：不加入 Spring 管理的事务，MySQL 仍可能逐条自动提交 SQL。


## 6. 共享事务与调用前置要求

### REQUIRED：加入已有事务，共享最终成败

**required** 意为“必需的”：方法需要事务，当前有事务就加入，没有就新建。它是 `@Transactional` 的默认传播行为。

以下以同线程、同数据源、同事务管理器，且调用经过 Spring 代理为前提。

**inner 使用 `REQUIRED` 时，会加入 outer 已开启的事务，两者共享提交与回滚结果。**

如果 inner 抛出运行时异常，异常经过它的事务代理，整个事务就会被标记为只能回滚。即使 outer 捕获异常并正常返回，最终也无法提交，而会回滚并抛出 `UnexpectedRollbackException`。

这个异常表示：**外层期待提交，但共享事务已经被内层标记为必须回滚。**

`REQUIRED` 的核心是**共享成败**。业务要求一起成功或一起失败时，应让异常继续向外传播；可以捕获后记录日志再抛出，不应吞掉异常后按成功处理。


### SUPPORTS：有事务就加入，没有就直接执行

**supports** 意为“支持”：加入当前已有事务；没有事务时，不主动开启 Spring 事务。

无外层事务时，普通 InnoDB 增删改通常由 MySQL 自动提交，后续 Java 异常不能撤销已提交的修改。多次写入若必须一起成功，应使用 REQUIRED 保证事务存在。

在常见 Spring + MyBatis 查询场景中，SUPPORTS 与不加事务注解通常没有明显区别，一般不必特意添加；它主要用于明确表达“事务可选”的约定。

### MANDATORY：必须在已有事务中执行

**mandatory** 意为“强制的”：当前有事务就加入，没有事务则抛出 `IllegalTransactionStateException`，方法体不会执行。

适合的业务是：**某个内层写入操作必须属于一个更大的业务事务，不允许单独执行。**

例如，一次转账包含三个步骤：扣除付款方余额、增加收款方余额、写入转账流水。这三个步骤应当一起成功或一起回滚，避免出现“钱已扣但未到账”或“只有流水却没有实际转账”的情况。

因此，可以将内部“写转账流水”的方法声明为 `MANDATORY`：调用者必须先开启事务，它才能执行，并参与该事务的整体成败。如果其他入口在没有事务时直接调用它，就会抛出异常。


## 7. 独立事务与事务外执行

### REQUIRES_NEW：独立事务，独立成败

**requires new** 意为“需要新的”：始终新建独立事务；当前已有事务时，先将其挂起，内层事务结束后再恢复。

outer 开启 T1，调用 inner 时挂起 T1，inner 在独立 T2 中完成；T2 提交或回滚后恢复 T1，outer 继续。

**挂起不是提交或回滚，外层事务持有的连接和锁通常仍然保留。** 在常规 JDBC 场景中，内层独立事务使用另一条数据库连接。

**异常传播与事务结果**

| 场景 | T1（outer） | T2（inner） |
|---|---|---|
| inner 成功，outer 随后失败 | 回滚 | 已提交，不受影响 |
| inner 失败，outer 捕获后正常返回 | 提交 | 回滚 |
| inner 失败，outer 不捕获 | 回滚 | 回滚 |

以上以运行时异常及默认回滚规则为前提。

REQUIRES_NEW 通常需要额外连接；外层仍持有资源，并发调用或更深层独立事务可能耗尽连接池，也可能因访问外层持锁的数据而等待。独立提交不是没有资源代价。

**独立事务不代表异常隔离。** outer 不捕获时，异常继续向外传播，分别触发 T2、T1 的回滚；不是 T2 回滚自动导致 T1 回滚。

**更深层调用的当前事务**

inner 调用 innermost 时，**当前事务是 T2，T1 仍处于挂起状态**：

- innermost 使用 `REQUIRED`、`SUPPORTS`、`MANDATORY`：加入 T2。
- innermost 使用 `REQUIRES_NEW`：挂起 T2，新建 T3。
- innermost 无事务注解：普通 MyBatis 操作通常参与 T2。

**业务场景**

适合需要独立保存结果的操作，例如记录业务失败的数据库审计记录：主业务回滚，已独立提交的审计记录仍然保留。

### NOT_SUPPORTED：挂起已有事务，在事务外执行

**not supported** 意为“不支持”：不参与 Spring 管理的事务；当前有事务则挂起，没有事务也不新建。

outer 开启 T1，调用 inner 时挂起 T1；inner 在事务外完成，随后恢复 T1。

在通常的 MySQL 自动提交配置下，inner 中的 SQL 各自提交，后续异常无法撤销已提交的修改。因此，**outer 回滚不会撤销 inner 已提交的记录，inner 的多次写入也不保证整体成败。**

inner 调用 innermost 时，**“当前”没有事务**，因为 T1 仍被挂起。之后如何执行，取决于 innermost 是否经过事务代理，以及它的传播配置；不会自动加入被挂起的 T1。

注意：**挂起不会释放外层事务已经持有的连接和锁。**

### NEVER：禁止在已有事务中执行

**never** 意为“绝不”：当前没有事务时直接执行，不新建事务；当前有事务时抛出 `IllegalTransactionStateException`，方法体不会执行。

它与 MANDATORY 的前置要求正好相反：

| 传播行为 | 要求 | 不满足时 |
|---|---|---|
| MANDATORY（mandatory，强制的） | 必须已有事务 | 没事务就抛异常 |
| NEVER（never，绝不） | 必须没有事务 | 有事务就抛异常 |

NEVER 在常规业务中使用较少，主要用于明确表达“禁止在事务中调用”的约束。

## 8. NESTED：局部回滚与保存点

**nested** 意为“嵌套的”：当前有事务时，通过保存点建立嵌套范围；没有事务时，新建事务。

**内层可以回滚到保存点，只撤销局部修改；但不能独立提交，最终仍受外层事务控制。** 在 JDBC/MyBatis 场景中，通常使用同一数据库连接上的保存点，需要事务管理器支持。

| 场景 | 外层结果 | 内层结果 |
|---|---|---|
| 内层失败，外层捕获后正常提交 | 保留 | 回滚到保存点，撤销 |
| 内层成功，外层最终回滚 | 回滚 | 一起回滚 |
| 内层失败，异常向外传播并导致外层回滚 | 回滚 | 回滚 |

以上以异常符合回滚规则、没有其他整体回滚标记为前提。

**更深层调用与保存点**

假设 outer 开启 T1，inner 使用 NESTED 创建保存点 S1，再调用 innermost：

- innermost 加入当前事务：仍使用 T1，不新建保存点；修改发生在 S1 之后，回滚到 S1 会将其撤销。
- innermost 使用 NESTED：仍在 T1 内，再创建保存点 S2。回滚到 S2 只撤销 S2 之后的修改；回滚到 S1 则撤销 S1 之后的修改。
- innermost 使用 REQUIRES_NEW：挂起 T1，开启独立事务 T2；T2 已提交的修改不受回滚到 S1 影响。

**重点：不能只看保存点，还要看异常经过的事务代理。**

在同线程、同数据源、同事务管理器，异常符合回滚规则、事务管理器采用默认配置的前提下：

1. **innermost 没有事务注解**：异常传到 inner 的 NESTED 代理时，回滚到 S1，撤销 inner 和 innermost 的修改。外层若捕获异常并正常结束，且没有其他回滚因素，可以提交剩余修改。
2. **innermost 是 REQUIRED，且调用经过其代理**：异常先经过 innermost 的事务代理，会将共用的 T1 标记为只能回滚。即使 inner 再回滚到 S1，也无法清除这个标记，最终 T1 仍须整体回滚；外层若正常返回并尝试提交，通常会收到 UnexpectedRollbackException。

**事务决定整体提交范围，保存点决定同一事务内可以局部撤销到哪里；局部回退不能解除整体 rollback-only 标记。**


## 9. 综合调用与 UnexpectedRollbackException

**综合 Demo 1：独立事务回滚，外层仍可提交**

前提：不同 Spring Bean 之间通过代理调用，同线程、同数据源、同事务管理器，采用默认回滚配置。

| 调用层 | 传播与写入 |
|---|---|
| outer | REQUIRED 开启 T1，插入 A |
| inner | REQUIRES_NEW 挂起 T1，开启 T2，插入 B |
| innermost | REQUIRED 加入 T2，插入 C，抛 IllegalStateException |

inner 不捕获；outer 捕获后正常返回。

innermost 的代理将 T2 标记为只能回滚；异常继续传出 inner，inner 的代理回滚 T2。恢复 T1 后，outer 捕获异常并正常返回，T1 提交。

**结果：A 保留，B、C 回滚。** T2 的回滚标记不会自动影响独立的 T1；如果 outer 不捕获，异常继续传出 outer，才会进一步触发 T1 回滚。

**综合 Demo 2：保存点不能清除整个事务的回滚标记**

沿用上述调用前提，将 inner 改为 NESTED：

| 调用层 | 传播与写入 |
|---|---|
| outer | REQUIRED 开启 T1，插入 A |
| inner | NESTED 在 T1 建立 S1，插入 B |
| innermost | REQUIRED 加入 T1，插入 C，抛 IllegalStateException |

inner 不捕获；outer 捕获后正常返回。

innermost 的代理先将整个 T1 标记为只能回滚。inner 的代理随后回滚到 S1，撤销 B、C，但无法清除 T1 的 rollback-only 标记。outer 正常返回后，外层代理无法提交，最终撤销 A，并抛出 UnexpectedRollbackException。

**结果：A、B、C 全部回滚。** 决定整体回滚的是 T1 的回滚标记，不是 outer 使用 REQUIRED 就必然不能局部成功。

**UnexpectedRollbackException 的原因**

**事务已经被标记为只能回滚，但外层方法正常返回，外层代理按正常路径尝试提交，最终只能回滚，因此抛出 UnexpectedRollbackException，告知调用方“预期提交，实际回滚”。**

它通常不是原始业务异常：原异常可以已经被外层 catch 捕获，而此异常发生在外层方法返回后的事务完成阶段。外层若直接抛出符合回滚规则的异常，代理按异常路径回滚，通常不会仅因该回滚标记再抛出 UnexpectedRollbackException。

## 10. 代理边界与事务增强失效

以下以 Spring 默认的代理式事务管理为前提。

| 情况 | 原因 |
|---|---|
| 同类内部调用 `this.inner()` | 绕过 inner 的代理，其事务配置不生效 |
| 自己 `new` 一个 Service | 普通对象不会自动获得 Spring 事务代理 |
| 给 `private` 方法加注解 | JDK 无公开接口入口；CGLIB 无法重写私有方法 |
| 给 `static` 方法加注解 | 静态调用不经过实例代理拦截 |
| 类代理中的 `final` 方法加注解 | CGLIB 无法重写；**JDK 接口代理仍可增强** |

以上指**方法自身的事务增强不生效**，不代表其中的 SQL 一定没有外层事务。

本文保留原有规则、传播场景与综合推理；本次对照 Spring 文档核对语义，未重跑数据库与事务异常实验。示例代码是局部说明，不代表所有配置下的运行结果已验证。
