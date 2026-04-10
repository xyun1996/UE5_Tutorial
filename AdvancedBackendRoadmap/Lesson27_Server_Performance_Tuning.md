# 第27课：服务器性能调优

> 学习目标：掌握服务器CPU性能分析、内存优化策略、Tick优化技巧和异步处理模式。

---

## 27.1 CPU性能分析

### 27.1.1 Unreal Profiler使用

**启动性能分析：**

```bash
# 命令行启动
MyGameServer.exe -trace=cpu,frame,bookmark,loadtime -tracehost=127.0.0.1

# 控制台命令
stat fps
stat unit
stat game
stat net
stat tick

# 详细追踪
trace start cpu,frame
trace save
```

### 27.1.2 自定义性能分析子系统

```csharp
// ServerProfilerSubsystem.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "ServerProfilerSubsystem.generated.h"

USTRUCT(BlueprintType)
struct FPerformanceMetrics
{
    GENERATED_BODY()

    // FPS
    UPROPERTY(BlueprintReadOnly)
    float FPS = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    float FrameTime = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    float GameThreadTime = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    float RenderThreadTime = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    float RHIThreadTime = 0.0f;

    // 内存
    UPROPERTY(BlueprintReadOnly)
    int64 TotalMemoryMB = 0;

    UPROPERTY(BlueprintReadOnly)
    int64 UsedMemoryMB = 0;

    UPROPERTY(BlueprintReadOnly)
    int64 GCMemoryMB = 0;

    // 网络
    UPROPERTY(BlueprintReadOnly)
    int32 ConnectedPlayers = 0;

    UPROPERTY(BlueprintReadOnly)
    float NetworkUpdateTime = 0.0f;

    // Actor统计
    UPROPERTY(BlueprintReadOnly)
    int32 TotalActors = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 TickActors = 0;
};

USTRUCT(BlueprintType)
struct FFunctionProfileData
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString FunctionName;

    UPROPERTY(BlueprintReadOnly)
    int32 CallCount = 0;

    UPROPERTY(BlueprintReadOnly)
    float TotalTimeMs = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    float AverageTimeMs = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    float MaxTimeMs = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    float MinTimeMs = FLT_MAX;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnPerformanceWarning, const FString&, WarningMessage);

UCLASS()
class SERVERPROFILER_API UServerProfilerSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 开始/停止分析
    UFUNCTION(BlueprintCallable, Category = "Profiler")
    void StartProfiling();

    UFUNCTION(BlueprintCallable, Category = "Profiler")
    void StopProfiling();

    // 获取性能指标
    UFUNCTION(BlueprintPure, Category = "Profiler")
    FPerformanceMetrics GetCurrentMetrics() const { return CurrentMetrics; }

    // 函数性能分析
    UFUNCTION(BlueprintCallable, Category = "Profiler")
    void BeginFunctionProfile(const FString& FunctionName);

    UFUNCTION(BlueprintCallable, Category = "Profiler")
    void EndFunctionProfile(const FString& FunctionName);

    // 获取函数分析数据
    UFUNCTION(BlueprintCallable, Category = "Profiler")
    TArray<FFunctionProfileData> GetFunctionProfileData() const;

    // 导出报告
    UFUNCTION(BlueprintCallable, Category = "Profiler")
    FString GeneratePerformanceReport();

    // 设置警告阈值
    UFUNCTION(BlueprintCallable, Category = "Profiler")
    void SetWarningThresholds(float InFrameTimeMs, float InMemoryMB);

    UPROPERTY(BlueprintAssignable, Category = "Profiler")
    FOnPerformanceWarning OnPerformanceWarning;

private:
    bool bIsProfiling = false;
    FPerformanceMetrics CurrentMetrics;
    float FrameTimeWarningThreshold = 33.33f; // 30 FPS
    float MemoryWarningThresholdMB = 4096.0f;

    // 函数分析数据
    TMap<FString, FFunctionProfileData> FunctionProfiles;
    TMap<FString, double> ActiveFunctionProfiles;

    // 采样定时器
    FTimerHandle SamplingTimerHandle;
    float SamplingInterval = 1.0f;

    void CaptureMetrics();
    void CheckWarnings();
    void UpdateMemoryStats();
    void UpdateNetworkStats();
    void UpdateActorStats();
};
```

