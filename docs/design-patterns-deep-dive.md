# Evrete 实现模式深度分析

> 参考模板：[prompt-template-design-analysis.md](./prompt-template-design-analysis.md)
> 本文聚焦实现层面的设计模式、数据结构和编程技巧，作为架构分析的补充。
> 分析版本：4.0.4-SNAPSHOT | 分析日期：2026-07-03

---

## 1. 自定义数据结构体系

### 1.1 ForkingArray + ForkingMap（父/子委派 + Copy-on-Write）

`ForkingArray` 和 `ForkingMap` 实现了同一设计哲学——父/子委派模式：

```java
// org.evrete.collections.ForkingArray
public class ForkingArray<V extends Indexed> {
    private Object[] array;
    private final ForkingArray<V> parent;
    private final int dataOffset;
    private int nextWriteIndex;

    private ForkingArray(int initialArraySize, ForkingArray<V> parent) {
        this.array = new Object[initialArraySize];
        this.parent = parent;
        this.dataOffset = parent == null ? 0
            : parent.dataOffset + parent.nextWriteIndex;
    }

    public ForkingArray<V> newBranch() {
        return new ForkingArray<>(this.initialArraySize, this);
    }
}
```

```java
// org.evrete.util.ForkingMap
public class ForkingMap<K, V> {
    private final Map<K, V> map = new HashMap<>();
    private final ForkingMap<K, V> parent;

    public ForkingMap<K, V> nextBranch() {
        return new ForkingMap<>(this);
    }

    public V get(K key) {
        V found = map.get(key);
        if (found == null && parent != null) {
            return parent.get(key);  // 读时委派
        }
        return found;
    }
}
```

**设计意图**：一个 `Knowledge` 可以创建多个 `RuleSession`，每个会话可能添加临时规则。通过 Forking 结构，会话共享未修改的父节点数据，只在差异部分持有副本。这等价于 Git 的分支——写时复制到本地，读时委派到父节点。

**值得注意的细节**：
- `ForkingArray` 在父/子之间传递 `dataOffset` 偏移量，确保父子数据的索引空间不冲突
- `ForkingMap.replace()` 向上传播修改，用于需要同步回父链的场景
- 类 Javadoc 中有 TODO 注释："review the whole package, or even delete it as it's not exported"——说明作者自己也在评估保留的必要性

### 1.2 LongKeyMap（原始类型特化）

```java
// org.evrete.collections.LongKeyMap
public class LongKeyMap<T> implements Iterable<T> {
    private Entry<T>[] table;
    private int size;
    private final Object lock = new Object();

    private static class Entry<T> {
        final long key;
        T value;
        Entry<T> next;
    }

    public T computeIfAbsent(long key, Supplier<T> supplier) {
        T result = get(key);
        if (result == null) {
            synchronized (lock) {
                result = get(key);
                if (result == null) {
                    result = supplier.get();
                    put(key, result);
                }
            }
        }
        return result;
    }
}
```

**动机**：避免 `HashMap<Long, V>` 的自动装箱。在 RETE 网络内部，事实句柄（`FactHandle`）的 ID 是 `long`，引擎需要频繁按 ID 查找事实。`long` 原始类型哈希避免了每次 `Long.valueOf()` 的对象分配和 GC 压力。

**工程复杂度**：

| 结构 | 行数 | 标准库替代 | 收益判断 |
|---|---|---|---|
| `ForkingArray` | ~200 | `CopyOnWriteArrayList` | 合理（父/子委派语义不同） |
| `ForkingMap` | ~60 | `HashMap` 浅拷贝 | 合理（分叉语义需要） |
| `LongKeyMap` | ~241 | `HashMap<Long, V>` | 合理（原始类型特化） |
| `ArrayMap` | ~100 | `ArrayList` + 手动 KV | 边缘 |
| `IndexingArrayMap` | ~184 | `LinkedHashMap` | 可替代 |
| `ForkingArrayMap` | ~137 | `HashMap` | 边缘 |

### 局限/风险

- `LongKeyMap` 的哈希函数 `Integer.hashCode((int) key)` 丢失高 32 位，在 key 分布集中于高 32 位时碰撞严重
- `LongKeyMap` 的迭代器不是快照迭代器，遍历中结构变更行为未定义
- `ArrayMap` 和 `IndexingArrayMap` 的用途可以完全用标准库替代，新增的工程复杂度不值得
- 只有 `ForkingArrayTest` 和 `LongKeyMapTest` 两个测试文件，缺少边界测试

