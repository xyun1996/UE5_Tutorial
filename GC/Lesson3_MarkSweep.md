# Lesson 3: 标记-清除算法实现

## 课程概述

本节课深入讲解UE5的标记-清除(Mark-Sweep)算法实现，包括可达性分析、根对象集合、以及清除操作的细节。

**学习目标：**
- 理解标记-清除算法的基本原理
- 掌握根对象集合的构成和作用
- 了解可达性分析的具体实现
- 熟悉清除阶段的执行流程

---

## 一、标记-清除算法基础

### 1.1 算法原理

```
┌─────────────────────────────────────────────────────────────────┐
│                    标记-清除算法原理                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ===== 标记阶段 (Mark) =====                                     │
│  步骤:                                                          │
│  1. 从根对象集合出发                                             │
│  2. 遍历所有可达对象 (图的遍历)                                   │
│  3. 标记遍历到的对象为"可达"                                      │
│                                                                 │
│  ===== 清除阶段 (Sweep) =====                                    │
│  步骤:                                                          │
│  1. 遍历所有对象                                                 │
│  2. 未标记为"可达"的对象即为垃圾                                  │
│  3. 回收这些对象的内存                                           │
│                                                                 │
│  ===== 核心不变式 =====                                          │
│  - 可达对象 ⊂ 存活对象                                           │
│  - 不可达对象 = 垃圾对象                                         │
│  - 存活对象 = 可达对象 + 浮动垃圾                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 UE5的改进

```cpp
// UE5对传统算法的改进
// Source: GarbageCollection.cpp

/*
 * ===== 传统问题 =====
 * 1. 标记阶段需要暂停所有线程 (Stop-The-World)
 * 2. 清除阶段也会阻塞
 * 3. 大对象集导致明显卡顿
 * 4. 单线程处理效率低
 *
 * ===== UE5改进 =====
 * 1. 增量可达性分析 - 分帧完成标记
 * 2. 并行标记 - 多线程加速遍历
 * 3. 聚类优化 - 批量处理相关对象
 * 4. 增量清除 - 分批销毁对象
 * 5. 写屏障 - 处理并发修改
 */
```

---

## 二、根对象集合

### 2.1 根对象的定义

```
┌─────────────────────────────────────────────────────────────────┐
│                    根对象集合 (Root Set)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  定义:                                                          │
│  根对象是GC遍历的起点, 永远不会被回收                             │
│                                                                 │
│  构成:                                                          │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  1. 永久对象 (Permanent Objects)                       │    │
│  │     - 索引 0 ~ ObjFirstGCIndex-1                       │    │
│  │     - Engine核心对象, Class对象等                       │    │
│  │     - 永不参与GC                                       │    │
│  ├────────────────────────────────────────────────────────┤    │
│  │  2. 根集合对象 (Root Set Objects)                      │    │
│  │     - 调用 AddToRoot() 的对象                          │    │
│  │     - RF_RootSet 标志                                  │    │
│  │     - 手动管理生命周期                                  │    │
│  ├────────────────────────────────────────────────────────┤    │
│  │  3. FGCObject引用 (FGCObject References)               │    │
│  │     - 继承 FGCObject 的类报告的引用                     │    │
│  │     - 非UObject持有的UObject引用                       │    │
│  ├────────────────────────────────────────────────────────┤    │
│  │  4. 自定义根对象 (Custom Root Objects)                  │    │
│  │     - FCoreDelegates::OnGetGCObjects 回调              │    │
│  │     - 引擎/游戏可扩展                                   │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 根对象收集代码

