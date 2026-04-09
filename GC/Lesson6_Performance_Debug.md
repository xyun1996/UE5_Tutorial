# Lesson 6: GC性能优化与调试

## 课程概述

本节课讲解UE5 GC系统的性能优化技巧和调试方法，帮助你解决实际开发中的GC相关问题。

**学习目标：**
- 掌握GC性能分析方法和工具
- 学会常见GC性能优化技巧
- 熟悉GC调试工具和问题排查
- 了解生产环境最佳实践

---

## 一、GC性能分析

### 1.1 关键性能指标

```
┌─────────────────────────────────────────────────────────────────┐
│                    GC 关键性能指标                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. GC延迟 (Latency)                                            │
│     - 定义: 单次GC造成的帧卡顿时间                               │
│     - 目标: < 2ms (60fps游戏)                                   │
│     - 影响: 玩家体验                                            │
│                                                                 │
│  2. GC频率 (Frequency)                                          │
│     - 定义: 多长时间触发一次GC                                   │
│     - 配置: gc.TimeBetweenGCs                                   │
│     - 影响: 内存回收效率                                         │
│                                                                 │
│  3. GC吞吐量 (Throughput)                                       │
│     - 定义: 每秒能处理的对象数                                   │
│     - 影响: 能处理多大规模场景                                   │
│     - 优化: 并行GC、聚类                                         │
│                                                                 │
│  4. 内存占用 (Memory Usage)                                     │
│     - 定义: GC期间的内存峰值                                     │
│     - 关注: GC后的内存回收量                                     │
│     - 目标: 稳定, 无泄漏                                        │
│                                                                 │
│  5. 对象数量 (Object Count)                                     │
│     - 定义: 当前活跃的UObject数量                                │
│     - 限制: gc.MaxObjectsInGame                                 │
│     - 影响: GC遍历时间                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 性能统计API

```cpp
// GC性能统计结构
// Source: GarbageCollection.cpp

struct FGCPerformanceStats
{
    // ===== 时间统计 =====
    double TotalGCTime;            // 总GC时间 (毫秒)
    double MarkTime;               // 标记阶段时间
    double SweepTime;              // 清除阶段时间
    double ClusterTime;            // 聚类处理时间

    // ===== 对象统计 =====
    int32 TotalObjects;            // 总对象数
    int32 ReachableObjects;        // 可达对象数
    int32 UnreachableObjects;      // 不可达对象数
    int32 ClustersProcessed;       // 处理的聚类数

    // ===== 内存统计 =====
    SIZE_T MemoryBefore;           // GC前内存 (字节)
    SIZE_T MemoryAfter;            // GC后内存
    SIZE_T MemoryFreed;            // 释放的内存

    // ===== 增量统计 =====
    int32 IncrementalSteps;        // 增量步骤数
    double AvgStepTime;            // 平均步骤时间
    double MaxStepTime;            // 最大步骤时间

    // ===== 计算方法 =====
    double GetMarkPercentage() const { return TotalGCTime > 0 ? MarkTime / TotalGCTime * 100.0 : 0.0; }
    double GetSweepPercentage() const { return TotalGCTime > 0 ? SweepTime / TotalGCTime * 100.0 : 0.0; }
    double GetMemoryFreedMB() const { return MemoryFreed / (1024.0 * 1024.0); }
};

// 获取统计
FGCPerformanceStats GetGCStats()
{
    FGCPerformanceStats Stats;
    FGarbageCollection::GetStats(Stats);
    return Stats;
}
```

### 1.3 使用stat命令分析

```
// 控制台stat命令

// ===== 查看GC统计 =====
stat gc

// 输出示例:
// ┌─────────────────────────────────┐
// │ GC Cycles: 127                  │
// │ Total Time: 2.34s               │
// │ Avg Time: 18.4ms                │
// │ Last GC: 12.3ms                 │
// │ Objects: 145,234                │
// │ Clusters: 1,234                 │
// │ Memory Freed: 256 MB            │
// └─────────────────────────────────┘

// ===== 查看详细内存 =====
stat memory

// ===== 实时监控 =====
stat fps      // 帧率
stat unit     // 各模块耗时
stat game     // 游戏线程耗时
```

### 1.4 自定义性能监控

```cpp
// 创建GC性能监控器