### 亮点评分：4/5

`LongKeyMap` 和 `ForkingArray` 有明确的领域动机，父/子委派是经过验证的模式。扣 1 分因为部分结构（`ArrayMap`, `IndexingArrayMap`）收益不高，且测试覆盖不够。

---

## 2. 惰性迭代器链

`org.evrete.util` 包含一组装饰器模式的惰性迭代器，共同特点是：零预缓存、零中间集合分配。

### 迭代器谱系

```java
// org.evrete.util.MappingIterator — 元素映射
public class MappingIterator<T, Z> implements Iterator<Z> {
    private final Iterator<T> delegate;
    private final Function<? super T, Z> mapper;

    @Override public Z next() {
        return mapper.apply(delegate.next());  // 仅 next() 时映射
    }
}

// org.evrete.util.FilteringIterator — 惰性过滤
public class FilteringIterator<T> implements Iterator<T> {
    private final Iterator<T> iterator;
    private final Predicate<T> predicate;
    private T nextElement;
    private boolean nextElementSet = false;

    private boolean setNextElement() {
        while (iterator.hasNext()) {
            T element = iterator.next();
            if (predicate.test(element)) {  // 逐个测试，找到为止
                nextElement = element;
                nextElementSet = true;
                return true;
            }
        }
        return false;
    }
}

// org.evrete.util.FlatMapIterator — 展平
public class FlatMapIterator<T, Z> implements Iterator<Z> {
    private final Iterator<T> source;
    private final Function<T, Iterator<Z>> flatMapFunction;
    private Iterator<Z> current = Collections.emptyIterator();

    @Override public boolean hasNext() {
        while (!current.hasNext() && source.hasNext()) {
            current = flatMapFunction.apply(source.next());
        }
        return current.hasNext();
    }
}

// org.evrete.util.CombinationIterator — 笛卡尔积
public class CombinationIterator<T> implements Iterator<T[]> {
    private final T[] combination;  // 共享结果数组

    @Override public T[] next() {
        // 直接修改共享数组，不创建新对象
        for (int i = size - 1; i >= readNextPosition; i--) {
            combination[i] = advanceIterator(i);
        }
        readNextPosition = computeNextPosition();
        return combination;
    }
}
```

**关键设计**：`CombinationIterator` 使用共享结果数组（`sharedResultArray`），每次 `next()` 复用同一个数组。在同时匹配 4 个事实类型的 Beta 评估中，这种方式避免了每次迭代创建 `T[4]` 的分配开销。

### 实际应用

这些迭代器在 Beta 节点评估中组合使用——`FlatMapIterator` 将事实的笛卡尔积惰性地传递给 `BetaEvaluator`，不产生中间集合：

```java
// BetaEvaluator 评估流程（推断的调用链）
FactIterator type1 = ...;  // Iterator<FactHandle>
FactIterator type2 = ...;
FlatMapIterator<FactHandle, Pair> flatMapped =
    new FlatMapIterator<>(type1, h1 -> mapToPair(h1, type2));
FilteringIterator<Pair> filtered =
    new FilteringIterator<>(flatMapped, pair -> evaluate(pair));
```

### 局限/风险

- **`CombinationIterator` 没有 `remove()` 支持**（大部分迭代器也没有，这是文档限制）
- **组合使用时代码可读性下降**——多个迭代器嵌套装饰时，调用链的调试体验不好
- **缺少短路优化**——在某些场景下，如果已经知道结果为空，迭代器仍会继续遍历直到耗尽
- **`EnumCombinationIterator`** 与普通 `CombinationIterator` 的功能重复，可以合并

### 亮点评分：4/5

实现清晰正确，`CombinationIterator` 的共享数组是 CPU vs 内存的优雅权衡。扣 1 分因为可调试性不足和 `EnumCombinationIterator` 的冗余。

---

## 3. `Mask<T>` 类型安全位集包装

`Mask<T>` 通过泛型包装 `BitSet`，将 RETE 网络中的集合运算从类型不安全变为类型安全：