```cpp
// 根对象收集实现
// Source: GarbageCollection.cpp

void GatherReachableObjects(TArray<UObject*>& Roots)
{
    TRACE_CPUPROFILER_EVENT_SCOPE(GC_GatherRoots);

    // ===== 1. 永久对象 (Never GCObjects) =====
    for (int32 i = 0; i < GUObjectArray.ObjFirstGCIndex; ++i)
    {
        FUObjectItem* Item = GUObjectArray.IndexToObject(i);
        if (Item && Item->Object)
        {
            Roots.Add(Item->Object);
        }
    }

    // ===== 2. AddToRoot() 标记的对象 =====
    {
        FScopeLock Lock(&GRootObjectsCritical);
        for (UObject* Obj : GRootObjects)
        {
            Roots.Add(Obj);
        }
    }

    // ===== 3. FGCObject 报告的对象 =====
    // UGCObjectReferencer 是一个特殊的UObject
    // 它的 AddReferencedObjects 会遍历所有 FGCObject
    UGCObjectReferencer::Get()->AddReferencedObjects(Collector);

    // ===== 4. 引用追踪回调 =====
    FCoreDelegates::OnGetGCObjects.Broadcast(Roots);

    UE_LOG(LogGarbage, Verbose,
        TEXT("Gathered %d root objects"), Roots.Num());
}
```

### 2.3 AddToRoot 机制

```cpp
// UObject::AddToRoot / RemoveFromRoot
// Source: UObject.cpp

void UObject::AddToRoot()
{
    // 设置Root标志
    AddFlags(RF_RootSet | RF_Standalone);

    // 加入根对象集合
    {
        FScopeLock Lock(&GRootObjectsCritical);
        GRootObjects.Add(this);
    }

    // 通知对象数组
    GUObjectArray.MarkObjectAsRoot(this);

    UE_LOG(LogGarbage, Verbose,
        TEXT("Object added to root: %s"), *GetName());
}

void UObject::RemoveFromRoot()
{
    // 清除Root标志
    ClearFlags(RF_RootSet | RF_Standalone);

    // 从根集合移除
    {
        FScopeLock Lock(&GRootObjectsCritical);
        GRootObjects.Remove(this);
    }

    // 通知对象数组
    GUObjectArray.UnmarkObjectAsRoot(this);

    UE_LOG(LogGarbage, Verbose,
        TEXT("Object removed from root: %s"), *GetName());
}
```

### 2.4 注册自定义根对象

```cpp
// 注册自定义根对象
// Source: CoreDelegates.h

// 定义回调类型
DECLARE_MULTICAST_DELEGATE_OneParam(FOnGetGCObjects, TArray<UObject*>& /*Roots*/);

// 注册示例
void RegisterCustomRoots()
{
    FCoreDelegates::OnGetGCObjects.AddLambda([](TArray<UObject*>& Roots)
    {
        // 添加单例对象作为根
        if (MySingleton::Get())
        {
            Roots.Add(MySingleton::Get());
        }

        // 添加全局管理器
        if (GAssetManager)
        {
            Roots.Add(GAssetManager);
        }
    });
}

// 取消注册
FDelegateHandle MyHandle;

void RegisterAndStoreHandle()
{
    MyHandle = FCoreDelegates::OnGetGCObjects.AddLambda([](TArray<UObject*>& Roots)
    {
        // ...
    });
}

void Unregister()
{
    FCoreDelegates::OnGetGCObjects.Remove(MyHandle);
}
```

---

## 三、可达性分析实现

### 3.1 标志位切换机制

```cpp
// UE5使用标志位切换而非清除
// Source: UObjectArray.h

// 为什么使用切换而非清除?
// 1. 避免遍历所有对象清除标志
// 2. O(1) 完成标志重置
// 3. 原子操作更安全

// 全局标志状态
static bool GReachabilityFlag0 = true;
static bool GReachabilityFlag1 = false;

void SwapReachableFlags()
{
    // 交换两大标志角色:
    // ReachabilityFlag0 <-> ReachabilityFlag1

    // 标记前:
    // RF0 = "上一轮可达"
    // RF1 = "上一轮可能不可达"

    // 标记后:
    // RF0 = "这一轮可能不可达" (将被清除)
    // RF1 = "这一轮可达" (新标记的)

    GReachabilityFlag0 = !GReachabilityFlag0;
    GReachabilityFlag1 = !GReachabilityFlag1;

    UE_LOG(LogGarbage, Verbose, TEXT("Swapped reachability flags"));
}
```

### 3.2 FReachabilityAnalysisState

