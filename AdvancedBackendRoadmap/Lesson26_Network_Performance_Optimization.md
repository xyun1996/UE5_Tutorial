# 第26课：网络性能优化

> 学习目标：掌握网络带宽分析方法，学习包大小优化、压缩算法选择和网络流量裁剪技术。

---

## 26.1 网络带宽分析

### 26.1.1 UE5网络分析工具

```csharp
// NetworkProfilerSubsystem.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "NetworkProfilerSubsystem.generated.h"

USTRUCT(BlueprintType)
struct FNetworkStat
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    float OutgoingBandwidthKBps = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    float IncomingBandwidthKBps = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    int32 PacketsSentPerSecond = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 PacketsReceivedPerSecond = 0;

    UPROPERTY(BlueprintReadOnly)
    float AveragePacketSizeBytes = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    int32 ActorReplicationCount = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 RPCCount = 0;

    UPROPERTY(BlueprintReadOnly)
    float NetUpdateTime = 0.0f;
};

USTRUCT(BlueprintType)
struct FActorNetStat
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString ActorName;

    UPROPERTY(BlueprintReadOnly)
    UClass* ActorClass;

    UPROPERTY(BlueprintReadOnly)
    int32 ReplicatedProperties = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 ReplicatedSizeBits = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 RPCCount = 0;

    UPROPERTY(BlueprintReadOnly)
    float LastReplicationTime = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    int32 Priority = 0;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnNetworkStatsUpdated, const FNetworkStat&, Stats);

UCLASS()
class NETWORKPROFILER_API UNetworkProfilerSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 开始分析
    UFUNCTION(BlueprintCallable, Category = "Network|Profiler")
    void StartProfiling();

    // 停止分析
    UFUNCTION(BlueprintCallable, Category = "Network|Profiler")
    void StopProfiling();

    // 获取当前统计
    UFUNCTION(BlueprintPure, Category = "Network|Profiler")
    FNetworkStat GetCurrentStats() const { return CurrentStats; }

    // 获取Actor统计
    UFUNCTION(BlueprintCallable, Category = "Network|Profiler")
    TArray<FActorNetStat> GetActorStats() const;

    // 获取带宽报告
    UFUNCTION(BlueprintCallable, Category = "Network|Profiler")
    FString GenerateBandwidthReport();

    // 导出分析数据
    UFUNCTION(BlueprintCallable, Category = "Network|Profiler")
    void ExportProfilingData(const FString& FilePath);

    // 设置分析间隔
    UFUNCTION(BlueprintCallable, Category = "Network|Profiler")
    void SetProfilingInterval(float IntervalSeconds);

    UPROPERTY(BlueprintAssignable, Category = "Network|Profiler")
    FOnNetworkStatsUpdated OnStatsUpdated;

private:
    bool bIsProfiling = false;
    FNetworkStat CurrentStats;
    TArray<FActorNetStat> ActorStats;
    FTimerHandle ProfilingTimerHandle;
    float ProfilingInterval = 1.0f;

    // 累计统计
    int64 TotalBytesSent = 0;
    int64 TotalBytesReceived = 0;
    int64 TotalPacketsSent = 0;
    int64 TotalPacketsReceived = 0;
    double ProfilingStartTime = 0.0;

    void CaptureStats();
    void UpdateActorStats();
    void CalculateBandwidth();
};
```

