# 第5课：垃圾回收与资产释放

本课深入讲解UE5的垃圾回收机制与资产释放策略，帮助开发者有效管理内存。

---

## 5.1 资产与GC的关系

### GC基本原理回顾

```
┌─────────────────────────────────────────────────────────────────┐
│                    UE5 GC 核心机制                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   标记-清除算法                                                  │
│   ─────────────                                                 │
│                                                                 │
│   步骤1：确定根集合（Root Set）                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  • 全局对象（GameInstance, GameMode等）                  │  │
│   │  • 静态变量                                              │  │
│   │  • 栈上的局部变量                                        │  │
│   │  • 显式添加到RootSet的对象                               │  │
│   └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│   步骤2：可达性分析                                              │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  从Root Set开始，遍历所有引用链                          │  │
│   │  标记所有可达对象                                        │  │
│   └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│   步骤3：清理不可达对象                                          │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  未被标记的对象 → 被回收                                 │  │
│   │  调用析构函数，释放内存                                  │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   资产引用与GC关系：                                             │
│   ─────────────────                                             │
│   • 硬引用 → 对象在引用链中 → 可达 → 不被回收                   │
│   • 软引用 → 只有路径字符串 → 不在引用链中 → 可被回收           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 资产生命周期

```
┌─────────────────────────────────────────────────────────────────┐
│                    资产生命周期                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   阶段1：创建/加载                                               │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  NewObject() / LoadObject() / AsyncLoad()              │  │
│   │      │                                                  │  │
│   │      ▼                                                  │  │
│   │  对象分配内存，加入全局对象池                            │  │
│   │  被标记为可达（新对象默认可达）                          │  │
│   └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│   阶段2：使用中                                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  被引用 = 可达 → GC不会回收                             │  │
│   │                                                         │  │
│   │  引用方式：                                              │  │
│   │  • UPROPERTY 硬引用                                     │  │
│   │  • AddReferencedObjects                                │  │
│   │  • Root Set                                            │  │
│   │  • TStrongObjectPtr                                    │  │
│   └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│   阶段3：解除引用                                                │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  置空指针 / 重置软引用 / 离开作用域                      │  │
│   │      │                                                  │  │
│   │      ▼                                                  │  │
│   │  无引用 = 不可达 → 等待GC回收                           │  │
│   └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│   阶段4：GC回收                                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  GC标记阶段 → 发现不可达对象                            │  │
│   │      │                                                  │  │
│   │      ▼                                                  │  │
│   │  GC清理阶段 → 调用析构函数 → 释放内存                   │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 引用管理

### UPROPERTY引用

```cpp
// UPROPERTY是最常用的引用管理方式
UCLASS()
class AItemContainer : public AActor
{
    GENERATED_BODY()

    // UPROPERTY引用会自动阻止GC
    UPROPERTY(EditDefaultsOnly)
    UStaticMesh* DefaultMesh;  // 被GC追踪

    // 数组引用
    UPROPERTY(EditDefaultsOnly)
    TArray<UStaticMesh*> MeshVariants;  // 所有元素被追踪

    // Map引用
    UPROPERTY()
    TMap<FName, UObject*> NamedAssets;  // 所有值被追踪

    // 非UPROPERTY指针（危险！）
    UObject* UntrackedPointer;  // 不被GC追踪，可能指向已销毁对象
};

// 正确的引用管理
void ManageReferences()
{
    UStaticMesh* Mesh = LoadObject<UStaticMesh>(...);

    // 正确：存储到UPROPERTY
    StoredMesh = Mesh;  // 现在被追踪

    // 错误：只存储到局部变量
    // 函数返回后，Mesh可能被GC
}
```

### AddReferencedObjects

```cpp
// 为非UPROPERTY引用添加GC追踪
UCLASS()
class UCustomAssetLoader : public UObject
{
    GENERATED_BODY()

private:
    // 这些是非UPROPERTY的引用
    TArray<UObject*> LoadedAssets;
    TMap<FName, UObject*> AssetCache;

public:
    // 重写此方法告诉GC额外的引用
    static void AddReferencedObjects(
        UObject* InThis,
        FReferenceCollector& Collector
    )
    {
        UCustomAssetLoader* This = CastChecked<UCustomAssetLoader>(InThis);

        // 添加手动引用
        Collector.AddReferencedObjects(This->LoadedAssets);
        Collector.AddReferencedObjects(This->AssetCache);

        // 调用父类
        Super::AddReferencedObjects(InThis, Collector);
    }
};
```