class FGCMonitor
{
    static int32 TotalGCCycles;
    static double TotalGCDuration;
    static double MaxGCDuration;
    static TQueue<FGCPerformanceStats> RecentStats;

public:
    static void Initialize()
    {
        FCoreDelegates::OnPostGarbageCollection.AddStatic(&FGCMonitor::OnGCComplete);
    }

    static void OnGCComplete()
    {
        TotalGCCycles++;

        FGCPerformanceStats Stats = GetGCStats();
        TotalGCDuration += Stats.TotalGCTime;
        MaxGCDuration = FMath::Max(MaxGCDuration, Stats.TotalGCTime);

        RecentStats.Enqueue(Stats);

        // 只保留最近100次
        if (RecentStats.Num() > 100)
        {
            FGCPerformanceStats Discard;
            RecentStats.Dequeue(Discard);
        }

        // 详细日志
        UE_LOG(LogGC, Log,
            TEXT("GC #%d: %.2fms (Mark: %.1f%%, Sweep: %.1f%%) Objects: %d, Freed: %.1fMB"),
            TotalGCCycles,
            Stats.TotalGCTime,
            Stats.GetMarkPercentage(),
            Stats.GetSweepPercentage(),
            Stats.TotalObjects,
            Stats.GetMemoryFreedMB());
    }

    static void PrintSummary()
    {
        UE_LOG(LogGC, Log, TEXT("=== GC Performance Summary ==="));
        UE_LOG(LogGC, Log, TEXT("  Total Cycles: %d"), TotalGCCycles);
        UE_LOG(LogGC, Log, TEXT("  Total Time: %.2fs"), TotalGCDuration);
        UE_LOG(LogGC, Log, TEXT("  Average: %.2fms"), TotalGCDuration / TotalGCCycles);
        UE_LOG(LogGC, Log, TEXT("  Maximum: %.2fms"), MaxGCDuration);

        // 分析最近GC趋势
        AnalyzeRecentTrends();
    }

private:
    static void AnalyzeRecentTrends()
    {
        // 分析最近10次GC的趋势
        // 检测是否有内存泄漏或性能退化
    }
};
```

---

## 二、性能优化技巧

### 2.1 减少对象数量

```cpp
// ===== 优化策略1: 减少UObject数量 =====

// 问题: 大量小对象增加GC开销
// 解决: 使用POD结构体或合并对象

// ===== 错误示范 =====
UCLASS()
class UProjectileData : public UObject
{
    GENERATED_BODY()

    UPROPERTY()
    float Damage;

    UPROPERTY()
    float Speed;

    UPROPERTY()
    FVector Direction;
};

// 创建10000个实例 = 10000个UObject!
void BadExample()
{
    TArray<UProjectileData*> Data;
    for (int32 i = 0; i < 10000; ++i)
    {
        Data.Add(NewObject<UProjectileData>());
    }
    // GC需要遍历10000个对象!
}

// ===== 正确做法 =====
USTRUCT(BlueprintType)
struct FProjectileData
{
    GENERATED_BODY()

    UPROPERTY()
    float Damage = 10.0f;

    UPROPERTY()
    float Speed = 1000.0f;

    UPROPERTY()
    FVector Direction = FVector::ForwardVector;
};

UCLASS()
class UProjectileManager : public UObject
{
    GENERATED_BODY()

    UPROPERTY()
    TArray<FProjectileData> AllData;  // 单个UObject包含所有数据
};

void GoodExample()
{
    UProjectileManager* Manager = NewObject<UProjectileManager>();
    Manager->AllData.SetNum(10000);

    // 仅1个UObject, GC开销大幅降低
}
```

### 2.2 优化引用关系

```cpp
// ===== 优化策略2: 减少引用遍历深度 =====

// 问题: 深层引用链增加标记时间
// 解决: 扁平化引用结构

// ===== 错误示范: 深层引用 =====
UCLASS()
class UTreeNode : public UObject
{
    GENERATED_BODY()

    UPROPERTY()
    UTreeNode* LeftChild;   // 每个节点一个引用

    UPROPERTY()
    UTreeNode* RightChild;
};

// 1000层深度的树 = 1000次引用遍历
// GC需要递归遍历每个节点

// ===== 正确做法: 扁平化 =====
UCLASS()
class UTreeData : public UObject
{
    GENERATED_BODY()

    // 使用索引而非指针
    TArray<int32> NodeData;

    // 或使用数组一次性遍历
    UPROPERTY()
    TArray<UTreeNode*> AllNodes;  // 一次遍历所有节点
};

