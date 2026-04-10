# Appendix 2: UE5 GC 源码导读

## 这份附录怎么用

这不是“源码清单”, 而是一份导读路线。  
目的不是让你一次把所有文件看完, 而是告诉你:

- 先看哪一批文件最容易建立全局感
- 哪些文件是概念入口
- 哪些文件是主流程核心
- 哪些文件适合在第二遍、第三遍再读

建议你在学习正文六节课时, 把这份导读和 UnrealEngine 仓库一起开着看。

本导读基于你本地仓库:

- `G:\workspace\repo\github\UnrealEngine`

---

## 一、先建立总地图

如果你只想先知道“GC 相关代码主要分布在哪”, 可以先记这一层:

### 1.1 公开接口与数据结构

- `Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectGlobals.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/GarbageCollection.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectArray.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/GCObject.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/GarbageCollectionSchema.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/FastReferenceCollector.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/ReachabilityAnalysis.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/ObjectPtr.h`

### 1.2 主实现文件

- `Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollection.cpp`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollectionSchema.cpp`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/ReachabilityAnalysisState.cpp`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/UObjectClusters.cpp`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/UObjectArray.cpp`

### 1.3 相关系统协作文件

- `Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncLoading.cpp`
- `Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncLoading2.cpp`
- `Engine/Source/Runtime/Engine/Private/Level.cpp`
- `Engine/Source/Runtime/Engine/Private/LevelActorContainer.cpp`

---

## 二、第一遍源码阅读: 只建立框架

第一遍不要追求“全部看懂”, 目标只有一个:

> 认识 GC 系统有哪些大块。

### 推荐顺序

1. `UObjectGlobals.h`
2. `GarbageCollection.h`
3. `UObjectArray.h`
4. `GCObject.h`
5. `GarbageCollectionSchema.h`
6. `ReachabilityAnalysis.h`

### 为什么这样排

因为这些文件主要回答:

- GC 的入口是什么
- 核心数据结构是什么
- 系统使用哪些公开术语

### 第一遍你只需要回答这些问题

1. GC 的主入口函数是什么?
2. `GUObjectArray` 是什么?
3. `FUObjectItem` 是什么?
4. `FGCObject` 解决什么问题?
5. `FSchemaView` 是什么?
6. 增量 reachability 暴露了哪些 API?

如果这 6 个问题能回答出来, 第一遍就算合格。

---

## 三、第二遍源码阅读: 主流程

当你已经知道大框架后, 第二遍重点读:

- `GarbageCollection.cpp`

### 建议的搜索顺序

在文件里按下面顺序搜:

1. `CollectGarbage`
2. `TryCollectGarbage`
3. `MarkObjectsAsUnreachable`
4. `PerformReachabilityAnalysis`
5. `IncrementalPurgeGarbage`
6. `SetIncrementalReachabilityAnalysisEnabled`
7. `PerformIncrementalReachabilityAnalysis`

### 读这一遍时要带的问题

1. 哪一段负责“开始一轮 GC”?
2. 哪一段负责“把对象放进可能不可达状态”?
3. 哪一段负责“从根扩展可达区域”?
4. 哪一段负责“后段 purge”?
5. 哪些 CVar 在控制不同阶段?

### 这一遍常见陷阱

陷阱 1:

- 一上来就想逐行看完 `GarbageCollection.cpp`

结果:

- 细节太多
- 主线反而丢了

更好的做法:

- 先用关键函数把骨架串出来

---

## 四、第三遍源码阅读: 引用图与 schema

当你已经能看懂主流程后, 第三遍重点补:

- `GarbageCollectionSchema.h`
- `GarbageCollectionSchema.cpp`
- `FastReferenceCollector.h`

### 这一遍重点看什么

#### 4.1 `GarbageCollectionSchema.h`

重点关注:

- `EMemberType`
- `FSchemaView`
- `FSchemaOwner`
- schema 中哪些成员类型对应哪些引用关系

