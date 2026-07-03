# Evrete 规则引擎设计分析

> 使用模板：[prompt-template-design-analysis.md](./prompt-template-design-analysis.md)
> 分析版本：4.0.4-SNAPSHOT | 分析日期：2026-07-03

---

## 1. 整体架构与模块划分

### 发现

Evrete 使用标准的扁平多模块 Maven 结构，5 个模块的职责清晰：

| 模块 | 源码文件 | 外部依赖 | 职责 |
|---|---|---|---|
| `evrete-core` | 217 | **无** | 核心引擎、API、SPI、默认实现 |
| `evrete-dsl-java` | 29 | `evrete-core` | 注解式 DSL 加载器 |
| `evrete-jsr94` | 15 | `evrete-core`, `jsr94:jsr94:1.1` | JSR94 标准适配 |
| `evrete-benchmarks` | 6 | `evrete-core`, JMH | 性能基准 |
| `evrete-code-samples` | 28 | `evrete-core`, `evrete-dsl-java` | 使用示例 |

### API/SPI/Runtime 三层分离

源码包结构清晰地映射了三层架构：

- **`org.evrete.api`** — 纯接口，43 个文件（直接位于 `api/` 下），外加 11 个 SPI 接口、9 个事件接口、4 个 Builder、3 个注解。定义 `Knowledge`, `RuleSession`, `Rule`, `FactHandle`, `Type` 等全量 API 契约。无 `Impl` 后缀，无实现逻辑。
- **`org.evrete.api.spi`** — SPI 纯接口。`DSLKnowledgeProvider`, `MemoryFactoryProvider`, `TypeResolverProvider`, `SourceCompilerProvider` 共 4 组扩展点。
- **`org.evrete.runtime`** — 私有实现，全在 `package-private` 可见性下。核心 RETE 网络、会话管理、Delta 机制。

```java
// 典型的三层分离模式：
// API:  org.evrete.api.Knowledge
// SPI:  org.evrete.api.spi.MemoryFactoryProvider
// Impl: org.evrete.spi.minimal.DefaultMemoryFactoryProvider (package-private)
```

### 零外部依赖的代价与收益

`evrete-core` 的 `pom.xml` 在 `<dependencies>` 中只有 JUnit 5（test scope）。这意味着：

**收益：**
- 无传递依赖冲突
- 无框架锁定（不依赖 Spring/Guice）
- 类加载器隔离简单
- JAR 体积小（~300KB）
- 潜在适配场景广（Android、OSGi、GraalVM）

**代价：**
- `java.util.logging` 而非 SLF4J/Log4j → 与主流日志框架集成需要桥接
- 自建所有基础设施（事件总线、SPI 发现、数据结构）→ 维护成本
- 无依赖注入 → 所有组合通过继承和委派手工完成

### 亮点评分：4/5

模块边界清晰，API/SPI/Runtime 分离彻底，零外部依赖是务实而非教条的选择。扣 1 分因为 `evrete-dsl-java` 和 `evrete-jsr94` 的职责边界可以更明确（DSL 模块既做注解解析又做动态编译）。

---

## 2. API 设计

### 自引用泛型实现类型安全链式调用

`RuleSession<S extends RuleSession<S>>` 是整个 API 层的核心模式：

```java
// org.evrete.api.RuleSession
public interface RuleSession<S extends RuleSession<S>>
    extends RuleSetContext<S, RuntimeRule>, SessionOps, MemoryStreaming {
    default S insert(Object... objects) {
        insert0(objects, true);
        return (S) this;  // 返回正确的子类型
    }
    Object fire();
}
```

这使得 `StatefulSession` 和 `StatelessSession` 各自返回自身类型：

```java
StatefulSession s = service.newStatefulSession();
s.insert(fact1)        // → StatefulSession
 .insert(fact2)        // → StatefulSession
 .fire();              // → StatefulSession 的 fire()

StatelessSession s2 = service.newStatelessSession();
s2.insert(fact1)       // → StatelessSession
  .insert(fact2);      // → StatelessSession
```

### Builder 模式的对称性

