# Evrete 学习指南：前置知识与文档路线图

> 本文回答两个问题：掌握 Evrete 需要哪些前置知识？还有哪些文档方向值得补充？
> 分析版本：4.0.4-SNAPSHOT | 分析日期：2026-07-03

已有参考文档：
- [design-analysis.md](./design-analysis.md) — 宏观架构分析（12章）
- [design-patterns-deep-dive.md](./design-patterns-deep-dive.md) — 实现模式分析（13章）
- [data-flow-trace.md](./data-flow-trace.md) — 运行时数据流追踪（从 insert 到 RHS 的完整路径）
- [known-issues.md](./known-issues.md) — 已知问题与技术债务汇总

---

## 一、前置知识清单

要从源码层面真正掌握 Evrete，以下知识会反复用到。按重要程度标星。

---

### 1. Java 泛型进阶（★★★ 频繁出现）

自引用泛型（Self-referential generic）是整个 API 层的核心模式：

```java
public interface RuleSession<S extends RuleSession<S>>
    extends RuleSetContext<S, RuntimeRule>, SessionOps, MemoryStreaming
```

为什么需要这个？

```java
// 没有自引用泛型的话：
RuleSession session = service.newStatefulSession();
session.insert(fact);        // 返回 RuleSession，不是 StatefulSession
session.insert(fact).fire(); // 需要强转

// 有了自引用泛型：
StatefulSession session = service.newStatefulSession();
session.insert(fact)         // 返回 StatefulSession，类型正确
       .fire();              // StatefulSession.fire()
```

Evrete 中用到的泛型模式：
- **自引用泛型** — `S extends RuleSession<S>`
- **泛型边界组合** — `CombinationIterator<T>` 的类型数组创建
- **幻影类型（Phantom Type）** — `Mask<T>` 的类型参数仅用于编译期类型标记

需要用到的 Java 泛型知识点：通配符边界、类型擦除边界、`@SuppressWarnings("unchecked")` 的合理使用场景。

---

### 2. Java SPI（ServiceLoader）机制（★★★）

Evrete 没有依赖注入框架，所有扩展点都通过 `java.util.ServiceLoader` 发现：

```java
static DSLKnowledgeProvider load(String dsl) {
    ServiceLoader<DSLKnowledgeProvider> loader =
        ServiceLoader.load(DSLKnowledgeProvider.class);
    // 遍历、匹配 DSL 名称、校验唯一性
}
```

需要理解：
- `META-INF/services/` 文件结构（全限定类名作为文件名）
- `ServiceLoader.load()` 与 `ServiceLoader.load(Class, ClassLoader)` 的区别（线程上下文类加载器）
- `OrderedServiceProvider` 的优先级排序机制
- ServiceLoader 的懒加载特性

Evrete 中所有 SPI 接口一览：

| SPI 接口 | 用途 | 默认实现 |
|---|---|---|
| `DSLKnowledgeProvider` | 插件化的 DSL 语言支持 | 无（由 DSL 模块提供） |
| `MemoryFactoryProvider` | 可替换的内存/存储策略 | `DefaultMemoryFactoryProvider` |
| `TypeResolverProvider` | 自定义类型解析 | `DefaultTypeResolverProvider` |
| `SourceCompilerProvider` | 自定义源码编译器 | `DefaultSourceCompilerProvider` |

---

### 3. RETE 算法（★★★）

Evrete 是一个 RETE 算法的 Java 实现。**不理解 RETE，就无法理解为什么代码要这样组织。**

#### 核心概念

| 概念 | 说明 | Evrete 对应类 |
|---|---|---|
| Alpha 节点 | 对单个事实类型的条件过滤 | `TypeAlphaConditions`, `TypeAlphaMemory` |
| Beta 节点 | 多事实模式之间的连接和条件评估 | `BetaEvaluator`, `ConditionMemory`, `LhsConditionDH` |
| 终端节点 | 规则的全部条件已满足，可以触发 | `SessionRule` |
| Token | 沿 RETE 网络传播的部分匹配结果 | `WorkMemoryActionBuffer`, `DeltaMemoryAction` |
| 冲突集 | 当前可触发的规则集合 | `ActivationContext`, `ActivationManager` |
| Agenda | 规则触发的顺序管理 | `ActivationMode` |