```cpp
// 可达性分析状态机
// Source: ReachabilityAnalysisState.h

class FReachabilityAnalysisState
{
public:
    // ===== 状态枚举 =====
    enum class EState
    {
        Idle,                    // 空闲, 无GC进行
        MarkingRoots,            // 正在标记根对象
        TraversingReferences,    // 正在遍历引用图
        GatheringUnreachable,    // 正在收集不可达对象
        Completed                // 已完成
    };

    // ===== 并发控制 =====
    std::atomic<EState> CurrentState{EState::Idle};

    // ===== 工作队列 (支持并行) =====
    static constexpr int32 MaxWorkers = 16;

    TArray<UObject*> ObjectsToProcess[MaxWorkers];  // 待处理对象
    TArray<UObject*> ReachableObjects[MaxWorkers];  // 已标记可达

    // ===== 增量控制 =====
    double IterationStartTime;      // 当前迭代开始时间
    double TimeLimit;               // 时间限制(秒)

    // ===== 进度追踪 =====
    std::atomic<int32> NextObjectIndex;  // 下一个要处理的对象索引
    std::atomic<int32> ProcessedCount;   // 已处理对象计数

    // ===== 主要方法 =====
    bool PerformIncrementalAnalysis(double TimeLimit);
    void MarkObjectAsReachable(UObject* Object);
    void GatherUnreachableObjects(TArray<UObject*>& OutUnreachable);
    bool IsTimeLimitExceeded() const;

private:
    void ProcessObjectBatch(int32 WorkerIndex, int32 BatchSize);
};
```

### 3.3 标记单个对象

```cpp
// 标记一个对象为可达
// Source: GarbageCollection.cpp

bool UObject::MarkAsReachableInternal()
{
    FUObjectItem* Item = GetItem();

    // 确定当前轮次的可达标志
    EInternalObjectFlags TargetFlag = GReachabilityFlag0 ?
        ReachabilityFlag0 : ReachabilityFlag1;

    // 获取当前标志 (原子操作)
    int64 CurrentFlagsAndRefCount = Item->FlagsAndRefCount.load();
    EInternalObjectFlags Flags = (EInternalObjectFlags)(CurrentFlagsAndRefCount & 0xFFFFFFFF);

    // 如果已经标记, 返回false (避免重复处理)
    if (Flags & TargetFlag)
    {
        return false;
    }

    // 原子操作设置标志
    int64 NewFlagsAndRefCount = CurrentFlagsAndRefCount;
    NewFlagsAndRefCount |= (int64)TargetFlag;
    NewFlagsAndRefCount &= ~(int64)Unreachable;  // 清除Unreachable

    // CAS操作
    if (Item->FlagsAndRefCount.compare_exchange_weak(
        CurrentFlagsAndRefCount, NewFlagsAndRefCount))
    {
        return true;  // 新标记, 需要处理其引用
    }

    // CAS失败, 重试
    return MarkAsReachableInternal();
}
```

### 3.4 完整标记阶段

```cpp
// 执行可达性分析
// Source: GarbageCollection.cpp

void FRealtimeGC::PerformReachabilityAnalysis()
{
    TRACE_CPUPROFILER_EVENT_SCOPE(GC_ReachabilityAnalysis);

    // ===== Phase 1: 标记所有对象为"可能不可达" =====
    // 通过交换标志位实现, O(1)
    SwapReachableFlags();

    // ===== Phase 2: 收集根对象引用 =====
    TArray<UObject*> RootSet;
    GatherReachableObjects(RootSet);

    UE_LOG(LogGarbage, Verbose,
        TEXT("Starting reachability with %d roots"), RootSet.Num());

    // ===== Phase 3: 遍历引用图 =====
    TFastReferenceCollector<FGCReferenceProcessor> Collector;

    for (UObject* Root : RootSet)
    {
        // 标记根对象为可达
        if (MarkAsReachable(Root))
        {
            // 加入工作队列
            Collector.AddObjectToProcess(Root);
        }
    }

    // ===== Phase 4: 并行处理引用图 =====
    Collector.ProcessAllReferences();

    // ===== Phase 5: 处理聚类 =====
    ProcessClusters();

    UE_LOG(LogGarbage, Verbose,
        TEXT("Reachability analysis complete"));
}
```

---

## 四、聚类处理

### 4.1 聚类的作用