`RuleSetBuilder<C extends RuntimeContext<C>>` 的 `.build()` 返回 `C`，即调用它的上下文类型。这意味着同一个 `RuleSetBuilder` 既可以作用于 `Knowledge` 也可以作用于 `RuleSession`：

```java
// org.evrete.api.builders.RuleSetBuilder
public interface RuleSetBuilder<C extends RuntimeContext<C>>
    extends FluentEnvironment<RuleSetBuilder<C>> {
    RuleBuilder<C> newRule(String name);
    C build();  // 返回构造时的上下文类型
}
```

### 接口默认方法驱动的 API 演进

`RuntimeContext` 统一了 `Knowledge` 和 `RuleSession` 的公共操作接口，利用默认方法提供便捷操作：

```java
// org.evrete.api.RuntimeContext
public interface RuntimeContext<C extends RuntimeContext<C>>
    extends FluentImports<C>, FluentEnvironment<C>, EventBus {

    RuleSetBuilder<C> builder(ClassLoader classLoader);

    default RuleSetBuilder<C> builder() {
        return builder(getClassLoader());  // 便捷重载
    }

    default C importRules(String dslName, Object source) throws IOException {
        return importRules(DSLKnowledgeProvider.load(dslName), source);
    }
}
```

### `@NonNull` / `@Nullable` 契约声明

API 层系统性地使用了自定义的 `@NonNull` 和 `@Nullable` 注解（非 JSR-305 或 checkerframework），在接口方法签名上显式声明空值契约：

```java
// org.evrete.api.RuleSession
<T> T getFact(@NonNull FactHandle handle);

// org.evrete.api.spi.DSLKnowledgeProvider
static DSLKnowledgeProvider load(@NonNull String dsl) {
    Objects.requireNonNull(dsl, "DSL identifier cannot be null");
    // ...
}
```

### 局限/风险

- `@SuppressWarnings("unchecked")` 在 API 层出现 11 处，因为自引用泛型的 `return (S) this` 无法避免 unchecked cast
- `@NonNull`/`@Nullable` 是自建注解，不与主流空值检测工具（IDE、SpotBugs、Checker Framework）直接集成
- 接口默认方法过多时，实现类可能难以追踪哪些行为是定制过的

### 亮点评分：5/5

自引用泛型的运用正确且一致，Builder 模式与泛型结合实现了 API 的对称性，接口默认方法既做兼容又做便捷重载。在 JDK 8 环境下能达到的类型安全上限。

---

## 3. 扩展性与 SPI 机制

### ServiceLoader 驱动的扩展发现

Evrete 没有依赖注入框架，所有扩展点通过标准的 `java.util.ServiceLoader` 发现：

```java
// org.evrete.api.spi.DSLKnowledgeProvider
static DSLKnowledgeProvider load(@NonNull String dsl) {
    ServiceLoader<DSLKnowledgeProvider> loader =
        ServiceLoader.load(DSLKnowledgeProvider.class);
    List<DSLKnowledgeProvider> found = new LinkedList<>();
    for (DSLKnowledgeProvider provider : loader) {
        if (dsl.equals(provider.getName())) {
            found.add(provider);
        }
    }
    if (found.isEmpty()) {
        throw new IllegalStateException("DSL provider '" + dsl
            + "' is not found. ...");
    }
    if (found.size() > 1) {
        throw new IllegalStateException("Multiple DSL providers found ...");
    }
    return found.iterator().next();
}
```

设计上有几个精妙之处：
1. **接口内静态方法** — `load()` 作为接口的静态方法，使用者无需再写一个 `ServiceLoader` 辅助类
2. **清晰的错误消息** — 找不到时列出所有可用的 provider，方便排查
3. **唯一性校验** — 避免多提供者意外冲突

### SPI 优先级排序

`OrderedServiceProvider` 提供了多提供者共存时的有序选择：

```java
// org.evrete.api.OrderedServiceProvider
public interface OrderedServiceProvider extends Comparable<OrderedServiceProvider> {
    int sortOrder();

    @Override
    default int compareTo(OrderedServiceProvider o) {
        return Integer.compare(this.sortOrder(), o.sortOrder());
    }
}
```

配套的 `LeastImportantServiceProvider` 提供了最低优先级的"兜底"实现：

