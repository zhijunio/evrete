# Evrete 规则引擎亮点总结

> 这份文档聚焦 Evrete 在设计和实现层面值得学习借鉴的地方，可作为代码评审和架构设计的参考。
>
> 每个亮点后附有 **跨场景应用** 建议，帮助在其他领域复用这些设计模式。

## 架构设计亮点

### 1. 零外部依赖的核心引擎

`evrete-core` 的 `pom.xml` 在 `<dependencies>` 中只有 JUnit 5（test scope）。200+ 个 Java 源文件的规则引擎无任何外部依赖，仅靠 JDK 标准库完成所有功能：

```
异步编排 → java.util.concurrent.CompletableFuture
SPI 发现 → java.util.ServiceLoader
动态编译 → javax.tools.JavaCompiler
事件传播 → 自建 BroadcastingPublisher
日志      → java.util.logging
```

**收益**：无传递依赖冲突、无框架锁定、类加载器隔离简单、JAR 体积 ~300KB、可嵌入 Android/OSGi/GraalVM。

**推演**：这个选择是有代价的——必须自建事件总线、SPI 发现等基础设施。但从规则引擎的嵌入场景看，零依赖的收益远高于成本。这也倒逼了内部代码的高质量——不能依赖第三方框架来弥补设计缺陷。

**跨场景应用**：嵌入式库（如模板引擎、表达式求值器、轻量级工作流引擎）、SDK 客户端库、Android 库、GraalVM Native Image 目标、IoT/边缘计算运行时。判断标准：如果用户需要在他们的项目中"加一个依赖就能用"，不希望被传递依赖"污染"类路径，零依赖就是正确选择。反例：需要集成 Spring/Quarkus 生态的应用层面项目，使用成熟框架比自建基础设施更务实。

### 2. 清晰的三层分层：API / SPI / Runtime

源码包结构映射了教科书级别的分层设计：

```
org.evrete.api        — 纯接口，43 个文件。定义 Knowledge, RuleSession, Rule, FactHandle, Type 等全量契约
org.evrete.api.spi    — 4 组扩展点接口：DSLKnowledgeProvider, MemoryFactoryProvider, TypeResolverProvider, SourceCompilerProvider
org.evrete.runtime    — 私有实现，全部 package-private 可见性
```

```java
// 典型模式
// API:  org.evrete.api.Knowledge
// SPI:  org.evrete.api.spi.MemoryFactoryProvider
// Impl: org.evrete.spi.minimal.DefaultMemoryFactoryProvider (package-private)
```

好在哪里：
- **最小公开表面** — `org.evrete.runtime` 中所有类都是 package-private，核心实现可自由重构而不影响用户代码
- **分离编译时与运行时关注点** — API 定义契约，SPI 定义扩展点，Runtime 实现具体算法
- **每个包的职责可独立理解** — 看 API 包就知道功能边界，看 SPI 包就知道可扩展方向

**跨场景应用**：任何需要"提供 API 让用户集成，但实现细节可能频繁变更"的库。例如 ORM 框架（API=Session/Query，SPI=Dialect/ConnectionProvider，Runtime=具体持久化实现）、配置中心客户端（API=ConfigSource，SPI=PropertySourceLoader，Runtime=不同后端实现）、消息队列客户端（API=Producer/Consumer，SPI=Serializer/Partitioner，Runtime=网络协议实现）。三层分界的核心信号是：当实现逻辑的变更频率和方向与接口契约不同时，就需要物理隔离。

### 3. 两阶段 RETE 节点分离

Evrete RETE 实现最突出的设计决策：将节点分为知识阶段（知识编译时）和会话阶段（运行时），通过 `ReteGraph.transform()` 连接。

```
知识编译阶段（Knowledge）     运行阶段（Session）
  ReteKnowledgeNode             ReteSessionNode
  ReteKnowledgeConditionNode    ReteSessionConditionNode
  ReteKnowledgeEntryNode        ReteSessionEntryNode
```

```java
// org.evrete.runtime.rete.ReteGraph
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
```

