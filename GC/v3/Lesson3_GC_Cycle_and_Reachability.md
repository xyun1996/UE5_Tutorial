# Lesson 3: GC 主流程与可达性分析

## 这节课解决什么问题

前两节已经建立了:

- GC 管理边界
- 引用图如何被建立

这一节开始回答主流程问题:

> 一次 UE5 GC 是如何从“开始”走到“对象被最终清理”的?

这是整套原理的主轴。  
如果你不掌握这条主轴, cluster、incremental、调优都只能是碎知识。

---

## 一、先纠正一个过度简化

很多资料会说:

> UE GC 就是 mark-sweep

这句话在算法家族层面成立, 但在工程实现层面不够用。  
当前 UE5 更像是这样一条流水线:

1. GC 入口与预处理
2. `MarkObjectsAsUnreachable`
3. 标记 keep / root / cluster root
4. `PerformReachabilityAnalysis`
5. 收集仍不可达对象
6. 清理弱引用
7. `ConditionalBeginDestroy`
8. `FinishDestroy` / `IncrementalPurgeGarbage`

所以更准确的记忆方式是:

> UE5 使用以可达性分析为核心的分阶段 GC 流程。

---

## 二、GC 开始前不是立刻扫描对象

### 2.1 为什么会有 `TryCollectGarbage`

公开接口里既有:

```cpp
CollectGarbage(...)
TryCollectGarbage(...)
```

这说明 GC 还受运行时状态影响:

- 当前是否能拿到 GC 锁
- 是否有线程在改 UObject 状态
- 是否要先处理上一轮 pending 流程

### 2.2 学习重点

GC 不只是“对象算法”, 还是“引擎同步与调度”问题。  
这也是为什么线上项目中 GC hitch 有时不仅仅是对象数量造成的。

---

## 三、`MarkObjectsAsUnreachable` 在做什么

### 3.1 源码位置

- `Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollection.cpp`

### 3.2 关键函数名

```cpp
MarkObjectsAsUnreachable
```

### 3.3 很多人对这个阶段的误解

误解:

- “这一步已经把垃圾精确找出来了”

更准确的理解:

- 它先把大批对象放入“可能不可达”的候选状态
- 然后再由后续 reachability traversal 把真正可达的对象翻回来

### 3.4 为什么这样做

因为 UE5 GC 的主判定问题不是:

- “哪个对象看起来像垃圾”

而是:

- “哪个对象能从根走到”

这件事必须通过后续 traversal 才能回答。

---

## 四、Root / Keep / Cluster Root 是 Reachability 的起点

在“铺底不可达”之后, 还需要把一批天然应保活的起点纳入 reachable 侧:

- root set 对象
- `KeepFlags` 命中的对象
- cluster root
- 某些外部根与全局保活路径

你可以把这个过程理解成:

```text
先把大地涂成“可能不可达”
再从一批已知起点向外扩展“真正存活区域”
```

这个比喻虽然简单, 但很适合理解 UE5 GC 的流程感。

---

## 五、`PerformReachabilityAnalysis` 是整轮 GC 的核心

### 5.1 核心类与函数

在源码里你会遇到:

- `FRealtimeGC`
- `PerformReachabilityAnalysis`
- `PerformReachabilityAnalysisOnObjectsInternal`

### 5.2 这一阶段做什么

从根集合出发, 沿着 GC 图不断扩展:

- 读取对象类的 `ReferenceSchema`
- 遍历普通对象引用
- 进入数组、struct、set/map 等容器
- 在需要时调用 `AddReferencedObjects`
- 处理 cluster 相关引用
- 处理外部 tracer / 其他桥接路径

最终得到:

- 哪些对象真正可达

### 5.3 为什么它是 GC 原理课的中心

因为它同时决定:

- GC 的正确性
- GC 的大部分前半段性能成本

后面所有“parallel / cluster / incremental”都和它直接有关。

---

## 六、一个对象图示意

```text
GameInstance (root)
  -> WorldSubsystem
      -> MatchStateObject
          -> CurrentPhaseAsset
              -> HUDTexture
```

如果这些边都在 GC 图里, `HUDTexture` 就可达。  
如果中间一段不是 GC 可见引用:

```text
GameInstance
  -> WorldSubsystem
      -> [native raw ptr, GC不可见]
          -> CurrentPhaseAsset
```

后半段对象就可能被误判为不可达。

这个图示值得反复记, 因为它把 Lesson 2 和 Lesson 3 串起来了。