### FGCObject

```cpp
// 对于非UObject类，使用FGCObject管理引用
class FAssetManager : public FGCObject
{
public:
    // 管理的资产
    TArray<UObject*> ManagedAssets;

    // 缓存的资产
    TMap<FName, UObject*> AssetCache;

    // 实现此方法
    virtual void AddReferencedObjects(FReferenceCollector& Collector) override
    {
        Collector.AddReferencedObjects(ManagedAssets);
        Collector.AddReferencedObjects(AssetCache);
    }

    // 实现此方法（用于调试）
    virtual FString GetReferencerName() const override
    {
        return TEXT("FAssetManager");
    }
};
```

### Root Set

```cpp
// 将对象添加到Root Set，永远不会被GC
void AddToRootSet(UObject* Object)
{
    if (Object && !Object->IsRooted())
    {
        Object->AddToRoot();  // 现在永远不会被GC
    }
}

// 从Root Set移除
void RemoveFromRootSet(UObject* Object)
{
    if (Object && Object->IsRooted())
    {
        Object->RemoveFromRoot();  // 现在可以被GC
    }
}

// 使用场景：全局单例、跨关卡的持久对象
UCLASS()
class UGlobalAssetCache : public UObject
{
    GENERATED_BODY()

public:
    static UGlobalAssetCache* Get()
    {
        if (!Instance)
        {
            Instance = NewObject<UGlobalAssetCache>();
            Instance->AddToRoot();  // 永不销毁
        }
        return Instance;
    }

private:
    static UGlobalAssetCache* Instance;
};
```

### TStrongObjectPtr

```cpp
// 使用强引用指针管理生命周期
void UseStrongObjectPtr()
{
    // 创建强引用
    TStrongObjectPtr<UObject> StrongPtr(NewObject<UObject>());

    // 引用计数增加，对象不会被GC
    // 适合跨线程或临时持有

    // 离开作用域，引用计数减少
    // 如果计数为0，对象可被GC
}

// 在异步操作中使用
void AsyncOperationWithStrongPtr()
{
    TStrongObjectPtr<UAssetData> AssetPtr(LoadedAsset);

    AsyncTask(ENamedThreads::AnyThread, [AssetPtr]()
    {
        // 异步操作期间，资产不会被GC
        DoWork(AssetPtr.Get());
    });
}
```

---

## 5.3 资产释放策略

### 手动释放

```cpp
// 资产管理器示例
UCLASS()
class UAssetCache : public UObject
{
    GENERATED_BODY()

public:
    // 缓存的资产
    UPROPERTY()
    TMap<FName, UObject*> CachedAssets;

    // 加载资产
    UObject* LoadAsset(FName AssetName, const FString& Path)
    {
        if (UObject** Found = CachedAssets.Find(AssetName))
        {
            return *Found;  // 返回缓存
        }

        UObject* Asset = LoadObject<UObject>(nullptr, *Path);
        if (Asset)
        {
            CachedAssets.Add(AssetName, Asset);
        }
        return Asset;
    }

    // 释放单个资产
    void ReleaseAsset(FName AssetName)
    {
        CachedAssets.Remove(AssetName);
        // 资产现在可被GC回收
    }

    // 释放所有资产
    void ReleaseAllAssets()
    {
        CachedAssets.Empty();
        // 请求GC
        GEngine->ForceGarbageCollection(false);
    }

    // 释放未使用的资产
    void ReleaseUnusedAssets()
    {
        TArray<FName> ToRemove;

        for (auto& Pair : CachedAssets)
        {
            // 检查引用计数
            FUObjectItem* Item = Pair.Value->GetGUObjectItem();
            if (Item->GetRefCount() == 0)
            {
                ToRemove.Add(Pair.Key);
            }
        }

        for (FName Name : ToRemove)
        {
            CachedAssets.Remove(Name);
        }
    }
};
```

### 引用计数检查

