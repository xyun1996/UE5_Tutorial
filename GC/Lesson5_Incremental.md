# Lesson 5: 增量GC实现

## 课程概述

本节课深入讲解UE5的增量GC(Incremental GC)系统，这是实现平滑帧率的关键技术。

**学习目标：**
- 理解增量GC解决什么问题
- 掌握增量可达性分析的实现
- 了解写屏障的工作原理
- 学会配置和调优增量GC

---

## 一、增量GC概述

### 1.1 问题背景

```
┌─────────────────────────────────────────────────────────────────┐
│                    为什么需要增量GC?                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  问题: 传统GC会导致帧卡顿                                        │
│                                                                 │
│  示例: 60fps游戏 (每帧16.67ms)                                  │
│                                                                 │
│  传统GC:                                                        │
│  ┌───────────────────────────────────────────────────┐         │
│  │ Frame 1  │ Frame 2  │ Frame 3  │ Frame 4 │       │         │
│  │  16.7ms  │  16.7ms  │  16.7ms  │  16.7ms │       │         │
│  └───────────────────────────────────────────────────┘         │
│                         ↓                                       │
│                  GC触发 (50ms)                                  │
│  ┌───────────────────────────────────────────────────┐         │
│  │ Frame 1  │ GC Block │ Frame 3  │ Frame 4 │       │         │
│  │  16.7ms  │  50ms    │  16.7ms  │  16.7ms │       │         │
│  └───────────────────────────────────────────────────┘         │
│                         ↓                                       │
│                   明显卡顿!                                      │
│                   丢失3帧!                                       │
│                                                                 │
│  增量GC:                                                        │
│  ┌───────────────────────────────────────────────────┐         │
│  │ Frame 1  │ Frame 2  │ Frame 3  │ Frame 4 │       │         │
│  │  18ms    │  18ms    │  18ms    │  18ms   │       │         │
│  │(+1.3ms)  │(+1.3ms)  │(+1.3ms)  │(+1.3ms) │       │         │
│  └───────────────────────────────────────────────────┘         │
│                         ↓                                       │
│                   平滑帧率!                                      │
│                   无明显卡顿                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 增量GC目标

```cpp
// 增量GC设计目标
// Source: ReachabilityAnalysis.h

/*
 * 1. 帧时间平滑 (Frame Time Smoothing)
 *    - 每帧GC时间限制可配置
 *    - 避免单帧长时间阻塞
 *    - 分摊GC开销到多帧
 *
 * 2. 正确性保证 (Correctness)
 *    - 增量期间对象状态一致
 *    - 正确处理并发修改
 *    - 写屏障保证可达性正确
 *
 * 3. 可预测性 (Predictability)
 *    - 开发者可控制GC时机
 *    - 可强制完成挂起的GC
 *    - 时间限制有保证
 *
 * 4. 性能开销小 (Low Overhead)
 *    - 增量机制本身开销低
 *    - 额外状态跟踪最小化
 *    - 写屏障高效实现
 */
```

### 1.3 增量GC vs 完整GC

| 特性 | 完整GC | 增量GC |
|------|--------|--------|
| 执行方式 | 单次阻塞 | 分帧执行 |
| 帧时间影响 | 大 (可能卡顿) | 小 (每帧均匀) |
| 总时间 | 较短 | 略长 (有开销) |
| 适用场景 | 加载时、暂停时 | 游戏运行时 |
| 配置 | 默认 | 需要启用 |

---

## 二、增量可达性分析

### 2.1 架构设计

```
┌─────────────────────────────────────────────────────────────────┐
│                    增量可达性分析架构                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              FReachabilityAnalysisState                 │   │
│  │              (增量状态机)                                │   │
│  │                                                          │   │
│  │  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐│   │
│  │  │  Idle   │──▶│Marking  │──▶│Traverse │──▶│Completed││   │
│  │  │         │   │ Roots   │   │  Refs   │   │         ││   │
│  │  └─────────┘   └─────────┘   └─────────┘   └─────────┘│   │
│  └─────────────────────────────────────────────────────────┘   │
│                            │                                    │
│                            ▼                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    写屏障系统                            │   │
│  │  ┌─────────────────┐    ┌─────────────────┐            │   │
│  │  │ 修改记录队列    │───▶│ 重新扫描处理    │            │   │
│  │  └─────────────────┘    └─────────────────┘            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  关键概念:                                                       │
│  - STW (Stop-The-World): 短暂暂停所有线程                       │
│  - 并发阶段: GC与游戏线程并发执行                                │
│  - 写屏障: 记录增量期间的修改                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 FReachabilityAnalysisState