#### 4.2 `GarbageCollectionSchema.cpp`

重点关注:

- schema 如何构建
- schema 如何被复用
- `gc.DumpSchemaStats` 背后的统计思路

#### 4.3 `FastReferenceCollector.h`

重点关注:

- 遍历器怎么消费 schema
- 并行 / 非并行选项如何区分
- 哪些对象路径会走 ARO

### 这一遍读完后, 你应该能回答

1. UE5 为什么需要 schema?
2. `AddReferencedObjects` 在 schema 体系里是什么位置?
3. 为什么 GC 成本不只是对象数量, 还和布局复杂度有关?

---

## 五、第四遍源码阅读: 增量 GC

重点读:

- `ReachabilityAnalysis.h`
- `ReachabilityAnalysisState.h`
- `ReachabilityAnalysisState.cpp`
- `ObjectPtr.h`
- `GarbageCollection.cpp` 中 incremental 相关部分

### 这一遍重点看什么

#### 5.1 状态机

看:

- `FReachabilityAnalysisState` 保存了哪些状态
- 如何暂停
- 如何继续

#### 5.2 时间预算

看:

- `gc.AllowIncrementalReachability`
- `gc.IncrementalReachabilityTimeLimit`
- `gc.AllowIncrementalGather`
- `gc.IncrementalBeginDestroyEnabled`

#### 5.3 barrier

看:

- `ObjectPtr.h` 里 `UE::GC::GIsIncrementalReachabilityPending`
- `UE::GC::MarkAsReachable(...)`

### 这一遍读完后, 你应该能回答

1. 为什么 incremental 不是“把 for 循环切几段”这么简单?
2. 为什么增量 GC 需要 barrier?
3. incremental reachability 和 incremental purge 分别在哪段?

---

## 六、第五遍源码阅读: cluster

重点读:

- `UObjectArray.h`
- `UObjectBaseUtility.h`
- `UObjectClusters.cpp`
- `Level.cpp`
- `LevelActorContainer.cpp`

### 这一遍重点看什么

#### 6.1 cluster 结构

看:

- `FUObjectCluster`
- `FUObjectClusterContainer`

#### 6.2 cluster 构建条件

看:

- `CanBeClusterRoot`
- `CanBeInCluster`
- `CreateCluster`
- `AddToCluster`

#### 6.3 cluster 失效与 dissolve

看:

- `DissolveCluster`
- `DissolveClusterAndMarkObjectsAsUnreachable`
- `DissolveClusters`

### 这一遍读完后, 你应该能回答

1. cluster 为什么更偏向稳定 cooked 资源图?
2. 为什么 cluster 会 dissolve?
3. `MutableObjects` 说明了什么?

---

## 七、文件级导读卡片

下面给你一组“文件看什么”的速查卡片。

### 7.1 `UObjectGlobals.h`

看点:

- `CollectGarbage`
- `TryCollectGarbage`
- `IsIncrementalReachabilityAnalysisPending`
- `PerformIncrementalReachabilityAnalysis`

一句话定位:

- GC 的公开入口与常用 API 门面

### 7.2 `GarbageCollection.h`

看点:

- `GARBAGE_COLLECTION_KEEPFLAGS`
- GC 基础声明

一句话定位:

- 公开层的 GC 基础定义文件

### 7.3 `UObjectArray.h`

看点:

- `FUObjectItem`
- `FUObjectArray`
- `FUObjectCluster`
- `FUObjectClusterContainer`

一句话定位:

- UObject 目录与 cluster 数据结构总表

### 7.4 `GCObject.h`

看点:

- `FGCObject`
- `UGCObjectReferencer`

一句话定位:

- native 世界进入 GC 图的桥

### 7.5 `GarbageCollectionSchema.h`

看点:

- `EMemberType`
- `FSchemaView`
- `FSchemaOwner`

一句话定位:

- UE5 GC 的引用布局描述核心