// 或者使用非UObject节点
USTRUCT()
struct FTreeNode
{
    int32 LeftIndex = INDEX_NONE;   // 索引引用
    int32 RightIndex = INDEX_NONE;  // 不参与GC
    int32 DataIndex = INDEX_NONE;
};
```

### 2.3 合理使用聚类

```cpp
// ===== 优化策略3: 利用聚类减少遍历 =====

// 检查聚类效率
void CheckClusterEfficiency()
{
    int32 ClusteredObjects = 0;
    int32 TotalObjects = GUObjectArray.GetObjectArrayNum();

    for (const FUObjectCluster& Cluster : GObjectClusters.Clusters)
    {
        if (Cluster.RootIndex != INDEX_NONE)
        {
            ClusteredObjects += Cluster.Objects.Num();
        }
    }

    float Efficiency = (float)ClusteredObjects / TotalObjects * 100.0f;

    UE_LOG(LogGC, Log, TEXT("Cluster efficiency: %.1f%%"), Efficiency);

    if (Efficiency < 30.0f)
    {
        UE_LOG(LogGC, Warning,
            TEXT("Low cluster efficiency! Consider enabling gc.CreateGCClusters"));
    }
}

// 确保大型资产使用聚类
void EnsureAssetClustered(UObject* Asset)
{
    if (!gc.CreateGCClusters)
        return;

    if (!Asset->HasAnyInternalFlags(ClusterRoot))
    {
        int32 SubObjectCount = CountSubObjects(Asset);

        if (SubObjectCount >= gc.MinGCClusterSize)
        {
            Asset->CreateCluster();

            UE_LOG(LogGC, Log,
                TEXT("Created cluster for %s with %d objects"),
                *Asset->GetName(), SubObjectCount);
        }
    }
}
```

### 2.4 优化GC时机

```cpp
// ===== 优化策略4: 在合适时机触发GC =====

// ===== 错误: 随机GC =====
void BadGameLoop()
{
    // GC可能在任何时候触发
    // 造成不可预测的卡顿
}

// ===== 正确: 控制GC时机 =====
class AMyGameMode : public AGameModeBase
{
    bool bJustLoadedLevel = false;
    bool bInPauseMenu = false;
    bool bPreparingForLevelChange = false;

public:
    void Tick(float DeltaTime) override
    {
        Super::Tick(DeltaTime);

        // 在关卡加载后触发完整GC
        if (bJustLoadedLevel)
        {
            CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS, true);
            bJustLoadedLevel = false;
        }

        // 在暂停菜单时完成挂起的GC
        if (bInPauseMenu && IsIncrementalGCPending())
        {
            FlushIncrementalGC();
        }

        // 在切换场景前预GC
        if (bPreparingForLevelChange)
        {
            CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS, true);
            bPreparingForLevelChange = false;
        }
    }

    void OnLevelLoaded()
    {
        bJustLoadedLevel = true;
    }

    void OnPauseMenuOpened()
    {
        bInPauseMenu = true;
    }

    void PrepareForLevelChange()
    {
        bPreparingForLevelChange = true;
    }
};
```

### 2.5 对象池化

```cpp
// ===== 优化策略5: 对象池减少GC压力 =====

template<typename T>
class TObjectPool
{
    TArray<T*> Pool;
    TArray<T*> Active;
    UClass* Class;
    UObject* Outer;

public:
    TObjectPool(UObject* InOuter, UClass* InClass, int32 InitialSize = 10)
        : Outer(InOuter), Class(InClass)
    {
        // 预创建对象
        for (int32 i = 0; i < InitialSize; ++i)
        {
            T* Obj = NewObject<T>(Outer, Class);
            Obj->AddToRoot();  // 防止GC回收
            Pool.Add(Obj);
        }
    }

    T* Acquire()
    {
        T* Obj;

        if (Pool.Num() > 0)
        {
            Obj = Pool.Pop();
        }
        else
        {
            // 池空, 创建新对象
            Obj = NewObject<T>(Outer, Class);
            Obj->AddToRoot();
        }

        Active.Add(Obj);
        return Obj;
    }

    void Release(T* Obj)
    {
        if (Active.Remove(Obj))
        {
            // 重置对象状态
            ResetObject(Obj);
            Pool.Add(Obj);
        }
    }

