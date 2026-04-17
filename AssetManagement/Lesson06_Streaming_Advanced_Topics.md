# 第6课：流式加载与高级主题

本课讲解UE5中的流式加载机制，包括关卡流式加载、World Partition系统，以及资产热更新等高级主题。

---

## 6.1 关卡流式加载

### 概述

关卡流式加载允许在运行时动态加载和卸载关卡部分，实现大型世界的无缝体验。

```
┌─────────────────────────────────────────────────────────────────┐
│                    关卡流式加载架构                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   持久关卡（Persistent Level）                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                                                         │  │
│   │   始终加载                                               │  │
│   │   • 玩家起始点                                           │  │
│   │   • 全局游戏逻辑                                         │  │
│   │   • 始终需要的资产                                       │  │
│   │                                                         │  │
│   └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│           ┌───────────────┼───────────────┐                    │
│           │               │               │                     │
│           ▼               ▼               ▼                     │
│   ┌───────────────┐ ┌───────────────┐ ┌───────────────┐       │
│   │ 流式关卡A     │ │ 流式关卡B     │ │ 流式关卡C     │       │
│   │ (Streaming)   │ │ (Streaming)   │ │ (Streaming)   │       │
│   │               │ │               │ │               │       │
│   │ 按需加载      │ │ 按需加载      │ │ 按需加载      │       │
│   │ 可卸载       │ │ 可卸载       │ │ 可卸载       │       │
│   └───────────────┘ └───────────────┘ └───────────────┘       │
│                                                                 │
│   加载触发条件：                                                 │
│   • 玩家接近（基于距离）                                        │
│   • 游戏事件触发                                                │
│   • 手动请求                                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### ULevelStreaming

```cpp
// 流式关卡的核心类
UCLASS()
class ULevelStreaming : public UObject
{
    GENERATED_BODY()

public:
    // 关卡资产引用
    UPROPERTY(EditAnywhere, Category = "Level")
    TSoftObjectPtr<UWorld> WorldAsset;

    // 是否应该被加载
    UPROPERTY(EditAnywhere, Category = "Level")
    uint8 bShouldBeLoaded : 1;

    // 是否应该可见
    UPROPERTY(EditAnywhere, Category = "Level")
    uint8 bShouldBeVisible : 1;

    // 是否在编辑器中预加载
    UPROPERTY(EditAnywhere, Category = "Level")
    uint8 bShouldBeLoadedInEditor : 1;

    // 关卡偏移
    UPROPERTY(EditAnywhere, Category = "Level")
    FVector LevelTransform;

    // 关卡颜色（用于调试）
    UPROPERTY(EditAnywhere, Category = "Level")
    FColor LevelColor;

    // ========== 核心方法 ==========

    // 设置是否应该加载
    void SetShouldBeLoaded(bool bInShouldBeLoaded);

    // 设置是否应该可见
    void SetShouldBeVisible(bool bInShouldBeVisible);

    // 获取已加载的关卡
    ULevel* GetLoadedLevel() const;

    // 获取关卡状态
    EStreamingLevelState GetStreamingState() const;

    // ========== 事件 ==========

    // 关卡加载完成事件
    FOnLevelLoaded OnLevelLoaded;

    // 关卡可见事件
    FOnLevelShown OnLevelShown;

    // 关卡卸载事件
    FOnLevelUnloaded OnLevelUnloaded;
};
```

### 关卡流式加载API

```cpp
#include "Kismet/GameplayStatics.h"

// ========== 加载流式关卡 ==========

// 方式1：使用蓝图可调用的API
void LoadStreamingLevel(UObject* WorldContext, FName LevelName, bool bMakeVisible)
{
    UGameplayStatics::LoadStreamLevel(
        WorldContext,
        LevelName,
        bMakeVisible,
        false,  // bShouldBlockOnLoad
        FLatentActionInfo()
    );
}