好在哪里：
- **编译时关注结构，运行时关注状态** — 知识阶段只需构建条件结构，不需要考虑内存分配
- **分离使热部署成为可能** — 新规则在知识阶段构建 RETE 子图，然后 transform 并 merge 到运行中的会话
- **类型安全** — `ReteGraph<B, E, C>` 的三个类型参数允许多种节点类型共享同一个图结构

**跨场景应用**：DSL 编译管道（AST → IR → 可执行代码，各阶段节点类型不同但结构相同）、查询引擎（逻辑计划 → 物理计划 → 执行，同一种优化器结构处理不同阶段的节点类型）、UI 组件树（声明式组件树 → 虚拟 DOM → 真实 DOM，transform 连接不同阶段）。这是"多阶段编译"的通用模式——用一个同构的图结构，通过 transform 映射不同阶段的节点类型，避免为每个阶段手写遍历器。

### 4. Forking 数据结构的领域特化

Evrete 自建了一套 Forking（分叉）数据结构，用于知识库与会话之间的写时隔离、读时共享。

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

`ForkingArray` 实现了同样的父/子委派语义，但基于数组索引：

```java
// ForkingArray 内部
public V get(int index) {
    int adjustedIndex = index - this.dataOffset;
    if (adjustedIndex < 0) {
        // 此索引属于父层级
        return parent == null ? null : parent.get(index);
    } else if (adjustedIndex >= this.nextWriteIndex) {
        return null;
    } else {
        return (V) array[adjustedIndex];
    }
}
```

好在哪里：
- **写时隔离，读时共享** — 一个 `Knowledge` 可以创建多个 `RuleSession`，共享未修改的父数据，只在差异部分持有副本
- **无锁读取** — 父节点数据不变，不需要同步
- **空间高效** — 不需要为每个 Session 复制全量数据

**跨场景应用**：版本控制系统（Git 的 commit DAG 本质就是 Forking 数据结构的树形父/子委派——每个分支共享祖先提交的数据，只在差异持有副本）、不可变配置系统的快照机制（每个请求拿到一份配置"视图"，修改不污染全局）、IDE 的 undo/redo 栈（每个操作生成一个 Forking 分支，回退时委派到父状态）、事务隔离（MVCC 的快照读 + 写时复制）。Forking 模式适用条件是：有明确的分叉点、读远多于写、父状态不可变。

`LongKeyMap` 则是另一类领域特化：原始类型 `long→V` 的哈希表，避免 `HashMap<Long, V>` 的热路径装箱开销。支持扩容和缩容，使用 `synchronized` 块级锁保证线程安全。

**跨场景应用**：任何以数字 ID 为主键的热路径查找。例如网络框架的连接 ID → Channel 映射、游戏引擎的 Entity ID → Component 映射、时间序列数据库的时间戳 → 数据点映射、内存缓存的哈希分片。判断标准：用 profiler 确认装箱出现在热点路径上再特化，过早优化不如直接用 `HashMap` + 良好的哈希分布。

### 5. SPI 三层回退 + 优先级排序

`KnowledgeService.Builder` 的 `loadCoreSPI()` 方法是 SPI 发现的优雅包装：

```java
// org.evrete.KnowledgeService.Builder
private <Z extends OrderedServiceProvider, I extends Z>
Z loadCoreSPI(Class<Z> clazz, String propertyName, Class<I> implClass) {
    // 1. 显式类参数（最高优先级）
    if (implClass != null) {
        return implClass.getConstructor().newInstance();
    }
    // 2. 配置属性名
    String className = conf.getProperty(propertyName);
    if (className != null) {
        return (Z) Class.forName(className).getConstructor().newInstance();
    }
    // 3. ServiceLoader 回退
    List<Z> providers = new LinkedList<>();
    ServiceLoader.load(clazz).iterator().forEachRemaining(providers::add);
    Collections.sort(providers);
    return providers.iterator().next();
}
```

配合 `OrderedServiceProvider` 接口，多个提供者共存时按优先级自动选择：

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

