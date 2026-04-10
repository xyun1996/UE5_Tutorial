# Lesson 3: 标记、可达性分析与清理阶段

## 本节定位

前两节解决了两个基础问题:

- GC 管什么
- 引用图怎么建立

这一节开始把它们串成时间线:

> 一次 GC 到底是怎么从“开始”走到“对象真正释放”的?

这一节会比第一、二节更像“主流程课”, 是后面 cluster 和 incremental 的基础。

---

## 学习目标

学完本节你应该能明确:

- 为什么 UE5 GC 不能被粗暴理解为“一遍 mark-sweep”
- `MarkObjectsAsUnreachable` 先做了什么
- `PerformReachabilityAnalysis` 才真正完成什么判定
- 不可达对象后面要经历哪些阶段
- `BeginDestroy`、`FinishDestroy`、purge 各在什么语义层面
- 为什么“对象死了”和“对象内存立刻释放”不是同一时刻

---

## 一、先纠正一个常见误解

很多资料会写:

> UE GC 就是 mark-sweep。

这句话在算法分类层面没错, 但在工程实现层面太粗。  
当前 UE5 的 GC 更像是一个分阶段流水线:

1. 预处理与锁管理
2. `MarkObjectsAsUnreachable`
3. 标记 root / keep / cluster root 等初始可达对象
4. `PerformReachabilityAnalysis`
5. 收集仍不可达对象
6. 清理弱引用与后续引用残留
7. `ConditionalBeginDestroy`
8. `FinishDestroy` / purge

所以你更应该记住的是:

> UE5 使用以可达性分析为核心的分阶段 GC 流程。

---

## 二、GC 入口不只是“开始扫描对象”

在真正遍历对象之前, 引擎还要处理很多运行时问题:

- GC 锁
- UObject hash table 锁
- 异步加载协作
- 统计与回调
- 上一轮 pending purge 是否已经完成

这也是为什么存在:

```cpp
TryCollectGarbage(...)
```

它的存在本身就说明:

- GC 受当前引擎运行环境制约
- 不是随时都能无条件执行

从学习角度你要接受一个事实:

> GC 不只是算法逻辑, 还是一个同步与调度逻辑。

---

## 三、`MarkObjectsAsUnreachable` 的真正意义

### 3.1 源码位置

- `Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollection.cpp`

### 3.2 关键函数

```cpp
void MarkObjectsAsUnreachable(const EObjectFlags KeepFlags)
```

### 3.3 初学者最容易误解的地方

很多人会把这个函数理解成:

- “把垃圾对象找出来并确认”

这不准确。

更准确的说法是:

- 先把大批对象放进“可能不可达”的候选状态
- 同时根据 `KeepFlags`、root 等保留一部分对象
- 然后把真正的可达判定留给后面的 traversal

也就是说, 这是一个“先做保守铺底”的阶段。

### 3.4 为什么不直接一开始就精确判断

因为真正的问题不是:

- “这个对象眼前看上去像不像垃圾”

而是:

- “它能不能从根集合走到”

这个问题必须依赖引用图遍历来回答。

---

## 四、root / keep / cluster root 为什么要先处理

在进入正式 traversal 之前, 一批天然应保活的对象会被先纳入可达起点, 比如:

- RootSet 对象
- `KeepFlags` 命中的对象
- cluster root
- 某些外部系统提供的根

这决定了整个 GC 的出发点不是“所有对象都同等可疑”, 而是:

```text
从一批已知应保活的起点开始
  -> 向外扩展整个可达区域
```

这跟很多新手脑中“GC 从垃圾开始找”是相反的。  
UE 更像是:

- 先铺一层不可达底色
- 再把真正可达区域翻回来

---

## 五、真正的核心阶段: `PerformReachabilityAnalysis`

### 5.1 相关入口

你会在 GC 源码里看到这些关键词:

- `FRealtimeGC`
- `PerformReachabilityAnalysis`
- `PerformReachabilityAnalysisOnObjectsInternal`

### 5.2 这一步到底做什么

从根集合出发, 按引用边向外扩展:

- 读取对象类的 `ReferenceSchema`
- 遍历反射可见成员
- 进入数组、结构体、set/map 等容器内容
- 在需要时调用 `AddReferencedObjects`
- 处理 cluster 引用
- 处理外部 tracer / Verse 桥接路径
- 把遍历到的对象重新标记为 reachable

### 5.3 为什么这一步最值得学

因为 GC 的“正确性”和“性能”很多都集中在这里:

- 引用图是否完整
- 遍历成本是否高
- 哪些类扫起来慢
- 哪些对象图适合 cluster 或 parallel

后面你看到的大多数优化器, 本质上都在帮这一段。

---

## 六、一个示意例子: 可达路径如何保活对象

看一个简化对象图:

```text
GameInstance (Root)
  -> UInventorySubsystem
      -> UInventoryProfile
          -> UItemDefinition
              -> UTexture2D Icon
```

只要这条链上的边都属于 GC 可见引用:

- 整条链都可达
- 最终 `Icon` 也会被保活

但如果中间一段变成裸指针:

```text
GameInstance
  -> UInventorySubsystem
      -> [裸指针，GC看不见]
          -> UItemDefinition
```

那对 GC 来说, 后面的对象就像“图断了”。