    ~TObjectPool()
    {
        // 清理所有对象
        for (T* Obj : Pool)
        {
            Obj->RemoveFromRoot();
        }
        for (T* Obj : Active)
        {
            Obj->RemoveFromRoot();
        }
    }

private:
    void ResetObject(T* Obj)
    {
        // 重置对象到初始状态
        // 具体实现取决于对象类型
    }
};

// 使用示例
TObjectPool<AProjectile>* ProjectilePool;

void SpawnProjectile(const FVector& Location)
{
    AProjectile* Proj = ProjectilePool->Acquire();
    Proj->SetActorLocation(Location);
    Proj->Activate();
}

void DestroyProjectile(AProjectile* Proj)
{
    Proj->Deactivate();
    ProjectilePool->Release(Proj);
    // 不会被GC, 可复用
}
```

---

## 三、调试工具与方法

### 3.1 控制台命令汇总

```
// GC调试命令完整列表

// ===== 基本命令 =====
gc                          // 显示GC帮助
stat gc                     // 显示GC统计

// ===== 强制操作 =====
gc.Flush                    // 立即完成所有挂起的GC
gc.CollectGarbage           // 强制触发一次GC

// ===== 调试开关 =====
gc.DumpAnalyticsToLog 1     // 输出GC分析日志
gc.GarbageReferenceTrackingEnabled 1  // 启用引用追踪
gc.LogGarbageEveryFrame 1   // 每帧记录垃圾对象

// ===== 验证选项 =====
gc.VerifyNoUnreachableObjects 1   // 验证不可达对象
gc.VerifyObjectsDestroyed 1       // 验证对象销毁

// ===== 对象命令 =====
obj list                    // 列出所有对象
obj list class=StaticMesh   // 列出特定类对象
obj refs <Name>             // 显示对象引用关系
obj garbage                 // 强制GC

// ===== 聚类命令 =====
gc.DumpGCClusters           // 输出聚类信息
gc.DebugGCClusters 1        // 调试聚类

// ===== 内存报告 =====
memreport -full             // 完整内存报告
memreport -lowmem           // 低内存报告
```

### 3.2 对象引用调试

```cpp
// 追踪对象引用关系

// ===== 方法1: 使用控制台命令 =====
// obj refs <ObjectName>

// ===== 方法2: 代码追踪 =====
void TraceObjectReferences(UObject* Target)
{
    UE_LOG(LogGC, Log, TEXT("=== Tracing references for: %s ==="),
        *Target->GetName());

    // 找到所有引用此对象的对象
    TArray<UObject*> Referencers;
    FReferenceFinder Finder(Referencers);
    Finder.FindReferences(Target);

    UE_LOG(LogGC, Log, TEXT("Referenced by %d objects:"), Referencers.Num());
    for (UObject* Ref : Referencers)
    {
        UE_LOG(LogGC, Log, TEXT("  - %s (%s)"),
            *Ref->GetName(), *Ref->GetClass()->GetName());
    }

    // 找到此对象引用的所有对象
    FSchemaView Schema = Target->GetClass()->GetSchema();
    UE_LOG(LogGC, Log, TEXT("References %d objects:"), Schema.NumMembers);

    Schema.ForEachReference(Target, [](UObject* Ref)
    {
        if (Ref)
        {
            UE_LOG(LogGC, Log, TEXT("  -> %s (%s)"),
                *Ref->GetName(), *Ref->GetClass()->GetName());
        }
    });
}
```

### 3.3 内存泄漏检测

```cpp
// 检测内存泄漏

// ===== 方法1: 对象计数追踪 =====
class FObjectCountTracker
{
    TMap<UClass*, int32> ObjectCounts;
    TMap<UClass*, int32> PreviousCounts;

public:
    void TakeSnapshot()
    {
        PreviousCounts = ObjectCounts;
        ObjectCounts.Empty();

        for (int32 i = 0; i < GUObjectArray.GetObjectArrayNum(); ++i)
        {
            FUObjectItem* Item = GUObjectArray.IndexToObject(i);
            if (Item && Item->Object)
            {
                UClass* Class = Item->Object->GetClass();
                ObjectCounts.FindOrAdd(Class)++;
            }
        }
    }

    void CompareSnapshots()
    {
        UE_LOG(LogGC, Log, TEXT("=== Object Count Changes ==="));

        for (auto& Pair : ObjectCounts)
        {
            int32 Current = Pair.Value;
            int32 Previous = PreviousCounts.FindRef(Pair.Key, 0);
            int32 Delta = Current - Previous;

            if (Delta != 0)
            {
                UE_LOG(LogGC, Log, TEXT("%s: %d -> %d (Delta: %+d)"),
                    *Pair.Key->GetName(), Previous, Current, Delta);
            }
        }
    }
};