好在哪里：
- **三路回退** — 显式指定 > 配置属性 > 自动发现，每个级别有明确的后备
- **无 DI 框架的 SPI 管理** — `ServiceLoader` + `OrderedServiceProvider` 足够覆盖大多数场景
- **清晰的错误消息** — 找不到时抛出 `IllegalStateException`，列出所有可用提供者

**跨场景应用**：日志框架（用户显式指定 Logger 实现 → 系统属性指定 → ServiceLoader 自动发现）、序列化库（显式注册 Serializer → 配置文件声明 → classpath 扫描）、数据库连接池（应用代码指定 → 环境变量 → 自动检测可用实现）。三层回退的精髓在于：每一层的优先级是自解释的，而不是用复杂的优先级规则让用户猜。如果只有两层（显式 vs 自动），通常就够用了。

## API 设计

### 6. 自引用泛型实现类型安全链式调用

```java
// org.evrete.api.RuleSession
public interface RuleSession<S extends RuleSession<S>>
    extends RuleSetContext<S, RuntimeRule>, SessionOps, MemoryStreaming {

    default S insert(Object... objects) {
        insert0(objects, true);
        return (S) this;  // 返回正确的子类型
    }
}
```

这使得 `StatefulSession` 和 `StatelessSession` 各自返回自身类型：

```java
StatefulSession s = service.newStatefulSession();
s.insert(fact1)        // returns StatefulSession
 .insert(fact2)        // returns StatefulSession
 .fire();              // returns StatefulSession.fire()

StatelessSession s2 = service.newStatelessSession();
s2.insert(fact1)       // returns StatelessSession
  .insert(fact2);      // returns StatelessSession
```

同样，`RuntimeContext<C extends RuntimeContext<C>>` 统一了 `Knowledge` 和 `RuleSession`，使得 `RuleSetBuilder` 的 `.build()` 返回构造时的上下文类型：

```java
// org.evrete.api.builders.RuleSetBuilder
public interface RuleSetBuilder<C extends RuntimeContext<C>>
    extends FluentEnvironment<RuleSetBuilder<C>> {
    RuleBuilder<C> newRule(String name);
    C build();  // 返回构造时的上下文类型
}
```

好在哪里：
- **同一 Builder 构造 Knowledge 或 RuleSession** — API 对称，学习成本低
- **类型安全** — 编译器能推导出正确的返回类型
- **在 JDK 8 下能达到的类型安全上限** — `@SuppressWarnings("unchecked")` 是 Java 类型系统的已知局限，项目在这个约束下做到了最好

**跨场景应用**：任何需要链式调用的 fluent API。例如：HTTP 客户端（`RequestBuilder<C extends RequestBuilder<C>>` + `GetBuilder extends RequestBuilder<GetBuilder>` → `client.get(url).header("k","v").send()` 返回 `GetResponse`）、查询构建器（`QueryBuilder<C extends QueryBuilder<C>>` + `SelectBuilder extends QueryBuilder<SelectBuilder>` → `select("name").from("users").where(...)`）、配置构建器（`ConfigBuilder<C extends ConfigBuilder<C>>` + `DataSourceConfig extends ConfigBuilder<DataSourceConfig>`）。这个模式的复杂度与收益成正比——如果只有一种子类型，`return (T) this` 比自引用泛型简单得多。

### 7. 接口默认方法驱动的 API 演进

`DSLKnowledgeProvider` 展示了精心的 API 演进管理。旧 API `create(KnowledgeService, URL...)` 标记为 `@Deprecated`，通过 `default` 方法自动委托到新 API：

```java
// org.evrete.api.spi.DSLKnowledgeProvider
@Deprecated
default Knowledge create(KnowledgeService service, URL... resources)
        throws IOException {
    Knowledge knowledge = service.newKnowledge();
    for (URL resource : resources) {
        this.appendTo(knowledge, resource);  // 委托到新 API
    }
    return knowledge;
}
```

好在哪里：
- **旧接口调用者无需修改代码** — `default` 方法自动适配
- **旧接口实现者仍然可用** — 虽然有编译警告，但兼容
- **新接口实现者不需要关心旧接口** — 只需要实现 `appendTo(RuntimeContext, Object)`