```java
// org.evrete.spi.minimal.LeastImportantServiceProvider
public interface LeastImportantServiceProvider extends OrderedServiceProvider {
    @Override
    default int order() {
        return Integer.MAX_VALUE;
    }
}
```

### 4 组 SPI 扩展点

| SPI 接口 | 抽象层级 | 默认实现 | 可替换性 |
|---|---|---|---|
| `DSLKnowledgeProvider` | 语言级 | 无（由模块提供） | 高：完整语言插件 |
| `MemoryFactoryProvider` | 存储级 | `DefaultMemoryFactoryProvider` | 中：替换内存策略 |
| `TypeResolverProvider` | 类型级 | `DefaultTypeResolverProvider` | 中：自定义类型解析 |
| `SourceCompilerProvider` | 编译级 | `DefaultSourceCompilerProvider` | 中：替换编译后端 |

### 局限/风险

- SPI 仅有 4 组扩展点，**缺少 `EvaluatorProvider`** —— 条件评估器的评估逻辑是硬编码在 `BetaEvaluator` 中的，无法通过 SPI 替换
- `ServiceLoader` 的发现是静态的，无法在运行时动态注册/注销提供者
- 没有 SPI 生命周期管理（初始化、销毁回调）
- 没有明确的 SPI 配置属性命名空间规约（依赖实现者和使用者之间的默契）

### 亮点评分：4/5

ServiceLoader 的使用方式简洁且可预测，优先级排序是重要的补充机制。扣 1 分因为扩展点数量偏少，缺少评估器 SPI 等关键扩展点。

---

## 4. 核心算法实现（RETE 网络）

### 两阶段节点分离

Evrete RETE 实现最突出的设计决策是将节点分为知识阶段和会话阶段，通过 `transform()` 连接：

```java
// org.evrete.runtime.rete.ReteGraph
public final class ReteGraph<B extends ReteNode<B>, E extends B, C extends B> {
    private final C terminalNode;

    public <B1 extends ReteNode<B1>, E1 extends B1, C1 extends B1>
    ReteGraph<B1, E1, C1> transform(
            Class<B1> nodeType,
            BiFunction<C, B1[], C1> conditionNodeMapper,
            Function<E, E1> entryNodeMapper) {
        C1 newTerminalNode = (C1) transformNode(
            nodeType, this.terminalNode,
            conditionNodeMapper, entryNodeMapper);
        return new ReteGraph<>(newTerminalNode);
    }
}
```

```
知识编译阶段（Knowledge）            运行阶段（Session）
ReteKnowledgeNode                  ReteSessionNode
  ReteKnowledgeConditionNode         ReteSessionConditionNode
  ReteKnowledgeEntryNode             ReteSessionEntryNode
```

这个设计的精妙之处在于：

1. **编译时关注结构，运行时关注状态** — 知识阶段只需要构建规则的条件结构，不需要考虑内存分配
2. **分离使 hot deploy 成为可能** — 新规则可以在知识阶段构建 RETE 子图，然后 transform 并 merge 到运行中的会话
3. **类型通用** — `ReteGraph<B, E, C>` 的三个类型参数允许不同类型的节点共享同一个图结构

### Alpha 内存的桶共享

`TypeAlphaConditions` 将一组 alpha 条件绑定到一个索引，多个规则声明可能共享同一条 alpha 内存：

```java
// org.evrete.runtime.TypeAlphaConditions
public class TypeAlphaConditions implements Indexed {
    private final Mask<AlphaConditionHandle> mask;  // 条件集合
    private final int index;                         // 内存桶索引
    private final ActiveType.Idx type;

    public TypeAlphaConditions(int index, ActiveType.Idx type,
            Set<AlphaConditionHandle> alphaConditions) {
        this.mask = Mask.alphaConditionsMask().set(alphaConditions);
        // ...
    }
}
```

当一个会话中有 N 个规则、M 个事实类型，但很多规则共享相同的条件时，这种设计大幅减少了重复评估。

### Beta 节点按复杂度排序评估

`BetaEvaluator` 在构造时对内部条件按复杂度排序：

