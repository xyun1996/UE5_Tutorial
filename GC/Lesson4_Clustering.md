# Lesson 4: GC聚类系统详解

## 课程概述

本节课深入讲解UE5的GC聚类(Clustering)系统，这是UE优化GC性能的核心技术之一。

**学习目标：**
- 理解聚类系统解决什么问题
- 掌握聚类的数据结构和创建流程
- 了解聚类在GC中的处理逻辑
- 学会正确使用聚类优化性能

---

## 一、聚类系统设计原理

### 1.1 问题背景

```
┌─────────────────────────────────────────────────────────────────┐
│                    为什么需要聚类?                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  问题: Asset包含大量子对象                                       │
│                                                                 │
│  示例: 一个材质Asset                                            │
│  ┌─────────────────────────────────────────────────────┐       │
│  │  UMaterial (主对象)                                  │       │
│  │  ├── UTexture2D (Diffuse)        [Obj 1]           │       │
│  │  ├── UTexture2D (Normal)         [Obj 2]           │       │
│  │  ├── UTexture2D (Roughness)      [Obj 3]           │       │
│  │  ├── UMaterialExpressionTexture  [Obj 4]           │       │
│  │  ├── UMaterialExpressionMultiply [Obj 5]           │       │
│  │  ├── UMaterialExpressionAdd      [Obj 6]           │       │
│  │  └── ... (可能数十到数百个子对象)                    │       │
│  └─────────────────────────────────────────────────────┘       │
│                                                                 │
│  传统方式: 每个对象单独GC                                        │
│  ┌─────────────────────────────────────────────────────┐       │
│  │  - 需要遍历每个对象的引用                            │       │
│  │  - 需要检查每个对象的可达性                          │       │
│  │  - 对象数N → O(N)复杂度                             │       │
│  │  - 大量小对象 = 大量GC开销                          │       │
│  └─────────────────────────────────────────────────────┘       │
│                                                                 │
│  聚类方式: 作为整体处理                                          │
│  ┌─────────────────────────────────────────────────────┐       │
│  │  - 只检查聚类根的可达性                              │       │
│  │  - 聚类内对象命运一致                                │       │
│  │  - 聚类数M (M << N) → O(M)复杂度                    │       │
│  │  - 大幅减少GC遍历开销                                │       │
│  └─────────────────────────────────────────────────────┘       │
│                                                                 │
│  性能对比:                                                       │
│  ┌─────────────────────────────────────────────────────┐       │
│  │  100个对象的材质:                                    │       │
│  │  - 无聚类: 100次可达性检查                           │       │
│  │  - 有聚类: 1次可达性检查                             │       │
│  │  - 性能提升: 100倍                                  │       │
│  └─────────────────────────────────────────────────────┘       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 聚类设计目标

```cpp
// 聚类系统设计目标
// Source: UObjectClusters.h

/*
 * 1. 减少GC遍历开销
 *    - 聚类内对象不需要单独遍历
 *    - 减少可达性检查次数
 *    - 从O(N)优化到O(M)
 *
 * 2. 保持语义正确性
 *    - 聚类对象生命周期一致
 *    - 正确处理跨聚类引用
 *    - 正确处理外部引用
 *
 * 3. 支持动态修改
 *    - 聚类可以溶解
 *    - 支持运行时对象加入/离开
 *    - 处理聚类状态变化
 *
 * 4. 最小化内存开销
 *    - 使用索引而非指针
 *    - 紧凑的数据结构
 *    - 高效的存储布局
 */
```

---

## 二、聚类数据结构

### 2.1 FUObjectCluster 结构

```cpp
// 聚类数据结构
// Source: UObjectClusters.h

struct FUObjectCluster
{
    // ===== 基本标识 =====
    int32 RootIndex;              // 聚类根对象索引

    // ===== 聚类内容 =====
    TArray<int32> Objects;        // 聚类内对象索引列表
    TArray<int32> MutableObjects; // 有外部引用的对象

    // ===== 聚类间关系 =====
    TArray<int32> ReferencedClusters;   // 本聚类引用的其他聚类
    TArray<int32> ReferencedByClusters; // 引用本聚类的其他聚类

    // ===== 状态标志 =====
    mutable std::atomic<bool> bNeedsDissolving; // 是否需要溶解