```cpp
// 检查对象的引用情况
void DebugObjectReferences(UObject* Object)
{
    if (!Object) return;

    UE_LOG(LogTemp, Log, TEXT("=== References for %s ==="), *Object->GetName());

    // 获取引用计数
    FUObjectItem* Item = Object->GetGUObjectItem();
    if (Item)
    {
        int32 RefCount = Item->GetRefCount();
        UE_LOG(LogTemp, Log, TEXT("  RefCount: %d"), RefCount);
    }

    // 检查是否在Root Set
    bool bIsRooted = Object->IsRooted();
    UE_LOG(LogTemp, Log, TEXT("  IsRooted: %s"), bIsRooted ? TEXT("Yes") : TEXT("No"));

    // 检查是否待销毁
    bool bIsPendingKill = Object->IsPendingKillPending();
    UE_LOG(LogTemp, Log, TEXT("  IsPendingKill: %s"), bIsPendingKill ? TEXT("Yes") : TEXT("No"));
}

// 使用控制台命令查看引用
// obj refs name=ObjectName
```

### 智能释放策略

```cpp
// 基于LRU的资产缓存
UCLASS()
class USmartAssetCache : public UObject
{
    GENERATED_BODY()

public:
    struct FCachedAsset
    {
        UObject* Asset;
        double LastAccessTime;
        int32 AccessCount;
        int32 Size;  // 预估大小
    };

    UPROPERTY(EditDefaultsOnly)
    int32 MaxCacheSize = 100;  // 最大缓存数量

    UPROPERTY(EditDefaultsOnly)
    int32 MaxMemoryMB = 512;  // 最大内存占用（MB）

    // 获取或加载资产
    UObject* GetOrLoad(FName AssetName, const FString& Path)
    {
        if (FCachedAsset* Cached = Cache.Find(AssetName))
        {
            Cached->LastAccessTime = FPlatformTime::Seconds();
            Cached->AccessCount++;
            return Cached->Asset;
        }

        // 检查是否需要清理
        CheckAndCleanup();

        // 加载新资产
        UObject* Asset = LoadObject<UObject>(nullptr, *Path);
        if (Asset)
        {
            FCachedAsset& NewCached = Cache.Add(AssetName);
            NewCached.Asset = Asset;
            NewCached.LastAccessTime = FPlatformTime::Seconds();
            NewCached.AccessCount = 1;
            NewCached.Size = EstimateAssetSize(Asset);
        }

        return Asset;
    }

private:
    UPROPERTY()
    TMap<FName, FCachedAsset> Cache;

    int32 CurrentMemoryKB = 0;

    void CheckAndCleanup()
    {
        // 检查数量限制
        if (Cache.Num() >= MaxCacheSize)
        {
            CleanupLRU(MaxCacheSize / 2);
        }

        // 检查内存限制
        int32 MaxMemoryKB = MaxMemoryMB * 1024;
        if (CurrentMemoryKB > MaxMemoryKB)
        {
            CleanupLRU(MaxCacheSize / 4);
        }
    }

    void CleanupLRU(int32 NumToRemove)
    {
        // 按访问时间排序
        TArray<FName> SortedKeys;
        Cache.GetKeys(SortedKeys);

        SortedKeys.Sort([this](const FName& A, const FName& B)
        {
            return Cache[A].LastAccessTime < Cache[B].LastAccessTime;
        });

        // 移除最少使用的
        for (int32 i = 0; i < NumToRemove && i < SortedKeys.Num(); i++)
        {
            FName Key = SortedKeys[i];
            CurrentMemoryKB -= Cache[Key].Size;
            Cache.Remove(Key);
        }

        // 请求GC
        GEngine->ForceGarbageCollection(false);
    }

    int32 EstimateAssetSize(UObject* Asset)
    {
        // 简单估算
        return Asset->GetResourceSizeBytes(EResourceSizeMode::Estimated) / 1024;
    }
};
```

---

## 5.4 内存监控

### 内存监控组件