---

## 七、Reachability 结束后, 才轮到“真正的垃圾处理”

当 traversal 完成后:

- 仍不可达的对象才会进入后续清理路径

后面的事情一般包括:

- 收集 unreachable objects
- 从一些查找结构解绑
- 清理弱引用
- 进入销毁阶段

也就是说:

> Reachability 负责“判活”, 后段流程负责“收尾”。

---

## 八、`ConditionalBeginDestroy`、`FinishDestroy` 与 purge

### 8.1 为什么对象不能一判死就直接 delete

因为真实引擎对象经常还涉及:

- 资源释放
- 渲染线程协作
- 延迟收尾
- 分帧销毁

所以 UE5 的对象销毁通常会拆成多步:

- `ConditionalBeginDestroy`
- `BeginDestroy`
- `FinishDestroy`
- purge

### 8.2 这意味着什么

对象生命周期里至少要分三层看:

1. 逻辑上是否还从根可达
2. 是否已进入 GC 销毁流程
3. 是否已完成物理释放

这三层不是同时发生的。

---

## 九、为什么很多 GC Bug 都会呈现“过一会儿才炸”

典型原因就是:

- 某对象已经不可达
- 但还处于销毁收尾阶段
- 某处逻辑还错误地继续使用它

所以你常会看到这种现象:

- 某段代码之前都能跑
- 某次 GC 后突然崩

根因往往并不是“这一帧突然发生了神秘变化”, 而是:

- GC 在这轮终于把之前就不安全的引用关系暴露出来了

---

## 十、`IncrementalPurgeGarbage` 为什么是单独一段

源码公开接口附近可以看到:

- 增量 purge 相关能力

关键理解:

- 即使 traversal 做完了
- 后段销毁和 purge 也可能成为大尖峰

所以引擎把 purge 也设计成可分批推进。  
这件事到第五课会和 incremental reachability 放在一起对照讲。

---

## 十一、主流程里最容易混淆的几组概念

### 11.1 `KeepFlags` vs RootSet

- `KeepFlags`: 本轮保留策略输入
- RootSet: 对象处于根集合状态

### 11.2 Unreachable vs Destroyed

- unreachable: 当前 GC 判定下不可达
- destroyed: 已完成销毁与释放

### 11.3 Reachability vs Purge

- reachability: 判活
- purge: 收尾

如果这三组概念混了, 你基本不可能准确讲明白 GC。

---

## 十二、第三轮源码导读

建议阅读顺序:

1. `GarbageCollection.cpp` 中搜索 `CollectGarbage`
2. 搜 `MarkObjectsAsUnreachable`
3. 搜 `PerformReachabilityAnalysis`
4. 搜 `IncrementalPurgeGarbage`

### 带着这些问题去看

1. 哪一段负责开始新一轮 GC?
2. 哪一段负责“铺底不可达”?
3. 哪一段负责“从根扩展可达”?
4. 哪一段负责开始真正的后段清理?

---

## 十三、常见面试题

### 面试题 1

一次 UE5 GC 周期大致有哪些阶段?

### 回答要点

- 入口和预处理
- `MarkObjectsAsUnreachable`
- 可达性分析
- 收集不可达对象
- 弱引用清理
- destroy / purge

### 面试题 2

`MarkObjectsAsUnreachable` 是不是已经找出垃圾了?

### 回答要点

- 不是
- 它更像“先铺设可能不可达候选状态”
- 真正判活核心仍然是后续 reachability analysis

### 面试题 3

为什么 `BeginDestroy` 不等于对象已经彻底释放?

### 回答要点

- 因为 UE 的销毁收尾是分阶段的
- 逻辑死亡和物理释放分离

---

## 十四、本节自测

1. 为什么不能只用“mark-sweep”四个字概括 UE5 GC?
2. `MarkObjectsAsUnreachable` 的真实角色是什么?
3. Reachability Analysis 和 Purge 分别在解决什么问题?
4. 为什么对象已经不可达, 但内存不一定立刻释放?
5. 为什么 RootSet、KeepFlags、cluster root 需要区分?

---

## 十五、本节的一个结论句

> UE5 GC 的主逻辑不是“直接找垃圾”, 而是先铺设不可达候选状态, 再从根集合做可达性分析, 最后把仍不可达的对象送入多阶段销毁与 purge 流程。

---

## 源码锚点

- `Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollection.cpp`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectGlobals.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/GarbageCollection.h`