    // ===== 辅助方法 =====
    bool IsObjectInCluster(int32 ObjIndex) const
    {
        return Objects.Contains(ObjIndex);
    }

    int32 GetNumObjects() const { return Objects.Num(); }

    bool IsValid() const { return RootIndex != INDEX_NONE; }
};
```

### 2.2 聚类管理器

```cpp
// 全局聚类管理
// Source: UObjectClusters.h

class FUObjectClusterContainer
{
public:
    // ===== 聚类存储 =====
    TArray<FUObjectCluster> Clusters;

    // ===== 索引映射 =====
    TArray<int32> ObjectToCluster;  // 每个对象所属聚类索引

    // ===== 空闲槽位 =====
    TArray<int32> AvailableClusterIndices;  // 可复用的聚类索引

    // ===== 聚类操作 =====

    // 创建新聚类
    int32 CreateCluster(UObject* RootObject);

    // 销毁聚类
    void DestroyCluster(int32 ClusterIndex);

    // 添加对象到聚类
    void AddObjectToCluster(int32 ClusterIndex, UObject* Object);

    // 从聚类移除对象
    void RemoveObjectFromCluster(int32 ClusterIndex, UObject* Object);

    // 溶解聚类 (GC时使用)
    void DissolveCluster(int32 ClusterIndex);

    // ===== 查询方法 =====

    // 检查对象是否在聚类中
    bool IsInCluster(UObject* Object) const;

    // 获取对象所属聚类
    int32 GetClusterIndex(UObject* Object) const;

    // 获取聚类指针
    FUObjectCluster* FindCluster(UObject* Object);

    // 获取统计信息
    void GetStats(int32& TotalClusters, int32& TotalObjects) const;
};

// 全局实例
extern FUObjectClusterContainer GObjectClusters;
```

### 2.3 聚类标志位

```cpp
// 对象的聚类相关标志
// Source: UObjectArray.h

enum EInternalObjectFlags
{
    // ... 其他标志 ...

    // ===== 聚类标志 =====
    ClusterRoot          = 0x10000000,  // 是聚类根对象
    ReachableInCluster   = 0x20000000,  // 聚类内可达
    WithinCluster        = 0x40000000,  // 在某个聚类内

    // ===== 标志组合 =====
    ClusterFlags = ClusterRoot | ReachableInCluster | WithinCluster
};

// 检查聚类状态的辅助方法
namespace UE::GC::Cluster
{
    FORCEINLINE bool IsClusterRoot(const UObject* Obj)
    {
        return Obj->HasAnyInternalFlags(ClusterRoot);
    }

    FORCEINLINE bool IsInCluster(const UObject* Obj)
    {
        return Obj->HasAnyInternalFlags(WithinCluster);
    }

    FORCEINLINE bool IsReachableInCluster(const UObject* Obj)
    {
        return Obj->HasAnyInternalFlags(ReachableInCluster);
    }
}
```

---

## 三、聚类创建

### 3.1 创建时机

```cpp
// 聚类创建触发点
// Source: UObjectClusters.cpp

void UObject::CreateCluster()
{
    // ===== 条件检查 =====

    // 1. 检查聚类功能是否启用
    if (!gc.CreateGCClusters)
    {
        return;
    }

    // 2. 检查是否已经是聚类根
    if (HasAnyInternalFlags(ClusterRoot))
    {
        return;
    }

    // 3. 检查是否已经在其他聚类中
    if (HasAnyInternalFlags(WithinCluster))
    {
        return;
    }

    // ===== 创建聚类 =====
    int32 ClusterIndex = GObjectClusters.CreateCluster(this);

    // 设置标志
    AddInternalFlags(ClusterRoot);

    // 递归添加相关对象
    AddObjectsToCluster(ClusterIndex);

    UE_LOG(LogGarbage, Verbose,
        TEXT("Created cluster %d for %s with %d objects"),
        ClusterIndex, *GetName(),
        GObjectClusters.Clusters[ClusterIndex].Objects.Num());
}

// 典型创建时机:
// 1. Package加载完成时
// 2. Asset创建时 (UMaterial, UStaticMesh等)
// 3. 手动调用 CreateCluster()
```

### 3.2 聚类创建流程

```cpp
// 聚类创建详细流程
// Source: UObjectClusters.cpp

