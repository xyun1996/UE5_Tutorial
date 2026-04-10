# 第9课：服务器架构模式

> 本课深入讲解UE5游戏服务器的各种架构模式，包括单一服务器、分区服务器、匹配服务器和集群架构设计。

---

## 课程目标

- 理解不同服务器架构模式的优缺点
- 掌握分区服务器（Zoning）设计
- 学会设计匹配服务器
- 了解大厅服务器架构
- 掌握服务器集群设计原则

---

## 一、服务器架构模式概述

### 1.1 架构模式对比

```
┌─────────────────────────────────────────────────────────────┐
│                    服务器架构模式对比                        │
│                                                              │
│  单一服务器                                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  ┌─────────────────────────────────────────────┐   │    │
│  │  │           单一游戏服务器                     │   │    │
│  │  │  - 所有游戏逻辑                             │   │    │
│  │  │  - 所有玩家                                 │   │    │
│  │  │  - 整个世界                                 │   │    │
│  │  └─────────────────────────────────────────────┘   │    │
│  │  优点：简单、延迟低                              │    │
│  │  缺点：扩展性差、单点故障                        │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  分区服务器                                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐            │    │
│  │  │ Zone A  │  │ Zone B  │  │ Zone C  │            │    │
│  │  │ 区域A   │  │ 区域B   │  │ 区域C   │            │    │
│  │  └─────────┘  └─────────┘  └─────────┘            │    │
│  │  优点：可扩展、分区管理                          │    │
│  │  缺点：跨区复杂、需同步                          │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  匹配+游戏服务器                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  ┌─────────────┐         ┌─────────────────────┐   │    │
│  │  │ 匹配服务器   │ ──────► │ 游戏服务器池        │   │    │
│  │  │ Matchmaker  │         │ Game Server Pool    │   │    │
│  │  └─────────────┘         └─────────────────────┘   │    │
│  │  优点：灵活、可扩展                              │    │
│  │  缺点：架构复杂                                  │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 选择指南

| 场景 | 推荐架构 | 原因 |
|------|---------|------|
| 小型游戏（<100人） | 单一服务器 | 简单够用 |
| 中型游戏（100-1000人） | 匹配+游戏服务器 | 按需分配 |
| 大型MMO | 分区服务器 | 分散负载 |
| 竞技游戏 | 匹配+游戏服务器 | 快速匹配 |
| 开放世界 | 分区/无缝分区 | 世界分割 |

---

## 二、单一服务器架构

### 2.1 架构设计

```cpp
// 单一服务器配置
UCLASS(Config = Game)
class USingleServerConfig : public UDeveloperSettings
{
    GENERATED_BODY()

    // 服务器限制
    UPROPERTY(Config, EditAnywhere, Category = "Capacity")
    int32 MaxPlayers = 64;

    UPROPERTY(Config, EditAnywhere, Category = "Capacity")
    int32 MaxTickRate = 30;

    // 资源限制
    UPROPERTY(Config, EditAnywhere, Category = "Resources")
    float MaxCPUUsage = 0.8f;

    UPROPERTY(Config, EditAnywhere, Category = "Resources")
    int64 MaxMemoryMB = 2048;
};

// 单一服务器GameMode
UCLASS()
class ASingleServerGameMode : public AGameModeBase
{
    GENERATED_BODY()

public:
    virtual void InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage) override;
    virtual void Tick(float DeltaSeconds) override;

    // 资源监控
    UFUNCTION(BlueprintPure, Category = "Server")
    float GetCPUUsage() const;

    UFUNCTION(BlueprintPure, Category = "Server")
    int64 GetMemoryUsage() const;

    UFUNCTION(BlueprintPure, Category = "Server")
    bool IsServerHealthy() const;

protected:
    void MonitorServerHealth();
    void HandleOverload();

private:
    float CurrentCPUUsage;
    int64 CurrentMemoryUsage;
    bool bIsOverloaded;
};

void ASingleServerGameMode::InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage)
{
    Super::InitGame(MapName, Options, ErrorMessage);

    // 设置服务器参数
    GetWorld()->GetTimerManager().SetTimer(
        HealthMonitorTimer,
        this,
        &ASingleServerGameMode::MonitorServerHealth,
        5.0f,
        true
    );
}

void ASingleServerGameMode::Tick(float DeltaSeconds)
{
    Super::Tick(DeltaSeconds);

    // 检查过载
    if (bIsOverloaded)
    {
        HandleOverload();
    }
}