**跨场景应用**：任何需要多版本 API 共存的库。例如 HTTP 框架的 Handler 接口演进（`handle(Request, Response)` → `handle(Request)`，用 default 方法把旧签名委托给新签名）、数据库迁移工具（`Migration.migrate(Connection)` → `migrate(DataSource)`，default 方法创建 Connection 后委托）。这是"Strangler Fig 模式"的代码级应用——旧接口不动，新接口生长，通过 default 方法桥接，最终旧接口可以安全移除。

需要硬断的场景则用 `throw UnsupportedOperationException`：

```java
// org.evrete.api.RuleSession
@Deprecated
default S addEventListener(SessionLifecycleListener listener) {
    throw new UnsupportedOperationException(
        "The library has moved from an Observer to a PubSub pattern. ...");
}
```

这种"软断兼容"的设计方式，让调用者立刻知道需要迁移，而非静默失效。

**跨场景应用**：框架层面的行为变更（例如从同步到异步、从回调到响应式）。`@Deprecated` + default 委托适合"加的"变更（增加新方法），`@Deprecated` + throw UnsupportedOperationException 适合"减的"变更（删除旧行为）。前者的迁移是透明的，后者需要用户介入——两者都不应该静默。

### 8. 注解驱动的 Java DSL

Evrete 的注解 DSL 是接近 Java 原生表达的方式——没有独立的 DSL 语言，用注解表达规则语义：

```java
// evrete-dsl-java
@RuleSet("my.rules")
public class MyRules {

    @Rule("high-value-order")
    @Where("$customer.credit > 1000")
    public void processHighValue(@Fact("$customer") Customer c) {
        // 规则动作
    }

    @Rule("complex-order")
    @Where({"$order.amount > 100.0", "$order.status == 'pending'"})
    public void processComplex(@Fact("$order") Order o) { ... }
}
```

与 Drools DRL 的对比：

```
Drools DRL:
  rule "high-value-order"
    when
      $customer: Customer(credit > 1000)
    then
      // ...
  end

Evrete 注解:
  @Rule("high-value-order")
  @Where("$customer.credit > 1000")
  void processHighValue(@Fact("$customer") Customer c) { ... }
```

好在哪里：
- **纯 Java** — IDE 语法高亮、重构、代码导航都可用
- **学习成本低** — 不需要学独立的 DSL 语法
- **编译时类型安全** — 方法参数有类型声明，IDE 可以检查

**跨场景应用**：配置框架（`@ConfigPrefix("app.db") @Key("url") void setDbUrl(@Value("${db.url}") String url)`——用注解 + 方法参数声明配置绑定，而不是 YAML 字符串）、工作流引擎（`@Workflow("order") @Step("validate") @Condition("$order.amount > 0") void validate(@Fact("$order") Order o)` — 用注解表达工作流步骤的条件）、校验框架（`@Validate @Rule("age >= 18") @Message("Too young") void checkAge(@Field("age") int age)`）。"注解 + 方法"作为 DSL 的关键收益是 IDE 支持零成本，关键成本是失去动态性——规则在代码编译期固定。

### 9. StringLiteralEncoder 的轻量级预处理

在没有 ANTLR 等解析器生成器的情况下，`StringLiteralEncoder` 是处理字符串常量的巧妙方案：

```java
// 输入: $c.status.equals("high priority") && $c.score > 100
// 编码后: $c.status.equals(${const0})&&$c.score>100
// 还原: ... "high priority"
```