int32 FUObjectClusterContainer::CreateCluster(UObject* RootObject)
{
    TRACE_CPUPROFILER_EVENT_SCOPE(CreateGCCluster);

    // ===== 1. 分配聚类索引 =====
    int32 ClusterIndex;
    if (AvailableClusterIndices.Num() > 0)
    {
        // 复用空闲槽位
        ClusterIndex = AvailableClusterIndices.Pop();
    }
    else
    {
        // 创建新槽位
        ClusterIndex = Clusters.AddDefaulted();
    }

    FUObjectCluster& Cluster = Clusters[ClusterIndex];

    // ===== 2. 初始化聚类 =====
    Cluster.RootIndex = RootObject->GetIndex();
    Cluster.Objects.Reset();
    Cluster.MutableObjects.Reset();
    Cluster.ReferencedClusters.Reset();
    Cluster.ReferencedByClusters.Reset();
    Cluster.bNeedsDissolving = false;

    // ===== 3. 设置对象到聚类映射 =====
    int32 RootIndex = RootObject->GetIndex();

    // 确保映射数组足够大
    if (ObjectToCluster.Num() <= RootIndex)
    {
        ObjectToCluster.AddDefaulted(RootIndex - ObjectToCluster.Num() + 1);
    }
    ObjectToCluster[RootIndex] = ClusterIndex;

    // ===== 4. 设置聚类根标志 =====
    RootObject->AddInternalFlags(ClusterRoot);

    return ClusterIndex;
}
```

### 3.3 添加对象到聚类

```cpp
// 添加对象到聚类
// Source: UObjectClusters.cpp

void FUObjectClusterContainer::AddObjectToCluster(
    int32 ClusterIndex,
    UObject* Object)
{
    FUObjectCluster& Cluster = Clusters[ClusterIndex];
    int32 ObjIndex = Object->GetIndex();

    // ===== 1. 添加到聚类对象列表 =====
    Cluster.Objects.Add(ObjIndex);

    // ===== 2. 设置对象到聚类映射 =====
    if (ObjectToCluster.Num() <= ObjIndex)
    {
        ObjectToCluster.AddDefaulted(ObjIndex - ObjectToCluster.Num() + 1);
    }
    ObjectToCluster[ObjIndex] = ClusterIndex;

    // ===== 3. 设置聚类内标志 =====
    Object->AddInternalFlags(WithinCluster);

    // ===== 4. 递归处理子对象 =====
    FSchemaView Schema = Object->GetClass()->GetSchema();

    Schema.ForEachReference(Object, [ClusterIndex, this](UObject* Ref)
    {
        if (Ref && CanAddToCluster(Ref))
        {
            AddObjectToCluster(ClusterIndex, Ref);
        }
    });
}

// 判断对象是否可加入聚类
bool CanAddToCluster(UObject* Object)
{
    // 不能加入聚类的条件:

    // 1. 已经在其他聚类中
    if (Object->HasAnyInternalFlags(WithinCluster | ClusterRoot))
    {
        return false;
    }

    // 2. 是根对象 (不应该被聚类管理)
    if (Object->HasAnyFlags(RF_RootSet | RF_Standalone))
    {
        return false;
    }

    // 3. 是运行时动态创建的对象
    // (某些情况下不希望聚类)

    return true;
}
```

---

## 四、聚类GC处理

### 4.1 聚类可达性传播

```cpp
// 聚类可达性处理
// Source: GarbageCollection.cpp