#### 两阶段网络

Evrete 的独特之处在于将 RETE 网络分为编译时和运行时两阶段：

```
知识编译阶段（Knowledge）            运行阶段（Session）
ReteKnowledgeNode                  ReteSessionNode
  ReteKnowledgeConditionNode          ReteSessionConditionNode
  ReteKnowledgeEntryNode              ReteSessionEntryNode

transform() 将知识编译阶段的图转换为运行阶段
```

#### 推荐的学习路径

1. 读 Forgy 1982 年原论文，"Rete: A Fast Algorithm for the Many Pattern/Many Object Pattern Match Problem"（~30 分钟可读完核心）
2. 或者读 Doorenbos 1995 年简化版，"Production Matching for Large Learning Systems"
3. 再回来看 Evrete 的 `ReteGraph` 和 transform 实现
4. 最后跟着一个 fact 的 insert 路径走一遍

---

### 4. Java Compiler API（★★☆）

Evrete 的条件编译（把 `@Where("$c.value > 100")` 变成可执行字节码）基于 `javax.tools.JavaCompiler`：

```java
JavaCompiler compiler = ToolProvider.getSystemJavaCompiler();
// InMemoryFileManager 将编译输出留在内存，不写磁盘
JavaFileManager fileManager = new InMemoryFileManager(
    compiler.getStandardFileManager(null, null, null));
JavaCompiler.CompilationTask task = compiler.getTask(
    null, fileManager, null, null, null, javaSources);
task.call();
```

需要理解：
- `JavaCompiler.getTask()` 的参数含义
- `SimpleJavaFileObject` 的子类化（`JavaSourceObject` 表示源码，`DestinationClassObject` 表示编译结果）
- `ForwardingJavaFileManager` 的实现
- 编译后的字节码通过自定义 `ClassLoader` 加载

相关类：
- `org.evrete.spi.minimal.compiler.DefaultSourceCompiler` — 编译入口
- `org.evrete.spi.minimal.compiler.InMemoryFileManager` — 内存文件管理
- `org.evrete.spi.minimal.compiler.ClassLoaderWrapper` — 类加载隔离
- `org.evrete.spi.minimal.compiler.PackageExplorer` — JAR 扫描

---

### 5. Java 并发工具（★★☆）

Evrete 使用 `CompletableFuture` 进行异步编排：

```java
// fireCycle 中的异步链式调用
CompletableFuture<ActivationContext.Status> memoryDeltaStatus =
    ctx.computeDelta(actions);

return memoryDeltaStatus
    .thenCompose(deltaStatus -> {
        WorkMemoryActionBuffer newActions =
            doAgenda(ctx, deltaStatus.getAgenda(), mode);
        return ctx.commitMemories(deltaStatus)
            .thenCompose(unused -> fireCycle(ctx, mode, newActions));
    });
```

需要理解：
- `CompletableFuture` 的链式编排（`thenCompose`, `thenApply`, `whenComplete`）
- `CompletableFuture.join()` 的阻塞特性
- `ConcurrentHashMap` 与 `synchronized` 块的混合使用
- `ExecutorService` 的线程池管理（`DelegatingExecutorService`）

关键类：
- `CompletionManager<K, T>` — Key 级别异步任务序列化
- `DelegatingExecutorService` — 内部/外部线程池生命周期管理
- `ExtendedFuture` — 异步结果包装
- `EventMessageBus.Handler` — 异步事件广播

---

### 6. BitSet 集合运算（★★☆）

`Mask<T>` 底层是 `java.util.BitSet`，RETE 网络的类型过滤完全靠它：