```csharp
// ServerProfilerSubsystem.cpp
#include "ServerProfilerSubsystem.h"
#include "HAL/PlatformMemory.h"
#include "Engine/World.h"
#include "GameFramework/PlayerController.h"
#include "TimerManager.h"

void UServerProfilerSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
}

void UServerProfilerSubsystem::Deinitialize()
{
    StopProfiling();
    Super::Deinitialize();
}

void UServerProfilerSubsystem::StartProfiling()
{
    if (bIsProfiling)
    {
        return;
    }

    bIsProfiling = true;
    FunctionProfiles.Empty();

    if (UWorld* World = GetWorld())
    {
        World->GetTimerManager().SetTimer(
            SamplingTimerHandle,
            this,
            &UServerProfilerSubsystem::CaptureMetrics,
            SamplingInterval,
            true
        );
    }

    UE_LOG(LogTemp, Log, TEXT("Server profiling started"));
}

void UServerProfilerSubsystem::StopProfiling()
{
    if (!bIsProfiling)
    {
        return;
    }

    bIsProfiling = false;

    if (UWorld* World = GetWorld())
    {
        World->GetTimerManager().ClearTimer(SamplingTimerHandle);
    }

    UE_LOG(LogTemp, Log, TEXT("Server profiling stopped"));
}

void UServerProfilerSubsystem::CaptureMetrics()
{
    // FPS和帧时间
    CurrentMetrics.FPS = 1.0f / FApp::GetDeltaTime();
    CurrentMetrics.FrameTime = FApp::GetDeltaTime() * 1000.0f;
    CurrentMetrics.GameThreadTime = FApp::GetGameThreadTimeMS();
    CurrentMetrics.RenderThreadTime = FApp::GetRenderThreadTimeMS();
    CurrentMetrics.RHIThreadTime = FApp().GetRHIThreadTimeMS();

    // 更新各种统计
    UpdateMemoryStats();
    UpdateNetworkStats();
    UpdateActorStats();

    // 检查警告
    CheckWarnings();
}

void UServerProfilerSubsystem::UpdateMemoryStats()
{
    FPlatformMemoryStats MemStats = FPlatformMemory::GetStats();

    CurrentMetrics.TotalMemoryMB = MemStats.TotalPhysical / (1024 * 1024);
    CurrentMetrics.UsedMemoryMB = MemStats.UsedPhysical / (1024 * 1024);
    CurrentMetrics.GCMemoryMB = GGetCurrentMemorySize(EScriptMemoryType::Game) / (1024 * 1024);
}

void UServerProfilerSubsystem::UpdateNetworkStats()
{
    CurrentMetrics.ConnectedPlayers = 0;
    CurrentMetrics.NetworkUpdateTime = 0.0f;

    if (UWorld* World = GetWorld())
    {
        for (auto It = World->GetPlayerControllerIterator(); It; ++It)
        {
            CurrentMetrics.ConnectedPlayers++;
        }

        UNetDriver* NetDriver = World->GetNetDriver();
        if (NetDriver)
        {
            CurrentMetrics.NetworkUpdateTime = NetDriver->GetNetUpdateTime() * 1000.0f;
        }
    }
}

void UServerProfilerSubsystem::UpdateActorStats()
{
    CurrentMetrics.TotalActors = 0;
    CurrentMetrics.TickActors = 0;

    if (UWorld* World = GetWorld())
    {
        for (TActorIterator<AActor> It(World); It; ++It)
        {
            CurrentMetrics.TotalActors++;

            if ((*It)->IsActorTickEnabled())
            {
                CurrentMetrics.TickActors++;
            }
        }
    }
}

void UServerProfilerSubsystem::BeginFunctionProfile(const FString& FunctionName)
{
    ActiveFunctionProfiles.Add(FunctionName, FPlatformTime::Seconds());
}

void UServerProfilerSubsystem::EndFunctionProfile(const FString& FunctionName)
{
    double* StartTime = ActiveFunctionProfiles.Find(FunctionName);
    if (!StartTime)
    {
        return;
    }

    double EndTime = FPlatformTime::Seconds();
    float DurationMs = (EndTime - *StartTime) * 1000.0f;

    ActiveFunctionProfiles.Remove(FunctionName);

    FFunctionProfileData& Data = FunctionProfiles.FindOrAdd(FunctionName);
    Data.FunctionName = FunctionName;
    Data.CallCount++;
    Data.TotalTimeMs += DurationMs;
    Data.AverageTimeMs = Data.TotalTimeMs / Data.CallCount;
    Data.MaxTimeMs = FMath::Max(Data.MaxTimeMs, DurationMs);
    Data.MinTimeMs = FMath::Min(Data.MinTimeMs, DurationMs);
}

void UServerProfilerSubsystem::CheckWarnings()
{
    // 帧时间警告
    if (CurrentMetrics.FrameTime > FrameTimeWarningThreshold)
    {
        FString Warning = FString::Printf(
            TEXT("Frame time %.2fms exceeds threshold %.2fms"),
            CurrentMetrics.FrameTime, FrameTimeWarningThreshold
        );
        OnPerformanceWarning.Broadcast(Warning);
    }

    // 内存警告
    if (CurrentMetrics.UsedMemoryMB > MemoryWarningThresholdMB)
    {
        FString Warning = FString::Printf(
            TEXT("Memory usage %lldMB exceeds threshold %.0fMB"),
            CurrentMetrics.UsedMemoryMB, MemoryWarningThresholdMB
        );
        OnPerformanceWarning.Broadcast(Warning);
    }
}

TArray<FFunctionProfileData> UServerProfilerSubsystem::GetFunctionProfileData() const
{
    TArray<FFunctionProfileData> Result;
    FunctionProfiles.GenerateValueArray(Result);

    // 按总时间排序
    Result.Sort([](const FFunctionProfileData& A, const FFunctionProfileData& B)
    {
        return A.TotalTimeMs > B.TotalTimeMs;
    });

    return Result;
}

FString UServerProfilerSubsystem::GeneratePerformanceReport()
{
    FString Report;

    Report += TEXT("=== Server Performance Report ===\n\n");

    Report += FString::Printf(TEXT("FPS: %.1f (%.2fms/frame)\n"), CurrentMetrics.FPS, CurrentMetrics.FrameTime);
    Report += FString::Printf(TEXT("Game Thread: %.2fms\n"), CurrentMetrics.GameThreadTime);
    Report += FString::Printf(TEXT("Render Thread: %.2fms\n"), CurrentMetrics.RenderThreadTime);
    Report += FString::Printf(TEXT("RHI Thread: %.2fms\n"), CurrentMetrics.RHIThreadTime);

    Report += TEXT("\n--- Memory ---\n");
    Report += FString::Printf(TEXT("Used: %lldMB / %lldMB\n"), CurrentMetrics.UsedMemoryMB, CurrentMetrics.TotalMemoryMB);
    Report += FString::Printf(TEXT("GC Memory: %lldMB\n"), CurrentMetrics.GCMemoryMB);

    Report += TEXT("\n--- Network ---\n");
    Report += FString::Printf(TEXT("Players: %d\n"), CurrentMetrics.ConnectedPlayers);
    Report += FString::Printf(TEXT("Net Update: %.2fms\n"), CurrentMetrics.NetworkUpdateTime);

    Report += TEXT("\n--- Actors ---\n");
    Report += FString::Printf(TEXT("Total: %d, Ticking: %d\n"), CurrentMetrics.TotalActors, CurrentMetrics.TickActors);

    Report += TEXT("\n--- Top Functions by Time ---\n");
    TArray<FFunctionProfileData> FuncData = GetFunctionProfileData();
    for (int32 i = 0; i < FMath::Min(10, FuncData.Num()); i++)
    {
        const FFunctionProfileData& Data = FuncData[i];
        Report += FString::Printf(TEXT("%s: %.2fms total, %.4fms avg, %d calls\n"),
            *Data.FunctionName, Data.TotalTimeMs, Data.AverageTimeMs, Data.CallCount);
    }

    return Report;
}
```

