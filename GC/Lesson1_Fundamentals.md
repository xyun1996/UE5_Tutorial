# Lesson 1: UE5 垃圾回收基础架构

## 课程概述

本节课介绍UE5垃圾回收系统的基本概念、设计目标和核心架构，为后续深入学习奠定基础。

**学习目标：**
- 理解UE5为什么需要GC系统
- 掌握GC的核心设计目标
- 了解GC系统的核心组件和架构
- 熟悉UObject的生命周期管理
- 掌握GC执行流程的五个阶段

---

## 一、为什么UE需要GC？

### 1.1 游戏引擎的内存管理挑战

UE5作为游戏引擎，面临独特的内存管理挑战：

```
┌─────────────────────────────────────────────────────────────────┐
│                    UE5 内存管理挑战                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 对象生命周期复杂                                             │
│     ├── Asset加载/卸载时机不确定                                 │
│     ├── Actor在关卡切换时销毁                                    │
│     ├── 组件动态创建/销毁                                        │
│     └── 蓝图动态创建的对象                                       │
│                                                                 │
│  2. 引用关系复杂                                                 │
│     ├── UObject之间相互引用                                      │
│     ├── 跨模块引用                                               │
│     ├── 蓝图动态创建的引用                                       │
│     └── 软引用/弱引用/延迟加载                                   │
│                                                                 │
│  3. 性能要求严苛                                                 │
│     ├── 60fps = 16.67ms/帧                                      │
│     ├── GC不能造成明显卡顿                                       │
│     ├── 需要增量执行                                             │
│     └── 内存占用需要可控                                         │
│                                                                 │
│  4. 开发效率需求                                                 │
│     ├── 蓝图开发者无需关心内存管理                               │
│     ├── 减少内存泄漏风险                                         │
│     └── 简化C++开发                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 GC vs 手动内存管理

| 方面 | 手动管理 | GC自动管理 |
|------|----------|------------|
| 内存泄漏 | 常见问题 | 自动避免 |
| 悬空指针 | 容易出现 | 理论上消除 |
| 开发效率 | 需要精心设计 | 简化开发 |
| 运行性能 | 更可控 | 需要优化 |
| 确定性 | 完全确定 | 非确定性销毁 |
| 学习曲线 | 陡峭 | 平缓 |

### 1.3 UE5 GC的历史演进

```
┌─────────────────────────────────────────────────────────────────┐
│                    UE GC 系统演进                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  UE4 早期                                                        │
│  ├── 基础标记-清除算法                                           │
│  ├── Stop-The-World 模式                                        │
│  └── 简单的聚类支持                                              │
│                                                                 │
│  UE4 中期                                                        │
│  ├── 引入增量GC                                                  │
│  ├── 并行标记支持                                                │
│  └── 改进的聚类系统                                              │
│                                                                 │
│  UE5 (当前)                                                      │
│  ├── Schema-based引用遍历                                        │
│  ├── 完善的增量可达性分析                                        │
│  ├── 高效的并行处理                                              │
│  ├── 智能聚类优化                                                │
│  └── Verse VM集成                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、UE5 GC设计目标

### 2.1 核心设计原则

```cpp
// UE5 GC的核心设计目标
// Source: Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollection.cpp

/*
 * 1. 安全性 (Safety)
 *    - 防止访问已销毁对象
 *    - WeakObjectPtr 提供安全的弱引用
 *    - 对象销毁时自动清空弱引用
 *    - 避免悬空指针和野指针
 *
 * 2. 性能 (Performance)
 *    - 最小化帧时间影响
 *    - 增量GC (Incremental GC) - 分帧执行
 *    - 并行GC (Parallel GC) - 多线程加速
 *    - 聚类优化 (Clustering) - 批量处理
 *
 * 3. 可预测性 (Predictability)
 *    - 确定的GC行为
 *    - 可配置的GC时机
 *    - 时间限制保证
 *    - 开发者可控
 *
 * 4. 兼容性 (Compatibility)
 *    - FGCObject 支持非UObject参与GC
 *    - Verse VM集成
 *    - 蓝图无缝支持
 *    - 跨版本兼容
 */
```

