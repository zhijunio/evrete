 # Evrete 运行时数据流追踪
 
 > 从 `session.insert(fact)` 到 `RHS 执行` 再到递归 fire cycle 的完整路径。
 > 分析版本：4.0.4-SNAPSHOT | 分析日期：2026-07-03
 
 已有参考文档：
 - [design-analysis.md](./design-analysis.md) — 宏观架构分析
 - [design-patterns-deep-dive.md](./design-patterns-deep-dive.md) — 实现模式分析
 - [learning-guide.md](./learning-guide.md) — 学习路线与前置知识
 
 ---
 
 ## 一、概览
 
 Evrete 的运行时核心是一个**异步递归循环**：
 
 ```
 外部 insert → 缓冲区 → fire 触发
                       ↓
                  ┌─ fireCycle ──────────┐
                  │  1. computeDelta     │ ← 增量计算（异步编排）
                  │  2. doAgenda         │ ← RHS 执行
                  │  3. commitMemories   │ ← 提交变更
                  └──────┬─┬─────────────┘
                   ┌─────┘ │
                   ↓       ↓
               RHS 产新动作  无新动作 → 结束
 ```
 
 ---
 
 ## 二、Phase 0：insert 入站路径
 
 ### 2.1 入口
 
 `insert()` 是 `RuleSession` 接口的 `default` 方法，直接委托：
 
 ```java
 // api/RuleSession.java:92
 default S insert(Object... objects) {
     insert0(objects, true);
     return (S) this;  // 返回自身，支持流式链
 }
 ```
 
 `insert0()` 声明在 `api/SessionOps.java:38` 中，实现在 `runtime/AbstractRuleSessionOps.java:42`。
 
 ### 2.2 类型解析与索引
 
 ```java
 // runtime/AbstractRuleSessionOps.java:154
 DefaultFactHandle bufferInsertMultiple(Object fact, boolean applyToStorage,
         boolean resolveCollections, WorkMemoryActionBuffer destination) {
     Collection<?> rawFacts = factToCollection(fact, resolveCollections);
     return bufferInsertMultiple(rawFacts, applyToStorage, destination);
 }
 ```
 
 `factToCollection()` 处理数组/Iterable 展开。之后对每个 fact：
 
 | 步骤 | 方法 | 作用 |
 |---|---|---|
 | 1 | `typeResolver.resolve(fact)` | 通过反射将 Java 类映射到 `Type`（若 `warnUnknownTypes` 开启且找不到，则跳过） |
 | 2 | `getCreateIndexedType(factType)` | 获取或创建 `ActiveType`，分配 `ActiveType.Idx` |
 | 3 | `bufferInsertSingle(type, activeType, ...)` | 创建 `DeltaMemoryAction.Insert` 并存入缓冲区 |
 
 ### 2.3 bufferInsertSingle 的核心逻辑
 
 ```java
 // runtime/AbstractRuleSessionOps.java:228-
 private void bufferInsertSingle(DefaultFactHandle factHandle, Type<?> type,
         ActiveType activeType, boolean applyToStorage, Object fact,
         WorkMemoryActionBuffer destination) {
     // 1. 提取事实的字段值
     TypeMemory memory = getMemory().getTypeMemory(activeType.getId());
     ValueIndexer<FactFieldValues> valueIndexer = memory.getFieldValuesIndexer();
     FactFieldValues fieldValues = activeType.readFactValue(type, fact);
 
     // 2. 为字段值计算索引 ID（用于快速比较和哈希）
     long valuesId = valueIndexer.getOrCreateId(fieldValues);
     FactHolder factHolder = new FactHolder(factHandle, valuesId, fact);
 
     // 3. 生成 DeltaMemoryAction.Insert 并加入缓冲区
     destination.addInsert(new DeltaMemoryAction.Insert(type, factHolder, applyToStorage));
 }
 ```
 
 **关键细节**：
 - `FactHandle` 是 `Long` ID（通过 `DefaultFactHandle` 分配），底层用 `AtomicLong` 生成
 - `ValueIndexer.getOrCreateId()` 实现了字段值的去重索引——相同字段值会映射到同一 `long` ID，这避免了 RETE 网络中重复存储
 - 此时事实尚未进入 RETE 网络，只是被包装为待处理的 `DeltaMemoryAction`
 
 ### 2.4 缓冲区结构
 
 `WorkMemoryActionBuffer` 内部使用 `LongKeyMap<State>`，以事实句柄 ID 为 key 存储每个事实的最新操作状态。每个 `State` 保留了上一个 insert 和第一个 delete，足以通过 update → delete + insert 的转换保证操作顺序正确。
 
 ```java
 // runtime/WorkMemoryActionBuffer.java
 class WorkMemoryActionBuffer {
     private final LongKeyMap<State> actionsPerFactHandle;
     // ...
 }
 ```
 
 ---
 
 ## 三、Phase 1：fire 入口
 
 ```java
 // runtime/StatefulSessionImpl.java:42
 public StatefulSession fire() {
     fireInner();
     return this;
 }
 
 // runtime/AbstractRuleSession.java:47
 void fireInner() {
     _assertActive();
     fireInnerAsync().join();  // 异步链同步等待
 }
 
 // runtime/AbstractRuleSession.java:52
 private CompletableFuture<Void> fireInnerAsync() {
     broadcast(SessionFireEvent.class, ...);  // 发布 fire 事件
     ActivationMode mode = getAgendaMode();
     ActivationContext context = new ActivationContext(
         this, ruleStorage.getList()    // 当前快照的规则列表
     );
     WorkMemoryActionBuffer buffer = getActionBuffer();  // 取缓冲
     return fireCycle(context, mode, buffer);
 }
 ```
 
 **重要设计决策**：
 - `ActivationContext` 在每次 `fire()` 时创建新实例。它持有当前规则列表的不可变快照，因此 `fire()` 期间新增的规则不会影响执行
 - 异步编排使用 `CompletableFuture.thenCompose` 链，但 `fireInner()` 通过 `.join()` 对外表现为同步
 
 ---
 
 ## 四、Phase 2：fireCycle 递归循环
 
 这是整个引擎的核心。文档中 `do{}while` 的语义通过 `fireCycle()` 的递归实现：
 
 ```java
 // runtime/AbstractRuleSession.java:72
 private CompletableFuture<Void> fireCycle(ActivationContext ctx,
         ActivationMode mode, WorkMemoryActionBuffer actions) {
     if (actions.hasData()) {
         // Step 1: 计算增量
         CompletableFuture<Status> deltaFuture = ctx.computeDelta(actions);
 
         return deltaFuture.thenCompose(deltaStatus -> {
             // Step 2: 执行议程（RHS）
             WorkMemoryActionBuffer newActions = doAgenda(
                 ctx, deltaStatus.getAgenda(), mode);
 
             // Step 3: 提交内存变更
             return ctx.commitMemories(deltaStatus)
                 .thenCompose(unused -> fireCycle(ctx, mode, newActions));  // ← 递归
         });
     } else {
         return CompletableFuture.completedFuture(null);
     }
 }
 ```
 
 ### 4.1 computeDelta — 增量计算
 
 这是最复杂的阶段，分两步进行。
 
 #### 4.1.1 拆分为 Type-SplitView
 
 `WorkMemoryActionBuffer.sinkToSplitView()` 将混杂所有类型的 action buffer 按 `ActiveType` 拆分为 `SplitView`。
 每个 `SplitView` 包含该类型的所有 insert 和 delete 操作。
 
 **关键设计**：sink 过程中自动解决了 update → delete + insert 的转换。
 
 #### 4.1.2 处理 delete 操作
 
 `ActivationContext.processDeleteActions()` 将 delete 操作分发到三处：
 
 | 目标 | 操作 | 目的 |
 |---|---|---|
 | `TypeAlphaMemory` | `memory.delete(valuesId, handle)` | 从 alpha 内存移除 |
 | `TypeMemory` | `typeMemory.remove(handle)` | 从事实现有存储移除（仅首次时） |
 | `SessionFactGroup` | `group.processDeleteDeltaActions()` | 从规则的条件匹配组移除 |
 
 每一步都是异步任务（`CompletableFuture.runAsync()`），通过 `CommonUtils.completeAll()` 并行等待。
 
 #### 4.1.3 处理 insert 操作 → alpha 内存
 
 `ActivationContext.processDeltaStatus()` 做两件事：
 
 **a）按 AlphaAddress 分组插入**
 
 ```java
 // 对每个 insert，计算它匹配哪些 AlphaAddress（即哪些 alpha 条件）
 Collection<AlphaAddress> matchingLocations =
     session.matchingAlphaLocations(insert.getHandle(), insert.getValues());
 for (AlphaAddress alpha : matchingLocations) {
     insertsByAlphaLocation.add(alpha, factHolder);
 }
 ```
 
 每个 `AlphaAddress` 对应一个 `TypeAlphaMemory`（即一组 alpha 条件的内存桶）。
 随后对每个 AlphaAddress 异步执行 `processInsertDeltaActions()`，将事实插入 `TypeAlphaMemory`。
 
 **b）建立 insertMask 并匹配规则**
 
 ```java
 Mask<AlphaAddress> insertMask = Mask.alphaAddressMask()
     .set(insertsByAlphaLocation.keySet());
 
 for (SessionRule rule : rules) {
     for (SessionFactGroup group : rule.getLhs().getFactGroups()) {
         if (group.getAlphaAddressMask().intersects(insertMask)) {
             // 此规则被激活
             result.addAffectedFactGroup(group);
             result.addAffectedRule(rule);
         }
     }
 }
 ```
 
 `Mask<AlphaAddress>` 的交集操作是 O(1) 的位运算，这是 Evrete 高效的秘诀之一——引擎不需要暴力匹配所有规则，只需要检查规则的条件组涉及的 AlphaAddress 是否有交集。
 
 #### 4.1.4 Beta 评估（buildDeltas）
 
 在所有 alpha 内存插入完成后，对被激活的规则条件组执行 Beta 评估：
 
 ```java
 // SessionFactGroupBeta.buildDeltas()
 CompletableFuture<Void> buildDeltas(DeltaMemoryMode mode) {
     return graph.terminalNode().computeDeltaMemoryAsync(mode);
 }
 ```
 
 `ReteSessionConditionNode.computeDeltaMemoryAsync()` 的行为取决于是否是终端节点：
 
 - **终端节点**（树的叶子）：将 alpha 内存的新数据读取到 `ConditionMemory`，计算新增/删除的 token 集合
 - **非终端节点**：递归地从子节点获取 token，用 `BetaEvaluator` 评估条件（类似 SQL JOIN + WHERE 的笛卡尔积过滤），将结果传播到父节点
 
 Beta 评估的最终产出是 `ConditionMemory` 中的 `ScopedValueId[]` 集合——每个 `ScopedValueId` 代表一组满足全部条件的事实句柄组合。
 
 ### 4.2 doAgenda — RHS 执行
 
 ```java
 // runtime/AbstractRuleSession.java:86
 private WorkMemoryActionBuffer doAgenda(ActivationContext context,
         List<SessionRule> agenda, ActivationMode mode) {
     if (agenda.isEmpty()) return WorkMemoryActionBuffer.EMPTY;
 
     activationManager.onAgenda(
         context.incrementFireCount(), Collections.unmodifiableList(agenda));
     WorkMemoryActionBuffer buffer = new WorkMemoryActionBuffer();
 
     switch (mode) {
         case DEFAULT:
             doAgendaDefault(agenda, buffer);     // 顺序模式
             break;
         case CONTINUOUS:
             doAgendaContinuous(agenda, buffer);  // 连续模式
             break;
     }
     return buffer;
 }
 ```
 
 **DEFAULT 模式**：逐个执行议程中的规则。一旦某个规则 RHS 产生了新动作（insert/update/delete），立即返回，把新动作放到下一个 fireCycle 处理。
 
 ```java
 // runtime/AbstractRuleSession.java:106
 private void doAgendaDefault(List<SessionRule> agenda,
         WorkMemoryActionBuffer destinationForRuleActions) {
     for (SessionRule rule : agenda) {
         if (activationManager.test(rule)) {
             long count = rule.callRhs(destinationForRuleActions);
             activationManager.onActivation(rule, count);
             if (destinationForRuleActions.hasData()) {
                 return;  // ← 关键：有新动作就停，下个循环继续
             }
         }
     }
 }
 ```
 
 **CONTINUOUS 模式**：执行所有议程规则，不管是否产生新动作。
 
 `rule.callRhs()` 内部执行用户定义的 RHS lambda，用户的 `insert()/update()/delete()` 调用通过 `RhsContextImpl` 重定向到 `destinationForRuleActions` 缓冲区，而不是直接操作会话内存。
 
 ### 4.3 commitMemories — 提交内存
 
 ```java
 // runtime/ActivationContext.java
 CompletableFuture<Void> commitMemories(Status status) {
     List<CompletableFuture<Void>> futures = new ArrayList<>();
     // 1. 提交 Beta 条件节点的变更
     for (SessionFactGroup group : status.affectedFactGroups) {
         futures.add(group.commitDeltas());
     }
     // 2. 提交 Alpha 内存的变更
     for (TypeAlphaMemory alphaMemory : status.affectedAlphaBuckets) {
         futures.add(alphaMemory.commit(executor));
     }
     return CommonUtils.completeAll(futures);
 }
 ```
 
 `commit()` 将 delta 状态合并到基线状态——Alpha 内存的 commit 将新插入的事实从 delta 集合转移到主集合；Beta 节点的 commit 将新增的 token 合并到条件匹配的结果集。
 
 提交完成后，递归回到 `fireCycle()` 检查是否还有新动作。
 
 ---
 
 ## 五、条件编译管道（附录）
 
 对应于 `@Where("$c.value > 100")` 注解到可执行 `MethodHandle` 的完整路径：
 
 ### 5.1 阶段示意
 
 ```
 @Where("$c.value > 100")
   ↓
 StringLiteralEncoder  — 将字符串常量替换为 ${constN} 占位符
   ↓
 DefaultLiteralSourceCompiler.RuleSource  — 生成 Java 源码文本
   ↓
 DefaultSourceCompiler.compile() — javac 编译
   |— InMemoryFileManager        — 输出保持在内存中
   |— JavaSourceObject           — 源码对象
   |— DestinationClassObject     — 字节码结果
   ↓
 ClassLoaderWrapper  — 自定义 ClassLoader 加载字节码
   ↓
 Class.getMethod → MethodHandle — 反射获取评估方法句柄
   ↓
 BetaEvaluator 运行时调用
 ```
 
 ### 5.2 关键类与职责
 
 | 类 | 包 | 职责 |
 |---|---|---|
 | `StringLiteralEncoder` | `runtime.compiler` | 将源码中的字符串常量编码为 `${constN}`，安全移除空白后还原 |
 | `DefaultLiteralSourceCompiler` | `runtime.compiler` | 生成符合 `javac` 语法要求的 Java 源文件（含 `@Where` 表达式的方法体） |
 | `DefaultSourceCompiler` | `spi.minimal.compiler` | 调用 JDK Compiler API 执行编译 |
 | `InMemoryFileManager` | `spi.minimal.compiler` | `ForwardingJavaFileManager` 子类，编译结果只写内存不写磁盘 |
 | `ClassLoaderWrapper` | `spi.minimal.compiler` | 隔离的类加载器，避免类冲突 |
 | `BetaEvaluator` | `runtime.evaluation` | 运行时通过 `MethodHandle` 调用编译后的条件方法 |
 
 ### 5.3 为什么需要 StringLiteralEncoder
 
 `@Where` 表达式经过字符串处理（空白移除、变量名替换等），这些操作会破坏字符串字面量内部的结构。例如：
 
 ```
 // 原始: @Where("$c.name.equals(\"John Doe\")")
 // 直接移除空白后: @Where($c.name.equals("JohnDoe"))  ← 两个单词间的空格也没了
 ```
 
 `StringLiteralEncoder` 在空白移除前将所有字符串常量替换为占位符，处理后还原，避免了这个问题。
 
 ---
 
 ## 六、关键时序图（文本版）
 
 ```
 insert(x)
   │
   ▼
 bufferInsertMultiple ──→ WorkMemoryActionBuffer
   │                        (actionBuffer)
   │
   ▼
 session.fire()
   │
   ▼
 fireCycle(ctx, mode, buffer)            ───────────────┐
   │                                                      │
   ▼                                                      │
 computeDelta(buffer)                                     │
   │                                                      │
   ├─ sinkToSplitView() → SplitView[]                     │
   ├─ processDeleteActions() →  [异步] 清理 alpha/beta    │
   └─ processDeltaStatus()                                │
        ├─ 分组 insert → TypeAlphaMemory [异步]           │
        ├─ insertMask & 规则匹配                           │
        └─ SessionFactGroup.buildDeltas() [异步]           │
             └─ ReteSessionConditionNode.computeDelta()    │
                  ├─ BetaEvaluator 评估条件                │
                  └─ ConditionMemory = 匹配结果集          │
   │                                                      │
   ▼                                                      │
 doAgenda(ctx, agenda, mode)                              │
   ├─ SessionRule.callRhs() → 用户代码执行                │
   └─ RHS 内 insert/update/delete → new buffer            │
   │                                                      │
   ▼                                                      │
 commitMemories(deltaStatus)                              │
   ├─ SessionFactGroup.commitDeltas()                     │
   └─ TypeAlphaMemory.commit()                            │
   │                                                      │
   ▼                                                      │
 fireCycle(ctx, mode, newBuffer) ─── buffer 空? ──→ 结束  │
   │                                                      │
   └──────────────────────────────────────────────────────┘
         (递归直到 RHS 不再产生新动作)
 ```
 
 ---
 
 ## 七、设计精要
 
 | 特性 | 效果 |
 |---|---|
 | **增量计算** | fireCycle 只处理缓冲区中的新变化，不重新计算整个 RETE 网络 |
 | **异步编排** | `CompletableFuture.thenCompose` + `CommonUtils.completeAll`，各阶段可并行 |
 | **Mask 位运算** | `Mask<AlphaAddress>.intersects()` 在 O(1) 时间内确定哪些规则被激活 |
 | **`LongKeyMap`** | 事实句柄 ID 查找避免 `Long` 自动装箱的 GC 开销 |
 | **递归 fireCycle** | 替代传统 `do{}while`，RHS 新动作自然进入下一个循环 |
 | **Copy-on-Write 队列** | SplitView 创建后消费，写入的新 buffer 不影响当前 cycle |
 
 ---
 
 *本文为学习路线图中"高优先级补充文档"的第一个实现，对应 learning-guide.md 中的"数据流全景追踪"方向。*