```
┌─────────────────────────────────────────────────────────────────┐
│                    GC 聚类优化                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  问题: 大型Asset包含数百个子对象                                  │
│        - 每个子对象都需要独立遍历                                │
│        - 引用检查开销大                                          │
│        - O(N) 复杂度                                            │
│                                                                 │
│  解决: 将相关对象聚成一组                                        │
│        - 只检查聚类根的可达性                                    │
│        - 组内对象命运一致                                        │
│        - O(M) 复杂度, M << N                                   │
│                                                                 │
│  示例: 材质Asset                                                │
│  ┌────────────────────────────────────┐                         │
│  │  UMaterial (ClusterRoot)           │                         │
│  │  ├── UTexture2D (Clustered)        │                         │
│  │  ├── UTexture2D (Clustered)        │                         │
│  │  ├── UMaterialExpression (Clustered)│                        │
│  │  └── ...                           │                         │
│  └────────────────────────────────────┘                         │
│                                                                 │
│  聚类前: 检查100个对象 → 100次遍历                               │
│  聚类后: 检查1个聚类根 → 1次遍历                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 聚类标记逻辑

```cpp
// 处理聚类
// Source: GarbageCollection.cpp

void ProcessClusters()
{
    TRACE_CPUPROFILER_EVENT_SCOPE(GC_ProcessClusters);

    for (FUObjectCluster& Cluster : GObjectClusters.Clusters)
    {
        if (Cluster.RootIndex == INDEX_NONE)
            continue;  // 空槽位

        UObject* RootObj = GUObjectArray.IndexToObject(Cluster.RootIndex);
        FUObjectItem* RootItem = GUObjectArray.GetItem(RootObj);

        // ===== 情况1: 聚类根可达 =====
        if (RootItem->IsReachable())
        {
            // 标记整个聚类为可达
            for (int32 ObjIndex : Cluster.Objects)
            {
                UObject* Obj = GUObjectArray.IndexToObject(ObjIndex);
                MarkAsReachable(Obj);
            }

            // 传播可达性到引用的聚类
            for (int32 RefClusterIndex : Cluster.ReferencedClusters)
            {
                FUObjectCluster& RefCluster = GObjectClusters.Clusters[RefClusterIndex];
                UObject* RefRoot = GUObjectArray.IndexToObject(RefCluster.RootIndex);
                MarkAsReachable(RefRoot);
            }
        }
        // ===== 情况2: 聚类根不可达 =====
        else
        {
            // 检查是否有外部引用到聚类内对象
            bool bHasExternalRef = false;

            for (int32 MutableIndex : Cluster.MutableObjects)
            {
                FUObjectItem* Item = GUObjectArray.GetItem(MutableIndex);
                if (Item->IsReachable())
                {
                    bHasExternalRef = true;
                    break;
                }
            }

            if (bHasExternalRef)
            {
                // 有外部引用, 需要溶解聚类
                Cluster.bNeedsDissolving = true;
            }
        }
    }
}
```

---

## 五、清除阶段

### 5.1 清除流程

```cpp
// 清除阶段实现
// Source: GarbageCollection.cpp

void IncrementalPurgeGarbage(bool bForceComplete)
{
    TRACE_CPUPROFILER_EVENT_SCOPE(GC_Purge);

    // ===== Phase 1: 收集不可达对象 =====
    TArray<UObject*> UnreachableObjects;
    GatherUnreachableObjects(UnreachableObjects);

    UE_LOG(LogGarbage, Verbose,
        TEXT("Found %d unreachable objects"), UnreachableObjects.Num());

    // ===== Phase 2: 开始销毁 (异步) =====
    for (UObject* Obj : UnreachableObjects)
    {
        if (!Obj->IsPendingKill())
        {
            // 调用 BeginDestroy
            Obj->ConditionalBeginDestroy();

            // 加入待销毁队列
            GObjectsPendingDestroy.Add(Obj);
        }
    }

    // ===== Phase 3: 完成销毁 =====
    while (GObjectsPendingDestroy.Num() > 0)
    {
        UObject* Obj = GObjectsPendingDestroy.Pop();

        // 等待对象准备好销毁
        if (Obj->IsReadyForFinishDestroy())
        {
            // 调用 FinishDestroy
            Obj->ConditionalFinishDestroy();

            // 从对象数组移除
            GUObjectArray.FreeUObjectIndex(Obj);

            // 调用析构函数
            Obj->~UObject();

            // 释放内存
            FMemory::Free(Obj);
        }
        else
        {
            // 还不能销毁, 放回队列末尾
            GObjectsPendingDestroy.Insert(Obj, 0);
        }

        // 增量控制
        if (!bForceComplete && TimeExceeded())
        {
            break;
        }
    }
}
```

### 5.2 BeginDestroy vs FinishDestroy

```cpp
// 对象销毁的两阶段
// Source: UObject.cpp