### 2.2 GC触发时机

```cpp
// GC自动触发条件
// Source: GarbageCollection.cpp

void FGCSystem::PerformGarbageCollection()
{
    // 1. 内存压力触发
    // 当内存使用超过阈值时触发
    if (GMalloc->GetMemoryUsed() > gc.MemoryTriggerThreshold)
    {
        CollectGarbage();
    }

    // 2. 时间间隔触发 (配置: gc.TimeBetweenGCs)
    // 定期执行GC，防止垃圾对象积累
    if (TimeSinceLastGC > gc.TimeBetweenGCs)
    {
        CollectGarbage();
    }

    // 3. 手动触发场景
    // - StreamableManager加载完成后
    // - 关卡切换时
    // - PIE结束后
    // - 开发者调用 CollectGarbage()
}
```

### 2.3 GC策略选择

```
┌─────────────────────────────────────────────────────────────────┐
│                    GC 策略选择指南                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  场景                    推荐策略                                │
│  ─────────────────────────────────────────────────────────────  │
│  高帧率游戏 (60fps+)     增量GC + 小时间片                        │
│  开放世界游戏           聚类优化 + 更频繁GC                      │
│  移动端游戏             增量GC + 内存优先策略                    │
│  关卡加载时             完整GC (非增量)                          │
│  暂停菜单               完成挂起的增量GC                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 三、核心架构概览

### 3.1 GC系统核心组件

```
┌─────────────────────────────────────────────────────────────────┐
│                    UE5 GC 核心架构图                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    CollectGarbage()                      │   │
│  │                    GC主入口函数                           │   │
│  └─────────────────────────┬───────────────────────────────┘   │
│                            │                                    │
│                            ▼                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  FGCObjectPool                           │   │
│  │                  对象池管理器                             │   │
│  └─────────────────────────┬───────────────────────────────┘   │
│                            │                                    │
│         ┌──────────────────┼──────────────────┐               │
│         │                  │                  │               │
│         ▼                  ▼                  ▼               │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐         │
│  │FUObjectArray│    │FUObjectItem│    │  Clusters  │         │
│  │ 全局对象池  │    │ 对象元数据  │    │  聚类系统   │         │
│  └──────┬─────┘    └────────────┘    └────────────┘         │
│         │                                                      │
│         ▼                                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │           FReachabilityAnalysisState                     │   │
│  │           可达性分析状态机                                │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │   │
│  │  │  Idle   │─▶│Marking  │─▶│Traverse │─▶│Completed│    │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘    │   │
│  └─────────────────────────┬───────────────────────────────┘   │
│                            │                                    │
│                            ▼                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              TFastReferenceCollector                     │   │
│  │              快速引用遍历器                               │   │
│  │  ┌─────────────────┐    ┌─────────────────┐            │   │
│  │  │  Schema View    │───▶│ Reference Batch │            │   │
│  │  │  引用Schema     │    │  引用批处理      │            │   │
│  │  └─────────────────┘    └─────────────────┘            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 关键类与职责

| 类名 | 职责 | 文件位置 |
|------|------|----------|
| `FUObjectArray` | 管理所有UObject的全局数组，索引分配与回收 | `UObjectArray.h` |
| `FUObjectItem` | 单个对象的GC元数据，包含标志位和引用计数 | `UObjectArray.h` |
| `FReachabilityAnalysisState` | 增量可达性分析状态机，控制GC阶段转换 | `ReachabilityAnalysisState.h` |
| `FRealtimeGC` | 执行标记阶段遍历的核心类 | `GarbageCollection.cpp` |
| `TFastReferenceCollector` | 高效引用图遍历模板类 | `FastReferenceCollector.h` |
| `FGCObject` | 非UObject类参与GC的基类 | `GCObject.h` |
| `FUObjectCluster` | 聚类数据结构，批量处理相关对象 | `UObjectClusters.h` |

### 3.3 GC入口函数