void ProcessClusterReachability()
{
    TRACE_CPUPROFILER_EVENT_SCOPE(GC_ProcessClusters);

    // 遍历所有聚类
    for (int32 ClusterIdx = 0; ClusterIdx < GObjectClusters.Clusters.Num(); ++ClusterIdx)
    {
        FUObjectCluster& Cluster = GObjectClusters.Clusters[ClusterIdx];

        // 跳过空槽位
        if (Cluster.RootIndex == INDEX_NONE)
        {
            continue;
        }

        UObject* RootObj = GUObjectArray.IndexToObject(Cluster.RootIndex);
        FUObjectItem* RootItem = GUObjectArray.GetItem(RootObj);

        // ===== 情况1: 聚类根可达 =====
        if (RootItem->IsReachable())
        {
            // 标记整个聚类为可达
            for (int32 ObjIdx : Cluster.Objects)
            {
                UObject* Obj = GUObjectArray.IndexToObject(ObjIdx);
                if (Obj)
                {
                    MarkAsReachable(Obj);
                }
            }

            // 传播到引用的聚类
            for (int32 RefClusterIdx : Cluster.ReferencedClusters)
            {
                PropagateReachabilityToCluster(RefClusterIdx);
            }
        }
        // ===== 情况2: 聚类根不可达 =====
        else
        {
            // 检查是否有外部引用到聚类内对象
            bool bHasExternalRef = false;

            for (int32 MutableIdx : Cluster.MutableObjects)
            {
                FUObjectItem* Item = GUObjectArray.GetItem(MutableIdx);
                if (Item && Item->IsReachable())
                {
                    bHasExternalRef = true;
                    break;
                }
            }

            if (bHasExternalRef)
            {
                // 有外部引用, 需要溶解聚类
                Cluster.bNeedsDissolving.store(true, std::memory_order_relaxed);
            }
        }
    }
}
```

### 4.2 聚类溶解

```cpp
// 聚类溶解
// Source: UObjectClusters.cpp

void FUObjectClusterContainer::DissolveCluster(int32 ClusterIndex)
{
    TRACE_CPUPROFILER_EVENT_SCOPE(DissolveCluster);

    FUObjectCluster& Cluster = Clusters[ClusterIndex];

    UE_LOG(LogGarbage, Verbose,
        TEXT("Dissolving cluster %d with %d objects"),
        ClusterIndex, Cluster.Objects.Num());

    // ===== 1. 清除聚类内对象的聚类标志 =====
    for (int32 ObjIdx : Cluster.Objects)
    {
        UObject* Obj = GUObjectArray.IndexToObject(ObjIdx);
        if (Obj)
        {
            Obj->ClearInternalFlags(WithinCluster);
            ObjectToCluster[ObjIdx] = INDEX_NONE;
        }
    }

    // ===== 2. 清除聚类根标志 =====
    UObject* RootObj = GUObjectArray.IndexToObject(Cluster.RootIndex);
    if (RootObj)
    {
        RootObj->ClearInternalFlags(ClusterRoot);
        ObjectToCluster[Cluster.RootIndex] = INDEX_NONE;
    }

    // ===== 3. 断开聚类间引用 =====
    for (int32 RefByClusterIdx : Cluster.ReferencedByClusters)
    {
        if (Clusters.IsValidIndex(RefByClusterIdx))
        {
            FUObjectCluster& RefByCluster = Clusters[RefByClusterIdx];
            RefByCluster.ReferencedClusters.Remove(ClusterIndex);
        }
    }

    for (int32 RefClusterIdx : Cluster.ReferencedClusters)
    {
        if (Clusters.IsValidIndex(RefClusterIdx))
        {
            FUObjectCluster& RefCluster = Clusters[RefClusterIdx];
            RefCluster.ReferencedByClusters.Remove(ClusterIndex);
        }
    }

    // ===== 4. 重置聚类 =====
    Cluster.RootIndex = INDEX_NONE;
    Cluster.Objects.Empty();
    Cluster.MutableObjects.Empty();
    Cluster.ReferencedClusters.Empty();
    Cluster.ReferencedByClusters.Empty();
    Cluster.bNeedsDissolving = false;

    // ===== 5. 回收聚类槽位 =====
    AvailableClusterIndices.Add(ClusterIndex);
}
```

### 4.3 可变对象处理

```cpp
// 可变对象 (Mutable Objects)
// 定义: 有指向聚类外部引用的对象

// 为什么需要追踪可变对象?
// 因为外部引用可能导致聚类内对象独立存活

void FUObjectClusterContainer::UpdateMutableObjects()
{
    for (FUObjectCluster& Cluster : Clusters)
    {
        if (Cluster.RootIndex == INDEX_NONE)
            continue;

        Cluster.MutableObjects.Reset();

        for (int32 ObjIdx : Cluster.Objects)
        {
            UObject* Obj = GUObjectArray.IndexToObject(ObjIdx);
            if (!Obj) continue;

            // 检查对象是否有指向聚类外的引用
            FSchemaView Schema = Obj->GetClass()->GetSchema();

            bool bHasExternalRef = false;
            Schema.ForEachReference(Obj, [&](UObject* Ref)
            {
                if (Ref && !IsInSameCluster(Ref, Obj))
                {
                    bHasExternalRef = true;
                }
            });

            if (bHasExternalRef)
            {
                // 有外部引用, 加入可变对象列表
                Cluster.MutableObjects.Add(ObjIdx);
            }
        }
    }
}