```cpp
// 增量可达性分析状态机
// Source: ReachabilityAnalysisState.h

class FReachabilityAnalysisState
{
public:
    // ===== 状态枚举 =====
    enum class EState : uint8
    {
        Idle,                       // 空闲, 无GC进行
        MarkingRoots,               // 正在标记根对象
        TraversingReferences,       // 正在遍历引用图 (增量)
        WaitingForFinalize,         // 等待最终完成时机
        Finalizing,                 // 最终完成中
        Completed                   // 已完成
    };

    // ===== 并发控制 =====
    std::atomic<EState> CurrentState{EState::Idle};

    // ===== 工作队列 =====
    static constexpr int32 MaxWorkers = 16;

    TArray<UObject*> ObjectsToProcess[MaxWorkers];  // 待处理对象
    TArray<UObject*> ReachableObjects[MaxWorkers];  // 已标记可达

    // ===== 增量控制 =====
    double IterationStartTime;      // 当前迭代开始时间
    double TimeLimit;               // 时间限制(秒)

    // ===== 进度追踪 =====
    std::atomic<int32> NextObjectIndex;  // 下一个要处理的对象索引
    std::atomic<int32> ProcessedCount;   // 已处理对象计数

    // ===== 写屏障记录 =====
    TArray<FWBRecord> WriteBarrierRecords;

    // ===== 主要方法 =====
    bool PerformIncrementalStep(double InTimeLimit);
    void FinalizeAnalysis();
    bool IsTimeLimitExceeded() const;

    // ===== 状态查询 =====
    EState GetState() const { return CurrentState.load(); }
    bool IsIdle() const { return GetState() == EState::Idle; }
    bool IsInProgress() const { return GetState() != EState::Idle && GetState() != EState::Completed; }
};
```

### 2.3 增量执行流程

```cpp
// 增量执行核心逻辑
// Source: GarbageCollection.cpp

bool FReachabilityAnalysisState::PerformIncrementalStep(double InTimeLimit)
{
    TRACE_CPUPROFILER_EVENT_SCOPE(GC_IncrementalStep);

    TimeLimit = InTimeLimit;
    IterationStartTime = FPlatformTime::Seconds();

    switch (CurrentState.load())
    {
    case EState::Idle:
        // ===== 开始新的GC周期 =====
        CurrentState.store(EState::MarkingRoots);
        PrepareForAnalysis();

        // 标记所有对象为"可能不可达"
        SwapReachableFlags();
        break;

    case EState::MarkingRoots:
        // ===== 标记根对象 =====
        if (MarkRootObjectsIncremental())
        {
            CurrentState.store(EState::TraversingReferences);
        }
        break;

    case EState::TraversingReferences:
        // ===== 遍历引用图(增量) =====
        if (TraverseReferencesIncremental())
        {
            // 处理写屏障记录
            ProcessWriteBarrierRecords();

            CurrentState.store(EState::WaitingForFinalize);
        }
        break;

    case EState::WaitingForFinalize:
        // ===== 等待合适的时机完成 =====
        if (ShouldFinalizeNow())
        {
            CurrentState.store(EState::Finalizing);
        }
        break;

    case EState::Finalizing:
        // ===== 最终完成 =====
        FinalizeAnalysis();
        CurrentState.store(EState::Completed);
        break;

    case EState::Completed:
        return true;  // 完成
    }

    return IsTimeLimitExceeded();
}
```

### 2.4 写屏障(Write Barrier)