```cpp
// GC主入口函数
// Location: Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollection.cpp

/**
 * 执行垃圾回收
 *
 * @param KeepFlags    - 需要保留的对象标志组合
 * @param bPerformFullPurge - 是否执行完整清除
 *                            true: 完整清除，阻塞直到完成
 *                            false: 可能增量执行
 *
 * 示例调用:
 *   CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS, true);  // 完整GC
 *   CollectGarbage(RF_Standalone, false);                // 增量GC
 */
void CollectGarbage(EObjectFlags KeepFlags = EObjectFlags::NoFlags,
                    bool bPerformFullPurge = false);

// 常用KeepFlags组合
#define GARBAGE_COLLECTION_KEEPFLAGS (RF_Standalone | RF_Public)
```

### 3.4 全局对象数组

```cpp
// 全局对象数组 - UE5 GC的核心数据结构
// Source: UObjectArray.h

// 全局实例声明
extern COREUOBJECT_API FUObjectArray GUObjectArray;

// 遍历所有对象的常用模式
for (int32 i = 0; i < GUObjectArray.GetObjectArrayNum(); ++i)
{
    FUObjectItem* Item = GUObjectArray.IndexToObject(i);
    if (Item && Item->Object)
    {
        UObject* Obj = Item->Object;
        // 处理对象...
    }
}
```

---

## 四、UObject生命周期

### 4.1 对象状态转换

```
┌─────────────────────────────────────────────────────────────────┐
│                    UObject 生命周期状态机                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                        NewObject()                              │
│                           │                                     │
│                           ▼                                     │
│   ┌──────────────────────────────────────────────────────┐    │
│   │                    Active (活跃)                       │    │
│   │  - 对象正常使用中                                      │    │
│   │  - 被根对象或其他可达对象引用                          │    │
│   │  - GC周期后仍存活                                      │    │
│   └──────────────────────┬───────────────────────────────┘    │
│                          │                                     │
│                          │ 引用断开 (成为垃圾)                  │
│                          ▼                                     │
│   ┌──────────────────────────────────────────────────────┐    │
│   │              Unreachable (不可达)                      │    │
│   │  - 没有任何根对象可达                                  │    │
│   │  - 被标记为待回收                                      │    │
│   │  - 等待BeginDestroy调用                                │    │
│   └──────────────────────┬───────────────────────────────┘    │
│                          │                                     │
│                          │ BeginDestroy()                      │
│                          ▼                                     │
│   ┌──────────────────────────────────────────────────────┐    │
│   │              PendingDestroy (待销毁)                   │    │
│   │  - BeginDestroy已调用                                 │    │
│   │  - 等待渲染线程等完成清理                              │    │
│   │  - 等待IsReadyForFinishDestroy()返回true              │    │
│   └──────────────────────┬───────────────────────────────┘    │
│                          │                                     │
│                          │ FinishDestroy()                     │
│                          ▼                                     │
│   ┌──────────────────────────────────────────────────────┐    │
│   │                  Destroyed (已销毁)                    │    │
│   │  - FinishDestroy已调用                                │    │
│   │  - 内存已释放                                         │    │
│   │  - 对象索引已回收                                      │    │
│   └──────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 对象标志位详解

```cpp
// 内部对象标志 - GC使用
// Source: Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectArray.h

enum EInternalObjectFlags
{
    // ===== 可达性标志 (在GC周期之间切换) =====
    ReachabilityFlag0    = 0x00400000,  // 当前"可达"标志
    ReachabilityFlag1    = 0x00800000,  // 当前"可能不可达"标志
    ReachabilityFlag2    = 0x01000000,  // 用于增量GC中间状态

    // ===== GC状态标志 =====
    Unreachable          = 0x02000000,  // 对象不可达，待回收
    Garbage              = 0x04000000,  // 对象已标记为垃圾
    Async                = 0x08000000,  // 异步处理中

    // ===== 聚类相关标志 =====
    ClusterRoot          = 0x10000000,  // 是聚类根对象
    ReachableInCluster   = 0x20000000,  // 聚类内对象可达
    WithinCluster        = 0x40000000,  // 在某个聚类内

    // ===== 标志组合 =====
    ReachabilityFlags = ReachabilityFlag0 | ReachabilityFlag1 | ReachabilityFlag2,
    ClusterFlags = ClusterRoot | ReachableInCluster | WithinCluster,
    RootFlags = Garbage | Async  // 根对象不应该有的标志
};
```

### 4.3 公开对象标志

```cpp
// 公开对象标志 - 开发者可使用
// Source: Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectBase.h