```java
// org.evrete.runtime.evaluation.BetaEvaluator
public BetaEvaluator(Mask<FactType> typeMask,
        Set<LhsConditionDH<FactType, ActiveField>> conditions) {
    List<LhsConditionDH<FactType, ActiveField>> sortedConditions =
        new ArrayList<>(conditions);
    sortedConditions.sort((c1, c2) -> {
        int cmp = Double.compare(
            c1.getCondition().getComplexity(),
            c2.getCondition().getComplexity());
        if (cmp == 0) {
            return Integer.compare(
                c1.getDescriptor().length(),
                c2.getDescriptor().length());
        }
        return cmp;
    });
    // ...
}
```

低成本条件先评估，尽早过滤不匹配的组合——这是 RETE 算法教科书中推荐的优化策略。

### 局限/风险

- **缺少索引网络** — 经典的 RETE 实现会维护 Beta 内存的索引（如哈希索引或树索引），Evrete 没有实现节点级别的索引
- **缺少否定条件节点（NCC）** — 经典 RETE 包含 `not` 和 `exists` 节点类型，Evrete 的 `BetaEvaluator` 用通用的条件评估器处理，可能在大规模数据下效率降低
- **`CombinationIterator` 的笛卡尔积是全量生成** — 对于大型事实集合，这可能成为性能瓶颈

### 亮点评分：4/5

两阶段节点分离是优雅且正确的抽象，Alpha 内存桶共享和 Beta 节点按复杂度排序说明了对 RETE 算法性能特征的深入理解。扣 1 分因为缺少索引网络实现，在大数据量场景下可能不足以达到最优性能。

---

## 5. 自定义数据结构

### LongKeyMap

从零实现的 long→value 哈希表，使用链表法解决冲突，支持扩容/缩容：

```java
// org.evrete.collections.LongKeyMap
public class LongKeyMap<T> implements Iterable<T> {
    private static final int INITIAL_CAPACITY = 16;
    private static final float LOAD_FACTOR = 0.75f;
    private static final float SHRINK_FACTOR = 0.25f;

    private Entry<T>[] table;
    private final Object lock = new Object();

    public T put(long key, T value) {
        synchronized (lock) {
            // 链表法冲突解决
            // 动态扩容（>LOAD_FACTOR）和缩容（<SHRINK_FACTOR）
        }
    }
}
```

**动机**：避免 `HashMap<Long, V>` 的自动装箱开销。在 RETE 网络内部，事实句柄（FactHandle）的 ID 是 `long` 类型，规则引擎需要频繁按 ID 查找事实。`long` 原始类型哈希+直接索引计算避免了每次 `Long.valueOf()` 的对象分配。

### ForkingArray + ForkingMap

Copy-on-Write 变体，专为知识库分叉设计：

```java
// org.evrete.util.ForkingMap
public class ForkingMap<K, V> {
    private final Map<K, V> map = new HashMap<>();
    private final ForkingMap<K, V> parent;

    public ForkingMap<K, V> nextBranch() {
        return new ForkingMap<>(this);  // 父/子链
    }

    public V get(K key) {
        V found = map.get(key);
        if (found == null && parent != null) {
            return parent.get(key);  // 委派给父节点
        }
        return found;
    }
}
```

一个 `Knowledge` 可以创建多个 `RuleSession`。通过 Forking 结构，会话共享未修改的父节点数据，只在差异部分持有自己的副本——实现了**写时隔离，读时共享**。

### 工程复杂度评估

| 数据结构 | 行数 | 与标准库替代对比 | 合理性 |
|---|---|---|---|
| `LongKeyMap` | ~180 | `HashMap<Long, V>` + 装箱 | 合理（热路径原始类型特化） |
| `ForkingArray` | ~200 | `CopyOnWriteArrayList` | 合理（父/子委派语义不同） |
| `ForkingMap` | ~60 | `HashMap` 浅拷贝 | 合理（分叉语义需要） |
| `ForkingArrayMap` | ~25 | `HashMap` | 边缘（功能简单） |
| `ArrayMap` | ~80 | `ArrayList` 的 KV 对 | 边缘（可被 `LinkedHashMap` 替代） |
| `IndexingArrayMap` | ~70 | `LinkedHashMap` | 可替代 |

### 局限/风险