bool FUObjectClusterContainer::IsInSameCluster(UObject* A, UObject* B) const
{
    int32 ClusterA = ObjectToCluster.IsValidIndex(A->GetIndex()) ?
        ObjectToCluster[A->GetIndex()] : INDEX_NONE;
    int32 ClusterB = ObjectToCluster.IsValidIndex(B->GetIndex()) ?
        ObjectToCluster[B->GetIndex()] : INDEX_NONE;

    return ClusterA != INDEX_NONE && ClusterA == ClusterB;
}
```

---

## 五、聚类配置与监控

### 5.1 控制台变量

```ini
; 聚类相关配置
; DefaultEngine.ini 或运行时控制台

; ===== 启用聚类 =====
gc.CreateGCClusters=1              ; 默认启用

; ===== 聚类大小阈值 =====
gc.MinGCClusterSize=2              ; 至少2个对象才创建聚类

; ===== 资产聚类 =====
gc.AssetClusteringEnabled=1        ; 自动聚类资产

; ===== 调试选项 =====
gc.DebugGCClusters=0               ; 调试聚类
gc.DumpGCClusters=0                ; 输出聚类信息
```

### 5.2 聚类统计

```cpp
// 获取聚类统计信息

struct FGCClusterStats
{
    int32 TotalClusters;           // 总聚类数
    int32 TotalClusteredObjects;   // 聚类内对象总数
    int32 MaxClusterSize;          // 最大聚类大小
    int32 AverageClusterSize;      // 平均聚类大小
    int32 DissolvedClusters;       // 已溶解聚类数
    float ClusterEfficiency;       // 聚类效率 (%)
};

FGCClusterStats GetClusterStats()
{
    FGCClusterStats Stats = {0};

    int32 TotalObjects = GUObjectArray.GetObjectArrayNum();

    for (const FUObjectCluster& Cluster : GObjectClusters.Clusters)
    {
        if (Cluster.RootIndex != INDEX_NONE)
        {
            Stats.TotalClusters++;
            Stats.TotalClusteredObjects += Cluster.Objects.Num();
            Stats.MaxClusterSize = FMath::Max(
                Stats.MaxClusterSize,
                Cluster.Objects.Num());
        }
    }

    Stats.AverageClusterSize = Stats.TotalClusters > 0 ?
        Stats.TotalClusteredObjects / Stats.TotalClusters : 0;

    Stats.ClusterEfficiency = TotalObjects > 0 ?
        (float)Stats.TotalClusteredObjects / TotalObjects * 100.0f : 0.0f;

    return Stats;
}

// 打印聚类统计
void PrintClusterStats()
{
    FGCClusterStats Stats = GetClusterStats();

    UE_LOG(LogGC, Log, TEXT("=== GC Cluster Statistics ==="));
    UE_LOG(LogGC, Log, TEXT("  Total Clusters: %d"), Stats.TotalClusters);
    UE_LOG(LogGC, Log, TEXT("  Clustered Objects: %d"), Stats.TotalClusteredObjects);
    UE_LOG(LogGC, Log, TEXT("  Max Cluster Size: %d"), Stats.MaxClusterSize);
    UE_LOG(LogGC, Log, TEXT("  Average Cluster Size: %d"), Stats.AverageClusterSize);
    UE_LOG(LogGC, Log, TEXT("  Cluster Efficiency: %.1f%%"), Stats.ClusterEfficiency);
}
```

---

## 六、聚类最佳实践

### 6.1 适合聚类的场景

```cpp
// ===== 适合聚类的Asset类型 =====

// 1. 材质 (UMaterial)
//    - 包含多个纹理、表达式
//    - 生命周期一致 (一起加载/卸载)
//    - 内部引用关系稳定

// 2. 静态网格 (UStaticMesh)
//    - 包含材质、物理资产
//    - 作为整体使用