```java
// org.evrete.runtime.compiler.StringLiteralEncoder
static StringLiteralEncoder of(String s, boolean stripWhiteSpaces) {
    // 先替换所有字符串常量为占位符 ${constN}
    for (char quote : QUOTES) {  // 处理 ', ", `
        // ... 查找引号对，替换为占位符
    }
    // 然后安全地移除空白
    if (stripWhiteSpaces) {
        current = current.replaceAll("\\s", "");
    }
    return new StringLiteralEncoder(s, new Encoded(current), stringConstantMap);
}
```

编译管道的默认策略是"先去空白，失败重试"——统计上大多数条件表达式可以安全去空白，这优化了最常见路径的性能：

```java
// org.evrete.runtime.compiler.DefaultLiteralSourceCompiler
String stripFlag = configuration.getProperty(SPI_LHS_STRIP_WHITESPACES);
if (stripFlag == null) {
    try {
        return compile(context, classLoader, sources, true);   // 先试去空白
    } catch (CompilationException e) {
        return compile(context, classLoader, sources, false);  // 保留空白重试
    }
}
```

好在哪里：
- **零依赖** — 不需要 ANTLR 等解析器生成器
- **正确的设计** — 先保护字符串常量，再做空白移除，顺序保证正确性
- **务实的性能优化** — "先试去空白"策略覆盖了最常见的路径

**跨场景应用**：任何需要预处理带字符串常量的表达式语言的场景。例如 SQL 模板引擎（先替换字符串常量为占位符，再重写参数化查询 → `WHERE name = ${const0}` → `WHERE name = ?`）、配置属性的变量插值（`${app.home}/data/${const0}` 中保护路径中的字面量）、国际化模板（先保护原文中的嵌入标签，再处理翻译变量）。关键约束：解析器只需要处理引号内文字（简单词法规则），一旦需要处理嵌套语法（字符串内的转义引号、注释等），就应该用正式的解析器。

**跨场景应用（编译策略）**：任何"大多数输入符合规范格式，少数需要宽松处理"的解析场景。例如 JSON 解析器（先按 RFC 严格模式解析，失败回退到 lenient 模式）、日期解析（先试 ISO 8601，再试常见区域格式）、CSV 解析（先试标准逗号分隔，再试引号内逗号等边界情况）。"先试严格模式"的隐含假设是检查"严格"的成本远低于"宽松"——如果严格模式检查本身就很贵，这个策略就不成立。

## 运行时能力

### 10. 热部署 / Delta 对齐

`AbstractRuleSessionDeployment.deployRules()` 实现了运行时规则热部署：

```java
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

热部署的核心在于 `allocateBetaNodes`：新规则的 Beta 节点需要与已存在的事实对齐（`buildDeltas(DeltaMemoryMode.HOT_DEPLOYMENT)`），这样新规则能立刻基于当前状态评估，无需销毁重建会话。

好在哪里：
- **`CompletableFuture` 链式编排** — 每个步骤异步执行，代码可读性强
- **热/冷部署共享同一代码路径** — `hotDeployment` 布尔参数控制行为差异
- **规则与事实分离** — 知识结构的变更不会影响已有事实的内存布局

**跨场景应用**：插件系统的热加载（新插件需要与已有运行时状态对齐，类似新规则与已有事实对齐）、微服务配置热更新（新配置规则需要在不影响当前请求的前提下与运行时状态合并）、流处理引擎的动态算子注入（新算子需要从当前状态快照开始处理，不是从头开始）。Delta 对齐模式的关键约束：需要明确知道"哪些已有状态需要与新组件对齐"，以及对齐操作本身必须是可增量执行的（而不是全量扫描）。

### 11. CompletableFuture 驱动的异步 fire 循环

`fireCycle()` 的编排展示了纯 JDK 标准库能实现的优雅异步设计：

```java
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

好在哪里：
- **清晰的阶段划分** — Delta 计算 → Agenda 执行 → 内存提交，三个阶段严格串行
- **递归直到稳定** — `fireCycle` 自我调用，直到没有新动作产生
- **无需外部线程框架** — `CompletableFuture` 的 thenCompose/thenAccept 链足够

**跨场景应用**：事件溯源系统的投射（事件 → 状态更新 → 产生新事件 → 递归直到一致）、UI 框架的变更传播（状态变更 → 重渲染 → 副作用 → 再变更 → 递归直到稳定）、编译器的优化循环（IR 优化 → 触发新优化 → 递归直到固定点）。递归直到稳定（fixpoint computation）是"增量式数据处理"的通用模式——关键设计决策是支持"空终止"：如果一轮计算没有产生新动作，计算自然结束；否则需要收敛性证明（不会无限循环）。Evrete 通过 `ActivationMode` 控制这一点：`DEFAULT` 模式一轮结束，`CONTINUOUS` 模式需要稳定性判断。

### 12. BroadcastingPublisher 的分层事件传播

`BroadcastingPublisher` 通过 `Hierarchy` 实现事件沿上下文链传播：

```
RuleSession 上广播的事件
  → 通知 RuleSession 的订阅者
  → 向上传播到 Knowledge 的订阅者