void ASingleServerGameMode::MonitorServerHealth()
{
    // 获取资源使用情况
    CurrentCPUUsage = GetCPUUsage();
    CurrentMemoryUsage = GetMemoryUsage();

    // 检查是否过载
    USingleServerConfig* Config = GetMutableDefault<USingleServerConfig>();
    bIsOverloaded = (CurrentCPUUsage > Config->MaxCPUUsage ||
                     CurrentMemoryUsage > Config->MaxMemoryMB * 1024 * 1024);

    UE_LOG(LogTemp, Log, TEXT("Server Health - CPU: %.2f%%, Memory: %lld MB, Overloaded: %s"),
        CurrentCPUUsage * 100.0f,
        CurrentMemoryUsage / (1024 * 1024),
        bIsOverloaded ? TEXT("Yes") : TEXT("No"));
}

void ASingleServerGameMode::HandleOverload()
{
    // 过载处理策略
    // 1. 拒绝新连接
    // 2. 降低更新频率
    // 3. 踢出空闲玩家
}
```

---

## 三、分区服务器架构

### 3.1 分区设计

```
┌─────────────────────────────────────────────────────────────┐
│                      分区服务器架构                          │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    协调服务器                        │    │
│  │  - 管理分区分配                                     │    │
│  │  - 处理跨区请求                                     │    │
│  │  - 玩家路由                                         │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│          ┌───────────────┼───────────────┐                  │
│          ▼               ▼               ▼                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   Zone A    │  │   Zone B    │  │   Zone C    │         │
│  │  (0,0)-(100 │  │ (100,0)-(200│  │ (200,0)-(300│         │
│  │   ,100)     │  │  ,100)      │  │  ,100)      │         │
│  │  城镇区域   │  │  森林区域   │  │  沙漠区域   │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│          │               │               │                  │
│          └───────────────┴───────────────┘                  │
│                          │                                   │
│                    边界同步区                                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 分区管理实现