```java
public final class Mask<T> {
    private final BitSet delegate;
    private final ToIntFunction<T> intMapper;  // T → bit position
}
```

需要理解的运算：
- `and()`, `or()` — 组合条件
- `intersects()` — 判断两个节点是否涉及同一组事实类型
- `containsAll()` — 判断类型子集关系（RETE 网络中判断 alpha 条件是否被满足）
- `cardinality()` — 条件数量统计

使用场景示例——Alpha 内存桶的索引分配：

```java
// 一组 alpha 条件共享同一个 alpha 内存桶
TypeAlphaConditions alphaBucket = new TypeAlphaConditions(
    index, type, setOfAlphaConditions);
// 用 Mask 记录该桶包含哪些 alpha 条件
this.mask = Mask.alphaConditionsMask().set(alphaConditions);
```

---

### 7. 相关设计模式（★★☆）

| 模式 | 出现位置 | 为什么重要 |
|---|---|---|
| Builder + 自引用泛型 | `RuleSetBuilder`, `RuleBuilder`, `LhsBuilder` | 类型安全的链式 API，返回正确子类型 |
| 装饰器/Wrapper | `AbstractSessionWrapper`, `KnowledgeWrapper` 等 6 个类 | SPI 层隔离，允许部分覆盖行为 |
| 模板方法 | `AbstractActiveRule`, `AbstractRuntime`, `AbstractRuleSession` | 规则执行生命周期骨架 |
| 空对象 | `DefaultActivationManager`, `LeastImportantServiceProvider` | 默认行为可替换，SPI 有回退 |
| Copy-on-Write | `ForkingArray`, `ForkingMap` | 会话间知识库隔离 |
| 工厂方法 | `Mask.factTypeMask()`, `DSLKnowledgeProvider.load()` | 静态工厂替代构造 |
| 策略 | `ActivationManager`, `MemoryFactory`, `TypeResolver` | SPI 扩展点 |

---

## 二、建议的学习切入顺序

如果从头开始学 Evrete 源码，推荐这个顺序：

```
Step 1. 读 evrete-code-samples 入门示例
        → 理解 API 长什么样，运行几个 demo
        → 推荐：HelloWorldType.java, StatelessSimple.java

Step 2. 读 RETE 算法入门（30 分钟）
        → 理解为什么代码要这样组织
        → 推荐：Forgy 1982 或 Doorenbos 1995

Step 3. 读 docs/design-analysis.md
        → 理解整体架构分层和模块职责

Step 4. 读 docs/data-flow-trace.md
        → 跟着 session.insert(fact) 走完运行时全路径
        → 理解 fireCycle 递归、delta 计算、alpha/Beta 评估
        → 从 @Where 注解到字节码的编译管道

Step 5. 读 docs/design-patterns-deep-dive.md
        → 理解实现层面的模式和数据结构

Step 6. 选一个 SPI 接口看其默认实现和扩展点
        → 推荐 MemoryFactory 或 TypeResolver
        → 理解可扩展性设计

Step 7. 读核心测试
        → TypeSystemTests — 类型系统行为
        → StatefulInsertTests — 有状态会话
        → HotDeploymentStatefulTests — 热部署
        → 理解边界和预期行为
```

---

## 三、还可以补充的文档

已有两份文档覆盖了架构层和实现层（共 1025 行）。以下是有价值的补充方向，按优先级排列。

### 高优先级

**1. RETE 算法入门 → Evrete 实现对照**
从零理解 RETE，然后映射到 Evrete 的具体类。读者看完就能在源码中对应着找。

| RETE 概念 | Evrete 类 |
|---|---|
| Alpha 节点 | `TypeAlphaConditions` + `TypeAlphaMemory` |
| Beta 节点 | `BetaEvaluator` + `ConditionMemory` |
| 终端节点 | `SessionRule` |
| Token 传播 | `DeltaMemoryAction` + `WorkMemoryActionBuffer` |
| 冲突集 | `ActivationContext` + `ActivationManager` |