```

好在哪里：
- **Observer → PubSub 的迁移是正确方向** — 事件的粒度从接口方法级别提升到事件类型级别
- **无需第三方事件总线** — 自建实现只花了 ~100 行代码
- **分层传播** — 事件沿 Hierarchy 向上自然传播，不需要手动桥接

**跨场景应用**：任何有层次结构的事件系统。例如 UI 组件树的事件冒泡（子组件事件 → 父组件 → 根组件）、微服务调用链的 Span 传播（子 Span → 父 Span → Trace）、嵌套事务的回滚（子事务失败 → 沿嵌套链向上回滚）。分层传播的关键设计：单向性（事件只能向上或向下传播，不能双向循环）、传播终止条件（要么到达根节点，要么有订阅者标记 consumed）、不引入循环依赖。

## 测试与代码质量

### 13. 测试覆盖与可测试性

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
│   └── SessionAsyncTests       — 异步测试
└── spi/minimal/          — SPI 实现测试
```

测试数量：`evrete-core` 344 个，`evrete-dsl-java` 99 个，`evrete-jsr94` 28 个。

**测试的可读性也展示了对用户的使用引导**：

```java
service
    .newKnowledge()
    .builder()
    .newRule("rule-name")
    .forEach("$a", TypeA.class)
    .where("$a.i > 10")
    .execute(ctx -> {
        TypeA a = ctx.get("$a");
        a.setActive(true);
    })
    .newRule(...)
    .build();
```

测试代码即示例代码——顺便展示了规则 API 的正确使用方式。

**跨场景应用**：任何需要兼顾"验证正确性"和"展示用法"的库。通过在测试中写可读性高的使用示例，测试同时承担了三重职责：功能验证、使用文档、回归防护。如果测试代码本身就是 API 使用的最佳实践，新用户看测试比看文档更快上手。约束条件：测试代码需要像产品代码一样维护——有命名规范、有注释、有场景分类。

### 14. 可见性控制的最小公开表面原则

`org.evrete.runtime` 中的所有类都是 `package-private`——这是整个代码库中最一致的工程纪律之一。

```java
// runtime 中所有类都是默认可见性
class AbstractRuntime { ... }      // package-private
class RuntimeRule { ... }           // package-private
class ReteKnowledgeNode { ... }     // package-private
class BetaEvaluator { ... }         // package-private

// 只有 API 包才公开
public interface RuleSession { ... }
public interface Knowledge { ... }
public class KnowledgeService { ... }
```

好在哪里：
- **封装是契约** — 用户不可能误用内部实现
- **重构自由** — 内部实现可以完全重写而不影响用户
- **文档聚焦** — Javadoc 只需要覆盖有限的公开 API

**跨场景应用**：任何有明确"框架 vs 应用"边界的项目。大型项目最一致的退化模式就是 internal 类泄漏为 public（因为懒或因为不够重视）。包级可见性 + module-info.java（Java 9+）是 Java 生态中最强的封装手段。核心实践：新类默认 package-private，只有明确需要公开的才加 public 关键字——"公开"的默认值应该是 false。

### 15. `@NonNull` / `@Nullable` 空值契约

API 层系统性地使用自定义的 `@NonNull` 和 `@Nullable` 注解，在接口方法签名上显式声明空值合约：

```java
// org.evrete.api.RuleSession
<T> T getFact(@NonNull FactHandle handle);

// org.evrete.api.spi.DSLKnowledgeProvider
static DSLKnowledgeProvider load(@NonNull String dsl) {
    Objects.requireNonNull(dsl, "DSL identifier cannot be null");
}
```

