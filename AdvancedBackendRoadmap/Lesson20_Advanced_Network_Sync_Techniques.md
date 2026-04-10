# 第20课：网络同步高级技巧

> 本课深入讲解UE5网络同步的高级技巧，包括时间同步、延迟补偿、防作弊和网络抖动处理。

---

## 课程目标

- 掌握时间同步机制实现
- 理解延迟补偿技术
- 学会防作弊同步验证
- 掌握网络抖动处理方法
- 理解时间线与插值算法

---

## 一、时间同步机制

### 1.1 时间同步原理

```
时间同步问题:

客户端时间                    服务器时间
    │                             │
    │  T1: 发送时间请求           │
    │  ──────────────────────►    │
    │                             │ T2: 服务器接收时间
    │                             │ T3: 服务器发送响应
    │  ◄──────────────────────    │
    │  T4: 接收时间               │
    │                             │

计算:
往返延迟(RTT) = (T4 - T1) - (T3 - T2)
时钟偏差 = (T2 + T3) / 2 - (T1 + T4) / 2
服务器时间 = 本地时间 + 时钟偏差 + RTT/2
```

### 1.2 时间同步实现

```cpp
// MyTimeSynchronizer.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "MyTimeSynchronizer.generated.h"

// 时间同步样本
USTRUCT()
struct FTimeSyncSample
{
    GENERATED_BODY()

    float ClientSendTime;
    float ServerReceiveTime;
    float ServerSendTime;
    float ClientReceiveTime;

    float GetRTT() const
    {
        return (ClientReceiveTime - ClientSendTime) - (ServerSendTime - ServerReceiveTime);
    }

    float GetClockOffset() const
    {
        return ((ServerReceiveTime + ServerSendTime) * 0.5f) -
               ((ClientSendTime + ClientReceiveTime) * 0.5f);
    }
};

// 时间同步配置
USTRUCT()
struct FTimeSyncConfig
{
    GENERATED_BODY()

    int32 SampleCount = 10;           // 用于计算平均值的样本数
    float SyncInterval = 1.0f;         // 同步间隔（秒）
    float MaxRTT = 0.5f;              // 最大可接受RTT（秒）
    float OutlierThreshold = 2.0f;    // 离群值检测阈值（标准差倍数）
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnTimeSyncComplete, float, EstimatedOffset);

UCLASS()
class UMyTimeSynchronizer : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 开始同步
    UFUNCTION(BlueprintCallable, Category = "Time Sync")
    void StartSync();

    // 停止同步
    UFUNCTION(BlueprintCallable, Category = "Time Sync")
    void StopSync();

    // 获取服务器时间
    UFUNCTION(BlueprintPure, Category = "Time Sync")
    float GetServerTime() const;

    UFUNCTION(BlueprintPure, Category = "Time Sync")
    float GetLocalToServerOffset() const { return EstimatedOffset; }

    // 获取延迟
    UFUNCTION(BlueprintPure, Category = "Time Sync")
    float GetRTT() const { return EstimatedRTT; }

    // 状态
    UFUNCTION(BlueprintPure, Category = "Time Sync")
    bool IsSynchronized() const { return bIsSynchronized; }

    UFUNCTION(BlueprintPure, Category = "Time Sync")
    float GetSyncAccuracy() const { return SyncAccuracy; }

    // 配置
    void SetConfig(const FTimeSyncConfig& Config) { CurrentConfig = Config; }

    // 委托
    UPROPERTY(BlueprintAssignable)
    FOnTimeSyncComplete OnSyncComplete;

protected:
    // 同步样本
    TArray<FTimeSyncSample> Samples;

    // 估计值
    float EstimatedOffset = 0.0f;
    float EstimatedRTT = 0.0f;
    float SyncAccuracy = 0.0f;

    // 状态
    bool bIsSynchronized = false;
    bool bIsSyncing = false;

    // 配置
    FTimeSyncConfig CurrentConfig;

    // 定时器
    FTimerHandle SyncTimer;

    // 方法
    void PerformSync();
    void ProcessSample(const FTimeSyncSample& Sample);
    void CalculateEstimates();
    void RemoveOutliers();
    void SendTimeRequest();

    // 服务器RPC
    UFUNCTION(Server, Reliable)
    void Server_TimeRequest(float ClientTime);

    UFUNCTION(Client, Reliable)
    void Client_TimeResponse(float ClientTime, float ServerReceiveTime, float ServerSendTime);
};

// MyTimeSynchronizer.cpp
#include "MyTimeSynchronizer.h"

void UMyTimeSynchronizer::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    CurrentConfig = FTimeSyncConfig();
}

void UMyTimeSynchronizer::Deinitialize()
{
    StopSync();
    Super::Deinitialize();
}

void UMyTimeSynchronizer::StartSync()
{
    if (bIsSyncing)
    {
        return;
    }

    bIsSyncing = true;
    Samples.Empty();

    // 开始定时同步
    GetWorld()->GetTimerManager().SetTimer(
        SyncTimer,
        this,
        &UMyTimeSynchronizer::PerformSync,
        CurrentConfig.SyncInterval,
        true
    );

    PerformSync();
}

void UMyTimeSynchronizer::StopSync()
{
    bIsSyncing = false;
    GetWorld()->GetTimerManager().ClearTimer(SyncTimer);
}

float UMyTimeSynchronizer::GetServerTime() const
{
    return GetWorld()->GetTimeSeconds() + EstimatedOffset;
}

void UMyTimeSynchronizer::PerformSync()
{
    SendTimeRequest();
}

void UMyTimeSynchronizer::SendTimeRequest()
{
    float ClientTime = GetWorld()->GetTimeSeconds();
    Server_TimeRequest(ClientTime);
}

void UMyTimeSynchronizer::Server_TimeRequest_Implementation(float ClientTime)
{
    float ServerReceiveTime = GetWorld()->GetTimeSeconds();

    // 处理请求
    // ...

    float ServerSendTime = GetWorld()->GetTimeSeconds();

    Client_TimeResponse(ClientTime, ServerReceiveTime, ServerSendTime);
}

void UMyTimeSynchronizer::Client_TimeResponse_Implementation(float ClientTime, float ServerReceiveTime, float ServerSendTime)
{
    FTimeSyncSample Sample;
    Sample.ClientSendTime = ClientTime;
    Sample.ServerReceiveTime = ServerReceiveTime;
    Sample.ServerSendTime = ServerSendTime;
    Sample.ClientReceiveTime = GetWorld()->GetTimeSeconds();

    ProcessSample(Sample);
}

void UMyTimeSynchronizer::ProcessSample(const FTimeSyncSample& Sample)
{
    // 检查RTT是否合理
    float RTT = Sample.GetRTT();
    if (RTT > CurrentConfig.MaxRTT)
    {
        UE_LOG(LogTemp, Warning, TEXT("Time sync sample rejected: RTT too high (%.3f)"), RTT);
        return;
    }

    Samples.Add(Sample);

    // 保持样本数量
    while (Samples.Num() > CurrentConfig.SampleCount)
    {
        Samples.RemoveAt(0);
    }

    // 计算估计值
    CalculateEstimates();
}

void UMyTimeSynchronizer::CalculateEstimates()
{
    if (Samples.Num() < 3)
    {
        return;
    }

    // 移除离群值
    RemoveOutliers();

    // 计算平均偏移和RTT
    float TotalOffset = 0.0f;
    float TotalRTT = 0.0f;

    for (const FTimeSyncSample& Sample : Samples)
    {
        TotalOffset += Sample.GetClockOffset();
        TotalRTT += Sample.GetRTT();
    }

    EstimatedOffset = TotalOffset / Samples.Num();
    EstimatedRTT = TotalRTT / Samples.Num();

    // 计算精度（标准差）
    float Variance = 0.0f;
    for (const FTimeSyncSample& Sample : Samples)
    {
        float Diff = Sample.GetClockOffset() - EstimatedOffset;
        Variance += Diff * Diff;
    }
    SyncAccuracy = FMath::Sqrt(Variance / Samples.Num());

    bIsSynchronized = true;
    OnSyncComplete.Broadcast(EstimatedOffset);

    UE_LOG(LogTemp, Log, TEXT("Time synchronized: Offset=%.3fms, RTT=%.3fms, Accuracy=%.3fms"),
        EstimatedOffset * 1000.0f, EstimatedRTT * 1000.0f, SyncAccuracy * 1000.0f);
}

void UMyTimeSynchronizer::RemoveOutliers()
{
    if (Samples.Num() < 5)
    {
        return;
    }

    // 计算均值和标准差
    float Mean = 0.0f;
    for (const FTimeSyncSample& Sample : Samples)
    {
        Mean += Sample.GetClockOffset();
    }
    Mean /= Samples.Num();

    float StdDev = 0.0f;
    for (const FTimeSyncSample& Sample : Samples)
    {
        float Diff = Sample.GetClockOffset() - Mean;
        StdDev += Diff * Diff;
    }
    StdDev = FMath::Sqrt(StdDev / Samples.Num());

    // 移除超出阈值的样本
    Samples.RemoveAll([Mean, StdDev, this](const FTimeSyncSample& Sample)
    {
        float Diff = FMath::Abs(Sample.GetClockOffset() - Mean);
        return Diff > StdDev * CurrentConfig.OutlierThreshold;
    });
}
```