/**
 * BeginDestroy - 异步销毁开始
 *
 * 调用时机: 游戏线程
 * 目的: 执行游戏逻辑相关的清理
 *
 * 注意事项:
 * - 可以检测是否可以继续销毁
 * - 可能被渲染线程阻塞
 * - 必须调用 Super::BeginDestroy()
 */
void UObject::BeginDestroy()
{
    // 1. 标记为正在销毁
    AddFlags(RF_BeginDestroyed);

    // 2. 通知所有监听者
    UninitializeSingletonIfNeeded(this);

    // 3. 取消所有异步任务
    CancelAsyncTasks();

    // 4. 清除弱引用
    ClearWeakReferences();
}

/**
 * FinishDestroy - 销毁完成
 *
 * 调用时机: 渲染线程完成后
 * 目的: 执行最终资源释放
 *
 * 注意事项:
 * - 此时对象已从所有容器移除
 * - 可以安全释放所有资源
 * - 必须调用 Super::FinishDestroy()
 */
void UObject::FinishDestroy()
{
    // 1. 标记为销毁完成
    AddFlags(RF_FinishDestroyed);

    // 2. 清除属性值
    ClearPropertyValues();

    // 3. 释放外部资源
    ReleaseResources();

    // 4. 调用父类
    Super::FinishDestroy();
}

/**
 * IsReadyForFinishDestroy - 检查是否准备好完成销毁
 *
 * 返回值:
 * - true: 可以调用 FinishDestroy
 * - false: 还需要等待
 *
 * 子类可重写此方法延迟销毁
 */
bool UObject::IsReadyForFinishDestroy()
{
    // 默认实现: 检查渲染代理
    return !HasRenderProxy();
}
```

### 5.3 重写销毁方法示例

```cpp
// 自定义销毁行为示例

class UMyResourceObject : public UObject
{
    // GPU资源
    FBufferRHIRef GPUBuffer;

    // 异步任务
    FGraphEventRef AsyncTask;

public:
    virtual void BeginDestroy() override
    {
        // 1. 先调用父类
        Super::BeginDestroy();

        // 2. 取消异步任务
        if (AsyncTask.IsValid())
        {
            AsyncTask->Wait();
            AsyncTask = nullptr;
        }

        // 3. 释放CPU端资源
        // ...
    }

    virtual bool IsReadyForFinishDestroy() override
    {
        // 等待GPU资源释放
        if (GPUBuffer.IsValid())
        {
            // 检查渲染命令是否完成
            return !GPUBuffer->IsPendingRender();
        }

        return Super::IsReadyForFinishDestroy();
    }

    virtual void FinishDestroy() override
    {
        // 释放GPU资源
        if (GPUBuffer.IsValid())
        {
            GPUBuffer.SafeRelease();
        }

        Super::FinishDestroy();
    }
};
```

---

## 六、弱引用处理

### 6.1 弱引用清空

```cpp
// GC时清空弱引用
// Source: GarbageCollection.cpp

void ClearWeakReferences()
{
    TRACE_CPUPROFILER_EVENT_SCOPE(GC_ClearWeakRefs);

    // 遍历所有对象
    for (int32 i = 0; i < GUObjectArray.GetObjectArrayNum(); ++i)
    {
        FUObjectItem* Item = GUObjectArray.IndexToObject(i);
        if (!Item || !Item->Object)
            continue;

        // 检查是否为垃圾对象
        if (Item->HasAnyFlags(Unreachable | Garbage))
        {
            UObject* Obj = Item->Object;

            // 设置垃圾标志
            Item->SetFlags(Garbage);

            // 清空指向此对象的所有弱引用
            // (弱引用系统内部维护了反向映射)
            ClearObjectWeakReferences(Obj);

            // 通知弱引用所有者
            OnObjectGarbageCollected(Obj);
        }
    }
}
```

### 6.2 TWeakObjectPtr 实现

```cpp
// 弱引用指针实现
// Source: UObjectWeakReferences.h