```cpp
// 写屏障: 处理增量期间的并发修改
// Source: GarbageCollection.cpp

/*
 * ===== 问题场景 =====
 * 1. GC正在遍历对象O的引用
 * 2. 游戏线程修改了O, 添加了对新对象N的引用
 * 3. 如果N还没被遍历到, 可能被错误标记为不可达
 * 4. N被回收 → 访问无效引用 → 崩溃!
 *
 * ===== 解决方案 =====
 * 使用写屏障记录所有修改
 */

// 写屏障记录结构
struct FWBRecord
{
    int32 ObjectIndex;       // 被修改的对象索引
    UObject** ReferencePtr;  // 被修改的引用指针
    UObject* OldValue;       // 旧值
    UObject* NewValue;       // 新值
};

// 写屏障实现
void UObject::WriteBarrier(UObject** RefPtr, UObject* NewValue)
{
    // 原子更新引用
    UObject* OldValue = InterlockedExchangePointer((void**)RefPtr, NewValue);

    // 如果增量GC正在进行
    if (IsIncrementalGCRunning())
    {
        // 记录修改
        FWBRecord Record;
        Record.ObjectIndex = GetIndex();
        Record.ReferencePtr = RefPtr;
        Record.OldValue = OldValue;
        Record.NewValue = NewValue;

        FReachabilityAnalysisState::RecordWriteBarrier(Record);
    }
}

// 处理写屏障记录
void FReachabilityAnalysisState::ProcessWriteBarrierRecords()
{
    TArray<FWBRecord>& Records = GetWriteBarrierRecords();

    for (const FWBRecord& Record : Records)
    {
        // 检查新引用的对象
        if (Record.NewValue && !Record.NewValue->IsReachable())
        {
            // 标记为可达并加入处理队列
            MarkAsReachable(Record.NewValue);
            AddToProcessQueue(Record.NewValue);
        }
    }

    Records.Empty();
}
```

### 2.5 三色标记算法

```cpp
// 三色标记算法 (Tri-Color Marking)
// 增量GC的理论基础

/*
 * ===== 三色定义 =====
 * White (白色): 未访问, 可能是垃圾
 * Gray  (灰色): 已发现, 待处理引用
 * Black (黑色): 已处理, 确定存活
 *
 * ===== 不变式 =====
 * 1. 黑色对象不会直接指向白色对象
 * 2. 所有灰色对象最终会变成黑色
 * 3. 白色对象就是垃圾
 *
 * ===== 写屏障保证 =====
 * Dijkstra写屏障:
 *   当创建 Black -> White 引用时
 *   将 White 对象标记为 Gray
 *
 * 这保证了不变式1不被破坏
 */

enum class EGCColor
{
    White,  // 未访问
    Gray,   // 待处理
    Black   // 已处理
};

// UE5使用ReachabilityFlag实现三色
// RF0/RF1 + Unreachable 组合表示三种状态
```

---

## 三、增量清除

### 3.1 增量清除架构

```cpp
// 增量清除配置
// Source: GarbageCollection.cpp

/*
 * 清除阶段也需要增量执行
 *
 * Phase 1: BeginDestroy
 *   - 可以异步执行
 *   - 游戏线程调用
 *   - 清理游戏逻辑
 *
 * Phase 2: FinishDestroy
 *   - 需要同步
 *   - 释放资源
 *   - 销毁对象
 */

class FIncrementalPurgeState
{
public:
    // 待销毁对象队列
    TArray<UObject*> PendingDestroyQueue;

    // 当前处理位置
    int32 CurrentDestroyIndex;

    // 时间限制
    double PurgeTimeLimit;

    // 统计
    int32 TotalDestroyed;
    int32 TotalPending;

    // 增量清除
    bool PerformIncrementalPurge(double TimeLimit);

private:
    void ProcessBeginDestroyBatch(int32 Count);
    void ProcessFinishDestroyBatch(int32 Count);
};
```

### 3.2 增量清除实现