```java
// org.evrete.runtime.Mask
public final class Mask<T> {
    private final BitSet delegate = new BitSet();
    private final ToIntFunction<T> intMapper;  // T → bit position

    public boolean containsAll(Mask<T> value) {
        BitSet other = value.delegate;
        BitSet otherCloned = (BitSet) other.clone();
        otherCloned.and(delegate);
        return otherCloned.equals(other);
    }

    public boolean intersects(Mask<T> other) {
        return delegate.intersects(other.delegate);
    }

    // 工厂方法绑定特定的类型-索引映射
    public static Mask<FactType> factTypeMask() {
        return new Mask<>(FactType::getInRuleIndex);
    }
    public static Mask<ActiveType> typeMask() {
        return new Mask<>(value -> value.getId().getIndex());
    }
    public static Mask<AlphaConditionHandle> alphaConditionsMask() {
        return Mask.instance(AlphaConditionHandle::getIndex);
    }
    public static Mask<AlphaAddress> alphaAddressMask() {
        return Mask.instance(AlphaAddress::getIndex);
    }
}
```

**为什么是亮点**：`Mask` 使用了 Java 泛型的"幻影类型"（phantom type）模式——类型参数 `T` 不用于数据存储，只用于编译期类型标记：

```java
Mask<FactType> factMask = Mask.factTypeMask();
Mask<AlphaAddress> addrMask = Mask.alphaAddressMask();
factMask.intersects(addrMask);  // ❌ 编译错误，类型不匹配
```

RETE 网络的 Alpha 内存桶索引、类型过滤、条件分组都重度依赖 `Mask`：

```java
// org.evrete.runtime.TypeAlphaConditions
public class TypeAlphaConditions implements Indexed {
    private final Mask<AlphaConditionHandle> mask;
    // 一组 alpha 条件对应一个内存桶
    this.mask = Mask.alphaConditionsMask().set(alphaConditions);
}

// org.evrete.runtime.evaluation.BetaEvaluator
public class BetaEvaluator implements WorkUnit {
    private final Mask<FactType> typeMask;
    // 标记该 Beta 评估器涉及哪些事实类型
}
```

### 局限/风险

- `Mask` 的 `equals()` 和 `hashCode()` 委托给内部的 `BitSet`，但 `Mask` 的泛型参数不在等值比较中——两个不同 `T` 的 `Mask` 实例如果 `BitSet` 一致会相等
- `containsAll()` 每次克隆 `BitSet`，在小规模位上性能可以接受，但在大规模位上可能成为热点
- 没有 `isEmpty()` 的便捷方法，调用方需要 `mask.cardinality() == 0`

### 亮点评分：5/5

`BitSet` 包装为类型安全的集合操作是优雅的设计，工厂方法模式锦上添花。"少即是多"的典范——整个类 ~129 行，解决的问题却关键。

---

## 4. 并发模型与异步工具

### 4.1 CompletionManager（Key 级异步序列化）

```java
// org.evrete.util.CompletionManager
public class CompletionManager<K, T> {
    private final ConcurrentHashMap<K, CompletableFuture<T>> completions;

    public CompletableFuture<T> enqueue(K key,
            Function<K, CompletableFuture<T>> mappingFunction) {
        synchronized (this.completions) {
            CompletableFuture<T> existing = this.completions.get(key);
            CompletableFuture<T> newFuture;
            if (existing == null) {
                newFuture = mappingFunction.apply(key);       // 新任务
            } else {
                newFuture = existing.thenCompose(             // 排队
                    t -> mappingFunction.apply(key));
            }
            // 完成自动清理
            CompletableFuture<T> chained = newFuture.whenComplete(
                (t, throwable) -> completions.remove(key));
            this.completions.put(key, chained);
            return chained;
        }
    }
}
```

**设计意图**：同一 key 的任务必须按提交顺序执行（例如对同一类型的 Delta 更新），但不同 key 的任务可以并行。`CompletableFuture.thenCompose()` 天然做到了这一点。

### 4.2 DelegatingExecutorService（选择性生命周期管理）

```java
// org.evrete.util.DelegatingExecutorService
public class DelegatingExecutorService implements ExecutorService {
    private final ExecutorService delegate;
    private final boolean externallySupplied;

    public DelegatingExecutorService(ExecutorService delegate) {
        this.delegate = delegate;
        this.externallySupplied = true;  // 外部提供，不接管生命周期
    }

    public DelegatingExecutorService(int threads, boolean daemonThreads) {
        this.delegate = Executors.newFixedThreadPool(threads,
            new CustomThreadFactory(daemonThreads));
        this.externallySupplied = false;  // 内部创建，管理生命周期
    }

    @Override
    public void shutdown() {
        if (!externallySupplied) {
            delegate.shutdown();  // 只关内部的
        }
    }

    @Override
    public List<Runnable> shutdownNow() {
        if (externallySupplied) {
            return Collections.emptyList();  // 外部传入，不动
        }
        return delegate.shutdownNow();
    }
}
```