---

## 二、延迟补偿技术

### 2.1 延迟补偿原理

```cpp
// MyLagCompensation.h
#pragma once

#include "CoreMinimal.h"
#include "MyLagCompensation.generated.h"

// 历史状态记录
USTRUCT()
struct FActorHistoryState
{
    GENERATED_BODY()

    float Timestamp;
    FVector Location;
    FRotator Rotation;
    FVector Velocity;
    FBox BoundingBox;
};

// 历史记录管理
UCLASS()
class UMyLagCompensationManager : public UObject
{
    GENERATED_BODY()

public:
    // 记录状态
    void RecordActorState(AActor* Actor);

    // 获取历史状态
    bool GetActorStateAtTime(AActor* Actor, float Time, FActorHistoryState& OutState);

    // 回滚Actor到历史状态
    void RollbackActorToTime(AActor* Actor, float Time);

    // 恢复Actor当前状态
    void RestoreActor(AActor* Actor);

    // 延迟补偿射线检测
    bool LagCompensatedLineTrace(
        const FVector& Start,
        const FVector& End,
        float ClientTime,
        FHitResult& OutHit,
        const TArray<AActor*>& ActorsToConsider,
        AActor* IgnoreActor = nullptr
    );

    // 配置
    void SetHistoryDuration(float Duration) { HistoryDuration = Duration; }

private:
    // 每个Actor的历史记录
    TMap<AActor*, TArray<FActorHistoryState>> ActorHistory;

    // 备份当前状态（用于恢复）
    TMap<AActor*, FActorHistoryState> CurrentStateBackup;

    float HistoryDuration = 1.0f; // 保存1秒历史

    void CleanupOldHistory();
    bool InterpolateState(const FActorHistoryState& A, const FActorHistoryState& B, float Time, FActorHistoryState& OutState);
};

// MyLagCompensation.cpp
#include "MyLagCompensationManager.h"

void UMyLagCompensationManager::RecordActorState(AActor* Actor)
{
    if (!Actor)
    {
        return;
    }

    FActorHistoryState State;
    State.Timestamp = GetWorld()->GetTimeSeconds();
    State.Location = Actor->GetActorLocation();
    State.Rotation = Actor->GetActorRotation();
    State.Velocity = FVector::ZeroVector;

    // 如果有移动组件，获取速度
    // if (UMovementComponent* Movement = Actor->FindComponentByClass<UMovementComponent>())
    // {
    //     State.Velocity = Movement->Velocity;
    // }

    State.BoundingBox = Actor->GetComponentsBoundingBox();

    TArray<FActorHistoryState>& History = ActorHistory.FindOrAdd(Actor);
    History.Add(State);

    CleanupOldHistory();
}

bool UMyLagCompensationManager::GetActorStateAtTime(AActor* Actor, float Time, FActorHistoryState& OutState)
{
    if (!Actor)
    {
        return false;
    }

    TArray<FActorHistoryState>* History = ActorHistory.Find(Actor);
    if (!History || History->Num() == 0)
    {
        return false;
    }

    // 二分查找最近的两个状态
    for (int32 i = History->Num() - 1; i > 0; i--)
    {
        if ((*History)[i].Timestamp <= Time)
        {
            // 在两个状态之间插值
            if (i < History->Num() - 1)
            {
                return InterpolateState((*History)[i], (*History)[i + 1], Time, OutState);
            }
            else
            {
                OutState = (*History)[i];
                return true;
            }
        }
    }

    return false;
}

void UMyLagCompensationManager::RollbackActorToTime(AActor* Actor, float Time)
{
    if (!Actor)
    {
        return;
    }

    // 备份当前状态
    FActorHistoryState CurrentState;
    CurrentState.Timestamp = GetWorld()->GetTimeSeconds();
    CurrentState.Location = Actor->GetActorLocation();
    CurrentState.Rotation = Actor->GetActorRotation();
    CurrentStateBackup.Add(Actor, CurrentState);

    // 获取历史状态
    FActorHistoryState HistoricalState;
    if (GetActorStateAtTime(Actor, Time, HistoricalState))
    {
        Actor->SetActorLocation(HistoricalState.Location);
        Actor->SetActorRotation(HistoricalState.Rotation);
    }
}

void UMyLagCompensationManager::RestoreActor(AActor* Actor)
{
    if (!Actor)
    {
        return;
    }

    FActorHistoryState* BackupState = CurrentStateBackup.Find(Actor);
    if (BackupState)
    {
        Actor->SetActorLocation(BackupState->Location);
        Actor->SetActorRotation(BackupState->Rotation);
        CurrentStateBackup.Remove(Actor);
    }
}

bool UMyLagCompensationManager::LagCompensatedLineTrace(
    const FVector& Start,
    const FVector& End,
    float ClientTime,
    FHitResult& OutHit,
    const TArray<AActor*>& ActorsToConsider,
    AActor* IgnoreActor)
{
    // 1. 回滚所有相关Actor
    TArray<AActor*> RolledBackActors;

    for (AActor* Actor : ActorsToConsider)
    {
        if (Actor != IgnoreActor)
        {
            RollbackActorToTime(Actor, ClientTime);
            RolledBackActors.Add(Actor);
        }
    }

    // 2. 执行射线检测
    FCollisionQueryParams Params;
    Params.AddIgnoredActor(IgnoreActor);

    bool bHit = GetWorld()->LineTraceSingleByChannel(
        OutHit,
        Start,
        End,
        ECC_Pawn,
        Params
    );

    // 3. 恢复所有Actor
    for (AActor* Actor : RolledBackActors)
    {
        RestoreActor(Actor);
    }

    return bHit;
}

void UMyLagCompensationManager::CleanupOldHistory()
{
    float CurrentTime = GetWorld()->GetTimeSeconds();
    float CutoffTime = CurrentTime - HistoryDuration;

    for (auto& Pair : ActorHistory)
    {
        TArray<FActorHistoryState>& History = Pair.Value;

        while (History.Num() > 0 && History[0].Timestamp < CutoffTime)
        {
            History.RemoveAt(0);
        }
    }
}

bool UMyLagCompensationManager::InterpolateState(
    const FActorHistoryState& A,
    const FActorHistoryState& B,
    float Time,
    FActorHistoryState& OutState)
{
    float Alpha = (Time - A.Timestamp) / (B.Timestamp - A.Timestamp);
    Alpha = FMath::Clamp(Alpha, 0.0f, 1.0f);

    OutState.Timestamp = Time;
    OutState.Location = FMath::Lerp(A.Location, B.Location, Alpha);
    OutState.Rotation = FMath::Lerp(A.Rotation, B.Rotation, Alpha);
    OutState.Velocity = FMath::Lerp(A.Velocity, B.Velocity, Alpha);

    return true;
}
```

