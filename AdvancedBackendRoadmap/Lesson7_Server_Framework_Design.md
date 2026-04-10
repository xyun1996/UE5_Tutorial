# 第7课：服务器框架设计

> 本课深入讲解UE5服务器框架的设计模式，包括GameInstance、PlayerState生命周期管理和断线重连机制。

---

## 课程目标

- 掌握GameInstance的持久化数据管理
- 理解PlayerState的生命周期管理
- 学会设计断线重连机制
- 掌握服务器配置管理系统
- 理解玩家连接流程

---

## 一、GameInstance持久化管理

### 1.1 GameInstance概述

```
┌─────────────────────────────────────────────────────────────┐
│                    GameInstance生命周期                      │
│                                                              │
│  游戏启动                                                     │
│      │                                                       │
│      ▼                                                       │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                 GameInstance                          │   │
│  │  - 跨关卡持久化                                       │   │
│  │  - 存储全局数据                                       │   │
│  │  - 管理在线会话                                       │   │
│  │  - 处理玩家连接                                       │   │
│  └──────────────────────────────────────────────────────┘   │
│      │                                                       │
│      │ 存在于整个游戏运行期间                                 │
│      │                                                       │
│      ▼                                                       │
│  关卡切换 ──────► 关卡切换 ──────► 关卡切换                   │
│      │              │              │                         │
│      ▼              ▼              ▼                         │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐                    │
│  │ World 1 │   │ World 2 │   │ World 3 │                    │
│  └─────────┘   └─────────┘   └─────────┘                    │
│      │              │              │                         │
│      └──────────────┴──────────────┘                         │
│                      │                                       │
│                      ▼                                       │
│  游戏退出                                                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 自定义GameInstance

```cpp
// MyGameInstance.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/GameInstance.h"
#include "Interfaces/OnlineSessionInterface.h"
#include "MyGameInstance.generated.h"

// 玩家数据
USTRUCT(BlueprintType)
struct FPlayerGameData
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString PlayerName;

    UPROPERTY(BlueprintReadOnly)
    FString PlayerId;

    UPROPERTY(BlueprintReadOnly)
    int32 TotalKills = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 TotalDeaths = 0;

    UPROPERTY(BlueprintReadOnly)
    float TotalScore = 0.0f;
};

// 游戏设置
USTRUCT(BlueprintType)
struct FGameSettings
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadWrite)
    FString ServerAddress;

    UPROPERTY(BlueprintReadWrite)
    int32 PreferredTeam = -1;

    UPROPERTY(BlueprintReadWrite)
    bool bVoiceChatEnabled = true;

    UPROPERTY(BlueprintReadWrite)
    float MouseSensitivity = 1.0f;
};

// 委托声明
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnSessionCreatedDelegate, bool, bSuccess);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnSessionJoinedDelegate, bool, bSuccess);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnSessionSearchCompleteDelegate, const TArray<FOnlineSessionSearchResult>&, Results);

UCLASS()
class UMyGameInstance : public UGameInstance
{
    GENERATED_BODY()

public:
    UMyGameInstance();

    virtual void Init() override;
    virtual void Shutdown() override;

    // ========== 玩家数据管理 ==========

    UFUNCTION(BlueprintCallable, Category = "Player Data")
    void SetPlayerData(const FPlayerGameData& Data);

    UFUNCTION(BlueprintPure, Category = "Player Data")
    FPlayerGameData GetPlayerData() const { return PlayerData; }

    UFUNCTION(BlueprintCallable, Category = "Player Data")
    void SavePlayerData();

    UFUNCTION(BlueprintCallable, Category = "Player Data")
    bool LoadPlayerData();

    // ========== 游戏设置 ==========

    UFUNCTION(BlueprintCallable, Category = "Settings")
    void SetGameSettings(const FGameSettings& Settings);

    UFUNCTION(BlueprintPure, Category = "Settings")
    FGameSettings GetGameSettings() const { return GameSettings; }

    // ========== 会话管理 ==========

    UFUNCTION(BlueprintCallable, Category = "Session")
    void CreateSession(const FString& SessionName, int32 MaxPlayers);

    UFUNCTION(BlueprintCallable, Category = "Session")
    void FindSessions();

    UFUNCTION(BlueprintCallable, Category = "Session")
    void JoinSession(const FOnlineSessionSearchResult& SessionResult);

    UFUNCTION(BlueprintCallable, Category = "Session")
    void DestroySession();

    // ========== 连接管理 ==========

    UFUNCTION(BlueprintCallable, Category = "Connection")
    void ConnectToServer(const FString& ServerAddress);

    UFUNCTION(BlueprintCallable, Category = "Connection")
    void DisconnectFromServer();

    UFUNCTION(BlueprintCallable, Category = "Connection")
    void TravelToMap(const FString& MapName);

