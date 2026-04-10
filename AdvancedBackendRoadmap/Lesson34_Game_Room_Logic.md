# 第34课：游戏房间逻辑

> 学习目标：实现游戏房间创建、管理和完整游戏流程控制。

---

## 34.1 房间系统架构

### 34.1.1 房间生命周期

```
房间状态流程:

[创建] -> [等待] -> [准备] -> [加载] -> [游戏中] -> [结算] -> [销毁]
           │           │         │          │          │
           ▼           ▼         ▼          ▼          ▼
        玩家加入    全员准备   关卡加载   游戏进行    结果统计
        等待超时    有玩家离开 同步状态   游戏结束    返回大厅
```

---

## 34.2 游戏房间管理器

### 34.2.1 房间管理子系统

```csharp
// GameRoomSubsystem.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "GameRoomSubsystem.generated.h"

UENUM(BlueprintType)
enum class ERoomState : uint8
{
    None,
    Creating,
    Waiting,
    Ready,
    Loading,
    Playing,
    Finished,
    Closing
};

UENUM(BlueprintType)
enum class ERoomType : uint8
{
    Casual,
    Ranked,
    Custom,
    Tutorial
};

USTRUCT(BlueprintType)
struct FRoomPlayer
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString PlayerId;

    UPROPERTY(BlueprintReadOnly)
    FString DisplayName;

    UPROPERTY(BlueprintReadOnly)
    int32 TeamId = 0;

    UPROPERTY(BlueprintReadOnly)
    bool bIsReady = false;

    UPROPERTY(BlueprintReadOnly)
    bool bIsHost = false;

    UPROPERTY(BlueprintReadOnly)
    int32 SlotIndex = -1;
};

USTRUCT(BlueprintType)
struct FRoomSettings
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadWrite)
    FString MapName;

    UPROPERTY(BlueprintReadWrite)
    FString GameMode;

    UPROPERTY(BlueprintReadWrite)
    int32 MaxPlayers = 8;

    UPROPERTY(BlueprintReadWrite)
    bool bIsPrivate = false;

    UPROPERTY(BlueprintReadWrite)
    FString Password;

    UPROPERTY(BlueprintReadWrite)
    int32 RoundTimeMinutes = 10;

    UPROPERTY(BlueprintReadWrite)
    int32 ScoreToWin = 25;
};

USTRUCT(BlueprintType)
struct FRoomInfo
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString RoomId;

    UPROPERTY(BlueprintReadOnly)
    FString RoomName;

    UPROPERTY(BlueprintReadOnly)
    ERoomType RoomType = ERoomType::Casual;

    UPROPERTY(BlueprintReadOnly)
    ERoomState RoomState = ERoomState::None;

    UPROPERTY(BlueprintReadOnly)
    FString HostPlayerId;

    UPROPERTY(BlueprintReadOnly)
    TArray<FRoomPlayer> Players;

    UPROPERTY(BlueprintReadOnly)
    FRoomSettings Settings;

    UPROPERTY(BlueprintReadOnly)
    FDateTime CreatedAt;

    UPROPERTY(BlueprintReadOnly)
    FDateTime GameStartTime;
};

USTRUCT(BlueprintType)
struct FGameResult
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString MatchId;

    UPROPERTY(BlueprintReadOnly)
    int32 WinningTeam = 0;

    UPROPERTY(BlueprintReadOnly)
    TArray<FString> WinningPlayers;

    UPROPERTY(BlueprintReadOnly)
    int32 MatchDurationSeconds = 0;

    UPROPERTY(BlueprintReadOnly)
    TArray<FPlayerMatchStats> PlayerStats;
};

USTRUCT(BlueprintType)
struct FPlayerMatchStats
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString PlayerId;

    UPROPERTY(BlueprintReadOnly)
    int32 Kills = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 Deaths = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 Assists = 0;

    UPROPERTY(BlueprintReadOnly)
    float DamageDealt = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    float DamageTaken = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    int32 Score = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 TeamId = 0;

    UPROPERTY(BlueprintReadOnly)
    bool bIsWinner = false;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnRoomCreated, const FRoomInfo&, Room);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnRoomJoined, const FRoomInfo&, Room);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnRoomLeft, const FString&, RoomId);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnRoomUpdated, const FRoomInfo&, Room);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnRoomStateChanged, ERoomState, NewState);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnGameResult, const FGameResult&, Result);

UCLASS()
class GAMEROOM_API UGameRoomSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 房间管理
    UFUNCTION(BlueprintCallable, Category = "GameRoom")
    void CreateRoom(const FRoomSettings& Settings, ERoomType RoomType = ERoomType::Casual);

    UFUNCTION(BlueprintCallable, Category = "GameRoom")
    void JoinRoom(const FString& RoomId, const FString& Password = TEXT(""));

    UFUNCTION(BlueprintCallable, Category = "GameRoom")
    void LeaveRoom();

    UFUNCTION(BlueprintCallable, Category = "GameRoom")
    void KickPlayer(const FString& PlayerId);

    // 房间状态
    UFUNCTION(BlueprintPure, Category = "GameRoom")
    bool IsInRoom() const { return bIsInRoom; }

    UFUNCTION(BlueprintPure, Category = "GameRoom")
    FRoomInfo GetCurrentRoom() const { return CurrentRoom; }

    UFUNCTION(BlueprintPure, Category = "GameRoom")
    ERoomState GetRoomState() const { return CurrentRoom.RoomState; }

    // 玩家操作
    UFUNCTION(BlueprintCallable, Category = "GameRoom")
    void SetReady(bool bReady);

    UFUNCTION(BlueprintCallable, Category = "GameRoom")
    void ChangeTeam(int32 TeamId);

    UFUNCTION(BlueprintCallable, Category = "GameRoom")
    void ChangeSlot(int32 SlotIndex);

    // 游戏流程
    UFUNCTION(BlueprintCallable, Category = "GameRoom")
    void StartGame();

    UFUNCTION(BlueprintCallable, Category = "GameRoom")
    void EndGame();

    UFUNCTION(BlueprintCallable, Category = "GameRoom")
    void ReportGameResult(const FGameResult& Result);

    // 房间设置
    UFUNCTION(BlueprintCallable, Category = "GameRoom")
    void UpdateRoomSettings(const FRoomSettings& NewSettings);

    // 事件
    UPROPERTY(BlueprintAssignable, Category = "GameRoom")
    FOnRoomCreated OnRoomCreated;

    UPROPERTY(BlueprintAssignable, Category = "GameRoom")
    FOnRoomJoined OnRoomJoined;

    UPROPERTY(BlueprintAssignable, Category = "GameRoom")
    FOnRoomLeft OnRoomLeft;

    UPROPERTY(BlueprintAssignable, Category = "GameRoom")
    FOnRoomUpdated OnRoomUpdated;

    UPROPERTY(BlueprintAssignable, Category = "GameRoom")
    FOnRoomStateChanged OnRoomStateChanged;

    UPROPERTY(BlueprintAssignable, Category = "GameRoom")
    FOnGameResult OnGameResult;

private:
    bool bIsInRoom = false;
    FRoomInfo CurrentRoom;
    FString LocalPlayerId;
    FString ApiBaseUrl;

    // HTTP请求
    void MakeRoomRequest(const FString& Endpoint, const FString& Method, const FString& Body);
    void HandleCreateRoomResponse(const FString& ResponseJson);
    void HandleJoinRoomResponse(const FString& ResponseJson);
    void HandleRoomUpdate(const FString& ResponseJson);
    void HandleGameResult(const FString& ResponseJson);

    // WebSocket更新
    void SubscribeToRoomUpdates();
    void HandleRoomUpdateMessage(const FString& Message);
};
```