如果用户传入自己的 `ExecutorService`，引擎不会关闭它；如果引擎自己创建，引擎负责生命周期。这种"看谁创建谁负责"的模式避免了资源泄漏又提供了灵活性。

### 4.3 Fire 循环的异步编排

`AbstractRuleSession.fireCycle()` 使用 `CompletableFuture` 编排 Delta 计算、议程执行和内存提交的流水线：

```java
// org.evrete.runtime.AbstractRuleSession
private CompletableFuture<Void> fireCycle(ActivationContext ctx,
        ActivationMode mode, WorkMemoryActionBuffer actions) {
    if (actions.hasData()) {
        return ctx.computeDelta(actions)
            .thenCompose(deltaStatus -> {
                WorkMemoryActionBuffer newActions =
                    doAgenda(ctx, deltaStatus.getAgenda(), mode);
                return ctx.commitMemories(deltaStatus)
                    .thenCompose(unused ->
                        fireCycle(ctx, mode, newActions));
            });
    } else {
        return CompletableFuture.completedFuture(null);
    }
}
```

这是一个典型的**生产者-消费者-递归**模式：Delta 变更产生议程，议程执行产生新的动作，新动作触发下一轮周期。

### 局限/风险

- `CompletionManager` 使用 `synchronized (this.completions)` 作为外部锁，但用的是 `ConcurrentHashMap`——两者的组合可能让人困惑，应该用文档说明
- `DelegatingExecutorService` 的 `CustomThreadFactory` 线程名固定为 `evrete-thread-N`，多知识库场景无法区分
- 异步路径的异常传播路径较长，`CompletableFuture` 链中任一环节抛异常后的错误消息可能不够直观

### 亮点评分：4/5

`CompletionManager` 是"所需即所得"的优雅工具。`DelegatingExecutorService` 的生命周期策略务实。扣 1 分因为线程命名和异常诊断的局限性。

---

## 5. 事件广播架构

### 5.1 Hierarchy（层级遍历工具）

```java
// org.evrete.util.Hierarchy
public class Hierarchy<T> {
    private final Hierarchy<T> parent;
    private final T value;

    public final void walkUp(Consumer<T> consumer) {
        consumer.accept(this.value);
        if (parent != null) {
            parent.walkUp(consumer);  // 递归向上遍历
        }
    }
}
```

一个极简（~30 行）但正确的递归树结构，只支持从当前节点向上的单向遍历。

### 5.2 BroadcastingPublisher（分层事件广播）

```java
// org.evrete.util.BroadcastingPublisher
public class BroadcastingPublisher<E extends Events.Event>
        implements Events.Publisher<E> {
    private final Hierarchy<SplitSubscriptions<E>> innerSubscriptions;

    // 子节点绑定到父节点的 Hierarchy
    public BroadcastingPublisher(BroadcastingPublisher<E> other) {
        this.innerSubscriptions = new Hierarchy<>(
            new SplitSubscriptions<>(), other.innerSubscriptions);
    }

    public void broadcast(E event) {
        this.innerSubscriptions.walkUp(
            subs -> subs.broadcast(event, executor));
    }
}
```

当 `RuleSession` 上触发事件时，不仅通知本地订阅者，还通过 `walkUp()` 沿 `Hierarchy` 向上传播到 `Knowledge` 的订阅者。这样用户可以在 `Knowledge` 级别订阅一次，所有子会话的事件都能收到。

### 5.3 分槽订阅

`SplitSubscriptions` 将同步和异步订阅分开存储，广播时先异步后同步：

```java
// org.evrete.util.BroadcastingPublisher.SplitSubscriptions
void broadcast(H event, Executor executor) {
    // 1. 先发异步（不阻塞当前线程）
    for (InnerSubscription<H> s : subscriptionsAsync)
        executor.execute(() -> s.action.accept(event));
    // 2. 再发同步
    for (InnerSubscription<H> s : subscriptionsSync)
        s.action.accept(event);
}
```

这种顺序确保了同步订阅能看到异步副作用的最终状态。

### 5.4 EventMessageBus（事件总线）