```csharp
// NetworkProfilerSubsystem.cpp
#include "NetworkProfilerSubsystem.h"
#include "Engine/NetConnection.h"
#include "Engine/NetDriver.h"
#include "GameFramework/PlayerController.h"
#include "TimerManager.h"

void UNetworkProfilerSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    // 绑定网络统计回调
    if (UWorld* World = GetWorld())
    {
        World->OnWorldBeginPlay.AddDynamic(this, &UNetworkProfilerSubsystem::StartProfiling);
    }
}

void UNetworkProfilerSubsystem::Deinitialize()
{
    StopProfiling();
    Super::Deinitialize();
}

void UNetworkProfilerSubsystem::StartProfiling()
{
    if (bIsProfiling)
    {
        return;
    }

    bIsProfiling = true;
    ProfilingStartTime = FPlatformTime::Seconds();
    TotalBytesSent = 0;
    TotalBytesReceived = 0;
    TotalPacketsSent = 0;
    TotalPacketsReceived = 0;

    // 启动定时采样
    if (UWorld* World = GetWorld())
    {
        World->GetTimerManager().SetTimer(
            ProfilingTimerHandle,
            this,
            &UNetworkProfilerSubsystem::CaptureStats,
            ProfilingInterval,
            true
        );
    }

    UE_LOG(LogTemp, Log, TEXT("Network profiling started"));
}

void UNetworkProfilerSubsystem::StopProfiling()
{
    if (!bIsProfiling)
    {
        return;
    }

    bIsProfiling = false;

    if (UWorld* World = GetWorld())
    {
        World->GetTimerManager().ClearTimer(ProfilingTimerHandle);
    }

    UE_LOG(LogTemp, Log, TEXT("Network profiling stopped"));
}

void UNetworkProfilerSubsystem::CaptureStats()
{
    if (UWorld* World = GetWorld())
    {
        UNetDriver* NetDriver = World->GetNetDriver();
        if (!NetDriver)
        {
            return;
        }

        // 获取连接统计
        for (UNetConnection* Connection : NetDriver->ClientConnections)
        {
            if (Connection)
            {
                // 更新发送统计
                TotalBytesSent += Connection->OutBytes;
                TotalPacketsSent += Connection->OutPackets;

                // 更新接收统计
                TotalBytesReceived += Connection->InBytes;
                TotalPacketsReceived += Connection->InPackets;
            }
        }

        // 计算带宽
        CalculateBandwidth();

        // 更新Actor统计
        UpdateActorStats();

        // 广播更新
        OnStatsUpdated.Broadcast(CurrentStats);
    }
}

void UNetworkProfilerSubsystem::CalculateBandwidth()
{
    double CurrentTime = FPlatformTime::Seconds();
    double ElapsedTime = CurrentTime - ProfilingStartTime;

    if (ElapsedTime > 0)
    {
        CurrentStats.OutgoingBandwidthKBps = (TotalBytesSent / 1024.0) / ElapsedTime;
        CurrentStats.IncomingBandwidthKBps = (TotalBytesReceived / 1024.0) / ElapsedTime;
        CurrentStats.PacketsSentPerSecond = TotalPacketsSent / ElapsedTime;
        CurrentStats.PacketsReceivedPerSecond = TotalPacketsReceived / ElapsedTime;

        if (TotalPacketsSent > 0)
        {
            CurrentStats.AveragePacketSizeBytes = TotalBytesSent / (float)TotalPacketsSent;
        }
    }
}

void UNetworkProfilerSubsystem::UpdateActorStats()
{
    ActorStats.Empty();

    if (UWorld* World = GetWorld())
    {
        for (TActorIterator<AActor> It(World); It; ++It)
        {
            AActor* Actor = *It;
            if (Actor && Actor->GetIsReplicated())
            {
                FActorNetStat Stat;
                Stat.ActorName = Actor->GetName();
                Stat.ActorClass = Actor->GetClass();
                Stat.Priority = Actor->GetNetPriority();

                // 获取复制属性数量
                UClass* Class = Actor->GetClass();
                for (TFieldIterator<FProperty> PropIt(Class); PropIt; ++PropIt)
                {
                    if (PropIt->HasAnyPropertyFlags(CPF_Net))
                    {
                        Stat.ReplicatedProperties++;
                    }
                }

                ActorStats.Add(Stat);
            }
        }
    }
}

FString UNetworkProfilerSubsystem::GenerateBandwidthReport()
{
    FString Report;
    Report += TEXT("=== Network Bandwidth Report ===\n");
    Report += FString::Printf(TEXT("Outgoing: %.2f KB/s\n"), CurrentStats.OutgoingBandwidthKBps);
    Report += FString::Printf(TEXT("Incoming: %.2f KB/s\n"), CurrentStats.IncomingBandwidthKBps);
    Report += FString::Printf(TEXT("Packets Sent/sec: %d\n"), CurrentStats.PacketsSentPerSecond);
    Report += FString::Printf(TEXT("Packets Received/sec: %d\n"), CurrentStats.PacketsReceivedPerSecond);
    Report += FString::Printf(TEXT("Avg Packet Size: %.1f bytes\n"), CurrentStats.AveragePacketSizeBytes);
    Report += TEXT("\n=== Top Bandwidth Actors ===\n");

    // 按复制大小排序
    ActorStats.Sort([](const FActorNetStat& A, const FActorNetStat& B)
    {
        return A.ReplicatedSizeBits > B.ReplicatedSizeBits;
    });

    for (int32 i = 0; i < FMath::Min(10, ActorStats.Num()); i++)
    {
        const FActorNetStat& Stat = ActorStats[i];
        Report += FString::Printf(TEXT("%s: %d props, %d bits\n"),
            *Stat.ActorName, Stat.ReplicatedProperties, Stat.ReplicatedSizeBits);
    }

    return Report;
}
```

### 26.1.2 使用Unreal Insights

**启动网络追踪：**

```bash
# 命令行启动追踪
MyGame.exe -trace=net,cpu,frame -tracehost=127.0.0.1

# 或者在游戏中使用控制台命令
trace start net
trace start cpu
trace stop
trace save
```

**关键网络追踪事件：**

```csharp
// 自定义追踪事件
#include "Trace/Trace.h"

// 定义追踪通道
UE_TRACE_CHANNEL(CustomNetChannel)

// 记录自定义网络事件
void LogCustomNetworkEvent(const FString& EventName, int32 DataSize)
{
    TRACE_CPUPROFILER_EVENT_SCOPE(CustomNetworkEvent);
    UE_TRACE_LOG(CustomNetChannel, CustomNetworkEvent)
        << EventName
        << DataSize;
}

// 在关键网络操作中使用
void AMyActor::OnRep_ImportantData()
{
    TRACE_CPUPROFILER_EVENT_SCOPE(OnRep_ImportantData);
    // 处理逻辑
}
```

---

## 26.2 包大小优化

### 26.2.1 属性复制优化