```cpp
// MyZoneManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "MyZoneManager.generated.h"

// 分区定义
USTRUCT(BlueprintType)
struct FZoneDefinition
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString ZoneId;

    UPROPERTY(BlueprintReadOnly)
    FString ZoneName;

    UPROPERTY(BlueprintReadOnly)
    FVector MinBounds;

    UPROPERTY(BlueprintReadOnly)
    FVector MaxBounds;

    UPROPERTY(BlueprintReadOnly)
    FString ServerAddress;

    UPROPERTY(BlueprintReadOnly)
    int32 MaxPlayers;

    UPROPERTY(BlueprintReadOnly)
    int32 CurrentPlayers;

    bool ContainsPosition(const FVector& Position) const
    {
        return Position.X >= MinBounds.X && Position.X <= MaxBounds.X &&
               Position.Y >= MinBounds.Y && Position.Y <= MaxBounds.Y &&
               Position.Z >= MinBounds.Z && Position.Z <= MaxBounds.Z;
    }
};

// 玩家跨区数据
USTRUCT()
struct FPlayerZoneData
{
    GENERATED_BODY()

    FString PlayerId;
    FString CurrentZoneId;
    FVector LastKnownPosition;
    float LastUpdateTime;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnPlayerZoneChanged, const FString&, PlayerId, const FString&, NewZoneId);

UCLASS()
class UMyZoneManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 分区查询
    UFUNCTION(BlueprintPure, Category = "Zone")
    FZoneDefinition GetZoneForPosition(const FVector& Position) const;

    UFUNCTION(BlueprintPure, Category = "Zone")
    FZoneDefinition GetCurrentZone() const;

    UFUNCTION(BlueprintPure, Category = "Zone")
    TArray<FZoneDefinition> GetAllZones() const;

    // 玩家分区管理
    UFUNCTION(BlueprintCallable, Category = "Zone")
    void UpdatePlayerPosition(const FString& PlayerId, const FVector& Position);

    UFUNCTION(BlueprintPure, Category = "Zone")
    FString GetPlayerZone(const FString& PlayerId) const;

    // 分区迁移
    UFUNCTION(BlueprintCallable, Category = "Zone")
    void MigratePlayerToZone(const FString& PlayerId, const FString& TargetZoneId);

    // 委托
    UPROPERTY(BlueprintAssignable)
    FOnPlayerZoneChanged OnPlayerZoneChanged;

private:
    UPROPERTY()
    TArray<FZoneDefinition> ZoneDefinitions;

    TMap<FString, FPlayerZoneData> PlayerZoneMap;

    FString CurrentZoneId;

    void LoadZoneDefinitions();
    void HandleZoneTransition(const FString& PlayerId, const FString& OldZone, const FString& NewZone);
};

// MyZoneManager.cpp
#include "MyZoneManager.h"

void UMyZoneManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    LoadZoneDefinitions();
}

void UMyZoneManager::LoadZoneDefinitions()
{
    // 从配置加载分区定义
    ZoneDefinitions.Empty();

    // 示例分区
    FZoneDefinition ZoneA;
    ZoneA.ZoneId = TEXT("Zone_A");
    ZoneA.ZoneName = TEXT("Town");
    ZoneA.MinBounds = FVector(0, 0, 0);
    ZoneA.MaxBounds = FVector(10000, 10000, 1000);
    ZoneA.ServerAddress = TEXT("192.168.1.101:7777");
    ZoneA.MaxPlayers = 64;
    ZoneDefinitions.Add(ZoneA);

    FZoneDefinition ZoneB;
    ZoneB.ZoneId = TEXT("Zone_B");
    ZoneB.ZoneName = TEXT("Forest");
    ZoneB.MinBounds = FVector(10000, 0, 0);
    ZoneB.MaxBounds = FVector(20000, 10000, 1000);
    ZoneB.ServerAddress = TEXT("192.168.1.102:7777");
    ZoneB.MaxPlayers = 64;
    ZoneDefinitions.Add(ZoneB);

    FZoneDefinition ZoneC;
    ZoneC.ZoneId = TEXT("Zone_C");
    ZoneC.ZoneName = TEXT("Desert");
    ZoneC.MinBounds = FVector(20000, 0, 0);
    ZoneC.MaxBounds = FVector(30000, 10000, 1000);
    ZoneC.ServerAddress = TEXT("192.168.1.103:7777");
    ZoneC.MaxPlayers = 64;
    ZoneDefinitions.Add(ZoneC);
}

FZoneDefinition UMyZoneManager::GetZoneForPosition(const FVector& Position) const
{
    for (const FZoneDefinition& Zone : ZoneDefinitions)
    {
        if (Zone.ContainsPosition(Position))
        {
            return Zone;
        }
    }

    // 返回默认分区
    return ZoneDefinitions[0];
}

FZoneDefinition UMyZoneManager::GetCurrentZone() const
{
    for (const FZoneDefinition& Zone : ZoneDefinitions)
    {
        if (Zone.ZoneId == CurrentZoneId)
        {
            return Zone;
        }
    }
    return FZoneDefinition();
}

TArray<FZoneDefinition> UMyZoneManager::GetAllZones() const
{
    return ZoneDefinitions;
}

void UMyZoneManager::UpdatePlayerPosition(const FString& PlayerId, const FVector& Position)
{
    FZoneDefinition NewZone = GetZoneForPosition(Position);

    if (PlayerZoneMap.Contains(PlayerId))
    {
        FString OldZoneId = PlayerZoneMap[PlayerId].CurrentZoneId;

        if (OldZoneId != NewZone.ZoneId)
        {
            HandleZoneTransition(PlayerId, OldZoneId, NewZone.ZoneId);
        }
    }

    PlayerZoneMap.Add(PlayerId, {PlayerId, NewZone.ZoneId, Position, GetWorld()->GetTimeSeconds()});
}

FString UMyZoneManager::GetPlayerZone(const FString& PlayerId) const
{
    if (const FPlayerZoneData* Data = PlayerZoneMap.Find(PlayerId))
    {
        return Data->CurrentZoneId;
    }
    return TEXT("");
}

void UMyZoneManager::MigratePlayerToZone(const FString& PlayerId, const FString& TargetZoneId)
{
    // 执行玩家迁移到新分区
    for (const FZoneDefinition& Zone : ZoneDefinitions)
    {
        if (Zone.ZoneId == TargetZoneId)
        {
            // 连接到目标分区服务器
            APlayerController* PC = GetWorld()->GetFirstPlayerController();
            if (PC)
            {
                PC->ClientTravel(Zone.ServerAddress, ETravelType::TRAVEL_Absolute);
            }
            break;
        }
    }
}

void UMyZoneManager::HandleZoneTransition(const FString& PlayerId, const FString& OldZone, const FString& NewZone)
{
    UE_LOG(LogTemp, Log, TEXT("Player %s transitioning from %s to %s"), *PlayerId, *OldZone, *NewZone);

    OnPlayerZoneChanged.Broadcast(PlayerId, NewZone);

    // 保存玩家状态
    // SavePlayerState(PlayerId);

    // 迁移到新分区
    MigratePlayerToZone(PlayerId, NewZone);
}
```