### 7.6 `FastReferenceCollector.h`

看点:

- `EGCOptions`
- worker / block / queue 相关结构

一句话定位:

- 遍历器与 traversal 执行模型

### 7.7 `ReachabilityAnalysisState.h/.cpp`

看点:

- 增量 traversal 状态
- suspend / continue 流程

一句话定位:

- incremental reachability 的状态机

### 7.8 `ObjectPtr.h`

看点:

- incremental 期间的 reachable barrier 路径

一句话定位:

- 现代 UObject 指针与增量 GC 的连接点

### 7.9 `UObjectClusters.cpp`

看点:

- cluster 创建
- mutable object 处理
- dissolve 路径

一句话定位:

- cluster 的主实现

---

## 八、源码阅读时推荐的笔记模板

建议你自己准备一个 Markdown 或 Notion 页面, 每看一个文件记三类内容:

### 8.1 这个文件定义了什么

例如:

- 公开 API
- 数据结构
- 状态机

### 8.2 这个文件解决什么问题

例如:

- 管入口
- 管 traversal
- 管 cluster
- 管增量状态

### 8.3 这个文件最容易和谁混

例如:

- `KeepFlags` vs RootSet
- Reachability vs Purge
- ARO vs Schema
- Incremental Reachability vs Incremental Gather

这样你第二遍、第三遍回顾时会快很多。

---

## 九、一个推荐的完整阅读路线

如果你要花 2 到 3 天做一次系统源码导读, 推荐这样安排:

### Day 1

- `UObjectGlobals.h`
- `GarbageCollection.h`
- `UObjectArray.h`
- `GCObject.h`
- `GarbageCollectionSchema.h`

目标:

- 建立总框架

### Day 2

- `GarbageCollection.cpp`
- `FastReferenceCollector.h`
- `GarbageCollectionSchema.cpp`

目标:

- 看懂主流程和 traversal 骨架

### Day 3

- `ReachabilityAnalysisState.h/.cpp`
- `ObjectPtr.h`
- `UObjectClusters.cpp`
- `Level.cpp`

目标:

- 看懂 incremental 和 cluster 两个优化器

---

## 十、阅读时最常见的三个坑

### 10.1 坑 1: 逐行硬啃 `GarbageCollection.cpp`

后果:

- 细节很多
- 主线反而没建立起来

### 10.2 坑 2: 只看函数名, 不结合课件心智模型

后果:

- 知道有这个函数
- 但不知道它为什么存在

### 10.3 坑 3: 只看主流程, 不看 schema / pointer / cluster 配套文件

后果:

- 会把 UE5 GC 误学成“旧式 mark-sweep”

---

## 十一、阅读完成后的自检题

1. 如果你只允许读 5 个文件, 你会选哪 5 个建立 GC 总体认知?
2. `GarbageCollection.cpp` 里你最先该搜哪几个关键词?
3. 为什么 `ObjectPtr.h` 在增量 GC 学习中很重要?
4. 为什么看 cluster 时必须同时看 `UObjectBaseUtility.h` 和 `UObjectClusters.cpp`?
5. 为什么单看公开 API 不足以理解 UE5 GC 的真实成本来源?

---

## 十二、和正文六节课的对应关系

- Lesson 1 对应:
  `UObjectGlobals.h`、`GarbageCollection.h`、`UObjectArray.h`
- Lesson 2 对应:
  `GarbageCollectionSchema.h/.cpp`、`GCObject.h`、`FastReferenceCollector.h`
- Lesson 3 对应:
  `GarbageCollection.cpp`
- Lesson 4 对应:
  `UObjectClusters.cpp`、`UObjectBaseUtility.h`
- Lesson 5 对应:
  `ReachabilityAnalysisState.*`、`ObjectPtr.h`
- Lesson 6 对应:
  主流程文件加调试相关 CVar 与统计路径

如果你按这个映射读, 正文和源码会很容易互相对上。