```csharp
// OptimizedReplicationActor.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"

#include "OptimizedReplicationActor.generated.h"

UCLASS()
class NETWORKOPTIMIZATION_API AOptimizedReplicationActor : public AActor
{
    GENERATED_BODY()

public:
    AOptimizedReplicationActor();

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
    virtual void PreReplication(IRepChangedPropertyTracker& ChangedPropertyTracker) override;

protected:
    // ===== 策略1：条件复制 =====

    // 仅对所有者可见
    UPROPERTY(ReplicatedUsing = OnRep_PlayerData, meta = (AllowPrivateAccess = true))
    FPlayerData PlayerData;

    UFUNCTION()
    void OnRep_PlayerData();

    // 仅在相关时复制
    UPROPERTY(Replicated)
    FVector TargetLocation;

    // 使用条件属性
    UPROPERTY(Replicated)
    float Health;

    UPROPERTY(Replicated)
    float MaxHealth;

    // ===== 策略2：频率控制 =====

    // 低频更新数据
    UPROPERTY(Replicated)
    FString PlayerName;

    // 高频更新数据
    UPROPERTY(ReplicatedUsing = OnRep_Position)
    FVector CurrentPosition;

    UFUNCTION()
    void OnRep_Position();

    // ===== 策略3：量化优化 =====

    // 使用量化减少精度
    UPROPERTY(Replicated)
    FRotator Rotation; // 可量化为16位

    // 自定义量化
    UPROPERTY(ReplicatedUsing = OnRep_QuantizedVelocity)
    FVector QuantizedVelocity;

    UFUNCTION()
    void OnRep_QuantizedVelocity();

    // ===== 策略4：结构体优化 =====

    USTRUCT(BlueprintType)
    struct FCompactGameState
    {
        GENERATED_BODY()

        // 使用紧凑类型
        UPROPERTY()
        uint8 GameState : 3;  // 0-7

        UPROPERTY()
        uint8 RoundNumber : 5; // 0-31

        UPROPERTY()
        uint16 MatchTimeSeconds; // 0-65535

        UPROPERTY()
        uint8 PlayerCount;
    };

    UPROPERTY(ReplicatedUsing = OnRep_CompactGameState)
    FCompactGameState CompactGameState;

    // ===== 策略5：增量复制 =====

    UPROPERTY(ReplicatedUsing = OnRep_DeltaPosition)
    FVector DeltaPosition;

    FVector LastReplicatedPosition;

private:
    // 条件复制属性
    bool ShouldReplicateHealth() const;
    bool ShouldReplicatePosition() const;
};
```

```csharp
// OptimizedReplicationActor.cpp
#include "OptimizedReplicationActor.h"
#include "Net/UnrealNetwork.h"

AOptimizedReplicationActor::AOptimizedReplicationActor()
{
    bReplicates = true;
    NetUpdateFrequency = 30.0f; // 30Hz更新
    MinNetUpdateFrequency = 10.0f;
}

void AOptimizedReplicationActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // ===== 条件复制 =====

    // 仅所有者复制
    DOREPLIFETIME_CONDITION(AOptimizedReplicationActor, PlayerData, COND_OwnerOnly);

    // 仅在自动代理（客户端控制）时复制
    DOREPLIFETIME_CONDITION(AOptimizedReplicationActor, TargetLocation, COND_AutonomousOnly);

    // 基于自定义条件
    DOREPLIFETIME_CONDITION_NOTIFY(AOptimizedReplicationActor, Health, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(AOptimizedReplicationActor, MaxHealth, COND_None, REPNOTIFY_Always);

    // ===== 频率控制 =====

    // 低频更新
    DOREPLIFETIME_CONDITION(AOptimizedReplicationActor, PlayerName, COND_InitialOnly);

    // 高频更新（位置）
    DOREPLIFETIME_CONDITION(AOptimizedReplicationActor, CurrentPosition, COND_None);

    // ===== 量化设置 =====

    // 使用量化减少带宽
    DOREPLIFETIME(AOptimizedReplicationActor, Rotation);

    // 自定义量化速度
    DOREPLIFETIME(AOptimizedReplicationActor, QuantizedVelocity);

    // ===== 紧凑结构体 =====

    DOREPLIFETIME(AOptimizedReplicationActor, CompactGameState);

    // ===== 增量复制 =====

    DOREPLIFETIME(AOptimizedReplicationActor, DeltaPosition);
}

void AOptimizedReplicationActor::PreReplication(IRepChangedPropertyTracker& ChangedPropertyTracker)
{
    Super::PreReplication(ChangedPropertyTracker);

    // 计算增量位置
    if (CurrentPosition != LastReplicatedPosition)
    {
        DeltaPosition = CurrentPosition - LastReplicatedPosition;
        LastReplicatedPosition = CurrentPosition;
    }

    // 量化速度（限制精度）
    QuantizedVelocity = Velocity;
    QuantizedVelocity.X = FMath::RoundToFloat(QuantizedVelocity.X * 100.0f) / 100.0f;
    QuantizedVelocity.Y = FMath::RoundToFloat(QuantizedVelocity.Y * 100.0f) / 100.0f;
    QuantizedVelocity.Z = FMath::RoundToFloat(QuantizedVelocity.Z * 100.0f) / 100.0f;

    // 条件性禁用复制
    if (!ShouldReplicateHealth())
    {
        DOREPLIFETIME_CONDITION_OVERRIDE(AOptimizedReplicationActor, Health, COND_Never);
    }
}

bool AOptimizedReplicationActor::ShouldReplicateHealth() const
{
    // 仅在战斗中或有变化时复制
    return Health > 0.0f && Health < MaxHealth;
}
```