---

## 三、防作弊同步验证

### 3.1 状态验证系统

```cpp
// MyAntiCheatSystem.h
#pragma once

#include "CoreMinimal.h"
#include "MyAntiCheatSystem.generated.h"

// 验证结果
UENUM()
enum class ECheatDetectionType : uint8
{
    None,
    SpeedHack,
    TeleportHack,
    Aimbot,
    WallHack,
    TimestampManipulation,
    InvalidState
};

// 验证报告
USTRUCT()
struct FCheatDetectionReport
{
    GENERATED_BODY()

    ECheatDetectionType DetectionType = ECheatDetectionType::None;
    FString PlayerId;
    FString Description;
    float Severity = 0.0f;
    FDateTime DetectionTime;
    TArray<FString> Evidence;
};

UCLASS()
class UMyAntiCheatSystem : public UObject
{
    GENERATED_BODY()

public:
    // 验证移动
    bool ValidateMovement(const FString& PlayerId, const FVector& OldLocation, const FVector& NewLocation, float DeltaTime);

    // 验证射击
    bool ValidateShot(const FString& PlayerId, const FVector& Start, const FVector& End, float ClientTime);

    // 验证状态
    bool ValidatePlayerState(const FString& PlayerId, const TMap<FString, FString>& State);

    // 记录可疑行为
    void ReportSuspiciousActivity(const FString& PlayerId, ECheatDetectionType Type, const FString& Description);

    // 获取玩家可疑记录
    TArray<FCheatDetectionReport> GetPlayerReports(const FString& PlayerId);

    // 配置
    UPROPERTY(EditDefaultsOnly, Category = "Anti-Cheat")
    float MaxSpeedMultiplier = 1.5f;

    UPROPERTY(EditDefaultsOnly, Category = "Anti-Cheat")
    float MaxTeleportDistance = 1000.0f;

    UPROPERTY(EditDefaultsOnly, Category = "Anti-Cheat")
    float AimAssistThreshold = 5.0f; // 度

    UPROPERTY(EditDefaultsOnly, Category = "Anti-Cheat")
    int32 MaxWarningsBeforeKick = 3;

private:
    TMap<FString, TArray<FCheatDetectionReport>> PlayerReports;
    TMap<FString, int32> PlayerWarningCounts;

    // 历史记录用于检测
    TMap<FString, FVector> LastKnownPositions;
    TMap<FString, float> LastKnownTimes;

    // 检测方法
    bool DetectSpeedHack(const FString& PlayerId, float Distance, float DeltaTime, float MaxAllowedSpeed);
    bool DetectTeleportHack(const FString& PlayerId, const FVector& NewLocation);
    bool DetectAimbot(const FString& PlayerId, const FVector& AimDirection, const FVector& TargetLocation);
};

// MyAntiCheatSystem.cpp
#include "MyAntiCheatSystem.h"

bool UMyAntiCheatSystem::ValidateMovement(const FString& PlayerId, const FVector& OldLocation, const FVector& NewLocation, float DeltaTime)
{
    float Distance = FVector::Dist(OldLocation, NewLocation);

    // 获取最大允许速度
    float MaxAllowedSpeed = 600.0f * MaxSpeedMultiplier; // 基础速度 * 容差

    // 检测速度作弊
    if (DetectSpeedHack(PlayerId, Distance, DeltaTime, MaxAllowedSpeed))
    {
        ReportSuspiciousActivity(PlayerId, ECheatDetectionType::SpeedHack,
            FString::Printf(TEXT("Moved %.2f units in %.3f seconds"), Distance, DeltaTime));
        return false;
    }

    // 检测传送作弊
    if (DetectTeleportHack(PlayerId, NewLocation))
    {
        ReportSuspiciousActivity(PlayerId, ECheatDetectionType::TeleportHack,
            FString::Printf(TEXT("Unexpected position change to %s"), *NewLocation.ToString()));
        return false;
    }

    // 更新记录
    LastKnownPositions.Add(PlayerId, NewLocation);
    LastKnownTimes.Add(PlayerId, GetWorld()->GetTimeSeconds());

    return true;
}

bool UMyAntiCheatSystem::ValidateShot(const FString& PlayerId, const FVector& Start, const FVector& End, float ClientTime)
{
    // 验证射击的合法性
    // 1. 检查射击位置是否合理
    // 2. 检查瞄准方向是否合理（检测Aimbot）
    // 3. 使用延迟补偿验证命中

    return true;
}

bool UMyAntiCheatSystem::DetectSpeedHack(const FString& PlayerId, float Distance, float DeltaTime, float MaxAllowedSpeed)
{
    if (DeltaTime <= 0.0f)
    {
        return false;
    }

    float ActualSpeed = Distance / DeltaTime;
    return ActualSpeed > MaxAllowedSpeed;
}

bool UMyAntiCheatSystem::DetectTeleportHack(const FString& PlayerId, const FVector& NewLocation)
{
    FVector* LastPosition = LastKnownPositions.Find(PlayerId);

    if (!LastPosition)
    {
        return false;
    }

    float Distance = FVector::Dist(*LastPosition, NewLocation);
    return Distance > MaxTeleportDistance;
}

bool UMyAntiCheatSystem::DetectAimbot(const FString& PlayerId, const FVector& AimDirection, const FVector& TargetLocation)
{
    // 检测瞄准辅助
    // 计算瞄准方向与目标方向的偏差
    // 如果偏差持续很小，可能是Aimbot

    return false;
}

void UMyAntiCheatSystem::ReportSuspiciousActivity(const FString& PlayerId, ECheatDetectionType Type, const FString& Description)
{
    FCheatDetectionReport Report;
    Report.DetectionType = Type;
    Report.PlayerId = PlayerId;
    Report.Description = Description;
    Report.DetectionTime = FDateTime::UtcNow();
    Report.Severity = 1.0f;

    PlayerReports.FindOrAdd(PlayerId).Add(Report);

    // 增加警告计数
    int32& WarningCount = PlayerWarningCounts.FindOrAdd(PlayerId);
    WarningCount++;

    UE_LOG(LogTemp, Warning, TEXT("Anti-cheat detection: %s - %s"), *PlayerId, *Description);

    // 检查是否需要踢出
    if (WarningCount >= MaxWarningsBeforeKick)
    {
        // KickPlayer(PlayerId);
    }
}

TArray<FCheatDetectionReport> UMyAntiCheatSystem::GetPlayerReports(const FString& PlayerId)
{
    TArray<FCheatDetectionReport>* Reports = PlayerReports.Find(PlayerId);
    return Reports ? *Reports : TArray<FCheatDetectionReport>();
}
```

