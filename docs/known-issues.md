# Evrete 已知问题与技术债务

> 从代码审查和静态分析中汇总的已知问题、TODO 注释、设计取舍和潜在风险。
> 分析版本：4.0.4-SNAPSHOT | 分析日期：2026-07-03

参考文档：
- [design-analysis.md](./design-analysis.md) — 架构分析与设计评价
- [design-patterns-deep-dive.md](./design-patterns-deep-dive.md) — 实现模式分析
- [data-flow-trace.md](./data-flow-trace.md) — 运行时数据流追踪

---

## 分类说明

| 严重度 | 含义 |
|---|---|
| 🔴 **Critical** | 可能导致错误行为的正确性问题 |
| 🟡 **Moderate** | 设计缺陷，可能在边界条件下暴露 |
| 🔵 **Minor** | 技术债务、清理项、缺失功能 |
| ⚪ **Cleanup** | 代码整洁度、注释补全 |

---

## 🔴 Critical

### C1. LongKeyMap 哈希函数丢弃高 32 位

`LongKeyMap.hash()` 的实现：

```java
// org.evrete.collections.LongKeyMap.java:56
return Integer.hashCode((int)key) & (table.length - 1);
```

将 `long` 类型 key 强转为 `int` 丢弃了高 32 位。当 key 分布集中于高 32 位（如使用高位 ID 生成器时），所有 key 会映射到零散低位空间，导致严重哈希碰撞，查找退化为 O(n)。

**影响范围**：`WorkMemoryActionBuffer` 内部以 `FactHandle` 的 long ID 为 key 进行查找。在长时间运行产生大量事实句柄的场景下，碰撞概率增大。

**修复方向**：替换为 `(int)(key ^ (key >>> 32))`，混合高低位。

---

### C2. 被注释质疑的条件分支（AbstractRuleSessionOps.java:132）

```java
//TODO something is wrong here. Rewrite. The checks below will never fail
Class<?> expectedFactClass = activeType.getValue().getJavaClass();
Class<?> argClass = newValue.getClass();
if (expectedFactClass.isAssignableFrom(argClass)) {
    // ...
} else {
    throw new IllegalArgumentException("Argument type mismatch...");
}
```

**问题**：作者自述"检查永远不会失败"。如果该分支无法达到，这个 `IllegalArgumentException` 是死代码。但作为 API 契约的安全网，如果未来有改动改变前置条件，缺失此检查可能导致静默的 `ClassCastException`。

**修复方向**：确认条件是否真的不可达。如果不可达，移除死代码并添加断言；如果仍视为安全网，添加注释说明意图。

---

### C3. Flaky Test：HotDeploymentStatefulTests.testMixedMulti

**症状**：在多测试组合运行时偶现失败，独立运行时总是通过。

**根因推测**：测试间存在静态状态泄漏。部分测试依赖共享的 `KnowledgeService` 实例。

**影响**：CI 可靠性下降，需多次重试。

**修复方向**：为每个测试方法创建独立的 `KnowledgeService` 实例，或添加 `@Before` 重置静态状态。

---

## 🟡 Moderate

### M1. Configuration 继承 Properties（Hashtable 历史债务）

```java
// org.evrete.Configuration
public class Configuration extends Properties
        implements Copyable<Configuration>, FluentImports<Configuration> {
```

`Properties` 继承自 `Hashtable`，意味着：
- 所有方法都是 `synchronized` 的（不必要的性能开销）
- 遗留的 `defaults` 序列化机制
- 继承了大量不需要的方法（`elements()`, `keys()` 等）

这与 Evrete 其他地方的"最小公开表面"哲学不一致。

**修复方向**：改为内部持有 `Map<String, String>` 委托，提供类型安全访问器。

---

### M2. JSR94 RuleServiceProviderImpl 使用静态 KnowledgeService

```java
// org.evrete.jsr94.RuleServiceProviderImpl
private static final KnowledgeService service = new KnowledgeService();
```

`static final` 意味着所有 JSR94 客户端共享同一个 `KnowledgeService`。在多应用部署场景（如多个 Web 应用引用同一 JAR），这可能导致非预期的配置泄漏。

**修复方向**：改为 `final` 实例字段（移除 `static`），或使用懒加载的线程本地实例。

---

### M3. @Deprecated 方法抛出 UnsupportedOperationException（违 LSP）

```java
// org.evrete.api.RuleSession
@Deprecated
default S addEventListener(SessionLifecycleListener listener) {
    throw new UnsupportedOperationException(
        "The library has moved from an Observer to a PubSub pattern...");
}
```

意图明确（强制使用者迁移），但打破了 Liskov 替换原则——调用者不能无差别替换接口版本。属于"硬兼容断裂"。

**影响**：如果用户代码捕获 `UnsupportedOperationException` 不充分，可能导致运行时异常。

---

### M4. 缺少 EvaluatorProvider SPI 扩展点