### 26.2.2 RPC优化

```csharp
// OptimizedRPCActor.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"

#include "OptimizedRPCActor.generated.h"

USTRUCT(BlueprintType)
struct FCompactInputData
{
    GENERATED_BODY()

    UPROPERTY()
    uint8 MoveFlags = 0;  // 位标志打包

    UPROPERTY()
    int16 Pitch = 0;      // 量化俯仰角

    UPROPERTY()
    int16 Yaw = 0;        // 量化偏航角

    UPROPERTY()
    uint8 JumpPressed : 1;
    UPROPERTY()
    uint8 CrouchPressed : 1;
    UPROPERTY()
    uint8 FirePressed : 1;
    UPROPERTY()
    uint8 ReloadPressed : 1;
};

UCLASS()
class NETWORKOPTIMIZATION_API AOptimizedRPCActor : public AActor
{
    GENERATED_BODY()

public:
    // ===== 批量RPC =====

    // 不推荐：多次调用小RPC
    UFUNCTION(Server, Reliable)
    void ServerUpdatePosition_One(FVector Position);

    UFUNCTION(Server, Reliable)
    void ServerUpdateRotation_One(FRotator Rotation);

    UFUNCTION(Server, Reliable)
    void ServerUpdateVelocity_One(FVector Velocity);

    // 推荐：批量打包RPC
    UFUNCTION(Server, Reliable)
    void ServerUpdateTransform_Batch(const FVector& Position, const FRotator& Rotation, const FVector& Velocity);

    // ===== 紧凑数据结构 =====

    // 不推荐：大量参数
    UFUNCTION(Server, Unreliable)
    void ServerPlayerInput_Full(FVector MoveDirection, FRotator ViewRotation, bool bJumped, bool bFired, bool bCrouched, bool bReloaded);

    // 推荐：紧凑结构体
    UFUNCTION(Server, Unreliable)
    void ServerPlayerInput_Compact(const FCompactInputData& InputData);

    // ===== 不可靠RPC用于高频数据 =====

    // 高频位置更新使用不可靠
    UFUNCTION(Server, Unreliable)
    void ServerUpdateMovement(const FVector& Position, const FRotator& Rotation);

    // 重要事件使用可靠
    UFUNCTION(Server, Reliable)
    void ServerFireWeapon(const FVector& TargetLocation, int32 WeaponId);

    // ===== 多播优化 =====

    // 仅对相关玩家多播
    UFUNCTION(NetMulticast, Unreliable)
    void MulticastPlayEffect_InRange(const FVector& Location, float Radius, int32 EffectId);

    // 使用Throttled多播
    UFUNCTION(NetMulticast, Unreliable)
    void MulticastPlaySound_Throttled(const FVector& Location, int32 SoundId);

private:
    // 多播节流
    TMap<int32, float> LastMulticastTime;
    float MulticastThrottleInterval = 0.1f;

    bool ShouldSendMulticast(int32 EffectId);
};
```

```csharp
// OptimizedRPCActor.cpp
#include "OptimizedRPCActor.h"

void AOptimizedRPCActor::ServerPlayerInput_Compact_Implementation(const FCompactInputData& InputData)
{
    // 解包紧凑数据
    FVector MoveDirection;
    MoveDirection.X = (InputData.MoveFlags & 0x01) ? 1.0f : ((InputData.MoveFlags & 0x02) ? -1.0f : 0.0f);
    MoveDirection.Y = (InputData.MoveFlags & 0x04) ? 1.0f : ((InputData.MoveFlags & 0x08) ? -1.0f : 0.0f);
    MoveDirection.Z = 0.0f;

    FRotator ViewRotation;
    ViewRotation.Pitch = InputData.Pitch / 32767.0f * 90.0f; // 解量化
    ViewRotation.Yaw = InputData.Yaw / 32767.0f * 180.0f;
    ViewRotation.Roll = 0.0f;

    // 处理输入
    // ...
}

void AOptimizedRPCActor::MulticastPlayEffect_InRange_Implementation(const FVector& Location, float Radius, int32 EffectId)
{
    // 节流检查
    if (!ShouldSendMulticast(EffectId))
    {
        return;
    }

    // 获取范围内的连接
    TArray<UNetConnection*> RelevantConnections;

    if (UWorld* World = GetWorld())
    {
        for (auto It = World->GetPlayerControllerIterator(); It; ++It)
        {
            APlayerController* PC = It->Get();
            if (PC && PC->GetPawn())
            {
                float Distance = FVector::Dist(PC->GetPawn()->GetActorLocation(), Location);
                if (Distance <= Radius)
                {
                    RelevantConnections.Add(PC->GetNetConnection());
                }
            }
        }
    }

    // 仅发送给相关连接
    for (UNetConnection* Connection : RelevantConnections)
    {
        // 发送效果RPC
    }
}

bool AOptimizedRPCActor::ShouldSendMulticast(int32 EffectId)
{
    float CurrentTime = GetWorld()->GetTimeSeconds();
    float* LastTime = LastMulticastTime.Find(EffectId);

    if (LastTime && (CurrentTime - *LastTime) < MulticastThrottleInterval)
    {
        return false;
    }

    LastMulticastTime.Add(EffectId, CurrentTime);
    return true;
}
```