// ===== 方法2: 检测AddToRoot泄漏 =====
void CheckRootObjectLeaks()
{
    int32 RootCount = 0;
    TArray<UObject*> PotentialLeaks;

    for (int32 i = 0; i < GUObjectArray.GetObjectArrayNum(); ++i)
    {
        FUObjectItem* Item = GUObjectArray.IndexToObject(i);
        if (Item && Item->Object && Item->Object->HasAnyFlags(RF_RootSet))
        {
            RootCount++;

            // 检查是否是泄漏
            // (长时间存在但不应该存在的根对象)
            if (IsPotentialLeak(Item->Object))
            {
                PotentialLeaks.Add(Item->Object);
            }
        }
    }

    UE_LOG(LogGC, Log, TEXT("Total root objects: %d"), RootCount);
    UE_LOG(LogGC, Log, TEXT("Potential leaks: %d"), PotentialLeaks.Num());

    for (UObject* Leak : PotentialLeaks)
    {
        UE_LOG(LogGC, Warning, TEXT("  Potential leak: %s"), *Leak->GetPathName());
    }
}
```

### 3.4 GC崩溃调试

```cpp
// 调试GC相关崩溃

// ===== 问题1: 访问已销毁对象 =====
// 症状: Access violation, 对象指针无效

void DebugAccessViolation(UObject* Suspect)
{
    if (!Suspect)
    {
        UE_LOG(LogGC, Error, TEXT("Null pointer access"));
        return;
    }

    FUObjectItem* Item = GUObjectArray.GetItem(Suspect);
    if (!Item)
    {
        UE_LOG(LogGC, Error, TEXT("Object not in global array - already destroyed?"));
        return;
    }

    // 检查GC标志
    if (Item->HasAnyFlags(Garbage | Unreachable))
    {
        UE_LOG(LogGC, Error, TEXT("Accessing garbage/unreachable object!"));
        UE_LOG(LogGC, Error, TEXT("  Object: %s"), *Suspect->GetPathName());
        UE_LOG(LogGC, Error, TEXT("  Flags: 0x%X"), Item->GetFlags());
        UE_LOG(LogGC, Error, TEXT("  Class: %s"), *Suspect->GetClass()->GetName());
    }
}

// ===== 问题2: 对象意外被回收 =====
// 症状: 持有的对象突然变成null

void DebugUnexpectedGC(UObject* LostObject, UObject* Owner)
{
    // 启用引用追踪
    gc.GarbageReferenceTrackingEnabled = 1;

    // 检查是否有UPROPERTY
    FSchemaView Schema = Owner->GetClass()->GetSchema();

    bool bFound = false;
    Schema.ForEachReference(Owner, [&bFound, LostObject](UObject* Ref)
    {
        if (Ref == LostObject)
        {
            bFound = true;
        }
    });

    if (bFound)
    {
        UE_LOG(LogGC, Log, TEXT("Object is tracked by schema"));
    }
    else
    {
        UE_LOG(LogGC, Error, TEXT("Object is NOT tracked - missing UPROPERTY?"));

        // 列出Owner的所有引用成员
        UE_LOG(LogGC, Log, TEXT("Owner's tracked references:"));
        Schema.ForEachReference(Owner, [](UObject* Ref)
        {
            UE_LOG(LogGC, Log, TEXT("  - %s"), Ref ? *Ref->GetName() : TEXT("NULL"));
        });
    }
}
```

---

## 四、常见问题与解决方案

### 4.1 GC卡顿

```cpp
// 问题: GC造成明显卡顿
// 诊断步骤:

void DiagnoseGCLag()
{
    UE_LOG(LogGC, Log, TEXT("=== GC Lag Diagnosis ==="));

    // 1. 检查GC配置
    UE_LOG(LogGC, Log, TEXT("Configuration:"));
    UE_LOG(LogGC, Log, TEXT("  TimeBetweenGCs: %.1f"), gc.TimeBetweenGCs);
    UE_LOG(LogGC, Log, TEXT("  AllowParallelGC: %d"), gc.AllowParallelGC);
    UE_LOG(LogGC, Log, TEXT("  AllowIncrementalReachability: %d"), gc.AllowIncrementalReachability);

    // 2. 检查对象数量
    int32 TotalObjects = GUObjectArray.GetObjectArrayNum();
    UE_LOG(LogGC, Log, TEXT("  Total Objects: %d"), TotalObjects);

    if (TotalObjects > 100000)
    {
        UE_LOG(LogGC, Warning, TEXT("High object count - consider optimization"));
    }

    // 3. 检查聚类效率
    int32 ClusteredCount = 0;
    for (const FUObjectCluster& Cluster : GObjectClusters.Clusters)
    {
        ClusteredCount += Cluster.Objects.Num();
    }

    float ClusterEfficiency = TotalObjects > 0 ?
        (float)ClusteredCount / TotalObjects : 0.0f;
    UE_LOG(LogGC, Log, TEXT("  Cluster Efficiency: %.1f%%"), ClusterEfficiency * 100.0f);

    // 4. 给出建议
    UE_LOG(LogGC, Log, TEXT("Recommendations:"));

    if (!gc.AllowIncrementalReachability)
    {
        UE_LOG(LogGC, Log, TEXT("  - Enable incremental GC for smoother frames"));
    }

    if (ClusterEfficiency < 0.3f)
    {
        UE_LOG(LogGC, Log, TEXT("  - Increase clustering for large assets"));
    }

    if (!gc.AllowParallelGC)
    {
        UE_LOG(LogGC, Log, TEXT("  - Enable parallel GC for faster processing"));
    }
}
```

### 4.2 内存持续增长

```cpp
// 问题: 内存不释放, 持续增长

void DiagnoseMemoryGrowth()
{
    UE_LOG(LogGC, Log, TEXT("=== Memory Growth Diagnosis ==="));

    // ===== 原因1: AddToRoot泄漏 =====
    int32 RootObjectCount = 0;
    for (int32 i = 0; i < GUObjectArray.GetObjectArrayNum(); ++i)
    {
        FUObjectItem* Item = GUObjectArray.IndexToObject(i);
        if (Item && Item->Object && Item->Object->HasAnyFlags(RF_RootSet))
        {
            RootObjectCount++;
        }
    }
    UE_LOG(LogGC, Log, TEXT("Root objects: %d"), RootObjectCount);

    // ===== 原因2: FGCObject未正确报告引用 =====
    gc.GarbageReferenceTrackingEnabled = 1;

    // ===== 原因3: 增量GC未完成 =====
    if (IsIncrementalGCPending())
    {
        UE_LOG(LogGC, Warning, TEXT("Incremental GC is pending"));
        UE_LOG(LogGC, Log, TEXT("Consider calling FlushIncrementalGC()"));
    }

    // ===== 原因4: 对象池未正确释放 =====
    // 检查自定义对象池
}
```

### 4.3 对象意外销毁

```cpp
// 问题: 持有的对象被GC销毁

// ===== 解决方案汇总 =====

class AMyActor : public AActor
{
    GENERATED_BODY()

    // ===== 问题1: 缺少UPROPERTY =====
    UObject* UnsafeRef;  // GC不知道这个引用!

    // 解决方案
    UPROPERTY()
    TObjectPtr<UObject> SafeRef;

    // ===== 问题2: 使用了弱引用但期望强引用行为 =====
    UPROPERTY()
    TWeakObjectPtr<UObject> WeakRef;  // 不阻止GC

    // 解决方案: 如果需要阻止GC
    UPROPERTY()
    TObjectPtr<UObject> StrongRef;

    // ===== 问题3: 临时引用问题 =====
    void TempRefProblem()
    {
        UObject* Temp = GetSomeObject();
        Temp->DoSomething();
        // Temp没有被任何UPROPERTY持有
        // GC后可能为null
    }

    // 解决方案
    UPROPERTY()
    TObjectPtr<UObject> CachedRef;

    void TempRefSolution()
    {
        CachedRef = GetSomeObject();  // 安全持有
        CachedRef->DoSomething();     // 安全
    }

    // ===== 问题4: AddToRoot使用不当 =====
    void ProperRootUsage()
    {
        UObject* Obj = NewObject<UObject>();
        Obj->AddToRoot();

        // 使用...
        Obj->DoSomething();

        // 使用完后必须移除
        Obj->RemoveFromRoot();
        Obj = nullptr;  // 允许GC回收
    }
};
```

---

## 五、生产环境最佳实践

### 5.1 平台配置模板

```cpp
// 不同平台/场景的推荐配置