**2. ✅ 数据流全景追踪（已实现 → [data-flow-trace.md](./data-flow-trace.md)）**
时序图和流程图，完整走过运行时全路径。这是"代码导航地图"的核心。

```
insert(fact)
  → WorkMemoryActionBuffer 缓冲插入操作
  → StatefulSessionImpl.fire()
    → fireCycle()
      → computeDelta() — 计算增量
        → 处理 delete 操作（如果存在）
        → 处理 insert 操作
          → TypeAlphaMemory 插入
          → BetaEvaluator 笛卡尔积评估
          → SessionFactGroup 匹配
      → doAgenda() — 触发匹配的规则
        → RHS 执行 ctx -> { ... }
      → 如果 RHS 产生新动作，递归 fireCycle()
```

**2. 条件编译管道：@Where 字符串 → 字节码**
从注解字符串到 `javax.tools.JavaCompiler` 的完整路径。

```
@Where("$c.value > 100")
  → StringLiteralEncoder 占位符编码
  → Java 源码生成（DefaultLiteralSourceCompiler.RuleSource）
  → In-memory Java 编译（DefaultSourceCompiler + InMemoryFileManager）
  → 类加载（ClassLoaderWrapper）
  → Class.getMethod + MethodHandle 绑定
  → BetaEvaluator 运行时调用评估
```

### 中优先级

| 文档 | 内容 |
|---|---|
| **并发模型解析** | `CompletableFuture` 链、线程池策略、`synchronized` 边界、`CompletionManager` type 级别序列化 |
| **类型系统详解** | `NamedTypeResolver` → `DefaultTypeResolver` 解析流程、`TypeAlphaConditions` 索引分配、`FieldDeclaration` 推导字段 |
| **内存模型全景** | `SessionMemory` → `TypeMemory` → `TypeAlphaMemory` → `ConditionMemory` 的层级关系和生命周期 |
| **测试策略与覆盖分析** | 各模块测试覆盖率、边界场景、flaky test 原因分析 |

### 低优先级

| 文档 | 内容 |
|---|---|
| **SPI 扩展开发指南** | 写一个自定义 `MemoryFactory` 或 `TypeResolver` 的完整 demo |
| **代码导航地图** | "从哪开始读"按功能路径组织 |
| **evrete-code-samples 索引** | 23 个示例按场景分类（入门/DSL/JSR94/高级） |
| **API 演进研究** | 从 4.0.0 beta 到 4.0.4-SNAPSHOT 的废弃方法和 API 变化 |
| **性能基准解读** | JMH 测试场景和结果分析 |

---

## 四、关键文件速查表

作为学习时的快速导航：

| 你想理解什么 | 看哪个文件 |
|---|---|
| API 入口 | `KnowledgeService.java`, `StatefulSessionImpl.java`, `StatelessSessionImpl.java` |
| RETE 图 | `ReteGraph.java`, `ReteKnowledgeNode.java`, `ReteSessionNode.java` |
| 条件评估 | `BetaEvaluator.java`, `TypeAlphaConditions.java`, `LhsConditionDH.java` |
| 会话生命周期 | `AbstractRuleSession.java`, `AbstractRuleSessionOps.java`, `AbstractRuleSessionDeployment.java` |
| 热部署 | `DeltaMemoryAction.java`, `DeltaMemoryMode.java` |
| 事件 | `EventMessageBus.java`, `BroadcastingPublisher.java` |
| 动态编译 | `DefaultLiteralSourceCompiler.java`, `DefaultSourceCompiler.java`, `InMemoryFileManager.java` |
| 类型系统 | `DefaultTypeResolver.java`, `ActiveType.java`, `FactType.java` |
| 测试工具 | `TestUtils.java`, `RhsAssert.java`, `FactEntry.java` |
| 示例 | `evrete-code-samples/src/main/java/org/evrete/examples/` |