    // 委托
    UPROPERTY(BlueprintAssignable, Category = "Events")
    FOnSessionCreatedDelegate OnSessionCreated;

    UPROPERTY(BlueprintAssignable, Category = "Events")
    FOnSessionJoinedDelegate OnSessionJoined;

    UPROPERTY(BlueprintAssignable, Category = "Events")
    FOnSessionSearchCompleteDelegate OnSessionSearchComplete;

protected:
    // 持久化数据
    UPROPERTY()
    FPlayerGameData PlayerData;

    UPROPERTY()
    FGameSettings GameSettings;

    // 会话搜索
    TSharedPtr<FOnlineSessionSearch> SessionSearch;

    // 在线会话接口
    IOnlineSessionPtr OnlineSessionInterface;

    // 回调函数
    void OnCreateSessionComplete(FName SessionName, bool bSuccess);
    void OnFindSessionsComplete(bool bSuccess);
    void OnJoinSessionComplete(FName SessionName, EOnJoinSessionCompleteResult::Type Result);
    void OnDestroySessionComplete(FName SessionName, bool bSuccess);

private:
    FString SaveSlotName = TEXT("PlayerData");
    void InitializeOnlineSubsystem();
};

// MyGameInstance.cpp
#include "MyGameInstance.h"
#include "OnlineSubsystem.h"
#include "OnlineSessionSettings.h"
#include "Kismet/GameplayStatics.h"
#include "MySaveGame.h"

UMyGameInstance::UMyGameInstance()
{
    // 默认设置
    GameSettings.ServerAddress = TEXT("127.0.0.1:7777");
}

void UMyGameInstance::Init()
{
    Super::Init();

    InitializeOnlineSubsystem();

    // 加载保存的数据
    LoadPlayerData();

    UE_LOG(LogTemp, Log, TEXT("GameInstance initialized"));
}

void UMyGameInstance::Shutdown()
{
    // 保存数据
    SavePlayerData();

    // 清理会话
    DestroySession();

    Super::Shutdown();

    UE_LOG(LogTemp, Log, TEXT("GameInstance shutdown"));
}

// ========== 初始化 ==========

void UMyGameInstance::InitializeOnlineSubsystem()
{
    IOnlineSubsystem* OnlineSubsystem = IOnlineSubsystem::Get();
    if (OnlineSubsystem)
    {
        OnlineSessionInterface = OnlineSubsystem->GetSessionInterface();

        if (OnlineSessionInterface.IsValid())
        {
            // 绑定回调
            OnlineSessionInterface->OnCreateSessionCompleteDelegates.AddUObject(
                this, &UMyGameInstance::OnCreateSessionComplete);
            OnlineSessionInterface->OnFindSessionsCompleteDelegates.AddUObject(
                this, &UMyGameInstance::OnFindSessionsComplete);
            OnlineSessionInterface->OnJoinSessionCompleteDelegates.AddUObject(
                this, &UMyGameInstance::OnJoinSessionComplete);
            OnlineSessionInterface->OnDestroySessionCompleteDelegates.AddUObject(
                this, &UMyGameInstance::OnDestroySessionComplete);
        }
    }
}

// ========== 玩家数据管理 ==========

void UMyGameInstance::SetPlayerData(const FPlayerGameData& Data)
{
    PlayerData = Data;
}

void UMyGameInstance::SavePlayerData()
{
    UMySaveGame* SaveGame = NewObject<UMySaveGame>();
    SaveGame->PlayerData = PlayerData;
    SaveGame->GameSettings = GameSettings;

    UGameplayStatics::SaveGameToSlot(SaveGame, SaveSlotName, 0);
    UE_LOG(LogTemp, Log, TEXT("Player data saved"));
}

bool UMyGameInstance::LoadPlayerData()
{
    if (UGameplayStatics::DoesSaveGameExist(SaveSlotName, 0))
    {
        USaveGame* LoadedGame = UGameplayStatics::LoadGameFromSlot(SaveSlotName, 0);
        if (UMySaveGame* MySaveGame = Cast<UMySaveGame>(LoadedGame))
        {
            PlayerData = MySaveGame->PlayerData;
            GameSettings = MySaveGame->GameSettings;
            UE_LOG(LogTemp, Log, TEXT("Player data loaded"));
            return true;
        }
    }
    return false;
}

void UMyGameInstance::SetGameSettings(const FGameSettings& Settings)
{
    GameSettings = Settings;
    SavePlayerData();
}

// ========== 会话管理 ==========