enum EObjectFlags
{
    RF_NoFlags                   = 0x00000000,  // 无标志

    // ===== 生命周期标志 =====
    RF_Public                    = 0x00000001,  // 公开对象，任何包可访问
    RF_Standalone                = 0x00000002,  // 独立对象，不随Outer销毁
    RF_Transient                 = 0x00000004,  // 临时对象，不保存
    RF_RootSet                   = 0x00000008,  // 在根集合中，永不GC

    // ===== GC相关标志 =====
    RF_Native                    = 0x00000100,  // 原生对象
    RF_MarkAsNative              = 0x00000200,  // 标记为原生
    RF_Garbage                   = 0x00002000,  // 垃圾对象
    RF_PendingKill               = 0x00004000,  // 待销毁
    RF_BeginDestroyed            = 0x00010000,  // BeginDestroy已调用
    RF_FinishDestroyed           = 0x00020000,  // FinishDestroy已调用

    // ===== 常用组合 =====
    RF_AllFlags                  = 0xFFFFFFFF,  // 所有标志
    RF_ContextFlags              = RF_Public | RF_Standalone | RF_Transient,
};
```

---

## 五、GC执行流程总览

### 5.1 完整GC周期

```
┌─────────────────────────────────────────────────────────────────┐
│                    完整 GC 周期                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ══════════════════════════════════════════════════════════    │
│  Phase 1: PRE-COLLECT (准备阶段)                                │
│  ══════════════════════════════════════════════════════════    │
│  ├── AcquireGCLock()              获取GC锁                      │
│  ├── CheckImageIntegrity()        检查镜像完整性 (可选)          │
│  ├── FlushAsyncLoading()          刷新异步加载 (可选)            │
│  └── LockUObjectHashTables()      锁定UObject哈希表              │
│                                                                 │
│  ══════════════════════════════════════════════════════════    │
│  Phase 2: MARK (标记阶段) - 可增量执行                           │
│  ══════════════════════════════════════════════════════════    │
│  ├── MarkObjectsAsMaybeUnreachable()  标记所有对象为"可能不可达" │
│  ├── SwapReachabilityFlags()          交换可达性标志位           │
│  ├── GatherReachableObjects()         收集根对象引用             │
│  │   ├── 永久对象 (ObjFirstGCIndex之前的对象)                   │
│  │   ├── AddToRoot()的对象                                      │
│  │   ├── FGCObject报告的引用                                    │
│  │   └── 自定义根对象回调                                       │
│  └── TraverseReferenceGraph()         遍历引用图，标记可达对象    │
│      ├── 并行遍历 (如果启用)                                    │
│      ├── 处理聚类                                              │
│      └── 处理增量时间限制                                       │
│                                                                 │
│  ══════════════════════════════════════════════════════════    │
│  Phase 3: POST-REACHABILITY (后标记阶段)                         │
│  ══════════════════════════════════════════════════════════    │
│  ├── DissolveUnreachableClusters()    溶解不可达聚类             │
│  ├── ClearWeakReferences()            清空弱引用                 │
│  ├── GatherUnreachableObjects()       收集不可达对象             │
│  └── UnlockUObjectHashTables()        解锁UObject哈希表          │
│                                                                 │
│  ══════════════════════════════════════════════════════════    │
│  Phase 4: SWEEP (清除阶段) - 可增量执行                          │
│  ══════════════════════════════════════════════════════════    │
│  ├── UnhashUnreachableObjects()       从哈希表移除不可达对象      │
│  │   └── 调用 BeginDestroy()                                    │
│  ├── IncrementalDestroyGarbage()      增量销毁垃圾对象            │
│  │   ├── 等待 IsReadyForFinishDestroy()                         │
│  │   ├── 调用 FinishDestroy()                                   │
│  │   ├── FreeUObjectIndex() 释放索引                            │
│  │   └── 析构并释放内存                                         │
│  └── 检查时间限制，可能分帧执行                                  │
│                                                                 │
│  ══════════════════════════════════════════════════════════    │
│  Phase 5: CLEANUP (清理阶段)                                     │
│  ══════════════════════════════════════════════════════════    │
│  ├── ShrinkHashTables()               收缩哈希表 (完整GC时)       │
│  ├── DeletePendingLoaders()           删除待处理加载器            │
│  ├── MemoryTrim()                     内存整理                    │
│  ├── BroadcastGarbageCollectComplete() 广播GC完成事件             │
│  └── ReleaseGCLock()                  释放GC锁                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 关键函数调用链