// 3. 动画序列 (UAnimSequence)
//    - 包含骨骼、通知
//    - 整体加载

// 4. 蓝图类 (UBlueprintGeneratedClass)
//    - 包含组件、变量、函数
//    - 生命周期与类绑定

// 5. 用户控件 (UUserWidget)
//    - 包含多个子控件
//    - 作为整体销毁

// ===== 为什么适合? =====
// - 这些资产的对象生命周期一致
// - 一起加载, 一起卸载
// - 内部引用关系稳定
// - 不需要独立控制子对象生命周期
```

### 6.2 不适合聚类的场景

```cpp
// ===== 不适合聚类的场景 =====

// 1. 运行时动态创建的对象
UObject* DynamicObj = NewObject<UObject>();
// DynamicObj 可能在任何时候被引用或释放
// 不适合加入聚类

// 2. 被多个资产共享的对象
// UTexture2D SharedTexture
// 被多个Material引用
// 生命周期与单个Material不一致
// 应该独立存在

// 3. 有外部引用的对象
// 某些对象可能被游戏逻辑直接持有
// 聚类会阻止正确GC

// 4. 需要独立控制生命周期的对象
// Actor, Component等
// 需要动态创建/销毁

// 5. 小型资产 (< gc.MinGCClusterSize)
// 聚类开销大于收益
```

### 6.3 聚类优化技巧

```cpp
// ===== 技巧1: 在加载时创建聚类 =====
void OnAssetLoaded(UObject* Asset)
{
    // 检查是否适合聚类
    if (CanCreateCluster(Asset))
    {
        Asset->CreateCluster();
    }
}

// ===== 技巧2: 监控聚类效率 =====
void MonitorClusterEfficiency()
{
    FGCClusterStats Stats = GetClusterStats();

    // 如果平均聚类大小太小, 可能需要调整策略
    if (Stats.AverageClusterSize < 3)
    {
        UE_LOG(LogGC, Warning,
            TEXT("Cluster efficiency low: avg size = %d"),
            Stats.AverageClusterSize);
    }

    // 如果聚类效率太低, 检查是否有很多对象没被聚类
    if (Stats.ClusterEfficiency < 30.0f)
    {
        UE_LOG(LogGC, Warning,
            TEXT("Low cluster efficiency: %.1f%%"),
            Stats.ClusterEfficiency);
    }
}

// ===== 技巧3: 避免过度聚类 =====
// 不要对所有对象都创建聚类
// 只对确实能带来收益的对象创建

// ===== 技巧4: 正确处理聚类溶解 =====
void HandleClusterDissolution()
{
    // 如果聚类需要溶解, 确保子对象能正确处理
    for (FUObjectCluster& Cluster : GObjectClusters.Clusters)
    {
        if (Cluster.bNeedsDissolving)
        {
            // 溶解前可以做些清理工作
            PreDissolveCleanup(Cluster);

            GObjectClusters.DissolveCluster(
                GObjectClusters.ObjectToCluster[
                    GUObjectArray.IndexToObject(Cluster.RootIndex)->GetIndex()]);
        }
    }
}
```

---

## 七、实践练习

### 练习1: 分析Asset聚类

```cpp
// 分析特定Asset的聚类情况