void UMyGameInstance::CreateSession(const FString& SessionName, int32 MaxPlayers)
{
    if (!OnlineSessionInterface.IsValid())
    {
        OnSessionCreated.Broadcast(false);
        return;
    }

    FOnlineSessionSettings SessionSettings;
    SessionSettings.bIsLANMatch = true;
    SessionSettings.bUsesPresence = true;
    SessionSettings.NumPublicConnections = MaxPlayers;
    SessionSettings.bShouldAdvertise = true;
    SessionSettings.bAllowJoinInProgress = true;

    OnlineSessionInterface->CreateSession(0, FName(*SessionName), SessionSettings);
}

void UMyGameInstance::FindSessions()
{
    if (!OnlineSessionInterface.IsValid())
    {
        OnSessionSearchComplete.Broadcast(TArray<FOnlineSessionSearchResult>());
        return;
    }

    SessionSearch = MakeShareable(new FOnlineSessionSearch());
    SessionSearch->bIsLanQuery = true;
    SessionSearch->MaxSearchResults = 100;

    OnlineSessionInterface->FindSessions(0, SessionSearch.ToSharedRef());
}

void UMyGameInstance::JoinSession(const FOnlineSessionSearchResult& SessionResult)
{
    if (!OnlineSessionInterface.IsValid())
    {
        OnSessionJoined.Broadcast(false);
        return;
    }

    OnlineSessionInterface->JoinSession(0, NAME_GameSession, SessionResult);
}

void UMyGameInstance::DestroySession()
{
    if (OnlineSessionInterface.IsValid())
    {
        OnlineSessionInterface->DestroySession(NAME_GameSession);
    }
}

// ========== 连接管理 ==========

void UMyGameInstance::ConnectToServer(const FString& ServerAddress)
{
    GameSettings.ServerAddress = ServerAddress;

    FString URL = FString::Printf(TEXT("%s"), *ServerAddress);

    APlayerController* PC = GetFirstLocalPlayerController();
    if (PC)
    {
        PC->ClientTravel(URL, ETravelType::TRAVEL_Absolute);
    }
}

void UMyGameInstance::DisconnectFromServer()
{
    APlayerController* PC = GetFirstLocalPlayerController();
    if (PC)
    {
        PC->ClientReturnToMainMenuWithTextReason(FText::FromString("Disconnected"));
    }
}

void UMyGameInstance::TravelToMap(const FString& MapName)
{
    GetWorld()->ServerTravel(MapName + "?Listen");
}

// ========== 回调实现 ==========

void UMyGameInstance::OnCreateSessionComplete(FName SessionName, bool bSuccess)
{
    OnSessionCreated.Broadcast(bSuccess);
    UE_LOG(LogTemp, Log, TEXT("Session created: %s, Success: %d"), *SessionName.ToString(), bSuccess);
}

void UMyGameInstance::OnFindSessionsComplete(bool bSuccess)
{
    TArray<FOnlineSessionSearchResult> Results;

    if (bSuccess && SessionSearch.IsValid())
    {
        for (const FOnlineSessionSearchResult& Result : SessionSearch->SearchResults)
        {
            Results.Add(Result);
        }
    }

    OnSessionSearchComplete.Broadcast(Results);
    UE_LOG(LogTemp, Log, TEXT("Session search complete, found %d sessions"), Results.Num());
}

void UMyGameInstance::OnJoinSessionComplete(FName SessionName, EOnJoinSessionCompleteResult::Type Result)
{
    bool bSuccess = (Result == EOnJoinSessionCompleteResult::Success);

    if (bSuccess && OnlineSessionInterface.IsValid())
    {
        FString ConnectString;
        if (OnlineSessionInterface->GetResolvedConnectString(SessionName, ConnectString))
        {
            APlayerController* PC = GetFirstLocalPlayerController();
            if (PC)
            {
                PC->ClientTravel(ConnectString, ETravelType::TRAVEL_Absolute);
            }
        }
    }

    OnSessionJoined.Broadcast(bSuccess);
}

void UMyGameInstance::OnDestroySessionComplete(FName SessionName, bool bSuccess)
{
    UE_LOG(LogTemp, Log, TEXT("Session destroyed: %s"), *SessionName.ToString());
}
```

---

## 二、PlayerState生命周期

### 2.1 PlayerState生命周期图

```
┌─────────────────────────────────────────────────────────────┐
│                  PlayerState生命周期                         │
│                                                              │
│  玩家连接                                                     │
│      │                                                       │
│      ▼                                                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  1. 创建 PlayerState                                 │    │
│  │     - 构造函数                                       │    │
│  │     - 初始化默认值                                   │    │
│  └─────────────────────────────────────────────────────┘    │
│      │                                                       │
│      ▼                                                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  2. BeginPlay                                        │    │
│  │     - 注册到GameState                                │    │
│  │     - 初始化玩家数据                                 │    │
│  └─────────────────────────────────────────────────────┘    │
│      │                                                       │
│      ▼                                                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  3. 游戏进行中                                       │    │
│  │     - 更新统计数据                                   │    │
│  │     - 处理分数变化                                   │    │
│  │     - 同步到所有客户端                               │    │
│  └─────────────────────────────────────────────────────┘    │
│      │                                                       │
│      ▼                                                       │
│  玩家断开连接                                                 │
│      │                                                       │
│      ▼                                                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  4. EndPlay                                          │    │
│  │     - 从GameState注销                                │    │
│  │     - 保存最终数据                                   │    │
│  └─────────────────────────────────────────────────────┘    │
│      │                                                       │
│      ▼                                                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  5. 销毁 PlayerState                                 │    │
│  │     - 析构函数                                       │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 自定义PlayerState