```cpp
// GC核心调用链 (简化版)
// Source: GarbageCollection.cpp

void CollectGarbage(EObjectFlags KeepFlags, bool bPerformFullPurge)
{
    // Phase 1: Pre-Collect
    FGCScopeGuard GCScopeGuard;
    FScopedUObjectHashTablesLock HashLock;

    // Phase 2: Mark
    FReachabilityAnalysisState& State = FReachabilityAnalysisState::Get();
    State.PerformReachabilityAnalysis(KeepFlags, bPerformFullPurge);

    // Phase 3: Post-Reachability
    ProcessClusters();
    ClearWeakReferences();

    // Phase 4: Sweep
    IncrementalPurgeGarbage(bPerformFullPurge);

    // Phase 5: Cleanup
    BroadcastGarbageCollectComplete();
}
```

---

## 六、控制台变量配置

### 6.1 核心GC配置

```ini
; GC核心配置 (DefaultEngine.ini 或运行时控制台)

; ===== 基本设置 =====
gc.TimeBetweenGCs=60.0              ; GC间隔时间(秒)，默认60秒
gc.MinGCInterval=8.0                ; 最小GC间隔(秒)，防止GC过于频繁
gc.MaxObjectsInGame=500000          ; 游戏中最大对象数
gc.MaxObjectsInEditor=1000000       ; 编辑器中最大对象数

; ===== 内存触发 =====
gc.MemoryTriggerThreshold           ; 内存触发阈值(字节)，超过时触发GC
gc.TargetGCBalanceToMemory=0.5      ; GC与内存平衡因子

; ===== 性能设置 =====
gc.AllowParallelGC=1                ; 启用并行GC (多线程)
gc.CreateGCClusters=1               ; 启用聚类优化
gc.MinGCClusterSize=2               ; 最小聚类大小
```

### 6.2 增量GC配置

```ini
; ===== 增量可达性分析 =====
gc.AllowIncrementalReachability=0   ; 启用增量可达性分析 (默认关闭)
gc.IncrementalReachabilityTimeLimit=0.005 ; 每次迭代时间限制(秒)

; ===== 增量收集 =====
gc.AllowIncrementalGather=0         ; 启用增量收集不可达对象
gc.IncrementalGatherTimeLimit=0.0   ; 收集时间限制

; ===== 增量销毁 =====
gc.IncrementalBeginDestroyEnabled=1 ; 启用增量销毁 (默认开启)
gc.IncrementalBeginDestroyGranularity=10 ; 每批销毁对象数
```

### 6.3 调试配置

```ini
; ===== 验证选项 =====
gc.VerifyNoUnreachableObjects=0     ; 验证不可达对象确实不可达
gc.VerifyObjectsDestroyed=0         ; 验证所有对象已销毁

; ===== 追踪选项 =====
gc.GarbageReferenceTrackingEnabled=0; 启用垃圾引用追踪
gc.DumpAnalyticsToLog=0             ; 输出GC分析日志
gc.LogGarbageEveryFrame=0           ; 每帧记录垃圾对象

; ===== 调试命令 =====
; gc.Flush               - 完成所有挂起的GC工作
; gc.CollectGarbage      - 强制触发一次GC
; gc.DumpGCClusters      - 输出聚类信息
; gc.Status              - 显示GC状态
```

---

## 七、关键源码文件位置

### 7.1 目录结构