条件评估逻辑硬编码在 `BetaEvaluator` 中，没有 SPI 接口允许替换。用户如需自定义条件评估行为（如专用索引、外部规则引擎集成），无法通过标准 SPI 完成。

---

## 🔵 Minor

### N1. ForkingArray TODO — "review the whole package, or even delete it"

```java
// org.evrete.collections.ForkingArray.java:26
//TODO review the whole package, or even delete it as it's not exported
```

作者对 `ForkingArray` 的保留价值存疑。该包中 `ArrayMap`、`IndexingArrayMap`、`ForkingArrayMap` 的领域动机较弱，标准库替代方案足以满足。

**建议**：审计 `collections` 包中每个类的实际使用者，移除未使用的结构。

---

### N2. WorkMemoryActionBuffer — "the executor isn't actually used"

```java
// org.evrete.runtime.WorkMemoryActionBuffer.java:57
// TODO the executor isn't actually used
CompletableFuture<Collection<SplitView>> sinkToSplitView(ExecutorService executor) {
    return CompletableFuture.supplyAsync(this::sinkToSplitViewSync, executor);
}
```

操作本身轻量级（HashMap + 遍历），异步包装没有实质收益。

**建议**：消除异步包装，改为同步方法，简化调用链。

---

### N3. DefaultValueIndexer — "review usage, it must be used"

```java
// org.evrete.spi.minimal.DefaultValueIndexer.java:36
//TODO review usage, it must be used
```

作者对代码被调用与否存疑。可能意味着某个功能路径在测试中未被覆盖，或存在死代码。

---

### N4. DefaultFactStorage — "size config option"

```java
// org.evrete.spi.minimal.DefaultFactStorage.java:40
//TODO size config option
```

初始容量硬编码，没有配置化。大事实量场景下初始容量太小会导致频繁 rehash。

---

### N5. ReteKnowledgeEvaluator — "use a subclass of LhsConditionDH"

```java
// org.evrete.runtime.rete.ReteKnowledgeEvaluator.java:31
//TODO use a subclass of LhsConditionDH
```

当前直接实例化 `LhsConditionDH` 基类，但语义上应使用其子类。

---

### N6. evrete-benchmarks 缺少文档

`evrete-benchmarks` 模块有一个 JMH 测试（`Expressions`），比较编译后条件评估与原生 Java 调用的性能。但缺少测试场景说明和结果分析。

---

### N7. 测试隔离不足

部分测试依赖共享的 `KnowledgeService` 静态实例，导致：
- 测试无法完全并行化
- 测试间状态泄漏（flaky test 根因之一）
- 测试结果对执行顺序敏感

**影响文件**：`HotDeploymentStatefulTests`, `HotDeploymentStatelessTests`, `TypeSystemTests` 等。

---

### N8. StringLiteralEncoder 不支持 Java 15+ 文本块引号

```java
private static final char[] QUOTES = new char[]{'\'', '"', '`'};
```

只处理单引号、双引号和反引号，不支持 `"""` 文本块。如果 `@Where` 注解中使用文本块，会错误解析。

---

## ⚪ Cleanup

| 位置 | 注释 | 性质 |
|---|---|---|
| `AlphaAddress.java:16` | `//TODO check if it's ok w/o pre-hashing` | 设计确认 |
| `SessionMemory.java:202` | `//TODO fix the mess with types` | 代码混乱 |
| `TypeMemory.java:21` | `// TODO Fix the mess with class fields` | 代码混乱 |
| `ActiveField.java:36` | `//TODO add link to where the method is used` | 文档缺失 |
| `LongKeyMapTest.java:221` | `//TODO test multithreading` | 测试缺失 |
| `KnowledgeService.java` | 20 个 `@Deprecated` 方法 | API 演进遗留 |

---

## 各模块 TODO 数量统计

| 模块 | TODO 数量 |
|---|---|
| `evrete-core/runtime/` | 5 |
| `evrete-core/runtime/rete/` | 1 |
| `evrete-core/collections/` | 1 |
| `evrete-core/spi/minimal/` | 2 |
| `evrete-dsl-java/` | 0 |
| `evrete-jsr94/` | 0 |
| 测试代码 | 1 |

---

## 设计层面的缺失功能（非 Bug）

以下在 `design-analysis.md` 中已有讨论，此处汇总以便追踪：

| 功能 | 说明 |
|---|---|
| 规则级事件 | `RuleFireEvent`、`ConditionMatchEvent` 等精细事件 |
| 多 Delta 模式 | 目前只有 `HOT_DEPLOYMENT` 一个变体 |
| SPI 生命周期 | 没有初始化/销毁回调 |
| 动态 SPI 注册 | ServiceLoader 静态发现，无法运行时热注册 |
| 序列化支持 | `StatefulSession` 序列化边界未文档化 |

---

*本文基于 v4.0.4-SNAPSHOT 代码审查。列表只反映审查时发现的问题，并非完整的 bug 追踪器。*