template<typename T>
class TWeakObjectPtr
{
public:
    // 存储: 对象索引 + 序列号
    FWeakObjectPtr InternalWeakReference;

    // 访问对象
    T* Get() const
    {
        return InternalWeakReference.Get<T>();
    }

    // 检查有效性
    bool IsValid() const
    {
        return InternalWeakReference.IsValid();
    }

    // 重置
    void Reset()
    {
        InternalWeakReference.Reset();
    }
};

// 底层实现
class FWeakObjectPtr
{
    int32 ObjectIndex;        // 在GUObjectArray中的索引
    int32 ObjectSerialNumber; // 序列号(防止索引复用问题)

public:
    UObject* Get() const
    {
        // 1. 检查索引有效
        if (ObjectIndex < 0 || ObjectIndex >= GUObjectArray.GetObjectArrayNum())
        {
            return nullptr;
        }

        // 2. 获取对象条目
        FUObjectItem* Item = GUObjectArray.IndexToObject(ObjectIndex);
        if (!Item || !Item->Object)
        {
            return nullptr;
        }

        // 3. 检查序列号匹配 (防止索引复用导致误访问)
        if (Item->SerialNumber != ObjectSerialNumber)
        {
            return nullptr;
        }

        // 4. 检查对象未被销毁
        if (Item->HasAnyFlags(Garbage))
        {
            return nullptr;
        }

        return Item->Object;
    }

    bool IsValid() const
    {
        return Get() != nullptr;
    }

    void Reset()
    {
        ObjectIndex = INDEX_NONE;
        ObjectSerialNumber = 0;
    }
};
```

---

## 七、调试与验证

### 7.1 GC验证模式

```cpp
// 启用GC验证
// Source: GarbageCollection.cpp

void EnableGCVerification()
{
    // 验证可达性分析正确性
    gc.VerifyNoUnreachableObjects = 1;

    // 验证销毁完成
    gc.VerifyObjectsDestroyed = 1;

    // 详细日志
    gc.DumpAnalyticsToLog = 1;
}

// 验证逻辑
void VerifyGCResults()
{
    UE_LOG(LogGarbage, Log, TEXT("Verifying GC results..."));

    // 检查所有标记为不可达的对象确实不可达
    for (int32 i = 0; i < GUObjectArray.GetObjectArrayNum(); ++i)
    {
        FUObjectItem* Item = GUObjectArray.IndexToObject(i);
        if (!Item || !Item->Object)
            continue;

        if (Item->HasAnyFlags(Unreachable))
        {
            // 尝试从根集合到达此对象
            if (CanReachFromRoot(Item->Object))
            {
                UE_LOG(LogGarbage, Error,
                    TEXT("OBJECT MARKED UNREACHABLE BUT ACTUALLY REACHABLE: %s"),
                    *Item->Object->GetPathName());

                // 在开发版本中断言
                checkf(false, TEXT("GC correctness failure!"));
            }
        }
    }

    UE_LOG(LogGarbage, Log, TEXT("GC verification passed"));
}
```

### 7.2 控制台命令

```
// GC调试命令

// ===== 查看对象引用 =====
obj refs <ObjectName>
// 显示对象的所有引用和被引用关系

// ===== 查看GC统计 =====
gc.PrintStats
// 打印GC统计信息

// ===== 强制GC =====
gc.Flush
// 执行完整GC周期

// ===== 查看对象列表 =====
obj list
// 列出所有对象

obj list class=StaticMesh
// 列出特定类对象

// ===== 查看特定对象 =====
obj dump <ObjectName>
// 显示对象详细信息
```

---

## 八、实践练习

### 练习1: 监控GC性能

```cpp
// 创建GC性能监控器