---

## 27.2 内存优化策略

### 27.2.1 内存池管理

```csharp
// MemoryPoolManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "MemoryPoolManager.generated.h"

USTRUCT(BlueprintType)
struct FMemoryPoolConfig
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 InitialSize = 1024;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 GrowSize = 512;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 MaxSize = 4096;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString PoolName;
};

USTRUCT(BlueprintType)
struct FPoolStats
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    int32 ActiveCount = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 PooledCount = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 PeakCount = 0;

    UPROPERTY(BlueprintReadOnly)
    int64 TotalMemoryBytes = 0;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnPoolGrown, const FString&, PoolName, int32, NewSize);

UCLASS()
class MEMORYPOOLS_API UMemoryPoolManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    // 初始化对象池
    template<typename T>
    TSharedRef<TObjectPool<T>> CreatePool(const FString& PoolName, const FMemoryPoolConfig& Config);

    // 从池中获取对象
    template<typename T>
    TSharedPtr<T> Acquire(const FString& PoolName);

    // 归还对象到池
    template<typename T>
    void Release(const FString& PoolName, TSharedPtr<T> Object);

    // 获取池统计
    UFUNCTION(BlueprintPure, Category = "MemoryPool")
    FPoolStats GetPoolStats(const FString& PoolName) const;

    // 清空所有池
    UFUNCTION(BlueprintCallable, Category = "MemoryPool")
    void ClearAllPools();

    // 预热池
    UFUNCTION(BlueprintCallable, Category = "MemoryPool")
    void WarmUpPool(const FString& PoolName, int32 Count);

    UPROPERTY(BlueprintAssignable, Category = "MemoryPool")
    FOnPoolGrown OnPoolGrown;

private:
    TMap<FString, TSharedPtr<void>> Pools;
    TMap<FString, FPoolStats> PoolStats;
};

// 对象池模板类
template<typename T>
class TObjectPool
{
public:
    TObjectPool(const FMemoryPoolConfig& InConfig)
        : Config(InConfig)
    {
        Preallocate(Config.InitialSize);
    }

    TSharedPtr<T> Acquire()
    {
        if (PooledObjects.Num() == 0)
        {
            if (TotalCount >= Config.MaxSize)
            {
                UE_LOG(LogTemp, Warning, TEXT("Pool %s at max capacity"), *Config.PoolName);
                return nullptr;
            }

            GrowPool();
        }

        if (PooledObjects.Num() > 0)
        {
            TSharedPtr<T> Object = PooledObjects.Pop();
            ActiveObjects.Add(Object);
            Stats.ActiveCount = ActiveObjects.Num();
            Stats.PooledCount = PooledObjects.Num();
            Stats.PeakCount = FMath::Max(Stats.PeakCount, Stats.ActiveCount);
            return Object;
        }

        return nullptr;
    }

    void Release(TSharedPtr<T> Object)
    {
        if (ActiveObjects.Remove(Object))
        {
            // 重置对象状态
            ResetObject(*Object);
            PooledObjects.Add(Object);

            Stats.ActiveCount = ActiveObjects.Num();
            Stats.PooledCount = PooledObjects.Num();
        }
    }

    void Preallocate(int32 Count)
    {
        for (int32 i = 0; i < Count; i++)
        {
            TSharedPtr<T> NewObject = MakeShareable(new T());
            ResetObject(*NewObject);
            PooledObjects.Add(NewObject);
        }
        TotalCount += Count;
        Stats.PooledCount = PooledObjects.Num();
        Stats.TotalMemoryBytes = TotalCount * sizeof(T);
    }

    void Clear()
    {
        PooledObjects.Empty();
        ActiveObjects.Empty();
        TotalCount = 0;
        Stats = FPoolStats();
    }

    FPoolStats GetStats() const { return Stats; }

private:
    FMemoryPoolConfig Config;
    TArray<TSharedPtr<T>> PooledObjects;
    TArray<TSharedPtr<T>> ActiveObjects;
    int32 TotalCount = 0;
    FPoolStats Stats;

    void GrowPool()
    {
        int32 GrowAmount = FMath::Min(Config.GrowSize, Config.MaxSize - TotalCount);
        Preallocate(GrowAmount);
        UE_LOG(LogTemp, Log, TEXT("Pool %s grown by %d to %d"),
            *Config.PoolName, GrowAmount, TotalCount);
    }

    void ResetObject(T& Object)
    {
        // 重置对象状态，根据类型特化
        // 可以使用类型traits或重载
    }
};
```