---

## 四、网络抖动处理

### 4.1 抖动缓冲实现

```cpp
// MyJitterBuffer.h
#pragma once

#include "CoreMinimal.h"
#include "MyJitterBuffer.generated.h"

// 抖动缓冲配置
USTRUCT()
struct FJitterBufferConfig
{
    GENERATED_BODY()

    int32 BufferSize = 16;
    float TargetDelay = 0.05f;      // 目标延迟50ms
    float MinDelay = 0.02f;          // 最小延迟20ms
    float MaxDelay = 0.2f;           // 最大延迟200ms
    float AdaptationRate = 0.1f;     // 自适应速率
};

// 缓冲包
USTRUCT()
struct FJitterPacket
{
    GENERATED_BODY()

    float Timestamp;
    int32 SequenceNumber;
    TArray<uint8> Data;
};

UCLASS()
class UMyJitterBuffer : public UObject
{
    GENERATED_BODY()

public:
    UMyJitterBuffer();

    // 添加包
    void AddPacket(int32 SequenceNumber, float Timestamp, const TArray<uint8>& Data);

    // 获取下一个包
    bool GetNextPacket(TArray<uint8>& OutData);

    // 更新（每帧调用）
    void Update(float DeltaTime);

    // 配置
    void SetConfig(const FJitterBufferConfig& Config) { CurrentConfig = Config; }

    // 统计
    float GetCurrentDelay() const { return CurrentDelay; }
    int32 GetBufferCount() const { return Packets.Num(); }
    int32 GetDroppedPackets() const { return DroppedPackets; }

private:
    TArray<FJitterPacket> Packets;
    FJitterBufferConfig CurrentConfig;

    float CurrentDelay;
    int32 ExpectedSequence;
    int32 DroppedPackets;

    float LastPacketTime;
    float JitterEstimate;

    void AdaptDelay();
    void RemoveOldPackets();
    void ReorderPackets();
};

// MyJitterBuffer.cpp
#include "MyJitterBuffer.h"

UMyJitterBuffer::UMyJitterBuffer()
{
    CurrentDelay = 0.05f;
    ExpectedSequence = 0;
    DroppedPackets = 0;
    LastPacketTime = 0.0f;
    JitterEstimate = 0.0f;
}

void UMyJitterBuffer::AddPacket(int32 SequenceNumber, float Timestamp, const TArray<uint8>& Data)
{
    FJitterPacket Packet;
    Packet.SequenceNumber = SequenceNumber;
    Packet.Timestamp = Timestamp;
    Packet.Data = Data;

    // 检查是否是旧包
    if (SequenceNumber < ExpectedSequence)
    {
        return; // 丢弃旧包
    }

    Packets.Add(Packet);
    ReorderPackets();

    // 更新抖动估计
    float CurrentTime = GetWorld()->GetTimeSeconds();
    if (LastPacketTime > 0.0f)
    {
        float Interval = CurrentTime - LastPacketTime;
        JitterEstimate = JitterEstimate * 0.9f + FMath::Abs(Interval - CurrentDelay) * 0.1f;
    }
    LastPacketTime = CurrentTime;
}

bool UMyJitterBuffer::GetNextPacket(TArray<uint8>& OutData)
{
    if (Packets.Num() == 0)
    {
        return false;
    }

    // 检查是否有足够的延迟
    float CurrentTime = GetWorld()->GetTimeSeconds();

    for (int32 i = 0; i < Packets.Num(); i++)
    {
        float Age = CurrentTime - Packets[i].Timestamp;

        if (Age >= CurrentDelay)
        {
            OutData = Packets[i].Data;
            ExpectedSequence = Packets[i].SequenceNumber + 1;
            Packets.RemoveAt(i);
            return true;
        }
    }

    return false;
}

void UMyJitterBuffer::Update(float DeltaTime)
{
    AdaptDelay();
    RemoveOldPackets();
}

void UMyJitterBuffer::AdaptDelay()
{
    // 根据抖动调整延迟
    float TargetDelay = CurrentConfig.TargetDelay + JitterEstimate * 2.0f;
    TargetDelay = FMath::Clamp(TargetDelay, CurrentConfig.MinDelay, CurrentConfig.MaxDelay);

    // 平滑调整
    CurrentDelay = FMath::Lerp(CurrentDelay, TargetDelay, CurrentConfig.AdaptationRate * 0.016f);
}

void UMyJitterBuffer::RemoveOldPackets()
{
    float CurrentTime = GetWorld()->GetTimeSeconds();
    float MaxAge = CurrentConfig.MaxDelay;

    int32 RemovedCount = Packets.RemoveAll([CurrentTime, MaxAge](const FJitterPacket& Packet)
    {
        return (CurrentTime - Packet.Timestamp) > MaxAge;
    });

    DroppedPackets += RemovedCount;
}

void UMyJitterBuffer::ReorderPackets()
{
    Packets.Sort([](const FJitterPacket& A, const FJitterPacket& B)
    {
        return A.SequenceNumber < B.SequenceNumber;
    });
}
```

---

## 五、总结

本课我们学习了：

1. **时间同步**：时钟偏移计算和补偿
2. **延迟补偿**：历史状态回滚技术
3. **防作弊**：状态验证和异常检测
4. **抖动处理**：缓冲和自适应延迟

---

## 课程总结

恭喜你完成了UE5后端高级开发学习路线的第一至第四阶段！

**已完成的课程**：
- 第一阶段：网络架构基础 (第1-5课)
- 第二阶段：游戏服务器架构 (第6-10课)
- 第三阶段：数据持久化 (第11-15课)
- 第四阶段：高级网络特性 (第16-20课)

**后续课程预告**：
- 第五阶段：云服务与微服务 (第21-25课)
- 第六阶段：性能优化与安全 (第26-30课)
- 第七阶段：实战项目 (第31-40课)

---

*课程版本：1.0*
*最后更新：2026-04-10*