```cpp
// 内存监控组件
UCLASS()
class UMemoryMonitorComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    UMemoryMonitorComponent();

    virtual void TickComponent(float DeltaTime, ... ) override;

    // 内存警告阈值（MB）
    UPROPERTY(EditDefaultsOnly, Category = "Memory")
    float WarningThresholdMB = 1024.0f;

    // 内存危险阈值（MB）
    UPROPERTY(EditDefaultsOnly, Category = "Memory")
    float CriticalThresholdMB = 2048.0f;

    // 检查间隔
    UPROPERTY(EditDefaultsOnly, Category = "Memory")
    float CheckInterval = 1.0f;

    // 是否启用
    UPROPERTY(EditDefaultsOnly, Category = "Memory")
    bool bEnabled = true;

    // 当前内存使用（MB）
    UPROPERTY(BlueprintReadOnly, Category = "Memory")
    float CurrentMemoryMB = 0.0f;

    // 内存使用历史
    UPROPERTY(BlueprintReadOnly, Category = "Memory")
    TArray<float> MemoryHistory;

    // 内存警告事件
    UPROPERTY(BlueprintAssignable, Category = "Memory")
    FOnMemoryWarning OnMemoryWarning;

    // 内存危险事件
    UPROPERTY(BlueprintAssignable, Category = "Memory")
    FOnMemoryCritical OnMemoryCritical;

    // 手动触发检查
    UFUNCTION(BlueprintCallable, Category = "Memory")
    void ForceCheck();

    // 输出内存报告
    UFUNCTION(BlueprintCallable, Category = "Memory")
    void DumpMemoryReport();

private:
    float TimeSinceLastCheck = 0.0f;

    void CheckMemory();
    void HandleMemoryWarning(float MemoryMB);
    void HandleMemoryCritical(float MemoryMB);
    void CollectMemoryStats();
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnMemoryWarning, float, MemoryMB);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnMemoryCritical, float, MemoryMB);

// 实现
UMemoryMonitorComponent::UMemoryMonitorComponent()
{
    PrimaryComponentTick.bCanEverTick = true;
}

void UMemoryMonitorComponent::TickComponent(float DeltaTime, ...)
{
    if (!bEnabled) return;

    TimeSinceLastCheck += DeltaTime;

    if (TimeSinceLastCheck >= CheckInterval)
    {
        CheckMemory();
        TimeSinceLastCheck = 0.0f;
    }
}

void UMemoryMonitorComponent::CheckMemory()
{
    FPlatformMemoryStats MemStats = FPlatformMemory::GetStats();
    CurrentMemoryMB = MemStats.UsedPhysical / 1024.0f / 1024.0f;

    // 记录历史
    MemoryHistory.Add(CurrentMemoryMB);
    if (MemoryHistory.Num() > 60)  // 保留最近60个采样
    {
        MemoryHistory.RemoveAt(0);
    }

    // 检查阈值
    if (CurrentMemoryMB >= CriticalThresholdMB)
    {
        HandleMemoryCritical(CurrentMemoryMB);
    }
    else if (CurrentMemoryMB >= WarningThresholdMB)
    {
        HandleMemoryWarning(CurrentMemoryMB);
    }
}

void UMemoryMonitorComponent::HandleMemoryWarning(float MemoryMB)
{
    UE_LOG(LogTemp, Warning, TEXT("Memory Warning: %.2f MB used"), MemoryMB);
    OnMemoryWarning.Broadcast(MemoryMB);

    // 可以在这里触发清理
    // 例如：释放非必要资产
}

void UMemoryMonitorComponent::HandleMemoryCritical(float MemoryMB)
{
    UE_LOG(LogTemp, Error, TEXT("Memory Critical: %.2f MB used"), MemoryMB);
    OnMemoryCritical.Broadcast(MemoryMB);

    // 紧急清理
    GEngine->ForceGarbageCollection(true);
}

void UMemoryMonitorComponent::DumpMemoryReport()
{
    UE_LOG(LogTemp, Log, TEXT("=== Memory Report ==="));

    // 系统内存
    FPlatformMemoryStats MemStats = FPlatformMemory::GetStats();
    UE_LOG(LogTemp, Log, TEXT("System Memory:"));
    UE_LOG(LogTemp, Log, TEXT("  Total: %.2f MB"), MemStats.TotalPhysical / 1024.0f / 1024.0f);
    UE_LOG(LogTemp, Log, TEXT("  Used: %.2f MB"), MemStats.UsedPhysical / 1024.0f / 1024.0f);
    UE_LOG(LogTemp, Log, TEXT("  Available: %.2f MB"), MemStats.AvailablePhysical / 1024.0f / 1024.0f);

    // UObject统计
    int32 TotalObjects = GUObjectArray.GetObjectCount();
    UE_LOG(LogTemp, Log, TEXT("UObjects: %d"), TotalObjects);

    // 资产统计
    IAssetRegistry& AssetRegistry = UAssetManager::Get().GetAssetRegistry();
    TArray<FAssetData> AllAssets;
    AssetRegistry.GetAllAssets(AllAssets);
    UE_LOG(LogTemp, Log, TEXT("Registered Assets: %d"), AllAssets.Num());

    // 按类统计内存
    TMap<FName, SIZE_T> ClassMemory;
    for (TObjectIterator<UObject> It; It; ++It)
    {
        UObject* Obj = *It;
        FName ClassName = Obj->GetClass()->GetFName();
        SIZE_T Size = Obj->GetResourceSizeBytes(EResourceSizeMode::Estimated);
        ClassMemory.FindOrAdd(ClassName) += Size;
    }

    // 排序并输出前10
    ClassMemory.ValueSort([](SIZE_T A, SIZE_T B) { return A > B; });

    UE_LOG(LogTemp, Log, TEXT("Top Memory Consumers:"));
    int32 Count = 0;
    for (auto& Pair : ClassMemory)
    {
        if (Count++ >= 10) break;
        UE_LOG(LogTemp, Log, TEXT("  %s: %.2f MB"),
            *Pair.Key.ToString(), Pair.Value / 1024.0f / 1024.0f);
    }
}
```