```cpp
// 增量清除核心逻辑
// Source: GarbageCollection.cpp

bool FIncrementalPurgeState::PerformIncrementalPurge(double TimeLimit)
{
    TRACE_CPUPROFILER_EVENT_SCOPE(GC_IncrementalPurge);

    double StartTime = FPlatformTime::Seconds();
    int32 ProcessedCount = 0;

    // 配置: 每批处理的对象数
    int32 BatchSize = gc.IncrementalBeginDestroyGranularity;

    while (CurrentDestroyIndex < PendingDestroyQueue.Num())
    {
        // ===== 检查时间限制 =====
        if ((FPlatformTime::Seconds() - StartTime) >= TimeLimit)
        {
            return false;  // 未完成, 下次继续
        }

        UObject* Obj = PendingDestroyQueue[CurrentDestroyIndex];

        // ===== Phase 1: BeginDestroy =====
        if (!Obj->IsPendingKill())
        {
            Obj->ConditionalBeginDestroy();
        }

        // ===== Phase 2: FinishDestroy =====
        if (Obj->IsReadyForFinishDestroy())
        {
            Obj->ConditionalFinishDestroy();

            // 释放对象
            GUObjectArray.FreeUObjectIndex(Obj);
            Obj->~UObject();
            FMemory::Free(Obj);

            ++TotalDestroyed;
        }
        else
        {
            // 还不能销毁, 跳过
            // 下次检查
        }

        ++CurrentDestroyIndex;
        ++ProcessedCount;

        // 定期检查时间
        if (ProcessedCount % BatchSize == 0)
        {
            if ((FPlatformTime::Seconds() - StartTime) >= TimeLimit)
            {
                return false;
            }
        }
    }

    return true;  // 完成
}
```

### 3.3 增量收集不可达对象

```cpp
// 增量收集不可达对象
// Source: GarbageCollection.cpp

class FIncrementalGatherState
{
    int32 NextObjectToCheck;
    TArray<UObject*> UnreachableObjects;

public:
    bool PerformIncrementalGather(double TimeLimit)
    {
        double StartTime = FPlatformTime::Seconds();

        for (; NextObjectToCheck < GUObjectArray.GetObjectArrayNum();
             ++NextObjectToCheck)
        {
            // 检查时间
            if ((FPlatformTime::Seconds() - StartTime) >= TimeLimit)
            {
                return false;  // 未完成
            }

            FUObjectItem* Item = GUObjectArray.IndexToObject(NextObjectToCheck);
            if (!Item || !Item->Object)
                continue;

            // 检查是否不可达
            if (Item->HasAnyFlags(Unreachable) && !Item->HasAnyFlags(Garbage))
            {
                UnreachableObjects.Add(Item->Object);
                Item->SetFlags(Garbage);
            }
        }

        return true;  // 完成
    }

    const TArray<UObject*>& GetUnreachableObjects() const
    {
        return UnreachableObjects;
    }
};
```

---

## 四、配置与控制

### 4.1 控制台变量

```ini
; 增量GC配置
; DefaultEngine.ini 或运行时控制台

; ===== 可达性分析增量 =====
gc.AllowIncrementalReachability=0     ; 启用增量可达性(默认关闭)
gc.IncrementalReachabilityTimeLimit=0.005 ; 每次迭代时间限制(秒)

; ===== 对象收集增量 =====
gc.AllowIncrementalGather=0           ; 启用增量收集(默认关闭)
gc.IncrementalGatherTimeLimit=0.0     ; 收集时间限制

; ===== 销毁增量 =====
gc.IncrementalBeginDestroyEnabled=1   ; 启用增量销毁(默认开启)
gc.IncrementalBeginDestroyGranularity=10 ; 每批销毁对象数

; ===== GC间隔 =====
gc.TimeBetweenGCs=60.0                ; GC间隔(秒)
gc.MinGCInterval=8.0                  ; 最小间隔
gc.MaxObjectsInGame=500000            ; 对象上限
```

### 4.2 程序控制接口

```cpp
// 程序化控制增量GC

// 检查是否有挂起的GC
bool IsIncrementalGCPending()
{
    return FReachabilityAnalysisState::Get().IsInProgress();
}

// 执行一步增量GC
void TickIncrementalGC(float TimeLimit)
{
    if (!gc.AllowIncrementalReachability)
        return;

    FReachabilityAnalysisState& State = FReachabilityAnalysisState::Get();

    // 执行增量步骤
    bool bCompleted = State.PerformIncrementalStep(TimeLimit);

    if (bCompleted)
    {
        // 可达性完成, 开始增量清除
        if (gc.IncrementalBeginDestroyEnabled)
        {
            TickIncrementalPurge(TimeLimit);
        }
        else
        {
            // 完整清除
            IncrementalPurgeGarbage(true);
        }
    }
}

// 强制完成挂起的GC
void FlushIncrementalGC()
{
    FReachabilityAnalysisState& State = FReachabilityAnalysisState::Get();

    UE_LOG(LogGC, Log, TEXT("Flushing incremental GC..."));

    // 完成可达性分析
    while (State.IsInProgress())
    {
        State.PerformIncrementalStep(FLT_MAX);
    }

    // 完成清除
    IncrementalPurgeGarbage(true);

    UE_LOG(LogGC, Log, TEXT("Incremental GC flush complete"));
}
```