---

## 26.3 压缩算法选择

### 26.3.1 网络数据压缩

```csharp
// NetworkCompressionManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "NetworkCompressionManager.generated.h"

UENUM(BlueprintType)
enum class ECompressionAlgorithm : uint8
{
    None,
    Zlib,
    LZ4,
    Oodle,
    LZMA
};

USTRUCT(BlueprintType)
struct FCompressionResult
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    TArray<uint8> CompressedData;

    UPROPERTY(BlueprintReadOnly)
    int32 OriginalSize = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 CompressedSize = 0;

    UPROPERTY(BlueprintReadOnly)
    float CompressionRatio = 1.0f;

    UPROPERTY(BlueprintReadOnly)
    float CompressionTimeMs = 0.0f;
};

UCLASS()
class NETWORKCOMPRESSION_API UNetworkCompressionManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    // 压缩数据
    UFUNCTION(BlueprintCallable, Category = "Compression")
    FCompressionResult CompressData(const TArray<uint8>& Data, ECompressionAlgorithm Algorithm);

    // 解压数据
    UFUNCTION(BlueprintCallable, Category = "Compression")
    bool DecompressData(const TArray<uint8>& CompressedData, TArray<uint8>& OutData, int32 OriginalSize, ECompressionAlgorithm Algorithm);

    // 选择最佳算法
    UFUNCTION(BlueprintCallable, Category = "Compression")
    ECompressionAlgorithm SelectBestAlgorithm(const TArray<uint8>& SampleData);

    // 批量压缩
    UFUNCTION(BlueprintCallable, Category = "Compression")
    TArray<FCompressionResult> CompressBatch(const TArray<TArray<uint8>>& DataBatch, ECompressionAlgorithm Algorithm);

    // 获取算法性能指标
    UFUNCTION(BlueprintPure, Category = "Compression")
    void GetAlgorithmMetrics(ECompressionAlgorithm Algorithm, float& SpeedMBps, float& AvgRatio);

private:
    TMap<ECompressionAlgorithm, float> AlgorithmSpeedCache;
    TMap<ECompressionAlgorithm, float> AlgorithmRatioCache;

    void UpdateMetrics(ECompressionAlgorithm Algorithm, float TimeMs, float Ratio);
};
```