// ===== PC高配置 =====
void ConfigurePC_HighEnd()
{
    gc.AllowParallelGC = 1;
    gc.CreateGCClusters = 1;
    gc.TimeBetweenGCs = 60.0f;
    gc.MaxObjectsInGame = 1000000;
    gc.AllowIncrementalReachability = 0;  // 性能足够, 不需要增量
}

// ===== PC中配置 =====
void ConfigurePC_MidRange()
{
    gc.AllowParallelGC = 1;
    gc.CreateGCClusters = 1;
    gc.TimeBetweenGCs = 45.0f;
    gc.MaxObjectsInGame = 500000;
    gc.AllowIncrementalReachability = 1;
    gc.IncrementalReachabilityTimeLimit = 0.003f;
}

// ===== 移动端 =====
void ConfigureMobile()
{
    gc.AllowParallelGC = 0;  // 移动端可能只有2-4核心
    gc.CreateGCClusters = 1;
    gc.TimeBetweenGCs = 30.0f;
    gc.MaxObjectsInGame = 200000;
    gc.AllowIncrementalReachability = 1;
    gc.IncrementalReachabilityTimeLimit = 0.002f;
    gc.IncrementalBeginDestroyGranularity = 5;
}

// ===== 主机 =====
void ConfigureConsole()
{
    gc.AllowParallelGC = 1;
    gc.CreateGCClusters = 1;
    gc.TimeBetweenGCs = 60.0f;
    gc.MaxObjectsInGame = 500000;
    gc.AllowIncrementalReachability = 1;
    gc.IncrementalReachabilityTimeLimit = 0.004f;
}
```

### 5.2 运行时监控

```cpp
// 生产环境GC监控组件

UCLASS(ClassGroup = "Performance", meta = (BlueprintSpawnableComponent))
class UGCMonitorComponent : public UActorComponent
{
    GENERATED_BODY()

    // 阈值配置
    UPROPERTY(EditDefaultsOnly, Category = "Thresholds")
    float MaxGCTimeMs = 16.0f;

    UPROPERTY(EditDefaultsOnly, Category = "Thresholds")
    float MaxMemoryMB = 1024.0f;

    UPROPERTY(EditDefaultsOnly, Category = "Thresholds")
    bool bLogEveryGC = false;

    // 警告回调
    UPROPERTY(BlueprintAssignable, Category = "Events")
    FOnGCWarning OnGCWarning;

    double LastGCStartTime;

public:
    virtual void BeginPlay() override
    {
        Super::BeginPlay();

        FCoreDelegates::OnPreGarbageCollection.AddUObject(
            this, &UGCMonitorComponent::OnPreGC);
        FCoreDelegates::OnPostGarbageCollection.AddUObject(
            this, &UGCMonitorComponent::OnPostGC);
    }

    void OnPreGC()
    {
        LastGCStartTime = FPlatformTime::Seconds();
    }

    void OnPostGC()
    {
        float Duration = (FPlatformTime::Seconds() - LastGCStartTime) * 1000.0f;

        // 检查GC时间
        if (Duration > MaxGCTimeMs)
        {
            FString Warning = FString::Printf(
                TEXT("GC time %.2fms exceeded threshold %.2fms"),
                Duration, MaxGCTimeMs);

            UE_LOG(LogGC, Warning, TEXT("%s"), *Warning);
            OnGCWarning.Broadcast(Warning);
        }

        if (bLogEveryGC)
        {
            UE_LOG(LogGC, Log, TEXT("GC: %.2fms"), Duration);
        }
    }

    virtual void TickComponent(float DeltaTime, ELevelTick TickType,
        FActorComponentTickFunction* ThisTickFunction) override
    {
        Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

        // 检查内存
        SIZE_T MemoryMB = FPlatformMemory::GetStats().TotalUsed / (1024 * 1024);
        if (MemoryMB > MaxMemoryMB)
        {
            FString Warning = FString::Printf(
                TEXT("Memory %.0fMB exceeded threshold %.0fMB"),
                (float)MemoryMB, MaxMemoryMB);

            OnGCWarning.Broadcast(Warning);
        }
    }
};
```

### 5.3 自动化测试

```cpp
// GC压力测试