### 4.3 最佳配置策略

```cpp
// 不同场景的推荐配置

// ===== 场景1: 高帧率游戏 (60fps+) =====
void ConfigureHighFrameRate()
{
    // 启用所有增量选项
    gc.AllowIncrementalReachability = 1;
    gc.IncrementalReachabilityTimeLimit = 0.002;  // 2ms
    gc.AllowIncrementalGather = 1;
    gc.IncrementalBeginDestroyEnabled = 1;
    gc.IncrementalBeginDestroyGranularity = 5;

    // 更频繁GC, 但每次时间短
    gc.TimeBetweenGCs = 30.0;
}

// ===== 场景2: 内存受限设备 =====
void ConfigureLowMemory()
{
    // 更频繁GC, 较大时间片
    gc.TimeBetweenGCs = 20.0;
    gc.AllowIncrementalReachability = 1;
    gc.IncrementalReachabilityTimeLimit = 0.005;  // 5ms

    // 更激进的清除
    gc.IncrementalBeginDestroyGranularity = 20;
}

// ===== 场景3: 关卡加载时 =====
void ConfigureLevelLoading()
{
    // 加载时禁用增量, 完整GC
    gc.AllowIncrementalReachability = 0;
    gc.IncrementalBeginDestroyEnabled = 0;

    // 立即执行完整GC
    CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS, true);
}

// ===== 场景4: 暂停菜单 =====
void ConfigurePauseMenu()
{
    // 完成挂起的GC
    FlushIncrementalGC();

    // 可以做一次完整GC
    CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS, true);
}
```

---

## 五、增量GC的挑战与解决

### 5.1 对象存活问题

```cpp
// 问题: 增量期间对象状态变化
// 解决: 三色标记 + 写屏障

/*
 * ===== 危险场景 =====
 * 1. GC开始标记, 对象A标记为不可达 (White)
 * 2. 游戏线程创建了对A的新引用
 * 3. GC不知道这个新引用
 * 4. A被错误回收 → 崩溃!
 *
 * ===== 解决方案 =====
 * 三色不变式 + 写屏障:
 * - 任何 Black -> White 的引用创建
 * - 都会将 White 变成 Gray
 * - 保证 White 不会被错误回收
 */
```

### 5.2 浮动垃圾

```cpp
// 浮动垃圾 (Floating Garbage)
// 增量GC的副作用

/*
 * ===== 问题 =====
 * 增量期间变成垃圾的对象要等到下次GC
 *
 * ===== 场景 =====
 * 1. GC开始遍历, 对象B被标记为可达 (Gray -> Black)
 * 2. 之后B的所有引用都被释放
 * 3. 但B已经被标记为可达
 * 4. B会存活到下次GC
 *
 * ===== 影响 =====
 * - 内存回收延迟
 * - 通常可接受
 * - 比完整STW更平滑
 */

// 减少浮动垃圾的策略
void ReduceFloatingGarbage()
{
    // 1. 更短的GC周期
    gc.TimeBetweenGCs = 30.0;  // 更频繁GC

    // 2. 更大的时间片 (更快完成)
    gc.IncrementalReachabilityTimeLimit = 0.010;  // 10ms

    // 3. 在特定时机触发完整GC
    if (IsLoadingComplete())
    {
        CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS, true);
    }
}
```

### 5.3 增量期间同步