```cpp
// MyPlayerState.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/PlayerState.h"
#include "MyPlayerState.generated.h"

// 玩家统计数据
USTRUCT(BlueprintType)
struct FPlayerMatchStats
{
    GENERATED_BODY()

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
    float Score = 0.0f;

    float GetKDA() const
    {
        return Deaths > 0 ? static_cast<float>(Kills + Assists) / Deaths : Kills;
    }
};

// 玩家状态枚举
UENUM(BlueprintType)
enum class EPlayerStatus : uint8
{
    None,
    Connected,
    Playing,
    Spectating,
    Disconnected,
    Reconnecting
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnScoreChangedDelegate, float, OldScore, float, NewScore);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnKillDelegate, int32, OldKills, int32, NewKills);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnDeathDelegate, int32, TotalDeaths);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnStatusChangedDelegate, EPlayerStatus, NewStatus);

UCLASS()
class AMyPlayerState : public APlayerState
{
    GENERATED_BODY()

public:
    AMyPlayerState();

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;

    // ========== 基础信息 ==========

    UPROPERTY(ReplicatedUsing = OnRep_TeamId, BlueprintReadOnly, Category = "Player")
    int32 TeamId;

    UPROPERTY(ReplicatedUsing = OnRep_PlayerStatus, BlueprintReadOnly, Category = "Player")
    EPlayerStatus PlayerStatus;

    UPROPERTY(Replicated, BlueprintReadOnly, Category = "Player")
    FString SteamId;

    // ========== 统计数据 ==========

    UPROPERTY(ReplicatedUsing = OnRep_MatchStats, BlueprintReadOnly, Category = "Stats")
    FPlayerMatchStats MatchStats;

    // ========== 委托 ==========

    UPROPERTY(BlueprintAssignable, Category = "Events")
    FOnScoreChangedDelegate OnScoreChanged;

    UPROPERTY(BlueprintAssignable, Category = "Events")
    FOnKillDelegate OnKill;

    UPROPERTY(BlueprintAssignable, Category = "Events")
    FOnDeathDelegate OnDeath;

    UPROPERTY(BlueprintAssignable, Category = "Events")
    FOnStatusChangedDelegate OnStatusChanged;

    // ========== 公共函数 ==========

    // 统计更新
    UFUNCTION(BlueprintCallable, Category = "Stats")
    void AddKill();

    UFUNCTION(BlueprintCallable, Category = "Stats")
    void AddDeath();

    UFUNCTION(BlueprintCallable, Category = "Stats")
    void AddAssist();

    UFUNCTION(BlueprintCallable, Category = "Stats")
    void AddScore(float Amount);

    UFUNCTION(BlueprintCallable, Category = "Stats")
    void AddDamageDealt(float Amount);

    UFUNCTION(BlueprintCallable, Category = "Stats")
    void AddDamageTaken(float Amount);

    // 状态管理
    UFUNCTION(BlueprintCallable, Category = "Player")
    void SetTeam(int32 NewTeamId);

    UFUNCTION(BlueprintCallable, Category = "Player")
    void SetPlayerStatus(EPlayerStatus NewStatus);

    // 查询函数
    UFUNCTION(BlueprintPure, Category = "Stats")
    float GetKDA() const { return MatchStats.GetKDA(); }

    UFUNCTION(BlueprintPure, Category = "Stats")
    bool IsOnSameTeamAs(const AMyPlayerState* Other) const;

    UFUNCTION(BlueprintPure, Category = "Player")
    FString GetDisplayName() const;

    // 重置
    UFUNCTION(BlueprintCallable, Category = "Stats")
    void ResetMatchStats();

protected:
    UFUNCTION()
    void OnRep_TeamId(int32 OldTeamId);

    UFUNCTION()
    void OnRep_PlayerStatus(EPlayerStatus OldStatus);

    UFUNCTION()
    void OnRep_MatchStats(const FPlayerMatchStats& OldStats);

private:
    void BroadcastStatChanges(const FPlayerMatchStats& OldStats);
};

// MyPlayerState.cpp
#include "MyPlayerState.h"
#include "MyGameState.h"
#include "Net/UnrealNetwork.h"

AMyPlayerState::AMyPlayerState()
{
    TeamId = -1; // 无队伍
    PlayerStatus = EPlayerStatus::None;
}

void AMyPlayerState::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(AMyPlayerState, TeamId);
    DOREPLIFETIME(AMyPlayerState, PlayerStatus);
    DOREPLIFETIME(AMyPlayerState, SteamId);
    DOREPLIFETIME(AMyPlayerState, MatchStats);
}

void AMyPlayerState::BeginPlay()
{
    Super::BeginPlay();

    if (HasAuthority())
    {
        PlayerStatus = EPlayerStatus::Connected;

        // 注册到GameState
        if (AMyGameState* GS = GetWorld()->GetGameState<AMyGameState>())
        {
            // GS->RegisterPlayerState(this);
        }
    }

    UE_LOG(LogTemp, Log, TEXT("PlayerState BeginPlay: %s"), *GetPlayerName());
}

void AMyPlayerState::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    if (HasAuthority())
    {
        // 从GameState注销
        if (AMyGameState* GS = GetWorld()->GetGameState<AMyGameState>())
        {
            // GS->UnregisterPlayerState(this);
        }
    }

    UE_LOG(LogTemp, Log, TEXT("PlayerState EndPlay: %s"), *GetPlayerName());

    Super::EndPlay(EndPlayReason);
}

// ========== 统计更新 ==========

void AMyPlayerState::AddKill()
{
    if (HasAuthority())
    {
        MatchStats.Kills++;
        OnKill.Broadcast(MatchStats.Kills - 1, MatchStats.Kills);
    }
}

void AMyPlayerState::AddDeath()
{
    if (HasAuthority())
    {
        MatchStats.Deaths++;
        OnDeath.Broadcast(MatchStats.Deaths);
    }
}

void AMyPlayerState::AddAssist()
{
    if (HasAuthority())
    {
        MatchStats.Assists++;
    }
}

void AMyPlayerState::AddScore(float Amount)
{
    if (HasAuthority())
    {
        float OldScore = MatchStats.Score;
        MatchStats.Score += Amount;
        OnScoreChanged.Broadcast(OldScore, MatchStats.Score);
    }
}

void AMyPlayerState::AddDamageDealt(float Amount)
{
    if (HasAuthority())
    {
        MatchStats.DamageDealt += Amount;
    }
}

void AMyPlayerState::AddDamageTaken(float Amount)
{
    if (HasAuthority())
    {
        MatchStats.DamageTaken += Amount;
    }
}

// ========== 状态管理 ==========

void AMyPlayerState::SetTeam(int32 NewTeamId)
{
    if (HasAuthority())
    {
        int32 OldTeamId = TeamId;
        TeamId = NewTeamId;
        OnRep_TeamId(OldTeamId);
    }
}

void AMyPlayerState::SetPlayerStatus(EPlayerStatus NewStatus)
{
    if (HasAuthority())
    {
        EPlayerStatus OldStatus = PlayerStatus;
        PlayerStatus = NewStatus;
        OnRep_PlayerStatus(OldStatus);
    }
}

// ========== 查询函数 ==========

bool AMyPlayerState::IsOnSameTeamAs(const AMyPlayerState* Other) const
{
    if (!Other || TeamId < 0 || Other->TeamId < 0)
    {
        return false;
    }
    return TeamId == Other->TeamId;
}

FString AMyPlayerState::GetDisplayName() const
{
    if (!GetPlayerName().IsEmpty())
    {
        return GetPlayerName();
    }
    return FString::Printf(TEXT("Player_%d"), GetPlayerId());
}

void AMyPlayerState::ResetMatchStats()
{
    if (HasAuthority())
    {
        MatchStats = FPlayerMatchStats();
    }
}

// ========== OnRep回调 ==========

void AMyPlayerState::OnRep_TeamId(int32 OldTeamId)
{
    UE_LOG(LogTemp, Log, TEXT("Player %s team changed: %d -> %d"),
        *GetPlayerName(), OldTeamId, TeamId);
}

void AMyPlayerState::OnRep_PlayerStatus(EPlayerStatus OldStatus)
{
    OnStatusChanged.Broadcast(PlayerStatus);
    UE_LOG(LogTemp, Log, TEXT("Player %s status changed: %d -> %d"),
        *GetPlayerName(), static_cast<int32>(OldStatus), static_cast<int32>(PlayerStatus));
}

void AMyPlayerState::OnRep_MatchStats(const FPlayerMatchStats& OldStats)
{
    BroadcastStatChanges(OldStats);
}

void AMyPlayerState::BroadcastStatChanges(const FPlayerMatchStats& OldStats)
{
    if (MatchStats.Kills != OldStats.Kills)
    {
        OnKill.Broadcast(OldStats.Kills, MatchStats.Kills);
    }

    if (MatchStats.Deaths != OldStats.Deaths)
    {
        OnDeath.Broadcast(MatchStats.Deaths);
    }

    if (MatchStats.Score != OldStats.Score)
    {
        OnScoreChanged.Broadcast(OldStats.Score, MatchStats.Score);
    }
}
```