---

## 5.5 资产泄漏检测

### 检测方法

```cpp
// 资产泄漏检测器
UCLASS()
class UAssetLeakDetector : public UObject
{
    GENERATED_BODY()

public:
    struct FAssetInfo
    {
        FString Path;
        FString ClassName;
        double LoadTime;
        int32 RefCount;
        TArray<FString> Referencers;
    };

    // 开始追踪
    UFUNCTION(BlueprintCallable, Category = "Debug")
    void StartTracking()
    {
        TrackedAssets.Empty();
        bIsTracking = true;

        // 记录当前所有资产
        for (TObjectIterator<UObject> It; It; ++It)
        {
            UObject* Obj = *It;
            if (ShouldTrack(Obj))
            {
                RecordAsset(Obj);
            }
        }
    }

    // 结束追踪并检测泄漏
    UFUNCTION(BlueprintCallable, Category = "Debug")
    void StopTrackingAndDetectLeaks()
    {
        if (!bIsTracking) return;

        TArray<FAssetInfo> LeakedAssets;

        // 检查追踪的资产是否仍存在
        for (auto& Pair : TrackedAssets)
        {
            UObject* Obj = Pair.Key;
            if (Obj && !Obj->IsPendingKillPending())
            {
                // 资产仍存在，可能是泄漏
                FAssetInfo& Info = Pair.Value;
                Info.RefCount = Obj->GetGUObjectItem()->GetRefCount();

                if (Info.RefCount > 0)
                {
                    LeakedAssets.Add(Info);
                }
            }
        }

        // 输出泄漏报告
        if (LeakedAssets.Num() > 0)
        {
            UE_LOG(LogTemp, Warning, TEXT("=== Potential Asset Leaks ==="));
            for (const FAssetInfo& Info : LeakedAssets)
            {
                UE_LOG(LogTemp, Warning, TEXT("  %s (%s) - RefCount: %d"),
                    *Info.Path, *Info.ClassName, Info.RefCount);
            }
        }
        else
        {
            UE_LOG(LogTemp, Log, TEXT("No asset leaks detected"));
        }

        bIsTracking = false;
    }

    // 导出泄漏报告
    UFUNCTION(BlueprintCallable, Category = "Debug")
    void ExportLeakReport(const FString& FilePath)
    {
        FString Report;
        Report += TEXT("Asset Leak Report\n");
        Report += TEXT("=================\n\n");

        for (auto& Pair : TrackedAssets)
        {
            UObject* Obj = Pair.Key;
            if (Obj && !Obj->IsPendingKillPending())
            {
                FAssetInfo& Info = Pair.Value;
                Report += FString::Printf(TEXT("Path: %s\n"), *Info.Path);
                Report += FString::Printf(TEXT("Class: %s\n"), *Info.ClassName);
                Report += FString::Printf(TEXT("RefCount: %d\n"), Info.RefCount);
                Report += TEXT("Referencers:\n");
                for (const FString& Ref : Info.Referencers)
                {
                    Report += FString::Printf(TEXT("  - %s\n"), *Ref);
                }
                Report += TEXT("\n");
            }
        }

        FFileHelper::SaveStringToFile(Report, *FilePath);
    }

private:
    TMap<UObject*, FAssetInfo> TrackedAssets;
    bool bIsTracking = false;

    bool ShouldTrack(UObject* Obj)
    {
        // 排除一些不需要追踪的对象
        if (Obj->IsA<UClass>()) return false;
        if (Obj->IsA<UPackage>()) return false;
        if (Obj->GetOutermost() == GetTransientPackage()) return false;

        return true;
    }

    void RecordAsset(UObject* Obj)
    {
        FAssetInfo& Info = TrackedAssets.Add(Obj);
        Info.Path = Obj->GetPathName();
        Info.ClassName = Obj->GetClass()->GetName();
        Info.LoadTime = FPlatformTime::Seconds();

        // 获取引用者
        TArray<UObject*> Referencers;
        FArchiveFindCulprit FindCulprit(Obj, Referencers);
        for (UObject* Ref : Referencers)
        {
            Info.Referencers.Add(Ref->GetPathName());
        }
    }
};
```