### 3.3 边界同步

```cpp
// 分区边界同步处理
UCLASS()
class UZoneBoundaryManager : public UObject
{
    GENERATED_BODY()

public:
    // 边界缓冲区大小
    UPROPERTY(EditAnywhere, Category = "Zone")
    float BoundaryBufferSize = 1000.0f;

    // 检查是否在边界区域
    bool IsInBoundaryZone(const FVector& Position, const FZoneDefinition& Zone) const
    {
        FVector ZoneSize = Zone.MaxBounds - Zone.MinBounds;

        // 检查是否靠近边界
        bool bNearXMin = Position.X - Zone.MinBounds.X < BoundaryBufferSize;
        bool bNearXMax = Zone.MaxBounds.X - Position.X < BoundaryBufferSize;
        bool bNearYMin = Position.Y - Zone.MinBounds.Y < BoundaryBufferSize;
        bool bNearYMax = Zone.MaxBounds.Y - Position.Y < BoundaryBufferSize;

        return bNearXMin || bNearXMax || bNearYMin || bNearYMax;
    }

    // 获取相邻分区
    TArray<FString> GetAdjacentZones(const FVector& Position, const FZoneDefinition& CurrentZone) const
    {
        TArray<FString> AdjacentZones;

        // 根据位置确定相邻分区
        // 实际实现需要根据分区配置

        return AdjacentZones;
    }

    // 同步边界区域实体
    void SyncBoundaryEntities(const FString& TargetZoneId, const TArray<AActor*>& Entities)
    {
        // 将边界区域的实体信息同步到相邻分区
        // 用于实现无缝过渡
    }
};
```

---

## 四、匹配服务器设计

### 4.1 匹配服务器架构

```
┌─────────────────────────────────────────────────────────────┐
│                      匹配服务器架构                          │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   匹配服务器                         │    │
│  │  ┌─────────────────────────────────────────────┐   │    │
│  │  │              匹配队列                        │   │    │
│  │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐    │   │    │
│  │  │  │Ranked   │  │Casual   │  │Custom   │    │   │    │
│  │  │  │Queue    │  │Queue    │  │Queue    │    │   │    │
│  │  │  └─────────┘  └─────────┘  └─────────┘    │   │    │
│  │  └─────────────────────────────────────────────┘   │    │
│  │  ┌─────────────────────────────────────────────┐   │    │
│  │  │              匹配逻辑                        │   │    │
│  │  │  - 按技能分匹配                              │   │    │
│  │  │  - 按地区匹配                                │   │    │
│  │  │  - 按模式匹配                                │   │    │
│  │  └─────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│                          ▼                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                 游戏服务器池                        │    │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐            │    │
│  │  │Server 1 │  │Server 2 │  │Server N │            │    │
│  │  │Port 7777│  │Port 7778│  │Port XXXX│            │    │
│  │  └─────────┘  └─────────┘  └─────────┘            │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 匹配系统实现

```cpp
// MyMatchmakingSystem.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "MyMatchmakingSystem.generated.h"

// 玩家匹配数据
USTRUCT()
struct FMatchmakingPlayer
{
    GENERATED_BODY()

    FString PlayerId;
    FString PlayerName;
    float SkillRating;
    FString Region;
    FString PreferredMode;
    float QueueTime;
    FString TicketId;
};

// 匹配配置
USTRUCT()
struct FMatchConfig
{
    GENERATED_BODY()

    FString MatchType;
    int32 TeamSize = 5;
    int32 TotalPlayers = 10;
    float SkillRange = 100.0f;
    float MaxWaitTime = 300.0f;
    TArray<FString> RequiredRegions;
};

// 匹配结果
USTRUCT(BlueprintType)
struct FMatchResult
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString MatchId;

    UPROPERTY(BlueprintReadOnly)
    FString ServerAddress;

    UPROPERTY(BlueprintReadOnly)
    FString SessionToken;

    UPROPERTY(BlueprintReadOnly)
    TArray<FString> TeamA;

    UPROPERTY(BlueprintReadOnly)
    TArray<FString> TeamB;
};