```
Engine/Source/Runtime/CoreUObject/
│
├── Private/UObject/
│   ├── GarbageCollection.cpp           # 主GC实现 (~7000行)
│   ├── GarbageCollectionVerification.cpp # GC验证逻辑
│   ├── UObjectClusters.cpp             # 聚类管理实现
│   ├── UObjectArray.cpp                # 对象数组管理
│   ├── FastReferenceCollector.cpp      # 引用遍历实现
│   └── ReachabilityAnalysis.cpp        # 可达性分析实现
│
├── Public/UObject/
│   ├── GarbageCollection.h             # GC公共接口
│   ├── GCObject.h                      # FGCObject基类定义
│   ├── UObjectArray.h                  # FUObjectArray/FUObjectItem
│   ├── UObjectClusters.h               # 聚类结构定义
│   ├── GarbageCollectionSchema.h       # 引用Schema定义
│   ├── FastReferenceCollector.h        # 快速引用收集器
│   └── ReachabilityAnalysis.h          # 可达性分析接口
│
└── Private/UniversalObjectLocator/
    └── UniversalObjectLocator.cpp      # 对象定位器GC支持
```

### 7.2 关键文件说明

| 文件 | 行数 | 主要内容 |
|------|------|----------|
| `GarbageCollection.cpp` | ~7000 | GC主流程、各阶段实现 |
| `UObjectArray.cpp` | ~2000 | 对象池管理、索引分配 |
| `UObjectClusters.cpp` | ~1500 | 聚类创建、溶解、GC处理 |
| `FastReferenceCollector.h` | ~800 | 高效引用遍历模板 |
| `GarbageCollectionSchema.h` | ~500 | Schema定义、引用类型 |

---

## 八、实践练习

### 练习1: 观察GC行为

```cpp
// 创建一个监控GC的控制台命令
// 在游戏运行时观察GC行为

void MonitorGC()
{
    // 启用GC详细日志
    IConsoleManager::Get().FindConsoleVariable(
        TEXT("gc.DumpAnalyticsToLog"))->Set(1);

    // 检查当前GC状态
    bool bIsGCCollecting = IsGarbageCollecting();
    UE_LOG(LogTemp, Log, TEXT("GC Currently Running: %s"),
           bIsGCCollecting ? TEXT("Yes") : TEXT("No"));

    // 获取对象统计
    int32 TotalObjects = GUObjectArray.GetObjectArrayNum();
    UE_LOG(LogTemp, Log, TEXT("Total Objects: %d"), TotalObjects);

    // 获取内存状态
    FPlatformMemoryStats MemStats = FPlatformMemory::GetStats();
    UE_LOG(LogTemp, Log, TEXT("Memory Used: %.2f MB"),
           MemStats.TotalUsed / (1024.0 * 1024.0));
}

// 注册控制台命令
static FAutoConsoleCommand MonitorGCCommand(
    TEXT("MyGame.MonitorGC"),
    TEXT("Monitor GC status"),
    FConsoleCommandDelegate::CreateStatic(&MonitorGC)
);
```

### 练习2: 手动触发GC

```cpp
// 安全地手动触发GC
// 展示不同场景下的GC调用方式

class FGCExample
{
public:
    // 方式1: 完整GC (阻塞直到完成)
    static void PerformFullGC()
    {
        if (IsGarbageCollecting())
        {
            UE_LOG(LogTemp, Warning, TEXT("GC already in progress"));
            return;
        }

        UE_LOG(LogTemp, Log, TEXT("Starting full GC..."));
        double StartTime = FPlatformTime::Seconds();

        CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS, true);

        double Duration = FPlatformTime::Seconds() - StartTime;
        UE_LOG(LogTemp, Log, TEXT("GC completed in %.2f ms"), Duration * 1000.0);
    }

    // 方式2: 增量GC (非阻塞)
    static void PerformIncrementalGC()
    {
        if (!gc.AllowIncrementalReachability)
        {
            UE_LOG(LogTemp, Warning,
                TEXT("Incremental GC not enabled, enabling it..."));
            gc.AllowIncrementalReachability = 1;
        }

        // 检查是否有挂起的增量GC
        if (IsIncrementalGCPending())
        {
            UE_LOG(LogTemp, Log, TEXT("Incremental GC already pending"));
        }
        else
        {
            // 开始新的增量GC周期
            CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS, false);
            UE_LOG(LogTemp, Log, TEXT("Started incremental GC"));
        }
    }

    // 方式3: 完成挂起的GC
    static void FlushPendingGC()
    {
        if (IsIncrementalGCPending())
        {
            UE_LOG(LogTemp, Log, TEXT("Flushing pending GC..."));
            // gc.Flush 命令或调用
            FlushIncrementalGC();
        }
    }
};
```