```csharp
// NetworkCompressionManager.cpp
#include "NetworkCompressionManager.h"
#include "Compression/OodleDataCompressionUtil.h"
#include "Misc/Compression.h"

FCompressionResult UNetworkCompressionManager::CompressData(const TArray<uint8>& Data, ECompressionAlgorithm Algorithm)
{
    FCompressionResult Result;
    Result.OriginalSize = Data.Num();

    double StartTime = FPlatformTime::Seconds();

    switch (Algorithm)
    {
        case ECompressionAlgorithm::Zlib:
        {
            int32 CompressedSize = Data.Num();
            Result.CompressedData.SetNumUninitialized(CompressedSize);

            // Zlib压缩
            int32 Flags = COMPRESS_ZLIB;
            while (true)
            {
                int32 ActualSize = FCompression::CompressMemory(
                    (ECompressionFlags)Flags,
                    Result.CompressedData.GetData(),
                    CompressedSize,
                    Data.GetData(),
                    Data.Num()
                );

                if (ActualSize > 0)
                {
                    Result.CompressedData.SetNum(ActualSize);
                    Result.CompressedSize = ActualSize;
                    break;
                }

                // 缓冲区不够，扩大
                CompressedSize = FMath::Max(CompressedSize * 2, ActualSize * -1);
                Result.CompressedData.SetNumUninitialized(CompressedSize);
            }
            break;
        }

        case ECompressionAlgorithm::LZ4:
        {
            int32 MaxCompressedSize = FCompression::CompressMemoryBound(COMPRESS_LZ4, Data.Num());
            Result.CompressedData.SetNumUninitialized(MaxCompressedSize);

            int32 ActualSize = FCompression::CompressMemory(
                COMPRESS_LZ4,
                Result.CompressedData.GetData(),
                MaxCompressedSize,
                Data.GetData(),
                Data.Num()
            );

            Result.CompressedData.SetNum(ActualSize);
            Result.CompressedSize = ActualSize;
            break;
        }

        case ECompressionAlgorithm::Oodle:
        {
            // Oodle压缩（UE5内置）
            int32 MaxCompressedSize = FOodleDataCompression::CompressedDataSizeBound(Data.Num());
            Result.CompressedData.SetNumUninitialized(MaxCompressedSize);

            int32 ActualSize = FOodleDataCompression::Compress(
                Result.CompressedData.GetData(),
                MaxCompressedSize,
                Data.GetData(),
                Data.Num(),
                FOodleDataCompression::ECompressor::Kraken,
                FOodleDataCompression::ECompressionLevel::Optimal2
            );

            Result.CompressedData.SetNum(ActualSize);
            Result.CompressedSize = ActualSize;
            break;
        }

        default:
            Result.CompressedData = Data;
            Result.CompressedSize = Data.Num();
            break;
    }

    double EndTime = FPlatformTime::Seconds();
    Result.CompressionTimeMs = (EndTime - StartTime) * 1000.0f;

    if (Result.OriginalSize > 0)
    {
        Result.CompressionRatio = (float)Result.CompressedSize / Result.OriginalSize;
    }

    // 更新性能指标
    UpdateMetrics(Algorithm, Result.CompressionTimeMs, Result.CompressionRatio);

    return Result;
}

bool UNetworkCompressionManager::DecompressData(const TArray<uint8>& CompressedData, TArray<uint8>& OutData, int32 OriginalSize, ECompressionAlgorithm Algorithm)
{
    OutData.SetNumUninitialized(OriginalSize);

    switch (Algorithm)
    {
        case ECompressionAlgorithm::Zlib:
            return FCompression::UncompressMemory(
                COMPRESS_ZLIB,
                OutData.GetData(),
                OriginalSize,
                CompressedData.GetData(),
                CompressedData.Num()
            );

        case ECompressionAlgorithm::LZ4:
            return FCompression::UncompressMemory(
                COMPRESS_LZ4,
                OutData.GetData(),
                OriginalSize,
                CompressedData.GetData(),
                CompressedData.Num()
            );

        case ECompressionAlgorithm::Oodle:
            return FOodleDataCompression::Decompress(
                OutData.GetData(),
                OriginalSize,
                CompressedData.GetData(),
                CompressedData.Num()
            ) > 0;

        default:
            OutData = CompressedData;
            return true;
    }
}

ECompressionAlgorithm UNetworkCompressionManager::SelectBestAlgorithm(const TArray<uint8>& SampleData)
{
    // 小数据不压缩
    if (SampleData.Num() < 64)
    {
        return ECompressionAlgorithm::None;
    }

    // 实时数据优先速度
    // 批量数据优先压缩率

    float BestScore = -1.0f;
    ECompressionAlgorithm BestAlgorithm = ECompressionAlgorithm::LZ4;

    TArray<ECompressionAlgorithm> TestAlgorithms = {
        ECompressionAlgorithm::LZ4,
        ECompressionAlgorithm::Oodle,
        ECompressionAlgorithm::Zlib
    };

    for (ECompressionAlgorithm Alg : TestAlgorithms)
    {
        FCompressionResult Result = CompressData(SampleData, Alg);

        // 计算综合得分
        // 压缩率权重 0.7，速度权重 0.3
        float SpeedScore = 1.0f / (Result.CompressionTimeMs + 0.001f);
        float RatioScore = 1.0f / Result.CompressionRatio;

        float Score = RatioScore * 0.7f + SpeedScore * 0.3f;

        if (Score > BestScore)
        {
            BestScore = Score;
            BestAlgorithm = Alg;
        }
    }

    // 如果压缩率不够好，不压缩
    FCompressionResult BestResult = CompressData(SampleData, BestAlgorithm);
    if (BestResult.CompressionRatio > 0.9f)
    {
        return ECompressionAlgorithm::None;
    }

    return BestAlgorithm;
}
```

---

## 26.4 网络流量裁剪

### 26.4.1 空间相关性优化

```csharp
// SpatialRelevanceManager.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"

#include "SpatialRelevanceManager.generated.h"

USTRUCT(BlueprintType)
struct FRelevanceSettings
{
    GENERATED_BODY()

    // 基础可见距离
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float BaseVisibleDistance = 5000.0f;

    // 网络优先级
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float NetworkPriority = 1.0f;

    // 更新频率
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float NetUpdateFrequency = 30.0f;

    // 空间分区网格大小
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float GridCellSize = 1000.0f;
};

UCLASS()
class NETWORKOPTIMIZATION_API ASpatialRelevanceManager : public AActor
{
    GENERATED_BODY()

public:
    ASpatialRelevanceManager();

    virtual void Tick(float DeltaTime) override;

    // 更新玩家相关性
    void UpdatePlayerRelevance(APlayerController* PC);

    // 获取相关Actor
    TArray<AActor*> GetRelevantActors(APlayerController* PC, const FVector& Location);

    // 空间查询
    void GetActorsInRadius(const FVector& Center, float Radius, TArray<AActor*>& OutActors);

    // 配置
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Relevance")
    FRelevanceSettings Settings;

private:
    // 空间分区网格
    TMap<FIntVector, TArray<AActor*>> SpatialGrid;

    // 玩家相关性缓存
    TMap<APlayerController*, TArray<AActor*>> PlayerRelevanceCache;

    // 上次更新时间
    TMap<APlayerController*, float> LastUpdateTime;

    void UpdateSpatialGrid();
    FIntVector WorldToGrid(const FVector& Location) const;
    void AddActorToGrid(AActor* Actor);
    void RemoveActorFromGrid(AActor* Actor);
};
```