---

## 5.6 最佳实践

### 引用管理原则

```cpp
// ✅ 正确做法
UCLASS()
class AGoodExample : public AActor
{
    GENERATED_BODY()

    // 使用UPROPERTY管理引用
    UPROPERTY()
    TArray<UObject*> OwnedAssets;

    // 使用软引用引用可选资产
    UPROPERTY(EditDefaultsOnly)
    TSoftObjectPtr<UObject> OptionalAsset;

    void Cleanup()
    {
        // 清除引用，让GC可以回收
        OwnedAssets.Empty();
    }
};

// ❌ 错误做法
class ABadExample : public AActor
{
    // 非UPROPERTY指针，不被GC追踪
    UObject* UnsafePointer;

    void LoadAsset()
    {
        UnsafePointer = LoadObject<UObject>(...);
        // 函数返回后可能被GC！
    }
};
```

### 内存优化策略

```
┌─────────────────────────────────────────────────────────────────┐
│                    内存优化策略                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   1. 资产加载优化                                               │
│   ─────────────────                                             │
│   • 使用软引用减少初始内存占用                                  │
│   • 大型资产使用异步加载                                        │
│   • 按需加载，避免预加载过多                                    │
│                                                                 │
│   2. 资产释放优化                                               │
│   ─────────────────                                             │
│   • 及时清除不再使用的引用                                      │
│   • 使用智能缓存策略（LRU）                                     │
│   • 关卡切换时主动释放                                          │
│                                                                 │
│   3. 引用管理优化                                               │
│   ─────────────────                                             │
│   • 避免循环引用                                                │
│   • 使用软引用打破引用环                                        │
│   • 合理使用Root Set                                            │
│                                                                 │
│   4. 监控与调试                                                 │
│   ─────────────────                                             │
│   • 实时监控内存使用                                            │
│   • 定期检查资产泄漏                                            │
│   • 使用控制台命令分析                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 常用调试命令

```
内存调试命令：
─────────────────

stat memory          - 显示内存统计
stat memoryplatform  - 平台详细内存信息
stat objects         - 对象统计

obj list             - 列出所有对象
obj list class=Type  - 列出特定类型对象
obj refs name=Obj    - 查看对象引用
obj gc               - 强制GC

memreport            - 生成内存报告
dmp                  - 内存分析

资产调试命令：
─────────────────

assetmanager loaded  - 列出已加载的主资产
assetmanager dump    - 输出资产管理器状态
```

---

## 5.7 实践练习

### 练习1：实现资产缓存管理器

```cpp
// 创建一个带LRU淘汰策略的资产缓存
UCLASS()
class ULRUAssetCache : public UObject
{
    // 实现LRU缓存逻辑
};
```

### 练习2：实现内存监控HUD

```cpp
// 创建一个显示内存信息的UI组件
UCLASS()
class UMemoryMonitorWidget : public UUserWidget
{
    // 显示当前内存、历史图表、警告信息
};
```

---

## 5.8 小结

### 核心概念

| 概念 | 说明 |
|------|------|
| **GC可达性** | 决定对象是否被回收 |
| **UPROPERTY** | 自动管理引用 |
| **Root Set** | 永不回收 |
| **引用计数** | 判断对象是否被使用 |

### 最佳实践

1. **使用UPROPERTY管理引用**
2. **及时释放不再使用的资产**
3. **实现内存监控机制**
4. **定期检查资产泄漏**
5. **使用软引用打破循环引用**

### 下节课预告

第6课将学习流式加载与高级主题，掌握大型世界的资产管理技巧。