### 练习3: 对象生命周期观察

```cpp
// 创建一个观察对象生命周期的类

UCLASS()
class UMyGCObject : public UObject
{
    GENERATED_BODY()

public:
    UMyGCObject()
    {
        UE_LOG(LogTemp, Log, TEXT("UMyGCObject Created: %s"), *GetName());
    }

    virtual void BeginDestroy() override
    {
        UE_LOG(LogTemp, Log, TEXT("UMyGCObject BeginDestroy: %s"), *GetName());
        Super::BeginDestroy();
    }

    virtual bool IsReadyForFinishDestroy() override
    {
        bool bReady = Super::IsReadyForFinishDestroy();
        UE_LOG(LogTemp, Log, TEXT("UMyGCObject IsReadyForFinishDestroy: %s - %s"),
               *GetName(), bReady ? TEXT("true") : TEXT("false"));
        return bReady;
    }

    virtual void FinishDestroy() override
    {
        UE_LOG(LogTemp, Log, TEXT("UMyGCObject FinishDestroy: %s"), *GetName());
        Super::FinishDestroy();
    }
};

// 测试代码
void TestObjectLifecycle()
{
    // 创建对象
    UMyGCObject* Obj = NewObject<UMyGCObject>();

    // 添加到根集合 (防止GC)
    Obj->AddToRoot();
    UE_LOG(LogTemp, Log, TEXT("Object added to root set"));

    // 强制GC (对象不会被回收)
    CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS, true);
    UE_LOG(LogTemp, Log, TEXT("GC completed, object still alive: %s"),
           Obj->IsValidLowLevel() ? TEXT("Yes") : TEXT("No"));

    // 从根集合移除
    Obj->RemoveFromRoot();
    UE_LOG(LogTemp, Log, TEXT("Object removed from root set"));

    // 清空引用
    Obj = nullptr;

    // 强制GC (对象应该被回收)
    CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS, true);
    UE_LOG(LogTemp, Log, TEXT("GC completed, check logs for destruction messages"));
}
```

---

## 九、总结

### 本节课要点

1. **设计动机**
   - 游戏引擎面临复杂的内存管理挑战
   - GC自动管理简化开发，避免内存泄漏
   - UE5 GC系统经过多年演进优化

2. **设计目标**
   - 安全性：防止访问已销毁对象
   - 性能：增量、并行、聚类优化
   - 可预测性：可配置的GC时机
   - 兼容性：支持多种使用场景

3. **核心架构**
   - FUObjectArray：全局对象池
   - FReachabilityAnalysisState：可达性分析状态机
   - TFastReferenceCollector：高效引用遍历

4. **对象生命周期**
   - Active → Unreachable → PendingDestroy → Destroyed
   - 理解标志位的作用和含义

5. **GC流程**
   - 五阶段：准备→标记→后标记→清除→清理
   - 每个阶段的职责和关键操作

6. **配置选项**
   - 核心配置：时间间隔、对象上限
   - 增量配置：时间限制、批量大小
   - 调试配置：验证、追踪选项

### 下一节课预告

下一节课将深入讲解**对象追踪与引用图**，包括：
- FUObjectArray的详细实现
- 引用Schema系统的工作原理
- FGCObject机制的使用
- 并行引用遍历的实现

---

## 参考资料

### 源码文件
- `Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollection.cpp`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectArray.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/GarbageCollection.h`

### 官方文档
- [UE5 Memory Management](https://docs.unrealengine.com/5.0/en-US/memory-management-in-unreal-engine/)
- [UObject Memory Management](https://docs.unrealengine.com/5.0/en-US/API/Runtime/CoreUObject/UObject/UObject/)

### 相关课程
- Lesson 2: 对象追踪与引用图
- Lesson 3: 标记-清除算法实现
