# Lesson 5: 性能架构：Schema、Parallel、Cluster、Incremental

## 这节课解决什么问题

如果前四节都偏“正确性与语义”, 这一节就聚焦:

> UE5 是如何尽量把 GC 做快、做平滑的

GC 优化不是某一个开关的功劳, 而是一整组机制协同的结果。  
这一节把最重要的四个方向放在一起看:

- schema 驱动 traversal
- parallel GC
- cluster
- incremental GC

---

## 一、先建立一个性能总图

GC 主要会在哪些地方花时间?

### 1.1 前半段

- 遍历对象
- 扩展引用图
- 确认哪些对象可达

### 1.2 后半段

- gather / unhash unreachable objects
- 清理弱引用
- begin destroy / finish destroy / purge

### 1.3 对应的优化器

- schema: 让 traversal 更高效
- parallel: 让 traversal / 处理能并行
- cluster: 压缩稳定对象图
- incremental: 把大块工作摊到多帧

---

## 二、Schema 为什么本质上也是性能架构的一部分

很多初学者把 schema 只当成“正确遍历引用”的工具。  
其实它同时是性能优化的基础。

### 2.1 为什么

如果没有 schema:

- GC 遍历时要更依赖反射临时解释
- 成员路径更难优化
- 对数组、struct、ARO 的处理也更难形成稳定执行模型

### 2.2 所以 schema 的双重作用

1. 正确表达引用布局
2. 为 traversal 提供高效执行计划

这也是为什么 `gc.DumpSchemaStats` 这种工具是有价值的。

---

## 三、Parallel GC 在解决什么问题

### 3.1 相关 CVar

在 `GarbageCollection.cpp` 中你会看到:

- `gc.AllowParallelGC`

### 3.2 它想解决什么

当对象和引用非常多时:

- 单线程 traversal 的 wall time 会很长

parallel 的核心思路就是:

- 尽可能把 traversal 和部分处理工作分给多个 worker

### 3.3 它不是永远稳赚

并行也有成本:

- 任务拆分开销
- 同步开销
- 工作不均衡
- 平台线程数限制

所以正确理解是:

> parallel 提供吞吐提升机会, 不是无条件收益。

---

## 四、Cluster 在性能架构中的位置

第四课已经讲了 cluster 的语义边界。  
这一节从性能角度再强调一次:

### 4.1 cluster 解决的是哪类成本

- 稳定对象群的重复 traversal 成本

### 4.2 它依赖的前提

- 对象图稳定
- 生命周期相近
- 同域 / 同包关系明确

### 4.3 它不适合的场景

- 高频动态关系
- 经常改 owner / 改引用拓扑
- 还没先控制对象数量

所以 cluster 是很强的优化器, 但不是第一优先级。

---

## 五、Incremental Reachability 在性能架构中的位置

### 5.1 相关 API 与 CVar

公开 API:

- `IsIncrementalReachabilityAnalysisPending`
- `PerformIncrementalReachabilityAnalysis`
- `FinalizeIncrementalReachabilityAnalysis`

关键 CVar:

- `gc.AllowIncrementalReachability`
- `gc.IncrementalReachabilityTimeLimit`

### 5.2 它解决的是什么

不是减少总工作量, 而是:

- 把 traversal 主阶段拆成多次迭代
- 摊平单帧尖峰

### 5.3 为什么它和 parallel 不冲突

parallel 更关注:

- 并发处理更多工作

incremental 更关注:

- 单帧不要吃掉太多时间

两者的目标维度不同。

---

## 六、后半段优化：Incremental Gather 与 Incremental BeginDestroy

在 `GarbageCollection.cpp` 中还能看到:

- `gc.AllowIncrementalGather`
- `gc.IncrementalGatherTimeLimit`
- `gc.IncrementalBeginDestroyEnabled`
- `gc.IncrementalBeginDestroyGranularity`

### 6.1 为什么后半段也要切片

很多项目会误以为:

- 只要 traversal 不慢, GC hitch 就没了

实际上:

- 大量不可达对象 gather
- 大量对象 begin destroy
- 后续 purge

这些都可能单独成为尖峰。

### 6.2 所以 UE 的做法

- 前半段可增量
- 后半段也可增量

这才是完整的“平滑 GC”思路。

---

## 七、为什么说 GC 优化首先是对象图优化

这句话要反复记:

> 所有执行层优化器, 都建立在对象图设计之上。

### 7.1 如果对象数过大

无论 parallel、cluster、incremental 再怎么调:

- 总成本还是会大

### 7.2 如果引用布局过于复杂

- schema traversal 仍然会重
- ARO 路径仍然会慢

### 7.3 如果销毁成本极高

- 后半段切片也只能摊平, 不能消灭成本

所以真正成熟的调优顺序是:

1. 减对象
2. 理引用
3. 再调执行层优化器

---

## 八、一个实用调优顺序

### 8.1 第一步：测量

先判断:

- traversal 慢不慢
- gather / destroy / purge 慢不慢

### 8.2 第二步：优化对象图

- 减少短命 UObject
- 把纯数据改为 struct
- 清理无意义强引用

### 8.3 第三步：执行层优化

- 开 parallel
- 评估 cluster
- 开 incremental reachability
- 开 incremental gather / begin destroy

### 8.4 第四步：精调参数

- 时间预算
- 粒度
- full purge 时机

---

## 九、第五轮源码导读

建议阅读顺序:

1. `GarbageCollectionSchema.h/.cpp`
2. `FastReferenceCollector.h`
3. `GarbageCollection.cpp` 中 parallel / incremental CVar
4. `UObjectClusters.cpp`
5. `ReachabilityAnalysisState.*`

### 带着这些问题去看

1. 哪些结构在帮助 traversal 更快?
2. parallel 和 incremental 分别优化什么?
3. cluster 压缩的是哪一类对象图?
4. 后半段切片是如何展开的?

---

## 十、常见面试题

### 面试题 1

UE5 GC 的主要性能优化方向有哪些?

### 回答要点

- schema 驱动 traversal
- parallel GC
- cluster
- incremental reachability
- incremental gather / begin destroy / purge

### 面试题 2

为什么说 schema 也是性能优化的一部分?

### 回答要点

- 它不仅描述正确性
- 也给 traversal 提供稳定高效的布局访问方式

### 面试题 3

parallel 和 incremental 的区别是什么?

### 回答要点

- parallel 提高吞吐
- incremental 摊平单帧尖峰

### 面试题 4

为什么 cluster 不应该作为 GC 优化的第一反应?

### 回答要点

- 它依赖稳定对象图
- 先控制对象数量和引用结构通常收益更大

---

## 十一、本节自测

1. schema 为什么既是正确性机制, 也是性能机制?
2. parallel GC 解决的主要问题是什么?
3. incremental reachability 和 incremental begin destroy 分别优化哪段?
4. cluster 为什么更适合稳定 cooked 资源图?
5. 为什么对象图治理总是优先于参数调优?

---

## 十二、本节的一个结论句

> UE5 GC 的性能不是靠单个开关换来的, 而是依赖“对象图设计 + schema traversal + parallel + cluster + incremental”这一整套协同架构。

---

## 源码锚点

- `Engine/Source/Runtime/CoreUObject/Public/UObject/GarbageCollectionSchema.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/FastReferenceCollector.h`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollection.cpp`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/UObjectClusters.cpp`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/ReachabilityAnalysisState.cpp`