---

## 34.3 服务器端房间逻辑

### 34.3.1 游戏房间Actor

```csharp
// GameRoomActor.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"

#include "GameRoomActor.generated.h"

class AArenaPlayerController;
class AArenaGameState;

UCLASS()
class ARENAGAME_API AGameRoomActor : public AActor
{
    GENERATED_BODY()

public:
    AGameRoomActor();
    virtual void BeginPlay() override;
    virtual void Tick(float DeltaTime) override;
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    // 房间初始化
    UFUNCTION(BlueprintCallable, Category = "GameRoom")
    void InitializeRoom(const FString& InRoomId, const FString& InMatchId, const TArray<FString>& PlayerIds, const FString& InGameMode);

    // 玩家管理
    UFUNCTION(BlueprintCallable, Category = "GameRoom")
    void PlayerJoined(AArenaPlayerController* Player);

    UFUNCTION(BlueprintCallable, Category = "GameRoom")
    void PlayerLeft(AArenaPlayerController* Player);

    UFUNCTION(BlueprintCallable, Category = "GameRoom")
    void PlayerReady(AArenaPlayerController* Player, bool bReady);

    // 游戏流程
    UFUNCTION(BlueprintCallable, Category = "GameRoom")
    void StartGame();

    UFUNCTION(BlueprintCallable, Category = "GameRoom")
    void EndGame(int32 WinningTeam);

    // 状态查询
    UFUNCTION(BlueprintPure, Category = "GameRoom")
    FString GetRoomId() const { return RoomId; }

    UFUNCTION(BlueprintPure, Category = "GameRoom")
    FString GetMatchId() const { return MatchId; }

    UFUNCTION(BlueprintPure, Category = "GameRoom")
    bool IsGameInProgress() const { return bGameInProgress; }

    UFUNCTION(BlueprintPure, Category = "GameRoom")
    TArray<AArenaPlayerController*> GetPlayers() const { return Players; }

    // 配置
    UPROPERTY(EditDefaultsOnly, Category = "GameRoom")
    float PreGameCountdown = 10.0f;

    UPROPERTY(EditDefaultsOnly, Category = "GameRoom")
    float PostGameDuration = 30.0f;

protected:
    UPROPERTY(Replicated)
    FString RoomId;

    UPROPERTY(Replicated)
    FString MatchId;

    UPROPERTY(Replicated)
    FString GameMode;

    UPROPERTY(Replicated)
    bool bGameInProgress = false;

    UPROPERTY(Replicated)
    float CountdownTimer = 0.0f;

private:
    TArray<AArenaPlayerController*> Players;
    TArray<AArenaPlayerController*> ReadyPlayers;
    TMap<FString, bool> PlayerReadyStatus;
    AArenaGameState* GameState;

    void CheckAllPlayersReady();
    void StartCountdown();
    void UpdateCountdown(float DeltaTime);
    void BroadcastGameStart();
    void BroadcastGameEnd(int32 WinningTeam);
    void CleanupRoom();
    void ReportMatchResults(int32 WinningTeam);
};
```