// 方式2：异步加载带回调
void LoadStreamingLevelAsync(UObject* WorldContext, FName LevelName)
{
    FLatentActionInfo LatentInfo;
    LatentInfo.CallbackTarget = WorldContext;
    LatentInfo.ExecutionFunction = FName("OnLevelLoaded");
    LatentInfo.Linkage = 0;
    LatentInfo.UUID = 0;

    UGameplayStatics::LoadStreamLevel(
        WorldContext,
        LevelName,
        true,   // bMakeVisible
        false,  // bShouldBlockOnLoad
        LatentInfo
    );
}

// 回调函数
UFUNCTION()
void OnLevelLoaded();

// ========== 卸载流式关卡 ==========

void UnloadStreamingLevel(UObject* WorldContext, FName LevelName)
{
    FLatentActionInfo LatentInfo;
    LatentInfo.CallbackTarget = WorldContext;
    LatentInfo.ExecutionFunction = FName("OnLevelUnloaded");

    UGameplayStatics::UnloadStreamLevel(
        WorldContext,
        LevelName,
        LatentInfo,
        false  // bShouldBlockOnUnload
    );
}

// ========== 检查关卡状态 ==========

bool IsLevelLoaded(UObject* WorldContext, FName LevelName)
{
    UWorld* World = GEngine->GetWorldFromContextObject(WorldContext, EGetWorldErrorMode::ReturnNull);
    if (!World) return false;

    for (ULevelStreaming* StreamingLevel : World->GetStreamingLevels())
    {
        if (StreamingLevel->GetWorldAsset().GetAssetName() == LevelName)
        {
            return StreamingLevel->IsLevelLoaded();
        }
    }

    return false;
}

EStreamingLevelState GetLevelState(UObject* WorldContext, FName LevelName)
{
    UWorld* World = GEngine->GetWorldFromContextObject(WorldContext, EGetWorldErrorMode::ReturnNull);
    if (!World) return EStreamingLevelState::Unloaded;

    for (ULevelStreaming* StreamingLevel : World->GetStreamingLevels())
    {
        if (StreamingLevel->GetWorldAsset().GetAssetName() == LevelName)
        {
            return StreamingLevel->GetStreamingState();
        }
    }

    return EStreamingLevelState::Unloaded;
}
```

### 基于距离的流式加载

```cpp
// 基于玩家位置自动加载/卸载关卡
UCLASS()
class AProximityLevelStreamer : public AActor
{
    GENERATED_BODY()

public:
    AProximityLevelStreamer();

    // 流式关卡配置
    USTRUCT(BlueprintType)
    struct FProximityLevelConfig
    {
        GENERATED_BODY()

        // 关卡名称
        UPROPERTY(EditAnywhere, BlueprintReadWrite)
        FName LevelName;

        // 关卡中心位置
        UPROPERTY(EditAnywhere, BlueprintReadWrite)
        FVector LevelCenter;

        // 加载半径
        UPROPERTY(EditAnywhere, BlueprintReadWrite)
        float LoadRadius = 10000.0f;

        // 卸载半径
        UPROPERTY(EditAnywhere, BlueprintReadWrite)
        float UnloadRadius = 15000.0f;

        // 是否已加载
        UPROPERTY(BlueprintReadOnly)
        bool bIsLoaded = false;
    };

    // 配置列表
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Streaming")
    TArray<FProximityLevelConfig> LevelConfigs;

    // 更新间隔
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Streaming")
    float UpdateInterval = 0.5f;

    virtual void BeginPlay() override;
    virtual void Tick(float DeltaTime) override;

    // 手动强制刷新
    UFUNCTION(BlueprintCallable, Category = "Streaming")
    void ForceUpdate();

protected:
    void UpdateStreamingLevels();
    FVector GetPlayerLocation() const;

private:
    float TimeSinceLastUpdate = 0.0f;
    TArray<FName> PendingLoads;
    TArray<FName> PendingUnloads;
};

// 实现
AProximityLevelStreamer::AProximityLevelStreamer()
{
    PrimaryActorTick.bCanEverTick = true;
}

void AProximityLevelStreamer::BeginPlay()
{
    Super::BeginPlay();
    TimeSinceLastUpdate = 0.0f;
}