虽然是自建注解而非 JSpecify 标准，但这种习惯本身就是值得肯定的——它让接口的调用者知道哪些参数可以传 null、哪些返回值可能为 null，比 Javadoc 中的隐晦描述可靠得多。

**跨场景应用**：任何有公共 API 的 Java 库。空值契约是 API 文档中最容易被忽略但又最容易被违反的部分。即使不使用 checkerframework/SpotBugs/JSpecify，在接口方法上标注 `@Nullable` 也远好于不标——因为它降低了调用者的心智负担。如果项目已迁移到 Java 9+，考虑直接迁移到 `org.jspecify.annotations`（JSpecify 已被 JetBrains、Gradle、Guava 等采纳），而不是维护自建注解。

### 16. JSR94 适配器的"薄"

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

好在哪里：
- **适配器层"薄"意味着核心 API 能力足够** — 不需要为了 JSR94 兼容性扭曲内部设计
- **可分离** — 如果没有 JSR94 需求，可以完全不编译这个模块
- **高质量适配器模式** — 适配器只做协议转换，不做业务逻辑

**跨场景应用**：任何需要兼容外部标准的库（JSR、ECMA、ISO 等标准接口）。适配器层的理想厚度是"薄到可以删除"——如果有一天标准废弃了，删除适配器模块不伤核心。判断适配器是否过厚的标准：适配器层的代码中是否出现了领域逻辑（路由、缓存、事务逻辑）？如果出现了，说明核心 API 缺少表达能力，应该增强核心而非堆适配器。

### 17. 预计算哈希的一致性优化

`PreHashed` 和 `AbstractIndex` 确保 RETE 节点在 HashMap 中的查找性能：

```java
// org.evrete.runtime.PreHashed
public abstract class PreHashed {
    private final int hash;
    protected PreHashed(int hash) { this.hash = hash; }
    @Override
    public final int hashCode() { return hash; }
}
```

哈希码在构造时预计算并缓存为 `final` 字段。对于规则引擎这类频繁进行 HashMap 查找的系统，这是有意义的优化——RETE 网络的 Beta 内存查找、规则匹配、类型索引都依赖哈希性能。

**跨场景应用**：任何频繁进行 `equals()`/`hashCode()` 调用的不可变对象。例如 AST 节点（编译器的语法树节点在模式匹配中频繁比较）、值对象（金额、货币、坐标等业务值对象）、缓存键（复合缓存键的构造即确定哈希）。核心约束：对象的哈希依赖字段必须在构造后不可变。如果对象可变，预计算哈希会在字段变更时返回错误值——这种情况适用缓存哈希但需要失效机制。

## 小结

Evrete 是一个设计扎实、零外部依赖的规则引擎，值得学习的地方：

1. **零依赖的极致工程** — 200+ 源码文件、完备的 RETE 实现、异步编排、动态编译，全部靠 JDK 标准库
2. **Forking 数据结构** — 写时隔离、读时共享的父/子委派模式是规则引擎领域的经典设计
3. **两阶段 RETE 节点分离** — `transform()` 连接知识阶段和会话阶段，是热部署架构的基石
4. **接口默认方法驱动的 API 演进** — `@Deprecated` + `default` 委托 + `throw UnsupportedOperationException` 的兼容策略
5. **最小公开表面** — `package-private` 贯穿整个 runtime，封装即契约
6. **自引用泛型的正确用法** — `RuleSession<S extends RuleSession<S>>` 在 JDK 8 约束下做到了类型安全链式调用的极限
7. **"薄"适配器** — JSR94 适配层的简洁说明核心 API 足够表达
8. **有领域动机的自建数据结构** — `LongKeyMap`、`ForkingArray`、`PreHashed` 都有明确的性能或语义动机
9. **跨场景复用** — 本文 17 个亮点中，每一个模式都可以在规则引擎以外的场景中找到适用位置，识别这些模式的适用条件比记住模式名称更重要。