---

## 三、断线重连机制

### 3.1 重连机制设计

```
┌─────────────────────────────────────────────────────────────┐
│                      断线重连流程                            │
│                                                              │
│  客户端                                                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  1. 检测到连接断开                                   │    │
│  │     - 网络超时                                       │    │
│  │     - 服务器无响应                                   │    │
│  └─────────────────────────────────────────────────────┘    │
│      │                                                       │
│      ▼                                                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  2. 保存本地状态                                     │    │
│  │     - 玩家ID                                         │    │
│  │     - 游戏进度                                       │    │
│  │     - 重连令牌                                       │    │
│  └─────────────────────────────────────────────────────┘    │
│      │                                                       │
│      ▼                                                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  3. 显示重连UI                                       │    │
│  │     - 倒计时                                         │    │
│  │     - 取消按钮                                       │    │
│  └─────────────────────────────────────────────────────┘    │
│      │                                                       │
│      ▼                                                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  4. 尝试重新连接                                     │    │
│  │     - 指数退避重试                                   │    │
│  │     - 最大重试次数                                   │    │
│  └─────────────────────────────────────────────────────┘    │
│      │                                                       │
│      ▼                                                       │
│  服务器                                                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  5. 验证重连请求                                     │    │
│  │     - 检查令牌有效性                                 │    │
│  │     - 检查玩家是否仍在线                             │    │
│  └─────────────────────────────────────────────────────┘    │
│      │                                                       │
│      ▼                                                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  6. 恢复玩家状态                                     │    │
│  │     - 恢复统计数据                                   │    │
│  │     - 恢复位置                                       │    │
│  │     - 同步游戏状态                                   │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 实现断线重连

```cpp
// MyReconnectionManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "MyReconnectionManager.generated.h"