```java
// org.evrete.runtime.EventMessageBus
public class EventMessageBus implements Copyable<EventMessageBus> {
    private final Map<Class<? extends ContextEvent>, Handler<?>> handlers;

    public EventMessageBus(Executor executor) {
        handlers = new HashMap<>();
        register(KnowledgeCreatedEvent.class, new KnowledgeCreatedEventHandler(executor));
        register(SessionCreatedEvent.class, new SessionCreatedEventHandler(executor));
        register(SessionClosedEvent.class, new SessionClosedEventHandler(executor));
        register(SessionFireEvent.class, new SessionFireEventHandler(executor));
        register(EnvironmentChangeEvent.class, new EnvironmentChangeEventHandler(executor));
    }

    @Override
    public synchronized EventMessageBus copyOf() {
        return new EventMessageBus(this);  // 深拷贝所有 handler
    }
}
```

5 个事件类型覆盖了知识库和会话的生命周期。`copyOf()` 实现了配置的隔离复制——每个 session 拿到自己独立的事件总线，修改不会相互影响。

### 局限/风险

- **事件类型偏少**——缺少规则级事件（如 `RuleActivationEvent`, `ConditionMatchEvent`），精细化的监控不够
- `BroadcastingPublisher` 使用 `synchronized` 来保护 `subscribe()` 和 `removeSubscription()`，但广播时未加锁——读-写并发时可能出现订阅已取消但仍收到事件的情况
- `Hierarchy` 不支持从下到上的遍历（`walkDown` 或 `children()`），只能从任意节点向上到根

### 亮点评分：4/5

`Hierarchy + BroadcastingPublisher` 的组合是事件系统的亮点，实现了父子上下文的事件传播。`SplitSubscriptions` 的分槽设计周全。扣 1 分因为事件类型不足和并发设计的粗粒度。

---

## 6. 预计算与索引系统

### 6.1 PreHashed（哈希码预计算）

```java
// org.evrete.runtime.PreHashed
public abstract class PreHashed {
    private final int hash;

    protected PreHashed(int hash) {
        this.hash = hash;
    }

    @Override
    public final int hashCode() {
        return hash;  // 预计算，避免重复计算
    }
}
```

RETE 网络中的节点（`AlphaConditionHandle`、`FactType` 等）在运行时被频繁用于 HashMap 查找。将 `hashCode()` 在构造时预计算并将方法声明为 `final`，防止子类覆盖。

### 6.2 AbstractIndex（统一索引抽象）

```java
// org.evrete.util.AbstractIndex
public abstract class AbstractIndex extends PreHashed implements Indexed {
    private final int index;

    protected AbstractIndex(int index, int hashCode) {
        super(hashCode);
        this.index = index;
    }

    @Override
    public int getIndex() {
        return index;
    }
}
```

### 6.3 Indexed + IndexedValue（引用透明的相等性）

```java
// org.evrete.util.Indexed
public interface Indexed {
    int getIndex();
}

// org.evrete.util.IndexedValue
public class IndexedValue<T> implements Indexed {
    private final int index;
    private final T value;

    @Override
    public boolean equals(Object o) {
        return index == ((IndexedValue<?>) o).index;  // 仅比较索引
    }

    @Override
    public int hashCode() {
        return index;  // 索引即哈希
    }
}
```

`IndexedValue` 的相等性只比较 `index`，意味着两个不同的值如果被分配到同一个索引，则被视为相等的。这在 RETE 网络中是有意为之——节点由其位置（索引）而非其内容标识。

### 工程目的

这套索引系统为 RETE 网络的每个节点（alpha 条件、beta 条件、fact type）分配唯一整数索引，然后用 `Mask`（BitSet）进行高效的集合运算。索引 → Mask → 集合运算，这条链路是 RETE 引擎性能的基础。

### 局限/风险

- `PreHashed` 使用 `final` 方法防止子类覆盖 `hashCode()`——这限制了子类的灵活性。如果子类的相等性判定需要更复杂的逻辑，无法通过重写 `hashCode()` 实现
- `AbstractIndex` 子类的构造函数需要手动传入 `hashCode` 值，存在调用者传错的风险（运行时 bug）
- 整个索引分配机制缺少文档化的契约（索引的分配时机、回收机制、最大值约束）

### 亮点评分：4/5

`PreHashed` 是简单有效的微优化。`IndexedValue` 的索引即 equals 的设计引用透明。扣 1 分因为 `final hashCode()` 的限制和手动传入 hash 值的出错风险。