void AnalyzeAssetClustering(UObject* Asset)
{
    if (!Asset)
    {
        UE_LOG(LogTemp, Warning, TEXT("Null asset"));
        return;
    }

    // 检查是否是聚类根
    if (Asset->HasAnyInternalFlags(ClusterRoot))
    {
        int32 ClusterIndex = GObjectClusters.ObjectToCluster[Asset->GetIndex()];
        FUObjectCluster& Cluster = GObjectClusters.Clusters[ClusterIndex];

        UE_LOG(LogTemp, Log, TEXT("=== Cluster Analysis ==="));
        UE_LOG(LogTemp, Log, TEXT("Asset: %s"), *Asset->GetName());
        UE_LOG(LogTemp, Log, TEXT("Status: Cluster Root"));
        UE_LOG(LogTemp, Log, TEXT("Cluster Index: %d"), ClusterIndex);
        UE_LOG(LogTemp, Log, TEXT("Cluster Size: %d objects"), Cluster.Objects.Num());
        UE_LOG(LogTemp, Log, TEXT("Mutable Objects: %d"), Cluster.MutableObjects.Num());
        UE_LOG(LogTemp, Log, TEXT("Referenced Clusters: %d"), Cluster.ReferencedClusters.Num());
        UE_LOG(LogTemp, Log, TEXT("Referenced By: %d"), Cluster.ReferencedByClusters.Num());

        // 打印聚类内对象
        UE_LOG(LogTemp, Log, TEXT("Clustered Objects:"));
        for (int32 ObjIdx : Cluster.Objects)
        {
            UObject* Obj = GUObjectArray.IndexToObject(ObjIdx);
            if (Obj)
            {
                UE_LOG(LogTemp, Log, TEXT("  - %s (%s)"),
                    *Obj->GetName(), *Obj->GetClass()->GetName());
            }
        }
    }
    else if (Asset->HasAnyInternalFlags(WithinCluster))
    {
        UE_LOG(LogTemp, Log, TEXT("Asset is in a cluster but not root"));
    }
    else
    {
        UE_LOG(LogTemp, Log, TEXT("Asset is not clustered"));
    }
}
```

### 练习2: 自定义聚类策略

```cpp
// 为自定义Asset类型实现聚类

UCLASS()
class UMyCustomAsset : public UObject
{
    GENERATED_BODY()

    UPROPERTY()
    TArray<UObject*> SubObjects;

    UPROPERTY()
    TArray<TObjectPtr<UTexture2D>> Textures;

public:
    // 在加载完成后调用
    void OnAssetLoaded()
    {
        // 检查聚类条件
        int32 TotalSubObjects = SubObjects.Num() + Textures.Num();

        if (TotalSubObjects >= gc.MinGCClusterSize)
        {
            UE_LOG(LogTemp, Log,
                TEXT("Creating cluster for %s with %d objects"),
                *GetName(), TotalSubObjects);

            // 创建聚类
            CreateCluster();

            // 获取聚类索引
            int32 ClusterIndex = GObjectClusters.ObjectToCluster[GetIndex()];

            // 添加子对象
            for (UObject* Obj : SubObjects)
            {
                if (Obj && CanAddToCluster(Obj))
                {
                    GObjectClusters.AddObjectToCluster(ClusterIndex, Obj);
                }
            }

            for (TObjectPtr<UTexture2D>& Tex : Textures)
            {
                if (Tex && CanAddToCluster(Tex))
                {
                    GObjectClusters.AddObjectToCluster(ClusterIndex, Tex);
                }
            }

            UE_LOG(LogTemp, Log,
                TEXT("Cluster created with %d objects"),
                GObjectClusters.Clusters[ClusterIndex].Objects.Num());
        }
        else
        {
            UE_LOG(LogTemp, Log,
                TEXT("Not enough objects for clustering: %d < %d"),
                TotalSubObjects, gc.MinGCClusterSize);
        }
    }
};
```

---

## 八、总结

### 本节课要点

1. **设计原理**
   - 聚类将相关对象分组，减少GC遍历开销
   - 从O(N)优化到O(M)，M << N
   - 适合生命周期一致的对象组

2. **数据结构**
   - FUObjectCluster存储聚类信息
   - FUObjectClusterContainer管理所有聚类
   - 标志位标识聚类状态

3. **聚类创建**
   - Package加载时自动创建
   - 递归添加子对象
   - 条件检查确保正确性

4. **GC处理**
   - 聚类根可达 → 整个聚类可达
   - 聚类根不可达 → 检查外部引用
   - 需要时溶解聚类

5. **可变对象**
   - 追踪有外部引用的对象
   - 防止聚类内对象被错误回收
   - 触发聚类溶解

6. **最佳实践**
   - 材质、网格等适合聚类
   - 动态对象不适合聚类
   - 监控聚类效率

### 下一节课预告

下一节课将讲解**增量GC实现**，包括：
- 增量可达性分析的原理
- 写屏障处理并发修改
- 增量清除的实现
- 增量GC的配置与调优

---

## 参考资料

### 源码文件
- `Engine/Source/Runtime/CoreUObject/Private/UObject/UObjectClusters.cpp`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectClusters.h`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollection.cpp`

### 相关课程
- Lesson 3: 标记-清除算法实现
- Lesson 5: 增量GC实现