### 27.2.2 垃圾回收优化

```csharp
// GCOptimizationManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "GCOptimizationManager.generated.h"

USTRUCT(BlueprintType)
struct FGCSettings
{
    GENERATED_BODY()

    // GC间隔（秒）
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float GCInterval = 60.0f;

    // 是否启用增量GC
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bIncrementalGC = true;

    // 增量GC时间片（毫秒）
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float IncrementalGCTimeSlice = 2.0f;

    // 是否在关卡切换时全量GC
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bFullGCOnLevelChange = true;

    // 内存阈值触发GC（MB）
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float MemoryThresholdMB = 2048.0f;
};

UCLASS()
class GCOPTIMIZATION_API UGCOptimizationManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 应用GC设置
    UFUNCTION(BlueprintCallable, Category = "GC")
    void ApplySettings(const FGCSettings& Settings);

    // 手动触发GC
    UFUNCTION(BlueprintCallable, Category = "GC")
    void TriggerGC(bool bForceFull = false);

    // 暂停/恢复GC
    UFUNCTION(BlueprintCallable, Category = "GC")
    void PauseGC();

    UFUNCTION(BlueprintCallable, Category = "GC")
    void ResumeGC();

    // 获取GC统计
    UFUNCTION(BlueprintPure, Category = "GC")
    int64 GetLastGCDuration() const { return LastGCDuration; }

    UFUNCTION(BlueprintPure, Category = "GC")
    int32 GetGCCount() const { return GCCount; }

    UFUNCTION(BlueprintPure, Category = "GC")
    float GetTimeSinceLastGC() const;

private:
    FGCSettings CurrentSettings;
    FTimerHandle GCTimerHandle;

    bool bGCPaused = false;
    int64 LastGCDuration = 0;
    double LastGCTime = 0.0;
    int32 GCCount = 0;

    void ScheduledGC();
    void ConfigureIncrementalGC();
    void OnPreGarbageCollection();
    void OnPostGarbageCollection();
};
```

```csharp
// GCOptimizationManager.cpp
#include "GCOptimizationManager.h"
#include "TimerManager.h"
#include "HAL/PlatformMemory.h"

void UGCOptimizationManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    // 绑定GC回调
    FCoreDelegates::OnPreGarbageCollection.AddUObject(this, &UGCOptimizationManager::OnPreGarbageCollection);
    FCoreDelegates::OnPostGarbageCollection.AddUObject(this, &UGCOptimizationManager::OnPostGarbageCollection);
}

void UGCOptimizationManager::Deinitialize()
{
    if (UWorld* World = GetWorld())
    {
        World->GetTimerManager().ClearTimer(GCTimerHandle);
    }

    FCoreDelegates::OnPreGarbageCollection.RemoveAll(this);
    FCoreDelegates::OnPostGarbageCollection.RemoveAll(this);

    Super::Deinitialize();
}

void UGCOptimizationManager::ApplySettings(const FGCSettings& Settings)
{
    CurrentSettings = Settings;

    // 配置增量GC
    if (Settings.bIncrementalGC)
    {
        ConfigureIncrementalGC();
    }

    // 设置定时GC
    if (UWorld* World = GetWorld())
    {
        World->GetTimerManager().SetTimer(
            GCTimerHandle,
            this,
            &UGCOptimizationManager::ScheduledGC,
            Settings.GCInterval,
            true
        );
    }
}

void UGCOptimizationManager::ConfigureIncrementalGC()
{
    // 配置增量GC参数
    GConfig->SetFloat(TEXT("/Script/Engine.GarbageCollectionSettings"),
        TEXT("gc.TimeBetweenPurgingPendingKillObjects"),
        CurrentSettings.IncrementalGCTimeSlice,
        GEngineIni);

    GConfig->SetBool(TEXT("/Script/Engine.GarbageCollectionSettings"),
        TEXT("gc.Incremental"),
        true,
        GEngineIni);
}

void UGCOptimizationManager::ScheduledGC()
{
    if (bGCPaused)
    {
        return;
    }

    // 检查内存阈值
    FPlatformMemoryStats MemStats = FPlatformMemory::GetStats();
    float UsedMemoryMB = MemStats.UsedPhysical / (1024.0f * 1024.0f);

    if (UsedMemoryMB > CurrentSettings.MemoryThresholdMB)
    {
        TriggerGC(true);
    }
    else if (CurrentSettings.bIncrementalGC)
    {
        // 增量GC
        TriggerGC(false);
    }
}

void UGCOptimizationManager::TriggerGC(bool bForceFull)
{
    if (bGCPaused)
    {
        return;
    }

    double StartTime = FPlatformTime::Seconds();

    if (bForceFull || !CurrentSettings.bIncrementalGC)
    {
        // 全量GC
        GEngine->ForceGarbageCollection(true);
    }
    else
    {
        // 增量GC
        GEngine->ForceGarbageCollection(false);
    }

    LastGCDuration = static_cast<int64>((FPlatformTime::Seconds() - StartTime) * 1000000);
    LastGCTime = FPlatformTime::Seconds();
    GCCount++;
}

void UGCOptimizationManager::PauseGC()
{
    bGCPaused = true;

    // 在关键游戏时刻暂停GC
    UE_LOG(LogTemp, Log, TEXT("GC paused"));
}

void UGCOptimizationManager::ResumeGC()
{
    bGCPaused = false;
    UE_LOG(LogTemp, Log, TEXT("GC resumed"));
}

float UGCOptimizationManager::GetTimeSinceLastGC() const
{
    if (LastGCTime == 0.0)
    {
        return -1.0f;
    }

    return FPlatformTime::Seconds() - LastGCTime;
}

void UGCOptimizationManager::OnPreGarbageCollection()
{
    // GC开始前的处理
    TRACE_CPUPROFILER_EVENT_SCOPE(GarbageCollection);
}

void UGCOptimizationManager::OnPostGarbageCollection()
{
    // GC完成后的处理
    UE_LOG(LogTemp, Verbose, TEXT("GC completed, duration: %lld us"), LastGCDuration);
}
```