// 重连令牌
USTRUCT()
struct FReconnectionToken
{
    GENERATED_BODY()

    UPROPERTY()
    FString PlayerId;

    UPROPERTY()
    FString SessionId;

    UPROPERTY()
    FString AuthToken;

    UPROPERTY()
    FString ServerAddress;

    float Timestamp;
    float ExpirationTime = 300.0f; // 5分钟有效期

    bool IsValid() const
    {
        return !PlayerId.IsEmpty() && !AuthToken.IsEmpty();
    }

    bool IsExpired() const
    {
        return FPlatformTime::Seconds() - Timestamp > ExpirationTime;
    }
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnReconnectionResultDelegate, bool, bSuccess);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnReconnectionProgressDelegate, int32, AttemptNumber);

UCLASS()
class UMyReconnectionManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 生成重连令牌
    FString GenerateReconnectionToken(const FString& PlayerId, const FString& SessionId);

    // 保存重连数据
    void SaveReconnectionData(const FString& ServerAddress, const FString& SessionId);

    // 尝试重连
    UFUNCTION(BlueprintCallable, Category = "Reconnection")
    void AttemptReconnection();

    // 取消重连
    UFUNCTION(BlueprintCallable, Category = "Reconnection")
    void CancelReconnection();

    // 检查是否有保存的重连数据
    UFUNCTION(BlueprintPure, Category = "Reconnection")
    bool HasReconnectionData() const;

    // 清除重连数据
    UFUNCTION(BlueprintCallable, Category = "Reconnection")
    void ClearReconnectionData();

    // 委托
    UPROPERTY(BlueprintAssignable, Category = "Reconnection")
    FOnReconnectionResultDelegate OnReconnectionResult;

    UPROPERTY(BlueprintAssignable, Category = "Reconnection")
    FOnReconnectionProgressDelegate OnReconnectionProgress;

protected:
    // 重连配置
    UPROPERTY(EditDefaultsOnly, Category = "Reconnection")
    int32 MaxReconnectionAttempts = 5;

    UPROPERTY(EditDefaultsOnly, Category = "Reconnection")
    float InitialRetryDelay = 1.0f;

    UPROPERTY(EditDefaultsOnly, Category = "Reconnection")
    float MaxRetryDelay = 30.0f;

private:
    FReconnectionToken SavedToken;
    int32 CurrentAttempt;
    FTimerHandle ReconnectionTimerHandle;
    bool bIsReconnecting;

    void PerformReconnectionAttempt();
    void OnConnectionResult(bool bSuccess);
    float CalculateRetryDelay() const;
};