- 自建数据结构缺少全面的边界测试（只有 `ForkingArrayTest`、`LongKeyMapTest` 两个测试文件）
- `LongKeyMap` 的哈希函数 `Integer.hashCode((int)key)` 丢失高位信息，在 key 分布集中于高 32 位时哈希碰撞严重
- `LongKeyMap` 的 `iterator()` 不是快照迭代器，遍历过程中若发生结构变更行为未定义

### 亮点评分：4/5

`LongKeyMap` 和 `ForkingArray` 的创建有明确的领域动机，父/子委派是企业级规则引擎中验证过的成熟模式。扣 1 分因为 `ArrayMap` 和 `IndexingArrayMap` 的收益/成本比不高，它们原本可以用标准库实现。

---

## 6. DSL 与注解系统

### 注解设计

8 个注解覆盖了规则定义的全部要素：

```java
// evrete-dsl-java/src/main/java/org/evrete/dsl/annotation/
@RuleSet("my.rules")           // 规则容器类
@Rule("high-value-order")      // 规则定义，支持 name + salience
@Where("$customer.credit > 1000")  // 条件表达式
@Fact("$customer")             // 参数绑定
@FieldDeclaration              // 虚拟字段声明
@MethodPredicate               // 方法引用的条件
@PredicateImplementation       // 自定义谓词
@EventSubscription             // 事件绑定
```

最有表现力的设计是 `@Where` + `@Fact` 的组合——通过方法参数声明事实绑定，而非外部配置文件：

```java
@Rule("high-value-order")
@Where("$customer.credit > 1000")
public void processHighValue(@Fact("$customer") Customer c) { ... }
```

这是比 Drools DRL 更接近 Java 原生表达的方式：没有独立的 DSL 语言，用注解表达规则语义。

### 动态编译管道

从 `@Where` 字符串到可执行的字节码，路径如下：

```java
// org.evrete.runtime.compiler.DefaultLiteralSourceCompiler

// 1. 尝试去除空白编译，失败则保留空白重试
public <S extends RuleLiteralData<R, C>, R extends Rule, C extends LiteralPredicate>
Collection<RuleCompiledSources<S, R, C>> compile(
        RuntimeContext<?> context, ClassLoader classLoader, Collection<S> sources) {
    String stripFlag = configuration.getProperty(SPI_LHS_STRIP_WHITESPACES);
    if (stripFlag == null) {
        try {
            return compile(context, classLoader, sources, true);  // 先试去空白
        } catch (CompilationException e) {
            return compile(context, classLoader, sources, false); // 保留空白重试
        }
    } else {
        return compile(context, classLoader, sources,
            Boolean.parseBoolean(stripFlag));
    }
}

// 2. 使用 javax.tools.JavaCompiler 进行内存编译
SourceCompiler compiler = context.getService().getSourceCompilerProvider()
    .instance(classLoader);
Collection<SourceCompiler.Result<RuleSource<S, R, C>>> result =
    compiler.compile(javaSources);
```

编译管道的默认策略（先去空白失败再重试）展示了务实的工程思维——统计上大多数条件表达式可以安全去空白，这优化了最常见路径的性能。

### StringLiteralEncoder —— 轻量级预处理

在没有 ANTLR 等解析器生成器的情况下，这是处理字符串常量的巧妙方案：

```java
// org.evrete.runtime.compiler.StringLiteralEncoder
// 输入: $c.status.equals("high priority") && $c.score > 100
// 编码后: $c.status.equals(${const0})&&$c.score>100
// 还原: ... "high priority"
```

占位符 `${constN}` 的设计保证了在空白移除过程中字符串内容不会被破坏。

### 局限/风险

- **条件表达式无编译时校验** — `@Where` 字符串是运行时编译的，类型错误直到 `compile()` 时才能发现
- **IDE 无支持** — `@Where` 中的变量名（`$customer`）没有 IDE 补全或跳转
- **编译失败诊断困难** — 生成的 Java 源码包含编译器生成的类和方法签名，编译错误信息的行号指向生成代码而非原始注解
- **`MethodPredicate` 注解的用法有些笨拙** — 需要分别声明条件和方法实现，比直接使用 lambda 更冗长

### 亮点评分：4/5