// 游戏服务器状态
USTRUCT()
struct FGameServerStatus
{
    GENERATED_BODY()

    FString ServerId;
    FString Address;
    int32 Port;
    int32 CurrentPlayers;
    int32 MaxPlayers;
    FString MapName;
    FString GameMode;
    bool bIsAvailable;
    float LastHeartbeat;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnMatchFound, const FMatchResult&, Result);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnQueueStatusUpdate, int32, PlayersInQueue);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnMatchmakingFailed, const FString&, Reason);

UCLASS()
class UMyMatchmakingSystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 匹配操作
    UFUNCTION(BlueprintCallable, Category = "Matchmaking")
    void JoinQueue(const FString& MatchType, const FString& Region);

    UFUNCTION(BlueprintCallable, Category = "Matchmaking")
    void LeaveQueue();

    UFUNCTION(BlueprintPure, Category = "Matchmaking")
    bool IsInQueue() const;

    UFUNCTION(BlueprintPure, Category = "Matchmaking")
    float GetQueueTime() const;

    // 服务器管理
    void RegisterGameServer(const FGameServerStatus& Server);
    void UnregisterGameServer(const FString& ServerId);
    void UpdateServerHeartbeat(const FString& ServerId);

    // 委托
    UPROPERTY(BlueprintAssignable)
    FOnMatchFound OnMatchFound;

    UPROPERTY(BlueprintAssignable)
    FOnQueueStatusUpdate OnQueueStatusUpdate;

    UPROPERTY(BlueprintAssignable)
    FOnMatchmakingFailed OnMatchmakingFailed;

private:
    // 匹配队列
    TArray<FMatchmakingPlayer> PlayerQueue;
    TMap<FString, FMatchConfig> MatchConfigs;
    TArray<FGameServerStatus> AvailableServers;

    // 当前状态
    FMatchmakingPlayer CurrentPlayer;
    bool bIsInQueue;
    float QueueStartTime;
    FString CurrentTicketId;

    // 匹配循环
    FTimerHandle MatchmakingTimerHandle;

    void ProcessMatchmaking();
    bool TryCreateMatch(const FMatchConfig& Config);
    TArray<FMatchmakingPlayer> FindMatchCandidates(const FMatchmakingPlayer& Reference, const FMatchConfig& Config);
    FGameServerStatus FindAvailableServer(const FString& Region);
    FMatchResult CreateMatchOnServer(const TArray<FMatchmakingPlayer>& Players, FGameServerStatus& Server);
    void RemovePlayersFromQueue(const TArray<FMatchmakingPlayer>& Players);

    // 辅助
    float CalculateMatchQuality(const FMatchmakingPlayer& A, const FMatchmakingPlayer& B, const FMatchConfig& Config);
};

// MyMatchmakingSystem.cpp
#include "MyMatchmakingSystem.h"

void UMyMatchmakingSystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    bIsInQueue = false;

    // 初始化匹配配置
    FMatchConfig RankedConfig;
    RankedConfig.MatchType = TEXT("Ranked");
    RankedConfig.TeamSize = 5;
    RankedConfig.TotalPlayers = 10;
    RankedConfig.SkillRange = 100.0f;
    MatchConfigs.Add(TEXT("Ranked"), RankedConfig);

    FMatchConfig CasualConfig;
    CasualConfig.MatchType = TEXT("Casual");
    CasualConfig.TeamSize = 5;
    CasualConfig.TotalPlayers = 10;
    CasualConfig.SkillRange = 500.0f;
    MatchConfigs.Add(TEXT("Casual"), CasualConfig);

    // 启动匹配循环
    GetWorld()->GetTimerManager().SetTimer(
        MatchmakingTimerHandle,
        this,
        &UMyMatchmakingSystem::ProcessMatchmaking,
        1.0f,
        true
    );
}

void UMyMatchmakingSystem::Deinitialize()
{
    GetWorld()->GetTimerManager().ClearTimer(MatchmakingTimerHandle);
    Super::Deinitialize();
}