---

## 27.3 Tick优化技巧

### 27.3.1 智能Tick管理

```csharp
// SmartTickManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/WorldSubsystem.h"

#include "SmartTickManager.generated.h"

UENUM(BlueprintType)
enum class ETickPriority : uint8
{
    Critical,   // 每帧必须执行
    High,       // 高优先级
    Normal,     // 正常优先级
    Low,        // 低优先级
    Background  // 后台任务
};

USTRUCT(BlueprintType)
struct FTickConfig
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    ETickPriority Priority = ETickPriority::Normal;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float TickInterval = 0.0f;  // 0 = 每帧

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bTickWhenPaused = false;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float CullDistance = 0.0f;  // 超出距离停止Tick

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bTickOnlyWhenRelevant = false;
};

UCLASS()
class SMARTTICK_API USmartTickManager : public UWorldSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    virtual void Tick(float DeltaTime) override;

    // 注册Tick组
    void RegisterTickGroup(const FString& GroupName, const FTickConfig& Config);

    // 添加Actor到Tick组
    void AddActorToGroup(AActor* Actor, const FString& GroupName);

    // 移除Actor
    void RemoveActorFromGroup(AActor* Actor);

    // 更新Actor的Tick状态
    void UpdateActorTickState(AActor* Actor, bool bShouldTick);

    // 获取组的统计
    UFUNCTION(BlueprintPure, Category = "SmartTick")
    int32 GetGroupActorCount(const FString& GroupName) const;

    // 配置
    UPROPERTY(EditAnywhere, Category = "SmartTick")
    float HighPriorityTimeBudget = 5.0f;  // ms

    UPROPERTY(EditAnywhere, Category = "SmartTick")
    float NormalPriorityTimeBudget = 10.0f;  // ms

    UPROPERTY(EditAnywhere, Category = "SmartTick")
    float LowPriorityTimeBudget = 5.0f;  // ms

private:
    struct FTickGroup
    {
        FString Name;
        FTickConfig Config;
        TArray<TWeakObjectPtr<AActor>> Actors;
        float AccumulatedTime = 0.0f;
        int32 LastTickedIndex = 0;
    };

    TMap<FString, FTickGroup> TickGroups;
    TMap<AActor*, FString> ActorToGroup;

    // 按优先级排序的组名
    TArray<FString> CriticalGroups;
    TArray<FString> HighPriorityGroups;
    TArray<FString> NormalPriorityGroups;
    TArray<FString> LowPriorityGroups;
    TArray<FString> BackgroundGroups;

    void ProcessTickGroup(FTickGroup& Group, float DeltaTime, float TimeBudget);
    bool ShouldTickActor(AActor* Actor, const FTickConfig& Config);
    void SortGroupsByPriority();
};
```