注解设计的表达力强、接近 Java 原生体验。编译管道的"先去空白失败重试"策略务实。扣 1 分因为运行时的类型错误诊断体验不够好。

---

## 7. 运行时能力

### 热部署/Delta 更新

`AbstractRuleSessionDeployment.deployRules()` 实现了运行时规则热部署：

```java
// org.evrete.runtime.AbstractRuleSessionDeployment
void deployRules(List<KnowledgeRule> descriptors, boolean hotDeployment) {
    // 1. 分配 alpha 内存
    CompletableFuture<Void> memoryAllocation = malloc(descriptors);

    // 2. 将知识规则转换为会话规则
    CompletableFuture<List<SessionRule>> convertedRules = memoryAllocation
        .thenCompose(v -> CommonUtils.completeAndCollect(
            descriptors, this::deploySingleRule));

    // 3. 如果是热部署，执行 beta 节点的 delta 对齐
    CompletableFuture<List<SessionRule>> deployedRules = convertedRules
        .thenCompose(rules -> allocateBetaNodes(rules, hotDeployment));

    // 4. 排序并追加
    CompletableFuture<Void> finish = deployedRules
        .thenAccept(rules -> ruleStorage.addAllAndSort(rules, getRuleComparator()));

    finish.join();
}
```

`DeltaMemoryMode` 枚举控制两种模式：

```java
// org.evrete.runtime.DeltaMemoryMode
public enum DeltaMemoryMode {
    DEFAULT,          // 冷部署：直接添加规则
    HOT_DEPLOYMENT    // 热部署：额外对齐 beta 内存
}
```

热部署模式通过 `SessionRule.getLhs().buildDeltas(DeltaMemoryMode.HOT_DEPLOYMENT)` 让新规则的 Beta 节点与已存在的事实对齐，无需销毁重建会话。

### 事件系统与可观测性

`EventMessageBus` 使用 `BroadcastingPublisher` 实现分层事件传播：

```java
// org.evrete.runtime.EventMessageBus
public class EventMessageBus implements Copyable<EventMessageBus> {
    private final Map<Class<? extends ContextEvent>, Handler<?>> handlers;

    public EventMessageBus(Executor executor) {
        this.handlers = new HashMap<>();
        register(KnowledgeCreatedEvent.class, new KnowledgeCreatedEventHandler(executor));
        register(SessionCreatedEvent.class, new SessionCreatedEventHandler(executor));
        register(SessionClosedEvent.class, new SessionClosedEventHandler(executor));
        register(SessionFireEvent.class, new SessionFireEventHandler(executor));
        register(EnvironmentChangeEvent.class, new EnvironmentChangeEventHandler(executor));
    }
}
```

`BroadcastingPublisher` 通过 `Hierarchy` 实现事件沿上下文链传播——在 `RuleSession` 上创建的事件通知也会向上传播到 `Knowledge` 的订阅者：

```java
// org.evrete.util.BroadcastingPublisher
public void broadcast(E event) {
    // 从当前节点沿 Hierarchy 向上遍历，逐层广播
    this.innerSubscriptions.walkUp(
        subscriptions -> subscriptions.broadcast(event, executor));
}
```

### 异步 fire 循环

`fireCycle()` 使用 `CompletableFuture` 链式编排，将 delta 计算、议程执行、内存提交串行化：

```java
// org.evrete.runtime.AbstractRuleSession
private CompletableFuture<Void> fireCycle(
        ActivationContext ctx, ActivationMode mode,
        WorkMemoryActionBuffer actions) {
    if (actions.hasData()) {
        return ctx.computeDelta(actions)
            .thenCompose(deltaStatus -> {
                WorkMemoryActionBuffer newActions =
                    doAgenda(ctx, deltaStatus.getAgenda(), mode);
                return ctx.commitMemories(deltaStatus)
                    .thenCompose(unused ->
                        fireCycle(ctx, mode, newActions)); // 递归直到无新动作
            });
    } else {
        return CompletableFuture.completedFuture(null);
    }
}
```

### 局限/风险