```cpp
// 增量GC期间的同步问题

// ===== 问题 =====
// 游戏线程访问正在GC的对象

// ===== 解决方案 =====

// 方式1: 使用GC锁
void SafeObjectAccessWithLock()
{
    FGCScopeGuard GCGuard;  // 阻止GC
    UObject* Obj = GetSomeObject();
    // 安全访问 Obj
    Obj->DoSomething();
}  // 离开作用域, GC可以继续

// 方式2: 检查对象状态
void SafeObjectAccessWithCheck()
{
    UObject* Obj = GetSomeObject();

    // 检查有效性
    if (Obj && !Obj->IsPendingKill() && !Obj->IsUnreachable())
    {
        Obj->DoSomething();
    }
}

// 方式3: 使用弱引用
void SafeObjectAccessWithWeakRef()
{
    TWeakObjectPtr<UObject> WeakObj = GetWeakRef();

    if (UObject* Obj = WeakObj.Get())
    {
        Obj->DoSomething();
    }
}
```

---

## 六、调试增量GC

### 6.1 监控工具

```cpp
// 增量GC监控

class FIncrementalGCMonitor
{
    static TArray<float> StepDurations;
    static int32 TotalSteps;
    static float TotalDuration;
    static float MaxStepTime;

public:
    static void Initialize()
    {
        // 监听GC步骤
        FCoreDelegates::OnGCStep.AddStatic(&FIncrementalGCMonitor::OnStep);
    }

    static void OnStep(float Duration, const TCHAR* StepName)
    {
        StepDurations.Add(Duration);
        TotalSteps++;
        TotalDuration += Duration;
        MaxStepTime = FMath::Max(MaxStepTime, Duration);

        // 检测异常
        if (Duration > 0.010)  // > 10ms
        {
            UE_LOG(LogGC, Warning,
                TEXT("GC step '%s' took %.2fms - consider increasing time limit"),
                StepName, Duration * 1000.0f);
        }
    }

    static void PrintSummary()
    {
        UE_LOG(LogGC, Log, TEXT("=== Incremental GC Summary ==="));
        UE_LOG(LogGC, Log, TEXT("  Steps: %d"), TotalSteps);
        UE_LOG(LogGC, Log, TEXT("  Total: %.2fms"), TotalDuration * 1000.0f);
        UE_LOG(LogGC, Log, TEXT("  Avg: %.2fms"), TotalDuration / TotalSteps * 1000.0f);
        UE_LOG(LogGC, Log, TEXT("  Max: %.2fms"), MaxStepTime * 1000.0f);
    }
};
```

### 6.2 控制台命令

```
// 增量GC调试命令

// 查看当前状态
gc.Status
// 输出: GC状态、挂起的步骤、对象数等

// 强制完成
gc.Flush
// 完成所有挂起的GC工作

// 启用详细日志
gc.DumpAnalyticsToLog 1
// 输出详细GC分析

// 输出增量状态
gc.DumpIncrementalState
// 显示增量GC的详细状态
```

### 6.3 常见问题排查

```cpp
// ===== 问题1: 增量GC未完成导致内存增长 =====
void DebugMemoryGrowth()
{
    if (IsIncrementalGCPending())
    {
        UE_LOG(LogGC, Warning,
            TEXT("Incremental GC pending for too long, forcing completion"));
        FlushIncrementalGC();
    }
}

// ===== 问题2: 增量GC时间片太小导致GC周期过长 =====
void DebugLongGCCycle()
{
    if (gc.IncrementalReachabilityTimeLimit < 0.002)
    {
        UE_LOG(LogGC, Warning,
            TEXT("Time limit %.4fs is very small, GC cycle may take too long"),
            gc.IncrementalReachabilityTimeLimit);
    }
}

// ===== 问题3: 增量期间对象访问问题 =====
void DebugObjectAccess()
{
    // 启用验证
    gc.VerifyNoUnreachableObjects = 1;

    // 检查对象状态
    UObject* Obj = GetProblematicObject();
    if (Obj)
    {
        UE_LOG(LogGC, Log, TEXT("Object: %s"), *Obj->GetName());
        UE_LOG(LogGC, Log, TEXT("  Flags: 0x%X"), Obj->GetFlags());
        UE_LOG(LogGC, Log, TEXT("  Internal: 0x%X"), Obj->GetInternalFlags());
    }
}
```

---

## 七、实践练习

### 练习1: 实现增量GC监控