// MyReconnectionManager.cpp
#include "MyReconnectionManager.h"
#include "Kismet/GameplayStatics.h"
#include "GameFramework/PlayerController.h"

void UMyReconnectionManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    // 加载保存的重连数据
    // LoadSavedToken();

    bIsReconnecting = false;
    CurrentAttempt = 0;
}

void UMyReconnectionManager::Deinitialize()
{
    CancelReconnection();
    Super::Deinitialize();
}

FString UMyReconnectionManager::GenerateReconnectionToken(const FString& PlayerId, const FString& SessionId)
{
    // 生成唯一令牌
    FReconnectionToken Token;
    Token.PlayerId = PlayerId;
    Token.SessionId = SessionId;
    Token.AuthToken = FGuid::NewGuid().ToString();
    Token.Timestamp = FPlatformTime::Seconds();

    SavedToken = Token;

    return Token.AuthToken;
}

void UMyReconnectionManager::SaveReconnectionData(const FString& ServerAddress, const FString& SessionId)
{
    SavedToken.ServerAddress = ServerAddress;
    SavedToken.SessionId = SessionId;
    SavedToken.Timestamp = FPlatformTime::Seconds();

    // 可选：保存到本地存储
    // UGameplayStatics::SaveGameToSlot(...);
}

void UMyReconnectionManager::AttemptReconnection()
{
    if (bIsReconnecting)
    {
        return;
    }

    if (!SavedToken.IsValid() || SavedToken.IsExpired())
    {
        OnReconnectionResult.Broadcast(false);
        ClearReconnectionData();
        return;
    }

    bIsReconnecting = true;
    CurrentAttempt = 0;

    PerformReconnectionAttempt();
}

void UMyReconnectionManager::PerformReconnectionAttempt()
{
    CurrentAttempt++;
    OnReconnectionProgress.Broadcast(CurrentAttempt);

    if (CurrentAttempt > MaxReconnectionAttempts)
    {
        OnReconnectionResult.Broadcast(false);
        CancelReconnection();
        return;
    }

    // 尝试连接服务器
    FString URL = FString::Printf(TEXT("%s?ReconnectToken=%s?PlayerId=%s"),
        *SavedToken.ServerAddress,
        *SavedToken.AuthToken,
        *SavedToken.PlayerId);

    APlayerController* PC = GetWorld()->GetFirstPlayerController();
    if (PC)
    {
        PC->ClientTravel(URL, ETravelType::TRAVEL_Absolute);
    }

    // 设置超时检查
    float Delay = CalculateRetryDelay();
    GetWorld()->GetTimerManager().SetTimer(
        ReconnectionTimerHandle,
        this,
        &UMyReconnectionManager::OnConnectionResult,
        Delay,
        false,
        false  // 暂时传递false，实际应该检查连接状态
    );
}

void UMyReconnectionManager::OnConnectionResult(bool bSuccess)
{
    if (bSuccess)
    {
        OnReconnectionResult.Broadcast(true);
        CancelReconnection();
    }
    else
    {
        // 继续重试
        PerformReconnectionAttempt();
    }
}

float UMyReconnectionManager::CalculateRetryDelay() const
{
    // 指数退避
    float Delay = InitialRetryDelay * FMath::Pow(2.0f, CurrentAttempt - 1);
    return FMath::Min(Delay, MaxRetryDelay);
}

void UMyReconnectionManager::CancelReconnection()
{
    bIsReconnecting = false;
    GetWorld()->GetTimerManager().ClearTimer(ReconnectionTimerHandle);
    CurrentAttempt = 0;
}

bool UMyReconnectionManager::HasReconnectionData() const
{
    return SavedToken.IsValid() && !SavedToken.IsExpired();
}

void UMyReconnectionManager::ClearReconnectionData()
{
    SavedToken = FReconnectionToken();
}
```

### 3.3 服务器端重连处理

```cpp
// MyGameMode中的重连处理

void AMyGameMode::PreLogin(const FString& Options, const FString& Address, const FUniqueNetIdRepl& UniqueId, FString& ErrorMessage)
{
    // 检查是否是重连
    FString ReconnectToken = UGameplayStatics::ParseOption(Options, TEXT("ReconnectToken"));
    FString PlayerId = UGameplayStatics::ParseOption(Options, TEXT("PlayerId"));

    if (!ReconnectToken.IsEmpty() && !PlayerId.IsEmpty())
    {
        // 验证重连令牌
        if (ValidateReconnectionToken(PlayerId, ReconnectToken))
        {
            // 恢复玩家状态
            RestorePlayerState(PlayerId);
            UE_LOG(LogTemp, Log, TEXT("Player %s reconnecting"), *PlayerId);
            return;
        }
        else
        {
            ErrorMessage = TEXT("Invalid reconnection token");
            return;
        }
    }

    Super::PreLogin(Options, Address, UniqueId, ErrorMessage);
}