void UMyMatchmakingSystem::JoinQueue(const FString& MatchType, const FString& Region)
{
    if (bIsInQueue)
    {
        return;
    }

    // 创建匹配票据
    CurrentTicketId = FGuid::NewGuid().ToString();
    QueueStartTime = GetWorld()->GetTimeSeconds();

    CurrentPlayer.PlayerId = TEXT("LocalPlayer");
    CurrentPlayer.Region = Region;
    CurrentPlayer.PreferredMode = MatchType;
    CurrentPlayer.QueueTime = QueueStartTime;
    CurrentPlayer.TicketId = CurrentTicketId;
    CurrentPlayer.SkillRating = 1500.0f; // 从玩家数据获取

    PlayerQueue.Add(CurrentPlayer);
    bIsInQueue = true;

    UE_LOG(LogTemp, Log, TEXT("Player joined %s queue"), *MatchType);
}

void UMyMatchmakingSystem::LeaveQueue()
{
    if (!bIsInQueue)
    {
        return;
    }

    PlayerQueue.RemoveAll([this](const FMatchmakingPlayer& Player)
    {
        return Player.TicketId == CurrentTicketId;
    });

    bIsInQueue = false;
    CurrentTicketId.Empty();

    UE_LOG(LogTemp, Log, TEXT("Player left queue"));
}

bool UMyMatchmakingSystem::IsInQueue() const
{
    return bIsInQueue;
}

float UMyMatchmakingSystem::GetQueueTime() const
{
    if (bIsInQueue)
    {
        return GetWorld()->GetTimeSeconds() - QueueStartTime;
    }
    return 0.0f;
}

void UMyMatchmakingSystem::ProcessMatchmaking()
{
    // 更新队列状态
    OnQueueStatusUpdate.Broadcast(PlayerQueue.Num());

    // 检查当前玩家是否在队列中
    if (!bIsInQueue)
    {
        return;
    }

    // 获取匹配配置
    FMatchConfig* Config = MatchConfigs.Find(CurrentPlayer.PreferredMode);
    if (!Config)
    {
        OnMatchmakingFailed.Broadcast(TEXT("Invalid match type"));
        LeaveQueue();
        return;
    }

    // 尝试创建匹配
    if (TryCreateMatch(*Config))
    {
        // 匹配成功，清理队列
        LeaveQueue();
    }
}

bool UMyMatchmakingSystem::TryCreateMatch(const FMatchConfig& Config)
{
    // 找到匹配候选
    TArray<FMatchmakingPlayer> Candidates = FindMatchCandidates(CurrentPlayer, Config);

    if (Candidates.Num() >= Config.TotalPlayers)
    {
        // 找到可用服务器
        FGameServerStatus Server = FindAvailableServer(CurrentPlayer.Region);
        if (Server.bIsAvailable)
        {
            // 创建匹配
            FMatchResult Result = CreateMatchOnServer(Candidates, Server);

            // 通知所有玩家
            OnMatchFound.Broadcast(Result);

            // 从队列移除
            RemovePlayersFromQueue(Candidates);

            return true;
        }
    }

    return false;
}

TArray<FMatchmakingPlayer> UMyMatchmakingSystem::FindMatchCandidates(const FMatchmakingPlayer& Reference, const FMatchConfig& Config)
{
    TArray<FMatchmakingPlayer> Candidates;
    Candidates.Add(Reference);

    float CurrentTime = GetWorld()->GetTimeSeconds();

    for (const FMatchmakingPlayer& Player : PlayerQueue)
    {
        if (Player.TicketId == Reference.TicketId)
        {
            continue;
        }

        // 检查地区
        if (Player.Region != Reference.Region)
        {
            continue;
        }

        // 检查技能范围（随时间扩大）
        float WaitTime = CurrentTime - Player.QueueTime;
        float ExpandedSkillRange = Config.SkillRange * (1.0f + WaitTime / 60.0f);

        if (FMath::Abs(Player.SkillRating - Reference.SkillRating) > ExpandedSkillRange)
        {
            continue;
        }

        Candidates.Add(Player);

        if (Candidates.Num() >= Config.TotalPlayers)
        {
            break;
        }
    }

    return Candidates;
}

FGameServerStatus UMyMatchmakingSystem::FindAvailableServer(const FString& Region)
{
    for (FGameServerStatus& Server : AvailableServers)
    {
        if (Server.bIsAvailable && Server.CurrentPlayers < Server.MaxPlayers)
        {
            return Server;
        }
    }

    return FGameServerStatus();
}