void AProximityLevelStreamer::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    TimeSinceLastUpdate += DeltaTime;

    if (TimeSinceLastUpdate >= UpdateInterval)
    {
        UpdateStreamingLevels();
        TimeSinceLastUpdate = 0.0f;
    }
}

void AProximityLevelStreamer::ForceUpdate()
{
    UpdateStreamingLevels();
}

void AProximityLevelStreamer::UpdateStreamingLevels()
{
    FVector PlayerLoc = GetPlayerLocation();
    if (PlayerLoc.IsNearlyZero()) return;

    for (FProximityLevelConfig& Config : LevelConfigs)
    {
        float Distance = FVector::Dist2D(PlayerLoc, Config.LevelCenter);

        bool bShouldBeLoaded = Distance <= Config.LoadRadius;
        bool bShouldBeUnloaded = Distance > Config.UnloadRadius;

        // 加载逻辑
        if (bShouldBeLoaded && !Config.bIsLoaded)
        {
            if (!PendingLoads.Contains(Config.LevelName))
            {
                PendingLoads.Add(Config.LevelName);

                UGameplayStatics::LoadStreamLevel(
                    this,
                    Config.LevelName,
                    true,
                    false,
                    FLatentActionInfo()
                );

                Config.bIsLoaded = true;
                UE_LOG(LogTemp, Log, TEXT("Loading level: %s"), *Config.LevelName.ToString());
            }
        }

        // 卸载逻辑
        if (bShouldBeUnloaded && Config.bIsLoaded)
        {
            if (!PendingUnloads.Contains(Config.LevelName))
            {
                PendingUnloads.Add(Config.LevelName);

                UGameplayStatics::UnloadStreamLevel(
                    this,
                    Config.LevelName,
                    FLatentActionInfo(),
                    false
                );

                Config.bIsLoaded = false;
                UE_LOG(LogTemp, Log, TEXT("Unloading level: %s"), *Config.LevelName.ToString());
            }
        }
    }

    // 清理已完成的状态
    PendingLoads.Empty();
    PendingUnloads.Empty();
}

FVector AProximityLevelStreamer::GetPlayerLocation() const
{
    APlayerController* PC = GetWorld()->GetFirstPlayerController();
    if (PC && PC->GetPawn())
    {
        return PC->GetPawn()->GetActorLocation();
    }
    return FVector::ZeroVector;
}
```

---

## 6.2 World Partition 系统

### 概述

World Partition是UE5引入的新一代大世界管理技术，自动将世界划分为可管理的区块。

```
┌─────────────────────────────────────────────────────────────────┐
│                    World Partition 架构                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   传统关卡流式加载 vs World Partition                           │
│                                                                 │
│   传统方式：                                                    │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  手动划分多个子关卡                                      │  │
│   │  手动配置加载边界                                        │  │
│   │  需要预设关卡边界                                        │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   World Partition：                                             │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  单一大关卡                                              │  │
│   │  自动划分网格区块                                        │  │
│   │  基于运行时位置自动加载                                  │  │
│   │  支持数据层（Data Layers）                               │  │
│   │  支持HLOD自动生成                                        │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   区块划分示意：                                                 │
│   ┌────┬────┬────┬────┐                                        │
│   │    │    │    │    │  每个区块可独立加载/卸载               │
│   ├────┼────┼────┼────┤                                        │
│   │    │ ██ │ ██ │    │  ██ = 玩家所在区块（已加载）           │
│   ├────┼────┼────┼────┤                                        │
│   │    │    │    │    │                                        │
│   └────┴────┴────┴────┘                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### World Partition配置

```cpp
// World Partition 在编辑器中配置
// 项目设置 > Engine > World Partition

/*
关键配置项：

1. World Partition Runtime Hash
   - Grid2D：2D网格划分
   - DataLayer：数据层支持

2. Cell Size
   - 区块大小（默认12800单位）
   - 根据内容密度调整

3. Data Layers
   - 可切换的内容层
   - 支持运行时动态加载

4. HLOD
   - 层级细节
   - 自动生成
*/
```

### 运行时API