```csharp
// SmartTickManager.cpp
#include "SmartTickManager.h"
#include "GameFramework/PlayerController.h"
#include "Kismet/GameplayStatics.h"

void USmartTickManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    SortGroupsByPriority();
}

void USmartTickManager::Deinitialize()
{
    TickGroups.Empty();
    ActorToGroup.Empty();

    Super::Deinitialize();
}

void USmartTickManager::Tick(float DeltaTime)
{
    TRACE_CPUPROFILER_EVENT_SCOPE(SmartTickManager);

    double StartTime = FPlatformTime::Seconds();

    // 1. 处理关键组（无时间限制）
    for (const FString& GroupName : CriticalGroups)
    {
        if (FTickGroup* Group = TickGroups.Find(GroupName))
        {
            ProcessTickGroup(*Group, DeltaTime, -1.0f);
        }
    }

    // 2. 高优先级组
    float HighBudget = HighPriorityTimeBudget / 1000.0f;
    for (const FString& GroupName : HighPriorityGroups)
    {
        if (FTickGroup* Group = TickGroups.Find(GroupName))
        {
            ProcessTickGroup(*Group, DeltaTime, HighBudget);
        }
    }

    // 3. 正常优先级组
    float NormalBudget = NormalPriorityTimeBudget / 1000.0f;
    for (const FString& GroupName : NormalPriorityGroups)
    {
        if (FTickGroup* Group = TickGroups.Find(GroupName))
        {
            ProcessTickGroup(*Group, DeltaTime, NormalBudget);
        }
    }

    // 4. 低优先级组
    float LowBudget = LowPriorityTimeBudget / 1000.0f;
    for (const FString& GroupName : LowPriorityGroups)
    {
        if (FTickGroup* Group = TickGroups.Find(GroupName))
        {
            ProcessTickGroup(*Group, DeltaTime, LowBudget);
        }
    }

    // 5. 后台组（仅在有空闲时间时处理）
    double Elapsed = FPlatformTime::Seconds() - StartTime;
    float RemainingBudget = (33.33f / 1000.0f) - Elapsed;

    if (RemainingBudget > 0.001f)
    {
        for (const FString& GroupName : BackgroundGroups)
        {
            if (FTickGroup* Group = TickGroups.Find(GroupName))
            {
                ProcessTickGroup(*Group, DeltaTime, RemainingBudget);
            }
        }
    }
}

void USmartTickManager::RegisterTickGroup(const FString& GroupName, const FTickConfig& Config)
{
    FTickGroup NewGroup;
    NewGroup.Name = GroupName;
    NewGroup.Config = Config;

    TickGroups.Add(GroupName, NewGroup);
    SortGroupsByPriority();
}

void USmartTickManager::AddActorToGroup(AActor* Actor, const FString& GroupName)
{
    if (!Actor || !TickGroups.Contains(GroupName))
    {
        return;
    }

    // 从旧组移除
    RemoveActorFromGroup(Actor);

    // 添加到新组
    TickGroups[GroupName].Actors.Add(Actor);
    ActorToGroup.Add(Actor, GroupName);

    // 禁用Actor的原生Tick
    Actor->SetActorTickEnabled(false);
}

void USmartTickManager::RemoveActorFromGroup(AActor* Actor)
{
    if (!Actor)
    {
        return;
    }

    FString* GroupName = ActorToGroup.Find(Actor);
    if (GroupName)
    {
        if (FTickGroup* Group = TickGroups.Find(*GroupName))
        {
            Group->Actors.RemoveAll([Actor](const TWeakObjectPtr<AActor>& Ptr)
            {
                return !Ptr.IsValid() || Ptr.Get() == Actor;
            });
        }
        ActorToGroup.Remove(Actor);
    }
}

void USmartTickManager::ProcessTickGroup(FTickGroup& Group, float DeltaTime, float TimeBudget)
{
    double StartTime = FPlatformTime::Seconds();
    int32 TickedCount = 0;

    const FTickConfig& Config = Group.Config;

    // 检查Tick间隔
    Group.AccumulatedTime += DeltaTime;
    if (Config.TickInterval > 0.0f && Group.AccumulatedTime < Config.TickInterval)
    {
        return;
    }
    Group.AccumulatedTime = 0.0f;

    // 从上次位置继续Tick（公平调度）
    int32 ActorCount = Group.Actors.Num();
    int32 StartIndex = Group.LastTickedIndex;

    for (int32 i = 0; i < ActorCount; i++)
    {
        int32 Index = (StartIndex + i) % ActorCount;
        TWeakObjectPtr<AActor>& ActorPtr = Group.Actors[Index];

        if (!ActorPtr.IsValid())
        {
            continue;
        }

        AActor* Actor = ActorPtr.Get();

        // 检查是否应该Tick
        if (!ShouldTickActor(Actor, Config))
        {
            continue;
        }

        // 执行Tick
        Actor->Tick(DeltaTime);

        TickedCount++;
        Group.LastTickedIndex = (Index + 1) % ActorCount;

        // 检查时间预算
        if (TimeBudget > 0.0f)
        {
            double Elapsed = FPlatformTime::Seconds() - StartTime;
            if (Elapsed > TimeBudget)
            {
                break;
            }
        }
    }
}

bool USmartTickManager::ShouldTickActor(AActor* Actor, const FTickConfig& Config)
{
    if (!Actor)
    {
        return false;
    }

    // 检查暂停
    if (!Config.bTickWhenPaused && GetWorld()->IsPaused())
    {
        return false;
    }

    // 检查距离剔除
    if (Config.CullDistance > 0.0f)
    {
        APlayerController* PC = UGameplayStatics::GetPlayerController(GetWorld(), 0);
        if (PC && PC->GetPawn())
        {
            float Distance = FVector::Dist(Actor->GetActorLocation(), PC->GetPawn()->GetActorLocation());
            if (Distance > Config.CullDistance)
            {
                return false;
            }
        }
    }

    // 检查相关性
    if (Config.bTickOnlyWhenRelevant)
    {
        // 使用UE的NetRelevancy判断
        // 这里简化实现
        if (!Actor->IsNetRelevantFor(nullptr, nullptr, FVector::ZeroVector))
        {
            return false;
        }
    }

    return true;
}

void USmartTickManager::SortGroupsByPriority()
{
    CriticalGroups.Empty();
    HighPriorityGroups.Empty();
    NormalPriorityGroups.Empty();
    LowPriorityGroups.Empty();
    BackgroundGroups.Empty();

    for (const auto& Pair : TickGroups)
    {
        switch (Pair.Value.Config.Priority)
        {
            case ETickPriority::Critical:
                CriticalGroups.Add(Pair.Key);
                break;
            case ETickPriority::High:
                HighPriorityGroups.Add(Pair.Key);
                break;
            case ETickPriority::Normal:
                NormalPriorityGroups.Add(Pair.Key);
                break;
            case ETickPriority::Low:
                LowPriorityGroups.Add(Pair.Key);
                break;
            case ETickPriority::Background:
                BackgroundGroups.Add(Pair.Key);
                break;
        }
    }
}
```