```csharp
// GameRoomActor.cpp
#include "GameRoomActor.h"
#include "ArenaGameState.h"
#include "ArenaPlayerController.h"
#include "Net/UnrealNetwork.h"
#include "Kismet/GameplayStatics.h"

AGameRoomActor::AGameRoomActor()
{
    bReplicates = true;
    PrimaryActorTick.bCanEverTick = true;
}

void AGameRoomActor::BeginPlay()
{
    Super::BeginPlay();

    GameState = Cast<AArenaGameState>(UGameplayStatics::GetGameState(this));
}

void AGameRoomActor::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    if (!HasAuthority())
    {
        return;
    }

    if (CountdownTimer > 0.0f)
    {
        UpdateCountdown(DeltaTime);
    }
}

void AGameRoomActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(AGameRoomActor, RoomId);
    DOREPLIFETIME(AGameRoomActor, MatchId);
    DOREPLIFETIME(AGameRoomActor, GameMode);
    DOREPLIFETIME(AGameRoomActor, bGameInProgress);
    DOREPLIFETIME(AGameRoomActor, CountdownTimer);
}

void AGameRoomActor::InitializeRoom(const FString& InRoomId, const FString& InMatchId, const TArray<FString>& PlayerIds, const FString& InGameMode)
{
    RoomId = InRoomId;
    MatchId = InMatchId;
    GameMode = InGameMode;

    // 初始化玩家就绪状态
    for (const FString& PlayerId : PlayerIds)
    {
        PlayerReadyStatus.Add(PlayerId, false);
    }

    UE_LOG(LogTemp, Log, TEXT("Room initialized: %s, Match: %s, Players: %d"),
        *RoomId, *MatchId, PlayerIds.Num());
}

void AGameRoomActor::PlayerJoined(AArenaPlayerController* Player)
{
    if (!Player || Players.Contains(Player))
    {
        return;
    }

    Players.Add(Player);

    // 验证玩家是否在预期列表中
    FString PlayerId = Player->GetPlayerId();
    if (PlayerReadyStatus.Contains(PlayerId))
    {
        UE_LOG(LogTemp, Log, TEXT("Player %s joined room %s"), *PlayerId, *RoomId);

        // 检查是否所有玩家都已加入
        if (Players.Num() == PlayerReadyStatus.Num())
        {
            // 所有玩家已加入，等待准备
            StartCountdown();
        }
    }
    else
    {
        UE_LOG(LogTemp, Warning, TEXT("Unexpected player %s joined room %s"), *PlayerId, *RoomId);
    }
}

void AGameRoomActor::PlayerLeft(AArenaPlayerController* Player)
{
    if (!Player)
    {
        return;
    }

    Players.Remove(Player);
    ReadyPlayers.Remove(Player);

    FString PlayerId = Player->GetPlayerId();
    PlayerReadyStatus.Remove(PlayerId);

    UE_LOG(LogTemp, Log, TEXT("Player %s left room %s"), *PlayerId, *RoomId);

    // 检查是否应该结束游戏
    if (bGameInProgress)
    {
        // 玩家在游戏中离开
        if (Players.Num() < 2)
        {
            // 游戏结束
            int32 WinningTeam = Players.Num() > 0 ? Players[0]->GetTeamId() : 0;
            EndGame(WinningTeam);
        }
    }
    else if (Players.Num() == 0)
    {
        // 房间为空，清理
        CleanupRoom();
    }
}

void AGameRoomActor::PlayerReady(AArenaPlayerController* Player, bool bReady)
{
    if (!Player || !PlayerReadyStatus.Contains(Player->GetPlayerId()))
    {
        return;
    }

    FString PlayerId = Player->GetPlayerId();
    PlayerReadyStatus[PlayerId] = bReady;

    if (bReady)
    {
        ReadyPlayers.AddUnique(Player);
    }
    else
    {
        ReadyPlayers.Remove(Player);
    }

    UE_LOG(LogTemp, Log, TEXT("Player %s ready: %s"), *PlayerId, bReady ? TEXT("true") : TEXT("false"));

    CheckAllPlayersReady();
}

void AGameRoomActor::CheckAllPlayersReady()
{
    // 检查所有玩家是否都准备好了
    bool bAllReady = true;
    for (const auto& Pair : PlayerReadyStatus)
    {
        if (!Pair.Value)
        {
            bAllReady = false;
            break;
        }
    }

    if (bAllReady && Players.Num() > 0)
    {
        // 所有玩家已准备，可以开始游戏
        CountdownTimer = PreGameCountdown;
    }
}

void AGameRoomActor::StartCountdown()
{
    CountdownTimer = PreGameCountdown;

    // 广播倒计时开始
    for (AArenaPlayerController* Player : Players)
    {
        Player->ClientStartCountdown(CountdownTimer);
    }
}

void AGameRoomActor::UpdateCountdown(float DeltaTime)
{
    CountdownTimer -= DeltaTime;

    // 广播倒计时更新
    if (FMath::IsNearlyEqual(CountdownTimer, FMath::RoundToFloat(CountdownTimer)))
    {
        for (AArenaPlayerController* Player : Players)
        {
            Player->ClientUpdateCountdown(CountdownTimer);
        }
    }

    if (CountdownTimer <= 0.0f)
    {
        StartGame();
    }
}

void AGameRoomActor::StartGame()
{
    if (bGameInProgress)
    {
        return;
    }

    bGameInProgress = true;
    CountdownTimer = 0.0f;

    BroadcastGameStart();

    UE_LOG(LogTemp, Log, TEXT("Game started in room %s"), *RoomId);
}

void AGameRoomActor::EndGame(int32 WinningTeam)
{
    if (!bGameInProgress)
    {
        return;
    }

    bGameInProgress = false;

    BroadcastGameEnd(WinningTeam);

    // 上报比赛结果
    ReportMatchResults(WinningTeam);

    // 设置清理定时器
    FTimerHandle CleanupTimerHandle;
    GetWorldTimerManager().SetTimer(CleanupTimerHandle, this, &AGameRoomActor::CleanupRoom, PostGameDuration, false);
}

void AGameRoomActor::BroadcastGameStart()
{
    for (AArenaPlayerController* Player : Players)
    {
        Player->ClientGameStarted();
    }

    // 通知GameState
    if (GameState)
    {
        GameState->OnGameStarted();
    }
}

void AGameRoomActor::BroadcastGameEnd(int32 WinningTeam)
{
    for (AArenaPlayerController* Player : Players)
    {
        Player->ClientGameEnded(WinningTeam);
    }

    // 通知GameState
    if (GameState)
    {
        GameState->OnGameEnded(WinningTeam);
    }
}

void AGameRoomActor::ReportMatchResults(int32 WinningTeam)
{
    // 收集玩家统计
    TArray<FPlayerMatchStats> AllStats;

    for (AArenaPlayerController* Player : Players)
    {
        FPlayerMatchStats Stats;
        Stats.PlayerId = Player->GetPlayerId();
        Stats.TeamId = Player->GetTeamId();
        Stats.Kills = Player->GetKills();
        Stats.Deaths = Player->GetDeaths();
        Stats.Assists = Player->GetAssists();
        Stats.Score = Player->GetScore();
        Stats.bIsWinner = Player->GetTeamId() == WinningTeam;

        AllStats.Add(Stats);
    }

    // 发送到后端
    // 这里应该调用HTTP API上报结果
}

void AGameRoomActor::CleanupRoom()
{
    UE_LOG(LogTemp, Log, TEXT("Cleaning up room %s"), *RoomId);

    // 断开所有玩家连接
    for (AArenaPlayerController* Player : Players)
    {
        Player->ClientReturnToMainMenu();
    }

    // 销毁房间Actor
    Destroy();
}
```