```cpp
// 创建完整的增量GC监控系统

UCLASS()
class UGCMonitorSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

    int32 GCCycleCount = 0;
    float TotalGCTime = 0.0f;
    float MaxStepTime = 0.0f;
    TArray<float> StepTimes;
    double GCStartTime;

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override
    {
        Super::Initialize(Collection);

        // 监听GC事件
        FCoreDelegates::OnPreGarbageCollection.AddUObject(
            this, &UGCMonitorSubsystem::OnPreGC);
        FCoreDelegates::OnPostGarbageCollection.AddUObject(
            this, &UGCMonitorSubsystem::OnPostGC);
    }

    void OnPreGC()
    {
        GCStartTime = FPlatformTime::Seconds();
    }

    void OnPostGC()
    {
        float Duration = FPlatformTime::Seconds() - GCStartTime;
        GCCycleCount++;
        TotalGCTime += Duration;

        UE_LOG(LogGC, Log, TEXT("GC #%d: %.2fms"),
            GCCycleCount, Duration * 1000.0f);
    }

    UFUNCTION(BlueprintCallable, Category = "GC")
    void PrintStats()
    {
        UE_LOG(LogGC, Log, TEXT("=== GC Monitor Stats ==="));
        UE_LOG(LogGC, Log, TEXT("  Cycles: %d"), GCCycleCount);
        UE_LOG(LogGC, Log, TEXT("  Total: %.2fms"), TotalGCTime * 1000.0f);
        UE_LOG(LogGC, Log, TEXT("  Avg: %.2fms"),
            TotalGCTime / GCCycleCount * 1000.0f);
        UE_LOG(LogGC, Log, TEXT("  Max Step: %.2fms"), MaxStepTime * 1000.0f);
    }
};
```

### 练习2: 优化增量GC配置

```cpp
// 根据设备性能自动配置

void AutoConfigureIncrementalGC()
{
    // 检测设备性能
    float EstimatedFPS = EstimateDevicePerformance();

    UE_LOG(LogGC, Log, TEXT("Auto-configuring GC for estimated %.0f FPS"),
        EstimatedFPS);

    if (EstimatedFPS >= 60.0f)
    {
        // 高性能设备
        gc.AllowIncrementalReachability = 1;
        gc.IncrementalReachabilityTimeLimit = 0.002;
        gc.TimeBetweenGCs = 60.0;
        gc.IncrementalBeginDestroyGranularity = 5;
    }
    else if (EstimatedFPS >= 30.0f)
    {
        // 中等性能设备
        gc.AllowIncrementalReachability = 1;
        gc.IncrementalReachabilityTimeLimit = 0.004;
        gc.TimeBetweenGCs = 45.0;
        gc.IncrementalBeginDestroyGranularity = 10;
    }
    else
    {
        // 低性能设备
        gc.AllowIncrementalReachability = 1;
        gc.IncrementalReachabilityTimeLimit = 0.006;
        gc.TimeBetweenGCs = 30.0;
        gc.IncrementalBeginDestroyGranularity = 15;
    }
}
```

---

## 八、总结

### 本节课要点

1. **设计原理**
   - 增量GC分摊开销到多帧
   - 避免单帧长时间阻塞
   - 实现平滑帧率

2. **可达性分析**
   - FReachabilityAnalysisState状态机
   - 增量执行各阶段
   - 时间限制控制

3. **写屏障**
   - 记录增量期间的修改
   - 保证三色不变式
   - 防止错误回收

4. **增量清除**
   - 分批销毁对象
   - 时间限制控制
   - 异步处理

5. **配置控制**
   - 时间限制配置
   - 场景化配置策略
   - 程序化控制接口

6. **挑战与解决**
   - 对象存活问题
   - 浮动垃圾
   - 同步问题

### 下一节课预告

下一节课将讲解**GC性能优化与调试**，包括：
- 性能分析方法和工具
- 常见优化技巧
- 调试方法和问题排查
- 生产环境最佳实践

---

## 参考资料

### 源码文件
- `Engine/Source/Runtime/CoreUObject/Public/UObject/ReachabilityAnalysis.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/ReachabilityAnalysisState.h`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollection.cpp`

### 相关课程
- Lesson 4: GC聚类系统详解
- Lesson 6: GC性能优化与调试