class FGCMonitor
{
    static double LastGCTime;
    static int32 TotalGCCycles;
    static double TotalGCDuration;
    static double MaxGCDuration;

public:
    static void Initialize()
    {
        // 监听GC完成事件
        FCoreDelegates::OnPostGarbageCollection.AddStatic(&FGCMonitor::OnGCComplete);
    }

    static void OnGCComplete()
    {
        TotalGCCycles++;

        // 获取GC统计
        FGCPerfStats Stats = GetGCPerfStats();

        double Duration = Stats.TotalTime;
        TotalGCDuration += Duration;
        MaxGCDuration = FMath::Max(MaxGCDuration, Duration);

        // 记录详细统计
        UE_LOG(LogTemp, Log,
            TEXT("GC Cycle %d: %.2fms (Mark: %.2fms, Sweep: %.2fms, Clusters: %d)"),
            TotalGCCycles,
            Duration * 1000.0,
            Stats.MarkTime * 1000.0,
            Stats.SweepTime * 1000.0,
            Stats.ClustersProcessed);
    }

    static void PrintSummary()
    {
        UE_LOG(LogTemp, Log, TEXT("=== GC Summary ==="));
        UE_LOG(LogTemp, Log, TEXT("  Total Cycles: %d"), TotalGCCycles);
        UE_LOG(LogTemp, Log, TEXT("  Total Time: %.2fms"), TotalGCDuration * 1000.0);
        UE_LOG(LogTemp, Log, TEXT("  Avg Time: %.2fms"),
            TotalGCDuration / TotalGCCycles * 1000.0);
        UE_LOG(LogTemp, Log, TEXT("  Max Time: %.2fms"), MaxGCDuration * 1000.0);
    }
};
```

### 练习2: 处理GC相关问题

```cpp
// 解决对象被错误回收问题

class AMyActor : public AActor
{
    GENERATED_BODY()

    // ===== 问题1: 缺少UPROPERTY =====
    UObject* UnsafeRef;  // GC不知道这个引用!

    // 解决方案
    UPROPERTY()
    UObject* SafeRef;

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
        // Temp没有被任何UPROPERTY持有
        // GC后可能为null
        Temp->DoSomething();  // 可能崩溃!
    }

    // 解决方案
    UPROPERTY()
    TObjectPtr<UObject> CachedRef;

    void TempRefSolution()
    {
        CachedRef = GetSomeObject();  // 安全持有
        CachedRef->DoSomething();     // 安全
    }

    // ===== 问题4: AddToRoot泄漏 =====
    void LeakExample()
    {
        UObject* Obj = NewObject<UObject>();
        Obj->AddToRoot();
        // 忘记 RemoveFromRoot!
        // Obj永远不会被回收
    }

    void ProperRootUsage()
    {
        UObject* Obj = NewObject<UObject>();
        Obj->AddToRoot();

        // 使用完后
        Obj->RemoveFromRoot();
        Obj = nullptr;  // 允许GC回收
    }
};
```

---

## 九、总结

### 本节课要点

1. **算法原理**
   - 标记阶段从根对象遍历引用图
   - 清除阶段回收不可达对象
   - UE5增加了增量、并行、聚类优化

2. **根对象集合**
   - 永久对象 + AddToRoot对象 + FGCObject引用
   - 可通过回调注册自定义根对象

3. **可达性分析**
   - 标志位切换机制 O(1)
   - 原子操作保证线程安全
   - 聚类优化减少遍历开销

4. **清除阶段**
   - BeginDestroy/FinishDestroy两阶段
   - 异步销毁支持
   - IsReadyForFinishDestroy可控制时机

5. **弱引用**
   - 安全的对象访问
   - 序列号防止索引复用
   - GC时自动清空

6. **调试工具**
   - gc.VerifyNoUnreachableObjects验证正确性
   - obj refs查看引用关系
   - stat gc查看统计

### 下一节课预告

下一节课将深入讲解**GC聚类系统**，包括：
- 聚类的创建和管理
- 聚类在GC中的处理逻辑
- 聚类溶解机制
- 聚类最佳实践

---

## 参考资料

### 源码文件
- `Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollection.cpp`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/UObjectClusters.cpp`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/ReachabilityAnalysisState.h`

### 相关课程
- Lesson 2: 对象追踪与引用图
- Lesson 4: GC聚类系统详解