---

## 27.4 异步处理模式

### 27.4.1 任务调度系统

```csharp
// AsyncTaskScheduler.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "AsyncTaskScheduler.generated.h"

UENUM(BlueprintType)
enum class ETaskPriority : uint8
{
    Highest,
    High,
    Normal,
    Low,
    Background
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnTaskComplete, FString, TaskId, bool, bSuccess);

UCLASS()
class ASYNCTASKS_API UAsyncTaskScheduler : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 调度异步任务
    FString ScheduleTask(TFunction<void()> Task, ETaskPriority Priority = ETaskPriority::Normal);

    // 调度延迟任务
    FString ScheduleDelayedTask(TFunction<void()> Task, float DelaySeconds, ETaskPriority Priority = ETaskPriority::Normal);

    // 调度重复任务
    FString ScheduleRepeatingTask(TFunction<void()> Task, float IntervalSeconds, ETaskPriority Priority = ETaskPriority::Normal);

    // 取消任务
    UFUNCTION(BlueprintCallable, Category = "AsyncTask")
    bool CancelTask(const FString& TaskId);

    // 等待任务完成
    bool WaitForTask(const FString& TaskId, float TimeoutSeconds = 10.0f);

    // 获取任务状态
    UFUNCTION(BlueprintPure, Category = "AsyncTask")
    bool IsTaskComplete(const FString& TaskId) const;

    // 统计
    UFUNCTION(BlueprintPure, Category = "AsyncTask")
    int32 GetActiveTaskCount() const;

    UPROPERTY(BlueprintAssignable, Category = "AsyncTask")
    FOnTaskComplete OnTaskComplete;

private:
    struct FScheduledTask
    {
        FString TaskId;
        TFunction<void()> Task;
        ETaskPriority Priority;
        float DelaySeconds = 0.0f;
        float IntervalSeconds = 0.0f;
        bool bIsRepeating = false;
        bool bIsComplete = false;
        bool bIsCancelled = false;
        double ScheduleTime = 0.0;
    };

    TMap<FString, FScheduledTask> ActiveTasks;
    FCriticalSection TaskMutex;
    FTimerHandle ProcessTimerHandle;

    FString GenerateTaskId();
    void ProcessScheduledTasks();
    void ExecuteTask(FScheduledTask& Task);
    ENamedThreads::Type GetThreadForPriority(ETaskPriority Priority) const;
};
```