---

## 34.4 游戏流程控制

### 34.4.1 游戏模式实现

```csharp
// ArenaGameMode_Deathmatch.h
#pragma once

#include "CoreMinimal.h"
#include "ArenaGameMode.h"

#include "ArenaGameMode_Deathmatch.generated.h"

UCLASS()
class ARENAGAME_API AArenaGameMode_Deathmatch : public AArenaGameMode
{
    GENERATED_BODY()

public:
    AArenaGameMode_Deathmatch();

    virtual void BeginPlay() override;
    virtual void Tick(float DeltaTime) override;

    // 重写基类方法
    virtual void RecordKill(APlayerState* Killer, APlayerState* Victim) override;
    virtual void CheckWinCondition() override;

    // 生成控制
    virtual AActor* ChoosePlayerStart_Implementation(AController* Player) override;

    // 重生
    virtual void RespawnPlayer(AArenaPlayerController* Player) override;

protected:
    // 生成的起点
    UPROPERTY(EditDefaultsOnly, Category = "Spawning")
    TArray<AActor*> SpawnPoints;

    // 团队起点（如果有团队模式）
    UPROPERTY(EditDefaultsOnly, Category = "Spawning")
    TMap<int32, TArray<AActor*>> TeamSpawnPoints;

    // 检查是否可以生成
    bool CanSpawnAtLocation(const FVector& Location) const;

    // 获取最佳生成点
    AActor* GetBestSpawnPoint(int32 TeamId = -1) const;

private:
    // 记录玩家击杀数
    TMap<APlayerState*, int32> PlayerKillCounts;

    // 记录玩家死亡数
    TMap<APlayerState*, int32> PlayerDeathCounts;
};
```