```cpp
// World Partition 运行时控制
#include "WorldPartition/WorldPartitionSubsystem.h"

UCLASS()
class AWorldPartitionController : public AActor
{
    GENERATED_BODY()

public:
    // 获取World Partition子系统
    UFUNCTION(BlueprintCallable, Category = "World Partition")
    UWorldPartitionSubsystem* GetWorldPartitionSubsystem()
    {
        UWorld* World = GetWorld();
        if (World)
        {
            return World->GetSubsystem<UWorldPartitionSubsystem>();
        }
        return nullptr;
    }

    // 检查区块是否已加载
    UFUNCTION(BlueprintCallable, Category = "World Partition")
    bool IsCellLoaded(const FVector& Location)
    {
        UWorldPartitionSubsystem* Subsystem = GetWorldPartitionSubsystem();
        if (Subsystem)
        {
            // 检查指定位置的区块加载状态
            // 具体API取决于UE版本
        }
        return false;
    }

    // 获取已加载区块数量
    UFUNCTION(BlueprintCallable, Category = "World Partition")
    int32 GetLoadedCellCount()
    {
        UWorldPartitionSubsystem* Subsystem = GetWorldPartitionSubsystem();
        if (Subsystem)
        {
            // 返回已加载区块数量
        }
        return 0;
    }
};
```

---

## 6.3 Data Layers

### 概念

Data Layers允许在同一个关卡中管理不同的内容层，实现条件性加载。

```cpp
// 数据层用于组织和管理关卡内容

UCLASS()
class ADataLayerManager : public AActor
{
    GENERATED_BODY()

public:
    // 获取数据层子系统
    UDataLayerSubsystem* GetDataLayerSubsystem()
    {
        UWorld* World = GetWorld();
        if (World)
        {
            return World->GetSubsystem<UDataLayerSubsystem>();
        }
        return nullptr;
    }

    // 激活数据层
    UFUNCTION(BlueprintCallable, Category = "Data Layers")
    void SetDataLayerActive(FName DataLayerName, bool bActive)
    {
        UDataLayerSubsystem* Subsystem = GetDataLayerSubsystem();
        if (Subsystem)
        {
            if (bActive)
            {
                Subsystem->SetDataLayerState(DataLayerName, EDataLayerState::Loaded);
            }
            else
            {
                Subsystem->SetDataLayerState(DataLayerName, EDataLayerState::Unloaded);
            }
        }
    }

    // 切换数据层状态
    UFUNCTION(BlueprintCallable, Category = "Data Layers")
    void ToggleDataLayer(FName DataLayerName)
    {
        UDataLayerSubsystem* Subsystem = GetDataLayerSubsystem();
        if (Subsystem)
        {
            EDataLayerState CurrentState = Subsystem->GetDataLayerState(DataLayerName);

            if (CurrentState == EDataLayerState::Loaded)
            {
                Subsystem->SetDataLayerState(DataLayerName, EDataLayerState::Unloaded);
            }
            else
            {
                Subsystem->SetDataLayerState(DataLayerName, EDataLayerState::Loaded);
            }
        }
    }

    // 检查数据层状态
    UFUNCTION(BlueprintPure, Category = "Data Layers")
    bool IsDataLayerActive(FName DataLayerName)
    {
        UDataLayerSubsystem* Subsystem = GetDataLayerSubsystem();
        if (Subsystem)
        {
            return Subsystem->GetDataLayerState(DataLayerName) == EDataLayerState::Loaded;
        }
        return false;
    }
};
```

### Data Layers使用场景

```
┌─────────────────────────────────────────────────────────────────┐
│                    Data Layers 使用场景                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   场景1：时间变化                                                │
│   ─────────────                                                 │
│   • DayLayer     - 白天环境                                     │
│   • NightLayer   - 夜晚环境                                     │
│   • 根据游戏时间切换                                            │
│                                                                 │
│   场景2：游戏进度                                                │
│   ─────────────                                                 │
│   • Chapter1Layer - 第一章内容                                  │
│   • Chapter2Layer - 第二章内容                                  │
│   • 根据剧情进度加载                                            │
│                                                                 │
│   场景3：多人模式                                                │
│   ─────────────                                                 │
│   • SinglePlayerLayer - 单人模式专属                           │
│   • MultiplayerLayer  - 多人模式专属                           │
│   • 根据游戏模式切换                                            │
│                                                                 │
│   场景4：破坏状态                                                │
│   ─────────────                                                 │
│   • IntactLayer    - 完好状态                                   │
│   • DestroyedLayer - 破坏状态                                   │
│   • 根据游戏事件切换                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.4 资产热更新

### 打包配置

```ini
; DefaultGame.ini - 资产打包配置