---

## 7. Wrapper 装饰器模式家族

`org.evrete.util` 包含一套 Wrapper 类体系，用于 SPI 层的实现隔离和扩展：

| 类 | 包装对象 | 用途 |
|---|---|---|
| `AbstractSessionWrapper` | `RuleSession` | 拦截/装饰会话操作 |
| `KnowledgeWrapper` | `Knowledge` | 知识库行为代理 |
| `RuntimeContextWrapper` | `RuntimeContext` | 统一上下文装饰基类 |
| `TypeWrapper` | `Type` | 类型行为扩展 |
| `FactStorageWrapper` | `FactStorage` | 存储行为代理 |
| `GroupingReteMemoryWrapper` | `GroupingReteMemory` | RETE 内存代理 |

```java
// org.evrete.util.RuntimeContextWrapper（统一装饰基类）
public abstract class RuntimeContextWrapper<D, C extends RuntimeContext<C>,
        R extends RuntimeRule> implements RuntimeContext<C> {
    protected final D delegate;
    protected abstract C self();

    @Override
    public RuleSetBuilder<C> newRule(String name) {
        return delegate().newRule(name);  // 默认委托
    }
}

// org.evrete.util.AbstractSessionWrapper（会话装饰）
public abstract class AbstractSessionWrapper<S extends RuleSession<S>>
        extends RuntimeContextWrapper<S, S, RuntimeRule>
        implements RuleSession<S> {

    @Override
    public ActivationManager getActivationManager() {
        return delegate.getActivationManager();
    }

    @Override
    public <T> Collector<T, ?, S> asCollector() {
        return new SessionCollector<>(self());
    }
}
```

SPI 提供者可以通过继承 Wrapper 类来**部分覆盖**行为——只覆写关心的方法，其余委托给原实现：

```java
// 假设自定义 SPI 实现
public class MonitoringSessionWrapper extends AbstractSessionWrapper<StatefulSession> {
    @Override
    public void fire() {
        long start = System.nanoTime();
        super.fire();                  // 委托给原实现
        long elapsed = System.nanoTime() - start;
        log("Fire took " + elapsed + "ns");  // 添加监控
    }
}
```

### 局限/风险

- **Wrapper 膨胀**——6 个 Wrapper 类中，`KnowledgeWrapper` 的行数达到 ~150 行，但实际应用场景有限
- **Wrapper 和被包装对象的接口不完全同步**——当 API 新增方法时，需要更新所有 Wrapper 类，容易遗漏
- **缺少 Wrapper 组合工厂**——目前需要手动嵌套 Wrapper，没有一个组合工厂来按顺序应用多个装饰器
- Wrapper 模式在这里的收益（"可以只覆写部分方法"）在实际中很少使用——大多数 SPI 实现都会完整替换行为

### 亮点评分：3/5

设计意图合理，但在实际中应用有限。扣 2 分因为 Wrapper 维护成本高（API 变更时需要同步更新所有 Wrapper）和缺少组合工厂。

---

## 8. API 工具设计

### 8.1 StringLiteralEncoder（轻量级字符串预处理）

```java
// org.evrete.runtime.compiler.StringLiteralEncoder
final class StringLiteralEncoder {
    private static final String PREFIX = "${const";
    private static final String SUFFIX = "}";
    // 支持三种引号
    private static final char[] QUOTES = new char[]{'\'', '"', '`'};

    static StringLiteralEncoder of(String s, boolean stripWhiteSpaces) {
        // 1. 将字符串常量替换为 ${constN} 占位符
        // 2. 安全移除空白
        // 3. 保留常量映射以便后续还原
    }

    public String unwrapLiterals(final String arg) {
        // 将占位符还原为原始字符串常量
        String s = arg;
        for (Map.Entry<String, String> entry : stringConstantMap.entrySet()) {
            s = s.replace(entry.getKey(), "\"" + entry.getValue() + "\"");
        }
        return s;
    }
}
```

**为什么是亮点**：在没有 ANTLR 等解析器生成器的情况下，这是解决"空白敏感的源码文本中内嵌字面量"问题的巧妙方案。用占位符劫持了空白移除的安全问题。更妙的是，类 Javadoc 中明确说明了其定位：

```java
/**
 * ... this class is a very basic tool for parsing Java literal sources.
 * More advanced implementations of the Service Provider Interface (SPI)
 * may use more sophisticated tools like ANTLR.
 */