```csharp
// ArenaGameMode_Deathmatch.cpp
#include "ArenaGameMode_Deathmatch.h"
#include "ArenaPlayerController.h"
#include "ArenaPlayerState.h"
#include "GameFramework/PlayerStart.h"
#include "Kismet/GameplayStatics.h"

AArenaGameMode_Deathmatch::AArenaGameMode_Deathmatch()
{
    // 死亡赛配置
    MinPlayersToStart = 2;
    MaxPlayers = 8;
    ScoreToWin = 25;
    RespawnDelay = 3.0f;
    MatchDuration = 600.0f;  // 10分钟
}

void AArenaGameMode_Deathmatch::BeginPlay()
{
    Super::BeginPlay();

    // 收集所有生成点
    UGameplayStatics::GetAllActorsOfClass(this, APlayerStart::StaticClass(), SpawnPoints);

    UE_LOG(LogTemp, Log, TEXT("Deathmatch mode initialized with %d spawn points"), SpawnPoints.Num());
}

void AArenaGameMode_Deathmatch::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    // 可以添加自定义逻辑
}

void AArenaGameMode_Deathmatch::RecordKill(APlayerState* Killer, APlayerState* Victim)
{
    Super::RecordKill(Killer, Victim);

    if (Killer)
    {
        int32& KillCount = PlayerKillCounts.FindOrAdd(Killer);
        KillCount++;

        // 更新玩家得分
        AddScore(Killer, 1.0f);

        // 检查胜利条件
        CheckWinCondition();
    }

    if (Victim)
    {
        int32& DeathCount = PlayerDeathCounts.FindOrAdd(Victim);
        DeathCount++;
    }
}

void AArenaGameMode_Deathmatch::CheckWinCondition()
{
    // 检查是否有玩家达到胜利分数
    for (const auto& Pair : PlayerKillCounts)
    {
        if (Pair.Value >= ScoreToWin)
        {
            // 玩家获胜
            APlayerState* Winner = Pair.Key;

            if (AArenaPlayerController* WinnerPC = Cast<AArenaPlayerController>(Winner->GetOwner()))
            {
                // 通知获胜
                EndMatch();
                return;
            }
        }
    }

    // 检查时间是否用尽
    if (MatchTimer <= 0.0f && GetMatchState() == EMatchState::InProgress)
    {
        // 找出得分最高的玩家
        APlayerState* Winner = nullptr;
        int32 HighestScore = -1;

        for (const auto& Pair : PlayerKillCounts)
        {
            if (Pair.Value > HighestScore)
            {
                HighestScore = Pair.Value;
                Winner = Pair.Key;
            }
        }

        EndMatch();
    }
}

AActor* AArenaGameMode_Deathmatch::ChoosePlayerStart_Implementation(AController* Player)
{
    AArenaPlayerController* ArenaPlayer = Cast<AArenaPlayerController>(Player);
    int32 TeamId = ArenaPlayer ? ArenaPlayer->GetTeamId() : -1;

    return GetBestSpawnPoint(TeamId);
}

AActor* AArenaGameMode_Deathmatch::GetBestSpawnPoint(int32 TeamId) const
{
    TArray<AActor*> ValidSpawnPoints;

    if (TeamId >= 0 && TeamSpawnPoints.Contains(TeamId))
    {
        ValidSpawnPoints = TeamSpawnPoints[TeamId];
    }
    else
    {
        ValidSpawnPoints = SpawnPoints;
    }

    if (ValidSpawnPoints.Num() == 0)
    {
        return nullptr;
    }

    // 选择最安全的生成点（远离敌人）
    AActor* BestSpawn = nullptr;
    float BestScore = -1.0f;

    for (AActor* SpawnPoint : ValidSpawnPoints)
    {
        float Score = 1.0f;

        // 检查附近是否有敌人
        FVector SpawnLocation = SpawnPoint->GetActorLocation();

        for (FConstPlayerControllerIterator It = GetWorld()->GetPlayerControllerIterator(); It; ++It)
        {
            AArenaPlayerController* PC = Cast<AArenaPlayerController>(*It);
            if (PC && PC->GetPawn())
            {
                float Distance = FVector::Dist(SpawnLocation, PC->GetPawn()->GetActorLocation());

                // 距离越远分数越高
                Score += Distance / 1000.0f;
            }
        }

        if (Score > BestScore)
        {
            BestScore = Score;
            BestSpawn = SpawnPoint;
        }
    }

    return BestSpawn ? BestSpawn : ValidSpawnPoints[FMath::RandRange(0, ValidSpawnPoints.Num() - 1)];
}

void AArenaGameMode_Deathmatch::RespawnPlayer(AArenaPlayerController* Player)
{
    if (!Player || !HasAuthority())
    {
        return;
    }

    // 移除旧Pawn
    if (APawn* OldPawn = Player->GetPawn())
    {
        OldPawn->Destroy();
    }

    // 延迟重生
    FTimerHandle RespawnTimerHandle;

    FTimerDelegate RespawnDelegate;
    RespawnDelegate.BindLambda([this, Player]()
    {
        // 选择生成点
        AActor* SpawnPoint = ChoosePlayerStart_Implementation(Player);

        if (SpawnPoint)
        {
            FVector SpawnLocation = SpawnPoint->GetActorLocation();
            FRotator SpawnRotation = SpawnPoint->GetActorRotation();

            // 生成新Pawn
            FActorSpawnParameters SpawnParams;
            SpawnParams.Owner = Player;

            APawn* NewPawn = GetWorld()->SpawnActor<APawn>(DefaultPawnClass, SpawnLocation, SpawnRotation, SpawnParams);

            if (NewPawn)
            {
                Player->Possess(NewPawn);
            }
        }
    });

    GetWorldTimerManager().SetTimer(RespawnTimerHandle, RespawnDelegate, RespawnDelay, false);
}
```

---

## 34.5 实践任务

### 任务1：实现房间大厅

创建房间大厅UI：
- 房间列表显示
- 快速加入功能
- 房间筛选

### 任务2：实现房间内交互

创建房间内功能：
- 准备/取消准备
- 切换队伍
- 聊天功能

### 任务3：实现游戏结算

创建游戏结算界面：
- 显示比赛结果
- 玩家统计
- 返回大厅

---

## 34.6 总结

本课完成了：
- 游戏房间系统架构
- 房间管理子系统
- 服务器端房间Actor
- 游戏模式实现
- 游戏流程控制

**下一课预告**：结果统计与排行榜 - 实现比赛数据统计和排名系统。

---

*课程版本：1.0*
*最后更新：2026-04-10*