```csharp
// SpatialRelevanceManager.cpp
#include "SpatialRelevanceManager.h"
#include "GameFramework/PlayerController.h"
#include "Kismet/GameplayStatics.h"

ASpatialRelevanceManager::ASpatialRelevanceManager()
{
    PrimaryActorTick.bCanEverTick = true;
}

void ASpatialRelevanceManager::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    // 更新空间分区
    UpdateSpatialGrid();

    // 更新每个玩家的相关性
    if (UWorld* World = GetWorld())
    {
        for (auto It = World->GetPlayerControllerIterator(); It; ++It)
        {
            APlayerController* PC = It->Get();
            if (PC)
            {
                UpdatePlayerRelevance(PC);
            }
        }
    }
}

void ASpatialRelevanceManager::UpdateSpatialGrid()
{
    SpatialGrid.Empty();

    if (UWorld* World = GetWorld())
    {
        for (TActorIterator<AActor> It(World); It; ++It)
        {
            AActor* Actor = *It;
            if (Actor && Actor->GetIsReplicated())
            {
                AddActorToGrid(Actor);
            }
        }
    }
}

FIntVector ASpatialRelevanceManager::WorldToGrid(const FVector& Location) const
{
    return FIntVector(
        FMath::FloorToInt(Location.X / Settings.GridCellSize),
        FMath::FloorToInt(Location.Y / Settings.GridCellSize),
        FMath::FloorToInt(Location.Z / Settings.GridCellSize)
    );
}

void ASpatialRelevanceManager::AddActorToGrid(AActor* Actor)
{
    FIntVector GridPos = WorldToGrid(Actor->GetActorLocation());
    SpatialGrid.FindOrAdd(GridPos).Add(Actor);
}

TArray<AActor*> ASpatialRelevanceManager::GetRelevantActors(APlayerController* PC, const FVector& Location)
{
    TArray<AActor*> RelevantActors;

    // 获取玩家所在的网格
    FIntVector CenterGrid = WorldToGrid(Location);

    // 计算需要检查的网格数量
    int32 GridRadius = FMath::CeilToInt(Settings.BaseVisibleDistance / Settings.GridCellSize);

    // 遍历周围的网格
    for (int32 X = CenterGrid.X - GridRadius; X <= CenterGrid.X + GridRadius; X++)
    {
        for (int32 Y = CenterGrid.Y - GridRadius; Y <= CenterGrid.Y + GridRadius; Y++)
        {
            for (int32 Z = CenterGrid.Z - GridRadius; Z <= CenterGrid.Z + GridRadius; Z++)
            {
                FIntVector GridPos(X, Y, Z);
                if (TArray<AActor*>* Actors = SpatialGrid.Find(GridPos))
                {
                    for (AActor* Actor : *Actors)
                    {
                        // 检查实际距离
                        float Distance = FVector::Dist(Location, Actor->GetActorLocation());
                        if (Distance <= Settings.BaseVisibleDistance)
                        {
                            RelevantActors.Add(Actor);
                        }
                    }
                }
            }
        }
    }

    return RelevantActors;
}

void ASpatialRelevanceManager::UpdatePlayerRelevance(APlayerController* PC)
{
    if (!PC || !PC->GetPawn())
    {
        return;
    }

    float CurrentTime = GetWorld()->GetTimeSeconds();

    // 节流更新
    float* LastTime = LastUpdateTime.Find(PC);
    if (LastTime && (CurrentTime - *LastTime) < 0.1f)
    {
        return;
    }

    LastUpdateTime.Add(PC, CurrentTime);

    FVector PlayerLocation = PC->GetPawn()->GetActorLocation();
    TArray<AActor*> RelevantActors = GetRelevantActors(PC, PlayerLocation);

    // 更新相关性缓存
    PlayerRelevanceCache.Add(PC, RelevantActors);

    // 设置Actor的网络相关性
    for (AActor* Actor : RelevantActors)
    {
        // 增加相关Actor的优先级
        Actor->SetNetPriority(Settings.NetworkPriority);
    }
}
```

### 26.4.2 优先级队列管理

```csharp
// NetworkPriorityManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/WorldSubsystem.h"

#include "NetworkPriorityManager.generated.h"

USTRUCT(BlueprintType)
struct FPriorityActor
{
    GENERATED_BODY()

    UPROPERTY()
    AActor* Actor = nullptr;

    UPROPERTY()
    float Priority = 0.0f;

    UPROPERTY()
    float LastUpdateTime = 0.0f;

    UPROPERTY()
    float DistanceToPlayer = 0.0f;
};

UCLASS()
class NETWORKOPTIMIZATION_API UNetworkPriorityManager : public UWorldSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 计算Actor优先级
    float CalculatePriority(AActor* Actor, APlayerController* ForPlayer);

    // 更新优先级队列
    void UpdatePriorityQueue(APlayerController* PC);

    // 获取本轮应该更新的Actor
    TArray<AActor*> GetActorsToUpdate(APlayerController* PC, int32 MaxCount);

    // 配置
    UPROPERTY(EditAnywhere, Category = "Priority")
    float DistancePriorityScale = 1.0f;

    UPROPERTY(EditAnywhere, Category = "Priority")
    float TimeSinceUpdateScale = 10.0f;

    UPROPERTY(EditAnywhere, Category = "Priority")
    float BasePriorityScale = 1.0f;

    UPROPERTY(EditAnywhere, Category = "Priority")
    float VelocityPriorityScale = 0.5f;

private:
    TMap<APlayerController*, TArray<FPriorityActor>> PriorityQueues;

    void SortPriorityQueue(APlayerController* PC);
};
```