bool AMyGameMode::ValidateReconnectionToken(const FString& PlayerId, const FString& Token)
{
    // 从存储中获取玩家保存的令牌
    FString SavedToken = GetSavedTokenForPlayer(PlayerId);

    if (SavedToken.IsEmpty())
    {
        return false;
    }

    return SavedToken == Token;
}

void AMyGameMode::RestorePlayerState(const FString& PlayerId)
{
    // 获取之前保存的玩家数据
    FPlayerSavedData SavedData = GetSavedPlayerData(PlayerId);

    // 在Login中会创建新的PlayerController
    // 需要标记这是重连的玩家，以便恢复数据
    PendingReconnectionPlayerId = PlayerId;
}

APlayerController* AMyGameMode::Login(UPlayer* NewPlayer, ENetRole InRemoteRole, const FString& Portal, const FString& Options, const FUniqueNetIdRepl& UniqueId, FString& ErrorMessage)
{
    APlayerController* PC = Super::Login(NewPlayer, InRemoteRole, Portal, Options, UniqueId, ErrorMessage);

    if (PC && !PendingReconnectionPlayerId.IsEmpty())
    {
        // 恢复玩家数据
        if (AMyPlayerState* PS = PC->GetPlayerState<AMyPlayerState>())
        {
            ApplySavedPlayerData(PS, PendingReconnectionPlayerId);
        }

        PendingReconnectionPlayerId.Empty();
    }

    return PC;
}
```

---

## 四、服务器配置管理

### 4.1 配置文件结构

```ini
; Config/DefaultGame.ini

[/Script/MyProject.MyGameMode]
; 游戏规则
MaxPlayers=32
MatchDuration=600
ScoreToWin=100
bAllowTeamChange=true
bFriendlyFire=false

; 服务器设置
ServerName="My Game Server"
WelcomeMessage="Welcome to the server!"
bEnableCheats=false

[/Script/MyProject.MyGameSettings]
; 经济设置
StartingMoney=1000
KillReward=100
DeathPenalty=-50

; 物品设置
ItemSpawnInterval=30.0
MaxItemsOnMap=20
```

### 4.2 配置管理类

```cpp
// MyServerConfig.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/DeveloperSettings.h"
#include "MyServerConfig.generated.h"

UCLASS(Config = Game, DefaultConfig, meta = (DisplayName = "My Game Server Config"))
class UMyServerConfig : public UDeveloperSettings
{
    GENERATED_BODY()

public:
    // 游戏规则
    UPROPERTY(Config, EditAnywhere, Category = "Game Rules")
    int32 MaxPlayers = 32;

    UPROPERTY(Config, EditAnywhere, Category = "Game Rules")
    float MatchDuration = 600.0f;

    UPROPERTY(Config, EditAnywhere, Category = "Game Rules")
    int32 ScoreToWin = 100;

    UPROPERTY(Config, EditAnywhere, Category = "Game Rules")
    bool bAllowTeamChange = true;

    UPROPERTY(Config, EditAnywhere, Category = "Game Rules")
    bool bFriendlyFire = false;

    // 服务器设置
    UPROPERTY(Config, EditAnywhere, Category = "Server")
    FString ServerName = TEXT("My Game Server");

    UPROPERTY(Config, EditAnywhere, Category = "Server")
    FString WelcomeMessage = TEXT("Welcome!");

    UPROPERTY(Config, EditAnywhere, Category = "Server")
    FString MOTD = TEXT("Have fun!");

    UPROPERTY(Config, EditAnywhere, Category = "Server")
    bool bEnableCheats = false;

    // 经济设置
    UPROPERTY(Config, EditAnywhere, Category = "Economy")
    int32 StartingMoney = 1000;

    UPROPERTY(Config, EditAnywhere, Category = "Economy")
    int32 KillReward = 100;

    UPROPERTY(Config, EditAnywhere, Category = "Economy")
    int32 DeathPenalty = 50;

    // 获取单例
    static UMyServerConfig* Get()
    {
        return GetMutableDefault<UMyServerConfig>();
    }

    // 保存配置
    void SaveConfig()
    {
        UpdateDefaultConfigFile();
    }
};
```

---

## 五、总结

本课我们学习了：

1. **GameInstance**：跨关卡持久化数据管理
2. **PlayerState**：生命周期和统计数据管理
3. **断线重连**：令牌验证和状态恢复
4. **配置管理**：服务器设置系统

---

## 六、下节预告

**第8课：游戏会话管理**

将深入学习：
- OnlineSubsystem接口
- 会话创建与查找
- 玩家加入/退出流程

---

*课程版本：1.0*
*最后更新：2026-04-10*