[/Script/UnrealEd.ProjectPackagingSettings]
; 打包目录
+DirectoriesToAlwaysCook=(Path="/Game/Maps")
+DirectoriesToAlwaysCook=(Path="/Game/Blueprints")

; 基于主资产的打包
; 配置Primary Asset后自动处理

; 分包配置（用于DLC）
bSplitBaseGamePak=True

; Chunk分配
+ChunkList=(ChunkId=0, AssetPath="/Game")
+ChunkList=(ChunkId=1, AssetPath="/Game/DLC1")
+ChunkList=(ChunkId=2, AssetPath="/Game/DLC2")

; 压缩配置
bCompressed=True
CompressionFormat=Zlib
```

### 运行时检查资产可用性

```cpp
// 检查资产是否在已安装的包中
UCLASS()
class UAssetAvailabilityChecker : public UObject
{
    GENERATED_BODY()

public:
    // 检查资产是否可用
    UFUNCTION(BlueprintCallable, Category = "Asset")
    static bool IsAssetAvailable(const FString& AssetPath)
    {
        FSoftObjectPath SoftPath(AssetPath);

        // 检查资产注册表
        IAssetRegistry& AssetRegistry = UAssetManager::Get().GetAssetRegistry();

        FAssetData AssetData;
        if (AssetRegistry.GetAssetByObjectPath(SoftPath, AssetData))
        {
            return true;
        }

        return false;
    }

    // 检查Chunk是否已安装
    UFUNCTION(BlueprintCallable, Category = "Asset")
    static bool IsChunkInstalled(int32 ChunkId)
    {
        // 使用平台特定的API检查
        // 具体实现取决于目标平台

        return false;
    }

    // 获取资产所在的Chunk
    UFUNCTION(BlueprintCallable, Category = "Asset")
    static int32 GetAssetChunkId(const FString& AssetPath)
    {
        // 返回资产分配的Chunk ID
        // 需要在打包时正确配置

        return 0;
    }
};
```

### DLC下载管理

```cpp
// DLC下载管理器示例
UCLASS(BlueprintType)
class UDLCManager : public UObject
{
    GENERATED_BODY()

public:
    // DLC状态
    UENUM(BlueprintType)
    enum class EDLCState : uint8
    {
        NotInstalled,
        Downloading,
        Installed,
        Error
    };

    // DLC信息
    USTRUCT(BlueprintType)
    struct FDLCInfo
    {
        GENERATED_BODY()

        UPROPERTY(BlueprintReadOnly)
        FString DLCName;

        UPROPERTY(BlueprintReadOnly)
        FString Version;

        UPROPERTY(BlueprintReadOnly)
        int64 SizeBytes;

        UPROPERTY(BlueprintReadOnly)
        EDLCState State;
    };

    // 获取DLC列表
    UFUNCTION(BlueprintCallable, Category = "DLC")
    void GetAvailableDLCs(TArray<FDLCInfo>& OutDLCs);

    // 开始下载DLC
    UFUNCTION(BlueprintCallable, Category = "DLC")
    void StartDownloadDLC(const FString& DLCName);

    // 获取下载进度
    UFUNCTION(BlueprintPure, Category = "DLC")
    float GetDownloadProgress(const FString& DLCName);

    // 取消下载
    UFUNCTION(BlueprintCallable, Category = "DLC")
    void CancelDownload(const FString& DLCName);

    // 检查DLC是否已安装
    UFUNCTION(BlueprintPure, Category = "DLC")
    bool IsDLCInstalled(const FString& DLCName);