这也是为什么 GC 课程不应该只讲“对象数量”, 还要讲“图是否连通且可见”。

---

## 七、Reachability 完成后, 才轮到“真正的垃圾”

可达性分析完成后, 剩下仍然处于不可达状态的对象, 才会进入后续回收路径。

这时引擎接下来要处理的事情包括:

- 收集不可达对象
- 从一些查找结构中解绑
- 清理弱引用
- 进入销毁和最终 purge 流程

也就是说:

> Reachability 是“判活”, 后续阶段才是“收尸”。

这个比喻虽然粗糙, 但有助于你区分阶段职责。

---

## 八、为什么 `BeginDestroy` 和“内存释放”不能混为一谈

在 UObject 生命周期里, 至少要区分:

- `ConditionalBeginDestroy`
- `BeginDestroy`
- `FinishDestroy`

### 8.1 为什么要分层

因为现实中对象销毁不是一个简单 `delete` 就完了:

- 有些对象要等渲染线程或其他线程清理资源
- 有些对象清理动作很重, 不能全部压在当前帧
- 引擎需要给销毁流程留出中间态

### 8.2 这能解释很多“看起来古怪”的现象

例如:

- 对象已经不可达, 但某些日志里还看得到它
- 弱引用已经失效了, 但对象内存还没立刻回收
- 清理流程跨越多帧完成

所以你要形成的正确理解是:

> “判死” 和 “物理释放” 是两个阶段, 中间可能隔着一整段销毁流程。

---

## 九、`IncrementalPurgeGarbage` 在哪里

源码中有:

```cpp
void IncrementalPurgeGarbage(bool bUseTimeLimit, double TimeLimit)
```

它属于 GC 后半段的清理能力。

### 9.1 它不是判活阶段

它不负责回答:

- “对象是不是可达”

它负责的是:

- 在对象已经不可达之后
- 如何把销毁和 purge 分批做完

### 9.2 为什么需要它

如果大量对象一起进入销毁阶段:

- 单帧很容易产生严重尖峰

所以引擎把 purge 也做成了可分批推进的形式。

---

## 十、`KeepFlags`、RootSet、Cluster 三者不要混

这是学习和面试里都特别容易混的点。

### 10.1 `KeepFlags`

本轮 GC 的保留策略输入。  
本质是“本轮有哪些 flag 命中的对象默认要留”。

### 10.2 RootSet

对象自身所处的根状态。  
它决定对象在引用图中是否被视作根起点。

### 10.3 Cluster

一种对象图压缩与遍历优化机制。  
它会影响对象如何被快速保活和批处理, 但它不是 root set 本身。

### 10.4 为什么一定要分清

否则你在分析“对象为什么活着”时, 会把三种不同原因混成一团。

---

## 十一、如何从三个层次理解“对象死了”

### 11.1 逻辑层

对象已经不再从根可达。

### 11.2 GC 状态层

对象已经被标记为不可达, 进入回收流水线。

### 11.3 内存层

对象的销毁、finish、purge 都完成, 内存真正释放。

这三个层次在时间上可能相隔很远。  
如果你不分层理解, 很多调试现象都会误判。

---

## 十二、两个典型排查方向

### 12.1 对象“本该死却没死”

优先怀疑:

- 根链路没断
- `KeepFlags` 在当前环境下保留了它
- `FGCObject` 还在上报它
- cluster 让它跟着一批对象一起活着

### 12.2 对象“本该活却死了”

优先怀疑:

- 裸指针没有接入 GC 图
- 弱引用被误当强引用
- native 持有者没有用 `FGCObject`
- 关键引用没有进入 schema / ARO 路径

---

## 十三、如何配合源码读这一课

建议顺序:

1. 在 `GarbageCollection.cpp` 中搜索 `MarkObjectsAsUnreachable`
2. 看 `PerformReachabilityAnalysis`
3. 看 `IncrementalPurgeGarbage`
4. 对照 `UObjectGlobals.h` 中的公开 API

### 13.1 阅读时建议回答这几个问题

- 哪一段负责“铺底不可达”
- 哪一段负责“从根翻正可达”
- 哪一段开始处理真正的 unreachable objects
- 哪一段会进入 begin destroy / purge

只要这几个问题能回答出来, 你对主流程就算真正入门了。

---

## 十四、本节后的自检题

1. `MarkObjectsAsUnreachable` 为什么不等于“已经确认垃圾”?
2. 为什么 `PerformReachabilityAnalysis` 才是 GC 主流程的核心?
3. `BeginDestroy` 和最终释放为什么不能合并理解?
4. 增量 purge 和增量 reachability 为什么要分开看?
5. `KeepFlags`、RootSet、Cluster 三者分别是什么?
6. 为什么“GC 流程”比“mark-sweep 四个字”更值得你记住?

---

## 十五、进入第四课前你应该达到的状态

如果你现在已经能自然说出下面这句话, 就可以进入 cluster:

> UE5 GC 不是简单“一遍扫完”的回收器, 而是先铺设不可达候选, 再从根做可达性分析, 最后把剩余不可达对象送入销毁与 purge 流程。

---

## 源码锚点

- `Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollection.cpp`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectGlobals.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/GarbageCollection.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/ReachabilityAnalysis.h`