FMatchResult UMyMatchmakingSystem::CreateMatchOnServer(const TArray<FMatchmakingPlayer>& Players, FGameServerStatus& Server)
{
    FMatchResult Result;
    Result.MatchId = FGuid::NewGuid().ToString();
    Result.ServerAddress = FString::Printf(TEXT("%s:%d"), *Server.Address, Server.Port);
    Result.SessionToken = FGuid::NewGuid().ToString();

    // 分配队伍
    int32 Index = 0;
    for (const FMatchmakingPlayer& Player : Players)
    {
        if (Index < Players.Num() / 2)
        {
            Result.TeamA.Add(Player.PlayerId);
        }
        else
        {
            Result.TeamB.Add(Player.PlayerId);
        }
        Index++;
    }

    return Result;
}

void UMyMatchmakingSystem::RemovePlayersFromQueue(const TArray<FMatchmakingPlayer>& Players)
{
    for (const FMatchmakingPlayer& Player : Players)
    {
        PlayerQueue.RemoveAll([&Player](const FMatchmakingPlayer& QueuePlayer)
        {
            return QueuePlayer.TicketId == Player.TicketId;
        });
    }
}

void UMyMatchmakingSystem::RegisterGameServer(const FGameServerStatus& Server)
{
    AvailableServers.Add(Server);
}

void UMyMatchmakingSystem::UnregisterGameServer(const FString& ServerId)
{
    AvailableServers.RemoveAll([&ServerId](const FGameServerStatus& Server)
    {
        return Server.ServerId == ServerId;
    });
}

void UMyMatchmakingSystem::UpdateServerHeartbeat(const FString& ServerId)
{
    for (FGameServerStatus& Server : AvailableServers)
    {
        if (Server.ServerId == ServerId)
        {
            Server.LastHeartbeat = GetWorld()->GetTimeSeconds();
            break;
        }
    }
}
```

---

## 五、服务器集群架构

### 5.1 集群设计

```cpp
// MyClusterManager.h
#pragma once

#include "CoreMinimal.h"
#include "MyClusterManager.generated.h"

// 集群节点状态
USTRUCT()
struct FClusterNode
{
    GENERATED_BODY()

    FString NodeId;
    FString NodeType; // "Gateway", "Game", "Database", etc.
    FString Address;
    int32 Port;
    float CPULoad;
    float MemoryLoad;
    int32 ActiveConnections;
    float LastHeartbeat;
    bool bIsHealthy;
};

// 集群配置
UCLASS(Config = Game)
class UClusterConfig : public UDeveloperSettings
{
    GENERATED_BODY()

    UPROPERTY(Config, EditAnywhere, Category = "Cluster")
    FString CoordinatorAddress;

    UPROPERTY(Config, EditAnywhere, Category = "Cluster")
    int32 CoordinatorPort = 9000;

    UPROPERTY(Config, EditAnywhere, Category = "Cluster")
    float HeartbeatInterval = 5.0f;

    UPROPERTY(Config, EditAnywhere, Category = "Cluster")
    float NodeTimeout = 30.0f;

    UPROPERTY(Config, EditAnywhere, Category = "Cluster")
    int32 MaxNodesPerType = 10;
};

UCLASS()
class UMyClusterManager : public UObject
{
    GENERATED_BODY()

public:
    // 节点管理
    void RegisterNode(const FClusterNode& Node);
    void UnregisterNode(const FString& NodeId);
    void UpdateNodeHeartbeat(const FString& NodeId, const FClusterNode& Update);

    // 节点查询
    FClusterNode GetBestNode(const FString& NodeType);
    TArray<FClusterNode> GetAllNodes();
    TArray<FClusterNode> GetNodesByType(const FString& NodeType);

    // 健康检查
    void CheckNodeHealth();
    void RemoveUnhealthyNodes();

    // 负载均衡
    FString SelectNodeForPlayer(const FString& PlayerId, const FString& NodeType);

private:
    TMap<FString, FClusterNode> Nodes;
    FClusterConfig* Config;

    float CalculateNodeScore(const FClusterNode& Node);
};
```

---

## 六、总结

本课我们学习了：

1. **架构模式**：单一服务器、分区服务器、匹配服务器
2. **分区设计**：Zoning实现和边界同步
3. **匹配系统**：队列管理和匹配逻辑
4. **集群架构**：节点管理和负载均衡

---

## 七、下节预告

**第10课：高级服务器功能**

将深入学习：
- 服务器执行命令
- 远程管理接口
- 服务器日志与监控
- 热重载与热修复

---

*课程版本：1.0*
*最后更新：2026-04-10*