```csharp
// AsyncTaskScheduler.cpp
#include "AsyncTaskScheduler.h"
#include "Async/Async.h"
#include "TimerManager.h"

void UAsyncTaskScheduler::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    // 启动定时处理
    if (UWorld* World = GetWorld())
    {
        World->GetTimerManager().SetTimer(
            ProcessTimerHandle,
            this,
            &UAsyncTaskScheduler::ProcessScheduledTasks,
            0.1f,  // 每100ms检查一次
            true
        );
    }
}

void UAsyncTaskScheduler::Deinitialize()
{
    if (UWorld* World = GetWorld())
    {
        World->GetTimerManager().ClearTimer(ProcessTimerHandle);
    }

    ActiveTasks.Empty();
    Super::Deinitialize();
}

FString UAsyncTaskScheduler::ScheduleTask(TFunction<void()> Task, ETaskPriority Priority)
{
    return ScheduleDelayedTask(Task, 0.0f, Priority);
}

FString UAsyncTaskScheduler::ScheduleDelayedTask(TFunction<void()> Task, float DelaySeconds, ETaskPriority Priority)
{
    FScopeLock Lock(&TaskMutex);

    FString TaskId = GenerateTaskId();

    FScheduledTask NewTask;
    NewTask.TaskId = TaskId;
    NewTask.Task = Task;
    NewTask.Priority = Priority;
    NewTask.DelaySeconds = DelaySeconds;
    NewTask.ScheduleTime = FPlatformTime::Seconds();
    NewTask.bIsComplete = false;
    NewTask.bIsCancelled = false;

    ActiveTasks.Add(TaskId, NewTask);

    UE_LOG(LogTemp, Verbose, TEXT("Task scheduled: %s, delay: %.2f"), *TaskId, DelaySeconds);
    return TaskId;
}

FString UAsyncTaskScheduler::ScheduleRepeatingTask(TFunction<void()> Task, float IntervalSeconds, ETaskPriority Priority)
{
    FScopeLock Lock(&TaskMutex);

    FString TaskId = GenerateTaskId();

    FScheduledTask NewTask;
    NewTask.TaskId = TaskId;
    NewTask.Task = Task;
    NewTask.Priority = Priority;
    NewTask.IntervalSeconds = IntervalSeconds;
    NewTask.bIsRepeating = true;
    NewTask.ScheduleTime = FPlatformTime::Seconds();

    ActiveTasks.Add(TaskId, NewTask);

    return TaskId;
}

bool UAsyncTaskScheduler::CancelTask(const FString& TaskId)
{
    FScopeLock Lock(&TaskMutex);

    if (FScheduledTask* Task = ActiveTasks.Find(TaskId))
    {
        Task->bIsCancelled = true;
        UE_LOG(LogTemp, Verbose, TEXT("Task cancelled: %s"), *TaskId);
        return true;
    }

    return false;
}

bool UAsyncTaskScheduler::WaitForTask(const FString& TaskId, float TimeoutSeconds)
{
    double StartTime = FPlatformTime::Seconds();

    while (true)
    {
        if (IsTaskComplete(TaskId))
        {
            return true;
        }

        if (FPlatformTime::Seconds() - StartTime > TimeoutSeconds)
        {
            return false;
        }

        FPlatformProcess::Sleep(0.001f);
    }
}

bool UAsyncTaskScheduler::IsTaskComplete(const FString& TaskId) const
{
    FScopeLock Lock(&TaskMutex);

    if (const FScheduledTask* Task = ActiveTasks.Find(TaskId))
    {
        return Task->bIsComplete || Task->bIsCancelled;
    }

    return true;
}

int32 UAsyncTaskScheduler::GetActiveTaskCount() const
{
    FScopeLock Lock(&TaskMutex);

    int32 Count = 0;
    for (const auto& Pair : ActiveTasks)
    {
        if (!Pair.Value.bIsComplete && !Pair.Value.bIsCancelled)
        {
            Count++;
        }
    }

    return Count;
}

FString UAsyncTaskScheduler::GenerateTaskId()
{
    return FGuid::NewGuid().ToString();
}

void UAsyncTaskScheduler::ProcessScheduledTasks()
{
    FScopeLock Lock(&TaskMutex);

    double CurrentTime = FPlatformTime::Seconds();
    TArray<FString> CompletedTasks;

    for (auto& Pair : ActiveTasks)
    {
        FScheduledTask& Task = Pair.Value;

        if (Task.bIsCancelled || Task.bIsComplete)
        {
            CompletedTasks.Add(Pair.Key);
            continue;
        }

        // 检查是否到达执行时间
        double Elapsed = CurrentTime - Task.ScheduleTime;
        if (Task.DelaySeconds > 0.0f && Elapsed < Task.DelaySeconds)
        {
            continue;
        }

        // 执行任务
        ExecuteTask(Task);

        // 处理重复任务
        if (Task.bIsRepeating)
        {
            Task.ScheduleTime = CurrentTime;
        }
        else
        {
            Task.bIsComplete = true;
            CompletedTasks.Add(Pair.Key);
        }
    }

    // 清理完成的任务
    for (const FString& TaskId : CompletedTasks)
    {
        ActiveTasks.Remove(TaskId);
    }
}

void UAsyncTaskScheduler::ExecuteTask(FScheduledTask& Task)
{
    ENamedThreads::Type ThreadType = GetThreadForPriority(Task.Priority);

    if (ThreadType == ENamedThreads::GameThread)
    {
        // 主线程执行
        Task.Task();
    }
    else
    {
        // 异步线程执行
        AsyncTask(ThreadType, [TaskCopy = Task]() mutable
        {
            TaskCopy.Task();
        });
    }
}

ENamedThreads::Type UAsyncTaskScheduler::GetThreadForPriority(ETaskPriority Priority) const
{
    switch (Priority)
    {
        case ETaskPriority::Highest:
        case ETaskPriority::High:
            return ENamedThreads::GameThread;
        case ETaskPriority::Normal:
            return ENamedThreads::AnyThread;
        case ETaskPriority::Low:
            return ENamedThreads::AnyBackgroundThreadNormalTask;
        case ETaskPriority::Background:
            return ENamedThreads::AnyBackgroundThreadLowPriority;
        default:
            return ENamedThreads::AnyThread;
    }
}
```

---

## 27.5 实践任务

### 任务1：创建性能监控仪表盘

实现一个实时性能监控UI：
- FPS和帧时间显示
- 内存使用图表
- 网络统计

### 任务2：优化Actor Tick

为一个大型关卡实现智能Tick：
- 距离剔除
- 优先级分组
- 时间预算控制

### 任务3：实现异步数据加载

使用任务调度系统：
- 异步加载资源
- 优先级队列
- 进度回调

---

## 27.6 总结

本课学习了：
- CPU性能分析工具和方法
- 内存池管理技术
- 垃圾回收优化策略
- 智能Tick管理
- 异步任务调度系统

**下一课预告**：服务器安全 - 输入验证、反作弊系统和加密通信。

---

*课程版本：1.0*
*最后更新：2026-04-10*