- **热部署的 Delta 模式目前只有 HOT_DEPLOYMENT 一个变体** — 在复杂场景下（规则删除、事实类型变更），Delta 对齐可能会较慢
- **事件类型不完整** — 缺少规则级事件（如 `RuleFireEvent`、`ConditionMatchEvent`），精细化的可观测性不够
- `DelegatingExecutorService` 的线程命名前缀固定为 `evrete-thread-`，在多知识库场景下无法区分线程归属

### 亮点评分：4/5

热部署机制的实现是规则引擎中最难的问题之一，Delta 对齐的设计正确。异步 fire 循环的编排清晰。扣 1 分因为事件系统的粒度和热部署的变体数量还有提升空间。

---

## 8. 兼容性与适配

### JSR94 适配器

`evrete-jsr94` 模块实现了 JSR-94 标准的所有接口，但每个实现类都很薄：

```java
// org.evrete.jsr94.RuleServiceProviderImpl
public class RuleServiceProviderImpl extends RuleServiceProvider {
    private static final KnowledgeService service = new KnowledgeService();
    private final RuleSetRegistrations registrations = new RuleSetRegistrations();

    @Override
    public RuleRuntime getRuleRuntime() {
        return new RuleRuntimeImpl(registrations);
    }

    @Override
    public RuleAdministrator getRuleAdministrator() {
        return new RuleAdministratorImpl(service, registrations);
    }
}
```

适配器层的"薄"说明核心 API 已经有足够的表达能力，不需要为了 JSR94 兼容性而扭曲内部设计。这是一种高质量的适配器模式应用。

### API 演进的后向兼容

`DSLKnowledgeProvider` 展示了精心的 API 演进管理。旧 API `create(KnowledgeService, URL...)` 标记为 `@Deprecated`，通过 `default` 方法自动委托到新 API：

```java
// org.evrete.api.spi.DSLKnowledgeProvider
@Deprecated
default Knowledge create(KnowledgeService service, URL... resources)
        throws IOException {
    // 回退到新 API 实现
    Knowledge knowledge = service.newKnowledge();
    for (URL resource : resources) {
        this.appendTo(knowledge, resource);
    }
    return knowledge;
}
```

这样做的好处：
1. 旧接口的调用者无需修改代码
2. 旧接口的实现者仍然可用（虽然有警告）
3. 新接口的实现者不需要关心旧接口

另一个例子是 `RuleSession.addEventListener()`：
```java
// org.evrete.api.RuleSession
@Deprecated
default S addEventListener(SessionLifecycleListener listener) {
    throw new UnsupportedOperationException(
        "The library has moved from an Observer to a PubSub pattern. ...");
}
```
这里使用 `throw UnsupportedOperationException` 做了"软断兼容"——调用者会立刻知道需要迁移，而非静默失效。

### 局限/风险

- JSR94 适配器缺少 `RuleExecutionSetProvider` 从 XML/URL 解析规则的能力
- `RuleServiceProviderImpl` 使用了 `static` 的 `KnowledgeService`，在多应用场景下可能共享非预期的状态
- `@Deprecated` 方法抛出 `UnsupportedOperationException` 的方式虽然不静默，但也打破了 LSP（里氏替换原则）——调用者不能无差别替换接口版本

### 亮点评分：4/5

JSR94 适配器的"薄"是高质量适配器的最佳实践。API 演进策略展示了对向后兼容的认真态度。扣 1 分因为非预期共享状态的风险和"软断兼容"的方式。

---

## 9. 工程实践

### 测试覆盖

测试按模块和功能组织，覆盖了核心路径：

```
evrete-core/src/test/java/org/evrete/
├── api/                  — API 接口测试
├── collections/          — LongKeyMapTest, ForkingArrayTest
├── runtime/              — 核心 10 个测试文件
│   ├── StatefulInsertTests     — 插入逻辑
│   ├── StatelessInsertTests    — 无状态会话
│   ├── HotDeployment*Tests     — 热部署（2 个文件）
│   ├── TypeSystemTests         — 类型系统
│   ├── MaskTest                — 位集测试
│   └── SessionAsyncTests       — 异步测试
├── spi/minimal/          — SPI 实现测试
└── util/                 — 工具类测试
```

测试数量：`evrete-core` 344 个，`evrete-dsl-java` 99 个，`evrete-jsr94` 28 个。