```

这种诚实的自我定位——"我知道这不是最好的方案，但它是正确的，SPI 扩展可以替换它"——展示了务实的工程判断。

### 8.2 Copyable<T>（深拷贝契约）

```java
// org.evrete.api.Copyable
public interface Copyable<T> {
    /**
     * 类似于 Git 分支的机制。创建副本确保：
     * 1. 父上下文 (Knowledge) 的数据对子会话可用
     * 2. 子会话的修改不影响父上下文
     */
    T copyOf();
}
```

`Configuration` 实现了 `Copyable`，形成 `KnowledgeService → Knowledge → RuleSession` 的配置继承链。每个层级获取父级配置的深拷贝作为起点：

```java
// org.evrete.Configuration
public class Configuration extends Properties
        implements Copyable<Configuration>, FluentImports<Configuration> {
    @Override
    public Configuration copyOf() {
        return new Configuration(this, this.imports.copyOf());
    }
}
```

`Copyable` 作为接口比 `Cloneable` 更好——返回值类型确定，不会抛出 `CloneNotSupportedException`。

### 8.3 Configuration 的属性管理

```java
// org.evrete.Configuration
public class Configuration extends Properties
        implements Copyable<Configuration>, FluentImports<Configuration> {

    // 属性键按命名空间分层
    public static final String OBJECT_COMPARE_METHOD = "evrete.core.fact-identity-strategy";
    public static final String DAEMON_INNER_THREADS  = "evrete.core.daemon-threads";
    static final String SPI_MEMORY_FACTORY           = "evrete.spi.memory-factory";
    public static final String RULE_BASE_CLASS       = "evrete.impl.rule-base-class";

    // 显式管理的废弃属性集合
    private static final Set<String> OBSOLETE_PROPERTIES = Set.of(
        IDENTITY_METHOD_EQUALS, SPI_EXPRESSION_RESOLVER, ...
    );

    @Override
    public synchronized Object setProperty(String key, String value) {
        if (OBSOLETE_PROPERTIES.contains(key))
            LOGGER.warning("Property '" + key + "' is obsolete");
        return super.setProperty(key, value);
    }
}
```

命名空间分层（`evrete.core.*`、`evrete.spi.*`、`evrete.impl.*`）为使用者提供了清晰的配置范围指引。废弃属性集合 + `setProperty` 拦截是运行时友好的迁移方式。

### 8.4 空对象模式

`DefaultActivationManager` 和 `LeastImportantServiceProvider` 是两个空对象模式的应用：

```java
// org.evrete.util.DefaultActivationManager
public class DefaultActivationManager implements ActivationManager {
    // 零行为默认实现
}

// org.evrete.spi.minimal.LeastImportantServiceProvider
public interface LeastImportantServiceProvider extends OrderedServiceProvider {
    @Override
    default int order() {
        return Integer.MAX_VALUE;  // 返回最低优先级
    }
}
```

使用空对象而非 `null`，避免了调用处的 null 检查，且通过 SPI 优先级机制自然回退到最低优先级的实现。

### 局限/风险

- `StringLiteralEncoder` 只支持三种引号（`'`, `"`, 反引号），不支持 Java 15+ 的文本块引号（`"""`）
- `Configuration` 继承 `Properties`（Hashtable 子类），引入了序列化和线程安全的 legacy baggage
- `BaseRuleClass` 目前只有 `eq()` 两个方法，文档说用户可以扩展但不提供 SPI 或配置路径来替换
- `Copyable` 是手动深拷贝，如果一个类新增了字段但忘记了更新 `copyOf()`，会引入微妙的共享状态 bug

### 亮点评分：4/5

`StringLiteralEncoder` 定位诚实、实现正确。`Copyable` 比 `Cloneable` 更优。命名空间分层的配置键是好习惯。扣 1 分因为 `Configuration` 继承 `Properties` 的历史债务和空对象模式应用的不一致。

---

## 9. 测试与工程实践

### 测试组织

测试按包结构与源码对齐：