```csharp
// NetworkPriorityManager.cpp
#include "NetworkPriorityManager.h"
#include "GameFramework/PlayerController.h"
#include "GameFramework/Pawn.h"

void UNetworkPriorityManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
}

float UNetworkPriorityManager::CalculatePriority(AActor* Actor, APlayerController* ForPlayer)
{
    if (!Actor || !ForPlayer)
    {
        return 0.0f;
    }

    float Priority = 0.0f;

    // 1. 基础优先级
    Priority += Actor->GetNetPriority() * BasePriorityScale;

    // 2. 距离因素
    if (APawn* Pawn = ForPlayer->GetPawn())
    {
        float Distance = FVector::Dist(Actor->GetActorLocation(), Pawn->GetActorLocation());

        // 距离越近优先级越高
        float DistanceFactor = FMath::Max(0.0f, 1.0f - Distance / 10000.0f);
        Priority += DistanceFactor * DistancePriorityScale;
    }

    // 3. 时间因素
    float CurrentTime = GetWorld()->GetTimeSeconds();
    float TimeSinceUpdate = CurrentTime - Actor->GetLastRenderTime();
    Priority += TimeSinceUpdate * TimeSinceUpdateScale;

    // 4. 速度因素
    FVector Velocity = Actor->GetVelocity();
    float Speed = Velocity.Size();
    Priority += (Speed / 1000.0f) * VelocityPriorityScale;

    // 5. 特殊标记
    // 高优先级标记
    if (Actor->GetClass()->ImplementsInterface(UHighPriorityReplication::StaticClass()))
    {
        Priority *= 2.0f;
    }

    // 拥有者总是高优先级
    if (Actor->GetOwner() == ForPlayer->GetPawn())
    {
        Priority *= 5.0f;
    }

    return Priority;
}

void UNetworkPriorityManager::UpdatePriorityQueue(APlayerController* PC)
{
    TArray<FPriorityActor>& Queue = PriorityQueues.FindOrAdd(PC);
    Queue.Empty();

    if (UWorld* World = GetWorld())
    {
        for (TActorIterator<AActor> It(World); It; ++It)
        {
            AActor* Actor = *It;
            if (Actor && Actor->GetIsReplicated())
            {
                FPriorityActor PriorityActor;
                PriorityActor.Actor = Actor;
                PriorityActor.Priority = CalculatePriority(Actor, PC);
                PriorityActor.LastUpdateTime = World->GetTimeSeconds();

                if (PC->GetPawn())
                {
                    PriorityActor.DistanceToPlayer = FVector::Dist(
                        Actor->GetActorLocation(),
                        PC->GetPawn()->GetActorLocation()
                    );
                }

                Queue.Add(PriorityActor);
            }
        }
    }

    SortPriorityQueue(PC);
}

void UNetworkPriorityManager::SortPriorityQueue(APlayerController* PC)
{
    if (TArray<FPriorityActor>* Queue = PriorityQueues.Find(PC))
    {
        Queue->Sort([](const FPriorityActor& A, const FPriorityActor& B)
        {
            return A.Priority > B.Priority;
        });
    }
}

TArray<AActor*> UNetworkPriorityManager::GetActorsToUpdate(APlayerController* PC, int32 MaxCount)
{
    TArray<AActor*> Result;

    if (TArray<FPriorityActor>* Queue = PriorityQueues.Find(PC))
    {
        int32 Count = FMath::Min(MaxCount, Queue->Num());
        for (int32 i = 0; i < Count; i++)
        {
            if ((*Queue)[i].Actor.IsValid())
            {
                Result.Add((*Queue)[i].Actor);
            }
        }
    }

    return Result;
}
```

---

## 26.5 实践任务

### 任务1：带宽分析工具

创建一个完整的带宽分析工具：
- 实时显示网络统计
- 识别高带宽Actor
- 生成优化建议

### 任务2：压缩系统集成

实现智能压缩系统：
- 自动选择压缩算法
- 动态调整压缩级别
- 性能监控

### 任务3：相关性优化

优化大型场景的网络相关性：
- 空间分区实现
- 动态优先级调整
- 更新频率控制

---

## 26.6 总结

本课学习了：
- 网络带宽分析工具和方法
- 属性复制优化策略
- RPC优化和批量处理
- 压缩算法选择
- 网络流量裁剪技术
- 空间相关性和优先级管理

**下一课预告**：服务器性能调优 - CPU性能分析、内存优化和异步处理模式。

---

*课程版本：1.0*
*最后更新：2026-04-10*