class FGCTests
{
public:
    // 测试1: 大量对象创建销毁
    static void TestObjectChurn()
    {
        UE_LOG(LogGC, Log, TEXT("=== Object Churn Test ==="));

        TArray<UObject*> Objects;

        // 创建10000个对象
        for (int32 i = 0; i < 10000; ++i)
        {
            Objects.Add(NewObject<UObject>());
        }

        // 记录GC前状态
        SIZE_T MemBefore = FPlatformMemory::GetStats().TotalUsed;
        double TimeBefore = FPlatformTime::Seconds();

        // 清除引用
        Objects.Empty();

        // 强制GC
        CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS, true);

        // 检查结果
        SIZE_T MemAfter = FPlatformMemory::GetStats().TotalUsed;
        double TimeAfter = FPlatformTime::Seconds();

        UE_LOG(LogGC, Log, TEXT("Time: %.2fms"), (TimeAfter - TimeBefore) * 1000.0);
        UE_LOG(LogGC, Log, TEXT("Memory freed: %lld bytes"), MemBefore - MemAfter);
    }

    // 测试2: 聚类效率
    static void TestClusterEfficiency()
    {
        UE_LOG(LogGC, Log, TEXT("=== Cluster Efficiency Test ==="));

        // 创建带聚类的资产
        UMaterial* Mat = NewObject<UMaterial>();
        Mat->CreateCluster();

        // 添加子对象
        int32 ClusterIndex = GObjectClusters.ObjectToCluster[Mat->GetIndex()];
        for (int32 i = 0; i < 100; ++i)
        {
            UTexture2D* Tex = NewObject<UTexture2D>(Mat);
            GObjectClusters.AddObjectToCluster(ClusterIndex, Tex);
        }

        // 测试GC性能
        double StartTime = FPlatformTime::Seconds();
        CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS, false);
        double ClusteredTime = FPlatformTime::Seconds() - StartTime;

        UE_LOG(LogGC, Log, TEXT("Clustered GC: %.2fms for 101 objects"),
            ClusteredTime * 1000.0);
    }
};
```

---

## 六、课程总结

### 6.1 六节课回顾

| 课程 | 主题 | 核心内容 |
|------|------|----------|
| Lesson 1 | 基础架构 | GC设计目标、核心组件、执行流程 |
| Lesson 2 | 引用图 | FUObjectArray、Schema系统、FGCObject |
| Lesson 3 | 标记-清除 | 根对象集合、可达性分析、清除阶段 |
| Lesson 4 | 聚类系统 | 聚类创建、GC处理、溶解机制 |
| Lesson 5 | 增量GC | 增量可达性、写屏障、增量清除 |
| Lesson 6 | 性能调试 | 优化技巧、调试工具、最佳实践 |

### 6.2 关键知识点总结

```
┌─────────────────────────────────────────────────────────────────┐
│                    UE5 GC 核心知识点                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ===== 基础概念 =====                                            │
│  - UObject生命周期管理                                           │
│  - 标记-清除算法                                                 │
│  - 五阶段GC流程                                                  │
│                                                                 │
│  ===== 核心技术 =====                                            │
│  - FUObjectArray全局对象池                                       │
│  - Schema引用遍历                                                │
│  - 聚类批量优化                                                  │
│  - 增量GC分帧执行                                                │
│  - 并行GC多线程加速                                              │
│                                                                 │
│  ===== 优化策略 =====                                            │
│  - 减少UObject数量                                               │
│  - 扁平化引用结构                                                │
│  - 利用聚类                                                      │
│  - 对象池化                                                      │
│  - 控制GC时机                                                    │
│                                                                 │
│  ===== 调试工具 =====                                            │
│  - stat gc                                                      │
│  - obj refs                                                     │
│  - gc.VerifyNoUnreachableObjects                                │
│  - memreport                                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.3 实践建议

1. **理解GC原理** - 掌握标记-清除算法的工作方式
2. **使用UPROPERTY** - 确保所有引用被正确追踪
3. **合理使用聚类** - 对大型资产启用聚类
4. **配置增量GC** - 根据平台特点配置
5. **监控GC性能** - 使用stat gc持续监控
6. **避免常见陷阱** - AddToRoot泄漏、弱引用误用

---

## 参考资料

### 源码文件
- `Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollection.cpp`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/GarbageCollection.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectArray.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/GCObject.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectClusters.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/ReachabilityAnalysis.h`

### 官方文档
- [UE5 Memory Management](https://docs.unrealengine.com/5.0/en-US/memory-management-in-unreal-engine/)
- [Garbage Collection Reference](https://docs.unrealengine.com/5.0/en-US/garbage-collection-reference-in-unreal-engine/)