### 代码组织惯例

**可见性控制**是 Evrete 代码质量的亮点：

- `org.evrete.runtime` 中的所有类都是 `package-private`（即**不**声明 `public`）
- 只有 API 包（`org.evrete.api`）和少量顶层类（`KnowledgeService`, `Configuration`）是 `public` 的
- 抽象基类（`AbstractRuntime`, `AbstractRuleSession` 等）提供模板方法，子类只填充具体逻辑

这种"最小公开表面"的设计意味着核心实现可以被自由重构而不影响用户代码。

**包结构遵循功能聚合**：
- `org.evrete.runtime.rete` — RETE 网络节点
- `org.evrete.runtime.compiler` — 条件编译
- `org.evrete.runtime.evaluation` — 评估器
- `org.evrete.runtime.events` — 事件实现
- `org.evrete.spi.minimal.compiler` — 编译器的 SPI 实现
- `org.evrete.collections` — 自建数据结构

### 预计算哈希的一致性

`PreHashed` 和 `AbstractIndex` 确保 RETE 节点在 HashMap 中的查找性能：

```java
// org.evrete.runtime.PreHashed
public abstract class PreHashed {
    private final int hash;
    protected PreHashed(int hash) { this.hash = hash; }
    @Override
    public final int hashCode() { return hash; }
}

// org.evrete.util.AbstractIndex
public abstract class AbstractIndex extends PreHashed implements Indexed {
    private final int index;
    protected AbstractIndex(int index, int hashCode) {
        super(hashCode);
        this.index = index;
    }
}
```

哈希码在构造时预计算并缓存为 `final` 字段。这对于规则引擎这类频繁进行 HashMap 查找的系统是有意义的优化。

### 局限/风险

- **存在 flaky test** — `HotDeploymentStatefulTests.testMixedMulti` 在多测试组合时偶现失败，独立运行通过，说明测试间有状态泄漏
- **测试不够隔离** — 部分测试依赖静态状态，无法完全并行化
- `evrete-benchmarks` 模块缺少配套的文档说明测试场景和结果分析
- **缺少微基准测试** — `evrete-benchmarks` 的 JMH 测试未覆盖单个运行的详细指标

### 亮点评分：4/5

可见性控制和包结构体现了良好的工程纪律。测试覆盖了核心路径。扣 1 分因为 flaky test 和测试隔离性问题。

---

## Top 5 设计亮点

| # | 亮点 | 理由 | 核心代码 |
|---|---|---|---|
| 1 | **两阶段 RETE 节点分离** | `ReteGraph.transform()` 在编译时和安全运行之间划出清晰的边界，使知识重用和热部署成为可能。这是整个引擎架构的基石。 | `ReteGraph.java`, `ReteKnowledgeNode.java`, `ReteSessionNode.java` |
| 2 | **零外部依赖的核心** | 200+ 个 Java 源文件的规则引擎无任何外部依赖（仅 JDK），展示了自包含软件工程的极致。异步通过 `CompletableFuture`、SPI 通过 `ServiceLoader`，都是标准库功能。 | `pom.xml`（core 模块的 dependencies） |
| 3 | **自引用泛型 + Builder 对称性** | `RuleSession<S extends RuleSession<S>>` 结合 `RuleSetBuilder<C extends RuntimeContext<C>>` 的 `.build()` 返回 `C`，实现了流式 API 的类型安全。 | `RuleSession.java`, `RuleSetBuilder.java`, `RuntimeContext.java` |
| 4 | **SPI 驱动的可扩展架构** | 4 组 SPI 扩展点通过 ServiceLoader 发现，配合 `OrderedServiceProvider` 优先级排序和 `DefaultMethod` 驱动的 API 演进，实现了无需 DI 框架的模块化。 | `DSLKnowledgeProvider.java`, `OrderedServiceProvider.java` |
| 5 | **Forking 数据结构的领域特化** | `ForkingArray` + `ForkingMap` 的父/子委派模式专为知识库分叉设计，`LongKeyMap` 的原始类型特化避免了热路径上的装箱开销。这些自建数据结构都有明确的领域动机。 | `ForkingArray.java`, `ForkingMap.java`, `LongKeyMap.java` |