    // 下载完成委托
    UPROPERTY(BlueprintAssignable, Category = "DLC")
    FOnDLCDownloadComplete OnDownloadComplete;

private:
    TMap<FString, FDLCInfo> KnownDLCs;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnDLCDownloadComplete,
    FString, DLCName, bool, bSuccess);
```

---

## 6.5 性能优化

### 加载性能优化

```cpp
// 资产加载优化策略
UCLASS()
class UAssetLoadingOptimizer : public UObject
{
    GENERATED_BODY()

public:
    // 预测性加载
    void PredictiveLoading(const FVector& PlayerLocation, const FVector& Velocity)
    {
        // 根据玩家位置和速度预测将要到达的区域
        FVector PredictedLocation = PlayerLocation + Velocity * PredictionTime;

        // 预加载预测区域的资产
        PreloadAssetsInArea(PredictedLocation);
    }

    // 优先级加载
    void PriorityLoading(const TArray<FSoftObjectPath>& CriticalAssets)
    {
        // 高优先级加载关键资产
        for (const FSoftObjectPath& Path : CriticalAssets)
        {
            UAssetManager::Get().GetStreamableManager().RequestAsyncLoad(
                Path,
                FStreamableDelegate(),
                1000  // 最高优先级
            );
        }
    }

    // 后台预缓存
    void BackgroundCaching()
    {
        // 在空闲时预加载可能需要的资产
        TArray<FSoftObjectPath> CacheAssets = GetCacheCandidates();

        for (const FSoftObjectPath& Path : CacheAssets)
        {
            UAssetManager::Get().GetStreamableManager().RequestAsyncLoad(
                Path,
                FStreamableDelegate(),
                -100  // 最低优先级
            );
        }
    }

private:
    float PredictionTime = 5.0f;  // 预测未来5秒

    void PreloadAssetsInArea(const FVector& Location);
    TArray<FSoftObjectPath> GetCacheCandidates();
};
```

### 流式加载性能监控

```cpp
// 流式加载性能监控
UCLASS()
class UStreamingPerformanceMonitor : public UObject
{
    GENERATED_BODY()

public:
    // 性能指标
    USTRUCT(BlueprintType)
    struct FStreamingMetrics
    {
        GENERATED_BODY()

        UPROPERTY(BlueprintReadOnly)
        float AverageLoadTime = 0.0f;

        UPROPERTY(BlueprintReadOnly)
        int32 LoadedLevels = 0;

        UPROPERTY(BlueprintReadOnly)
        int32 PendingLoads = 0;

        UPROPERTY(BlueprintReadOnly)
        float MemoryUsedMB = 0.0f;
    };

    // 获取性能指标
    UFUNCTION(BlueprintCallable, Category = "Performance")
    FStreamingMetrics GetMetrics()
    {
        FStreamingMetrics Metrics;

        // 收集指标数据
        // ...

        return Metrics;
    }

    // 输出性能报告
    UFUNCTION(BlueprintCallable, Category = "Performance")
    void DumpPerformanceReport()
    {
        UE_LOG(LogTemp, Log, TEXT("=== Streaming Performance Report ==="));

        // 关卡加载统计
        UWorld* World = GetWorld();
        if (World)
        {
            TArray<ULevelStreaming*> StreamingLevels = World->GetStreamingLevels();

            UE_LOG(LogTemp, Log, TEXT("Streaming Levels: %d"), StreamingLevels.Num());

            for (ULevelStreaming* Level : StreamingLevels)
            {
                if (Level)
                {
                    UE_LOG(LogTemp, Log, TEXT("  %s: %s"),
                        *Level->GetWorldAsset().GetAssetName().ToString(),
                        Level->IsLevelLoaded() ? TEXT("Loaded") : TEXT("Unloaded"));
                }
            }
        }
    }
};
```

---

## 6.6 调试工具

### 控制台命令

```
流式加载调试命令：
─────────────────────────────────

stat streaming          - 流式加载统计
stat streamingdetails   - 详细流式加载信息
stat worldpartition     - World Partition统计

streaming pause         - 暂停流式加载
streaming unpaus        - 恢复流式加载
streaming list          - 列出流式关卡