```
evrete-core/src/test/java/org/evrete/
├── runtime/
│   ├── StatefulInsertTests         — 有状态插入
│   ├── StatelessInsertTests        — 无状态插入
│   ├── StatelessSessionImplTest    — 无状态会话实现
│   ├── HotDeploymentStatefulTests  — 热部署（有状态）
│   ├── HotDeploymentStatelessTests — 热部署（无状态）
│   ├── TypeSystemTests             — 类型系统
│   ├── SessionAsyncTests           — 异步会话
│   ├── SessionUpdateDeleteTests    — 更新/删除
│   ├── MaskTest                    — Mask 位集
│   ├── EventMessageBusTests        — 事件总线
│   └── EvaluationListenersTests    — 评估监听器
├── collections/    — ForkingArrayTest, LongKeyMapTest
├── spi/minimal/    — DefaultGroupingReteMemoryTest, TypeResolverTest
├── util/           — CombinationIteratorTest, FlatMapIteratorTest
└── api/            — API 测试
```

测试数量（在 Maven 转换后已验证）：
- `evrete-core`：344 个测试（排除一个 flaky test）
- `evrete-dsl-java`：99 个测试
- `evrete-jsr94`：28 个测试

### 测试设计特点

1. **类型系统的参数化测试** — `TypeSystemTests` 覆盖多种类型解析场景
2. **热部署的状态验证** — 规则添加后验证已有事实的匹配一致性
3. **异步会话的时序测试** — `SessionAsyncTests` 验证并发插入和触发

### 可见性控制

最值得称道的工程实践是可见性控制——`org.evrete.runtime` 中所有类都是 `package-private`：

```java
// 这些类在 runtime 包中，但没有 public 声明
class AbstractRuleSession { ... }
class ActivationContext { ... }
class BetaEvaluator { ... }
class Mask<T> { ... }
```

只有 API 包和少量顶层类是 `public` 的。这种"最小公开表面"的设计意味着核心实现可以被自由重构而不影响用户代码。

### 局限/风险

- **Flaky test** — `HotDeploymentStatefulTests.testMixedMulti` 在多测试组合时偶现失败，独立运行通过，说明测试间有静态状态泄漏
- **测试隔离不足** — 部分测试依赖共享的 `KnowledgeService` 实例
- **缺少微基准测试** — `evrete-benchmarks` 模块缺少配套文档说明测试场景和结果解读
- `ForkingArray` 类上有一个 TODO 注释："review the whole package, or even delete it"——暗示作者对某些自定义数据结构的保留有疑虑

### 亮点评分：3/5

可见性控制是工程纪律的亮点。核心路径测试覆盖合理。扣 2 分因为 flaky test、测试隔离问题和 TODO 中暗示的代码质量不确定性。

---

## Top 5 实现亮点

| # | 亮点 | 理由 | 核心文件 |
|---|---|---|---|
| 1 | **`Mask<T>` 幻影类型包装** | 用 ~129 行代码将 `BitSet` 包装为类型安全的集合操作。`Mask<FactType>` 和 `Mask<AlphaAddress>` 在编译期互不相容。RETE 网络的 Alpha 内存桶、类型过滤、条件分组全部依赖它。 | `org.evrete.runtime.Mask` |
| 2 | **`BroadcastingPublisher` + `Hierarchy` 分层广播** | `Hierarchy.walkUp()` + `BroadcastingPublisher` 的组合实现了父子上下文的事件传播。RuleSession 的事件自动上浮到 Knowledge 的订阅者，不需要额外的注册代码。 | `org.evrete.util.Hierarchy`, `org.evrete.util.BroadcastingPublisher` |
| 3 | **`CompletionManager` Key 级异步序列化** | 用 `ConcurrentHashMap<K, CompletableFuture<T>>` + `thenCompose` 链式调用来实现按 key 顺序执行的任务队列。同一 type 的 Delta 更新串行，不同 type 并行，恰好匹配规则引擎的并发需求。 | `org.evrete.util.CompletionManager` |
| 4 | **`ForkingArray` + `ForkingMap` 父/子委派** | 写时复制到本地，读时委派到父节点。一个 Knowledge 的多个 RuleSession 共享未修改数据，只在差异部分持有独立副本。等价于"数据结构层面"的 Git 分支。 | `org.evrete.collections.ForkingArray`, `org.evrete.util.ForkingMap` |
| 5 | **`StringLiteralEncoder` 字符串占位符预处理** | 在没有 ANTLR 的情况下，用 `${constN}` 占位符劫持了"空白敏感的 Java 源码中内嵌字符串字面量"的安全移除问题。代码 ~80 行，类 Javadoc 诚实说明了设计边界。 | `org.evrete.runtime.compiler.StringLiteralEncoder` |