wp.runtime.toggledraw   - 切换World Partition可视化
wp.runtime.drawruntime  - 绘制运行时状态

内存和资产调试命令：
─────────────────────────────────

stat memory             - 内存统计
stat objects            - 对象统计
obj list                - 列出所有对象
obj refs name=X         - 查看对象引用
assetmanager loaded     - 列出已加载资产
```

### 可视化调试

```cpp
// 调试可视化组件
UCLASS()
class UStreamingDebugVisualizer : public UActorComponent
{
    GENERATED_BODY()

public:
    // 是否启用可视化
    UPROPERTY(EditDefaultsOnly, Category = "Debug")
    bool bEnableVisualization = true;

    // 绘制加载边界
    UPROPERTY(EditDefaultsOnly, Category = "Debug")
    bool bDrawLoadBounds = true;

    // 绘制区块状态
    UPROPERTY(EditDefaultsOnly, Category = "Debug")
    bool bDrawCellStatus = true;

    // 绘制当前加载区域
    UFUNCTION(BlueprintCallable, Category = "Debug")
    void DrawStreamingStatus();

private:
    // 调试绘制
    void DrawDebugInfo();
};
```

---

## 6.7 实践练习

### 练习1：实现无缝世界加载系统

```cpp
// 创建一个支持无缝穿越的开放世界加载系统
UCLASS()
class ASeamlessWorldLoader : public AActor
{
    GENERATED_BODY()

    // 实现：
    // 1. 基于玩家位置的关卡加载
    // 2. 预测性加载
    // 3. 渐进式卸载
};
```

### 练习2：实现Data Layers动态切换

```cpp
// 创建一个基于游戏状态的Data Layer管理器
UCLASS()
class AGameStateLayerManager : public AActor
{
    GENERATED_BODY()

    // 实现：
    // 1. 时间变化（日/夜）
    // 2. 季节变化
    // 3. 事件触发的内容变化
};
```

---

## 6.8 小结

### 核心概念

| 概念 | 说明 |
|------|------|
| **Level Streaming** | 传统的关卡流式加载 |
| **World Partition** | UE5的新一代大世界系统 |
| **Data Layers** | 条件性加载的内容层 |
| **Hot Update** | 运行时资产更新 |

### 最佳实践

1. **合理划分关卡/区块大小**
2. **使用预测性加载提升体验**
3. **设置合适的加载/卸载边界**
4. **利用Data Layers管理条件内容**
5. **实现性能监控和调试工具**

---

## 课程总结

### 知识体系回顾

```
┌─────────────────────────────────────────────────────────────────┐
│                    UE5 资源管理知识体系                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   第1课              第2课              第3课                   │
│   资产基础           加载机制           延迟加载                 │
│   ┌─────────┐       ┌─────────┐       ┌─────────┐              │
│   │ 引用类型 │──────►│同步/异步│──────►│ 软引用  │              │
│   │ 生命周期 │       │ 加载策略 │       │ 按需加载 │              │
│   └─────────┘       └─────────┘       └─────────┘              │
│                                                                 │
│   第4课              第5课              第6课                   │
│   资产管理           内存管理           高级应用                 │
│   ┌─────────┐       ┌─────────┐       ┌─────────┐              │
│   │AssetMgr │──────►│ GC释放  │──────►│流式加载 │              │
│   │ Bundle  │       │ 内存监控 │       │热更新   │              │
│   └─────────┘       └─────────┘       └─────────┘              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 核心技能总结

| 技能 | 课程 | 应用场景 |
|------|------|---------|
| 引用管理 | 1 | 避免内存泄漏 |
| 同步/异步加载 | 2 | 性能优化 |
| 延迟加载 | 3 | 减少初始内存 |
| 资产管理器 | 4 | 大型项目管理 |
| 内存管理 | 5 | 内存优化 |
| 流式加载 | 6 | 大世界游戏 |

### 进阶学习建议

1. **深入源码**：阅读AssetManager、GarbageCollection源码
2. **实践项目**：构建一个完整的资产管理系统
3. **性能分析**：使用Profiler分析加载性能
4. **平台适配**：了解不同平台的加载特性
