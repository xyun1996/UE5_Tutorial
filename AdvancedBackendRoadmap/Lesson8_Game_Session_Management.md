# 第8课：游戏会话管理

> 本课深入讲解UE5的游戏会话管理系统，包括OnlineSubsystem接口、会话创建与查找、玩家加入退出流程。

---

## 课程目标

- 理解OnlineSubsystem架构
- 掌握会话创建与查找方法
- 学会管理玩家加入/退出流程
- 实现会话状态机设计
- 掌握跨平台会话处理

---

## 一、OnlineSubsystem架构

### 1.1 架构概述

```
┌─────────────────────────────────────────────────────────────┐
│                   OnlineSubsystem架构                        │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    应用层                            │    │
│  │  GameInstance / GameMode / PlayerController         │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│                          ▼                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              OnlineSubsystem接口                     │    │
│  │  - IOnlineSession (会话管理)                         │    │
│  │  - IOnlineIdentity (身份验证)                        │    │
│  │  - IOnlineFriends (好友系统)                         │    │
│  │  - IOnlineAchievements (成就)                        │    │
│  │  - IOnlineLeaderboards (排行榜)                      │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│          ┌───────────────┼───────────────┐                  │
│          ▼               ▼               ▼                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   Steam     │  │    EOS     │  │   Null      │         │
│  │  Subsystem  │  │  Subsystem │  │  Subsystem  │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 获取OnlineSubsystem接口

```cpp
// MySessionManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "Interfaces/OnlineSessionInterface.h"
#include "MySessionManager.generated.h"

// 会话搜索结果
USTRUCT(BlueprintType)
struct FBlueprintSessionResult
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString SessionId;

    UPROPERTY(BlueprintReadOnly)
    FString ServerName;

    UPROPERTY(BlueprintReadOnly)
    int32 CurrentPlayers;

    UPROPERTY(BlueprintReadOnly)
    int32 MaxPlayers;

    UPROPERTY(BlueprintReadOnly)
    int32 Ping;

    FOnlineSessionSearchResult NativeResult;
};

// 会话设置
USTRUCT(BlueprintType)
struct FBlueprintSessionSettings
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadWrite)
    FString SessionName;

    UPROPERTY(BlueprintReadWrite)
    int32 MaxPlayers = 16;

    UPROPERTY(BlueprintReadWrite)
    bool bIsLAN = false;

    UPROPERTY(BlueprintReadWrite)
    bool bAllowJoinInProgress = true;

    UPROPERTY(BlueprintReadWrite)
    FString MapName;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnSessionCreated, bool, bSuccess);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnSessionFound, const TArray<FBlueprintSessionResult>&, Results);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnSessionJoined, bool, bSuccess);
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnSessionDestroyed);

UCLASS()
class UMySessionManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 会话操作
    UFUNCTION(BlueprintCallable, Category = "Session")
    void CreateSession(const FBlueprintSessionSettings& Settings);

    UFUNCTION(BlueprintCallable, Category = "Session")
    void FindSessions(bool bFindLAN = false);

    UFUNCTION(BlueprintCallable, Category = "Session")
    void JoinSession(const FBlueprintSessionResult& SessionResult);

    UFUNCTION(BlueprintCallable, Category = "Session")
    void DestroySession();

    // 身份验证
    UFUNCTION(BlueprintCallable, Category = "Identity")
    bool Login(const FString& Id, const FString& Token);

    UFUNCTION(BlueprintCallable, Category = "Identity")
    void Logout();

    UFUNCTION(BlueprintPure, Category = "Identity")
    bool IsLoggedIn() const;

    UFUNCTION(BlueprintPure, Category = "Identity")
    FString GetPlayerName() const;

    // 委托
    UPROPERTY(BlueprintAssignable)
    FOnSessionCreated OnSessionCreated;

    UPROPERTY(BlueprintAssignable)
    FOnSessionFound OnSessionFound;

    UPROPERTY(BlueprintAssignable)
    FOnSessionJoined OnSessionJoined;

    UPROPERTY(BlueprintAssignable)
    FOnSessionDestroyed OnSessionDestroyed;

private:
    // 接口指针
    IOnlineSessionPtr SessionInterface;
    IOnlineIdentityPtr IdentityInterface;

    // 搜索结果
    TSharedPtr<FOnlineSessionSearch> LastSearch;

    // 回调处理
    void HandleCreateSessionComplete(FName SessionName, bool bSuccess);
    void HandleFindSessionsComplete(bool bSuccess);
    void HandleJoinSessionComplete(FName SessionName, EOnJoinSessionCompleteResult::Type Result);
    void HandleDestroySessionComplete(FName SessionName, bool bSuccess);
    void HandleLoginComplete(int32 LocalUserNum, bool bSuccess, const FUniqueNetId& UserId, const FString& Error);

    // 辅助函数
    FBlueprintSessionResult ConvertResult(const FOnlineSessionSearchResult& Result) const;
    TArray<FBlueprintSessionResult> ConvertResults(const TArray<FOnlineSessionSearchResult>& Results) const;
};

// MySessionManager.cpp
#include "MySessionManager.h"
#include "OnlineSubsystem.h"
#include "OnlineSessionSettings.h"
#include "Kismet/GameplayStatics.h"

void UMySessionManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    IOnlineSubsystem* OnlineSubsystem = IOnlineSubsystem::Get();
    if (OnlineSubsystem)
    {
        SessionInterface = OnlineSubsystem->GetSessionInterface();
        IdentityInterface = OnlineSubsystem->GetIdentityInterface();

        if (SessionInterface.IsValid())
        {
            // 绑定回调
            SessionInterface->OnCreateSessionCompleteDelegates.AddUObject(
                this, &UMySessionManager::HandleCreateSessionComplete);
            SessionInterface->OnFindSessionsCompleteDelegates.AddUObject(
                this, &UMySessionManager::HandleFindSessionsComplete);
            SessionInterface->OnJoinSessionCompleteDelegates.AddUObject(
                this, &UMySessionManager::HandleJoinSessionComplete);
            SessionInterface->OnDestroySessionCompleteDelegates.AddUObject(
                this, &UMySessionManager::HandleDestroySessionComplete);
        }

        if (IdentityInterface.IsValid())
        {
            IdentityInterface->OnLoginCompleteDelegates->AddUObject(
                this, &UMySessionManager::HandleLoginComplete);
        }
    }

    UE_LOG(LogTemp, Log, TEXT("SessionManager initialized"));
}

void UMySessionManager::Deinitialize()
{
    if (SessionInterface.IsValid())
    {
        SessionInterface->OnCreateSessionCompleteDelegates.RemoveAll(this);
        SessionInterface->OnFindSessionsCompleteDelegates.RemoveAll(this);
        SessionInterface->OnJoinSessionCompleteDelegates.RemoveAll(this);
        SessionInterface->OnDestroySessionCompleteDelegates.RemoveAll(this);
    }

    if (IdentityInterface.IsValid())
    {
        IdentityInterface->OnLoginCompleteDelegates->RemoveAll(this);
    }

    Super::Deinitialize();
}

// ========== 身份验证 ==========

bool UMySessionManager::Login(const FString& Id, const FString& Token)
{
    if (!IdentityInterface.IsValid())
    {
        return false;
    }

    FOnlineAccountCredentials Credentials;
    Credentials.Id = Id;
    Credentials.Token = Token;
    Credentials.Type = TEXT("epic"); // 或 "steam", "dev" 等

    return IdentityInterface->Login(0, Credentials);
}

void UMySessionManager::Logout()
{
    if (IdentityInterface.IsValid())
    {
        IdentityInterface->Logout(0);
    }
}

bool UMySessionManager::IsLoggedIn() const
{
    if (IdentityInterface.IsValid())
    {
        return IdentityInterface->GetLoginStatus(0) == ELoginStatus::LoggedIn;
    }
    return false;
}

FString UMySessionManager::GetPlayerName() const
{
    if (IdentityInterface.IsValid())
    {
        TSharedPtr<const FUniqueNetId> UserId = IdentityInterface->GetUniquePlayerId(0);
        if (UserId.IsValid())
        {
            return IdentityInterface->GetPlayerNickname(*UserId);
        }
    }
    return TEXT("");
}

// ========== 会话创建 ==========

void UMySessionManager::CreateSession(const FBlueprintSessionSettings& Settings)
{
    if (!SessionInterface.IsValid())
    {
        OnSessionCreated.Broadcast(false);
        return;
    }

    // 创建会话设置
    FOnlineSessionSettings SessionSettings;
    SessionSettings.NumPublicConnections = Settings.MaxPlayers;
    SessionSettings.bIsLANMatch = Settings.bIsLAN;
    SessionSettings.bShouldAdvertise = true;
    SessionSettings.bAllowJoinInProgress = Settings.bAllowJoinInProgress;
    SessionSettings.bUsesPresence = true;
    SessionSettings.bUseLobbiesIfAvailable = true;

    // 自定义设置
    SessionSettings.Set(FName("ServerName"), Settings.SessionName, EOnlineDataAdvertisementType::ViaOnlineService);
    SessionSettings.Set(FName("MapName"), Settings.MapName, EOnlineDataAdvertisementType::ViaOnlineService);

    // 创建会话
    bool bSuccess = SessionInterface->CreateSession(0, NAME_GameSession, SessionSettings);

    if (!bSuccess)
    {
        OnSessionCreated.Broadcast(false);
    }
}

void UMySessionManager::HandleCreateSessionComplete(FName SessionName, bool bSuccess)
{
    OnSessionCreated.Broadcast(bSuccess);

    if (bSuccess)
    {
        UE_LOG(LogTemp, Log, TEXT("Session created: %s"), *SessionName.ToString());

        // 启动服务器监听
        GetWorld()->ServerTravel("?Listen");
    }
    else
    {
        UE_LOG(LogTemp, Warning, TEXT("Failed to create session"));
    }
}

// ========== 会话搜索 ==========

void UMySessionManager::FindSessions(bool bFindLAN)
{
    if (!SessionInterface.IsValid())
    {
        OnSessionFound.Broadcast(TArray<FBlueprintSessionResult>());
        return;
    }

    LastSearch = MakeShareable(new FOnlineSessionSearch());
    LastSearch->bIsLanQuery = bFindLAN;
    LastSearch->MaxSearchResults = 100;

    // 搜索设置
    LastSearch->QuerySettings.Set(SEARCH_PRESENCE, true, EOnlineComparisonOp::Equals);

    SessionInterface->FindSessions(0, LastSearch.ToSharedRef());
}

void UMySessionManager::HandleFindSessionsComplete(bool bSuccess)
{
    TArray<FBlueprintSessionResult> Results;

    if (bSuccess && LastSearch.IsValid())
    {
        Results = ConvertResults(LastSearch->SearchResults);
        UE_LOG(LogTemp, Log, TEXT("Found %d sessions"), Results.Num());
    }
    else
    {
        UE_LOG(LogTemp, Warning, TEXT("Session search failed"));
    }

    OnSessionFound.Broadcast(Results);
}

// ========== 加入会话 ==========

void UMySessionManager::JoinSession(const FBlueprintSessionResult& SessionResult)
{
    if (!SessionInterface.IsValid())
    {
        OnSessionJoined.Broadcast(false);
        return;
    }

    SessionInterface->JoinSession(0, NAME_GameSession, SessionResult.NativeResult);
}

void UMySessionManager::HandleJoinSessionComplete(FName SessionName, EOnJoinSessionCompleteResult::Type Result)
{
    bool bSuccess = (Result == EOnJoinSessionCompleteResult::Success);

    if (bSuccess && SessionInterface.IsValid())
    {
        FString ConnectString;
        if (SessionInterface->GetResolvedConnectString(SessionName, ConnectString))
        {
            // 连接到服务器
            APlayerController* PC = GetWorld()->GetFirstPlayerController();
            if (PC)
            {
                PC->ClientTravel(ConnectString, ETravelType::TRAVEL_Absolute);
            }
        }
    }

    OnSessionJoined.Broadcast(bSuccess);
    UE_LOG(LogTemp, Log, TEXT("Join session result: %d"), static_cast<int32>(Result));
}

// ========== 销毁会话 ==========

void UMySessionManager::DestroySession()
{
    if (SessionInterface.IsValid())
    {
        SessionInterface->DestroySession(NAME_GameSession);
    }
}

void UMySessionManager::HandleDestroySessionComplete(FName SessionName, bool bSuccess)
{
    OnSessionDestroyed.Broadcast();
    UE_LOG(LogTemp, Log, TEXT("Session destroyed: %s, success: %d"), *SessionName.ToString(), bSuccess);
}

// ========== 辅助函数 ==========

FBlueprintSessionResult UMySessionManager::ConvertResult(const FOnlineSessionSearchResult& Result) const
{
    FBlueprintSessionResult BPResult;
    BPResult.SessionId = Result.GetSessionIdStr();
    BPResult.CurrentPlayers = Result.Session.NumOpenPublicConnections;
    BPResult.MaxPlayers = Result.Session.SessionSettings.NumPublicConnections;
    BPResult.Ping = Result.PingInMs;
    BPResult.NativeResult = Result;

    // 获取服务器名称
    FString ServerName;
    if (Result.Session.SessionSettings.Get(FName("ServerName"), ServerName))
    {
        BPResult.ServerName = ServerName;
    }

    return BPResult;
}

TArray<FBlueprintSessionResult> UMySessionManager::ConvertResults(const TArray<FOnlineSessionSearchResult>& Results) const
{
    TArray<FBlueprintSessionResult> BPResults;
    for (const auto& Result : Results)
    {
        BPResults.Add(ConvertResult(Result));
    }
    return BPResults;
}

void UMySessionManager::HandleLoginComplete(int32 LocalUserNum, bool bSuccess, const FUniqueNetId& UserId, const FString& Error)
{
    if (bSuccess)
    {
        UE_LOG(LogTemp, Log, TEXT("Login successful for user %d"), LocalUserNum);
    }
    else
    {
        UE_LOG(LogTemp, Warning, TEXT("Login failed: %s"), *Error);
    }
}
```

---

## 二、会话创建与查找

### 2.1 创建会话

```cpp
// 创建不同类型的会话

// LAN会话
void CreateLANSession(const FString& SessionName)
{
    FBlueprintSessionSettings Settings;
    Settings.SessionName = SessionName;
    Settings.bIsLAN = true;
    Settings.MaxPlayers = 16;
    Settings.MapName = TEXT("Lobby");

    CreateSession(Settings);
}

// 在线会话（通过Steam/EOS）
void CreateOnlineSession(const FString& SessionName)
{
    FBlueprintSessionSettings Settings;
    Settings.SessionName = SessionName;
    Settings.bIsLAN = false;
    Settings.MaxPlayers = 32;
    Settings.bAllowJoinInProgress = true;

    CreateSession(Settings);
}

// 私密会话（仅好友）
void CreatePrivateSession(const FString& SessionName)
{
    FBlueprintSessionSettings Settings;
    Settings.SessionName = SessionName;
    Settings.bIsLAN = false;
    Settings.MaxPlayers = 4;

    CreateSession(Settings);

    // 设置为私密需要在创建后修改
    if (SessionInterface.IsValid())
    {
        FOnlineSessionSettings* SessionSettings = SessionInterface->GetSessionSettings(NAME_GameSession);
        if (SessionSettings)
        {
            SessionSettings->bShouldAdvertise = false;
            SessionSettings->bAllowInvites = true;
        }
    }
}
```

### 2.2 会话搜索与过滤

```cpp
// 高级会话搜索
UCLASS()
class UAdvancedSessionSearcher : public UObject
{
    GENERATED_BODY()

public:
    // 按名称搜索
    void SearchByName(const FString& ServerName)
    {
        LastSearch = MakeShareable(new FOnlineSessionSearch());
        LastSearch->QuerySettings.Set(FName("ServerName"), ServerName, EOnlineComparisonOp::Equals);
        SessionInterface->FindSessions(0, LastSearch.ToSharedRef());
    }

    // 按游戏模式搜索
    void SearchByGameMode(const FString& GameMode)
    {
        LastSearch = MakeShareable(new FOnlineSessionSearch());
        LastSearch->QuerySettings.Set(FName("GameMode"), GameMode, EOnlineComparisonOp::Equals);
        SessionInterface->FindSessions(0, LastSearch.ToSharedRef());
    }

    // 搜索有空位的会话
    void SearchWithOpenSlots()
    {
        LastSearch = MakeShareable(new FOnlineSessionSearch());
        LastSearch->QuerySettings.Set(SEARCH_EMPTY_SERVERS, false, EOnlineComparisonOp::Equals);
        SessionInterface->FindSessions(0, LastSearch.ToSharedRef());
    }

    // 搜索特定地区
    void SearchByRegion(const FString& Region)
    {
        LastSearch = MakeShareable(new FOnlineSessionSearch());
        LastSearch->QuerySettings.Set(FName("Region"), Region, EOnlineComparisonOp::Equals);
        SessionInterface->FindSessions(0, LastSearch.ToSharedRef());
    }
};
```

---

## 三、玩家加入退出流程

### 3.1 完整流程图

```
┌─────────────────────────────────────────────────────────────┐
│                      玩家加入流程                            │
│                                                              │
│  客户端                                                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  1. 搜索会话 / 直接输入地址                          │    │
│  └─────────────────────────────────────────────────────┘    │
│      │                                                       │
│      ▼                                                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  2. 选择会话并请求加入                               │    │
│  └─────────────────────────────────────────────────────┘    │
│      │                                                       │
│      ▼                                                       │
│  服务器                                                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  3. PreLogin - 验证玩家                              │    │
│  └─────────────────────────────────────────────────────┘    │
│      │                                                       │
│      ▼                                                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  4. Login - 创建PlayerController                    │    │
│  └─────────────────────────────────────────────────────┘    │
│      │                                                       │
│      ▼                                                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  5. PostLogin - 初始化玩家                           │    │
│  │     - 设置玩家名称                                   │    │
│  │     - 分配队伍                                       │    │
│  │     - 生成Pawn                                       │    │
│  │     - 广播加入事件                                   │    │
│  └─────────────────────────────────────────────────────┘    │
│      │                                                       │
│      ▼                                                       │
│  客户端                                                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  6. 接收复制数据，游戏开始                           │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 GameMode中的玩家管理

```cpp
// MyGameMode.cpp - 完整的玩家加入退出处理

void AMyGameMode::PreLogin(const FString& Options, const FString& Address, const FUniqueNetIdRepl& UniqueId, FString& ErrorMessage)
{
    // 调用基类
    Super::PreLogin(Options, Address, UniqueId, ErrorMessage);

    // 如果已有错误，直接返回
    if (!ErrorMessage.IsEmpty())
    {
        return;
    }

    // 检查服务器状态
    if (bMatchInProgress && !bAllowJoinInProgress)
    {
        ErrorMessage = TEXT("Match already in progress");
        return;
    }

    // 检查玩家数量
    if (GetNumPlayers() >= MaxPlayers)
    {
        ErrorMessage = TEXT("Server is full");
        return;
    }

    // 验证玩家身份
    if (!ValidatePlayerIdentity(UniqueId))
    {
        ErrorMessage = TEXT("Player verification failed");
        return;
    }

    // 检查封禁列表
    if (IsPlayerBanned(UniqueId))
    {
        ErrorMessage = TEXT("You are banned from this server");
        return;
    }

    // 检查白名单（如果启用）
    if (bUseWhitelist && !IsPlayerWhitelisted(UniqueId))
    {
        ErrorMessage = TEXT("Server is invite only");
        return;
    }

    UE_LOG(LogTemp, Log, TEXT("PreLogin passed for connection from %s"), *Address);
}

APlayerController* AMyGameMode::Login(UPlayer* NewPlayer, ENetRole InRemoteRole, const FString& Portal, const FString& Options, const FUniqueNetIdRepl& UniqueId, FString& ErrorMessage)
{
    // 解析选项
    FString PlayerName = UGameplayStatics::ParseOption(Options, TEXT("Name"));
    FString TeamPref = UGameplayStatics::ParseOption(Options, TEXT("Team"));

    // 创建PlayerController
    APlayerController* PC = Super::Login(NewPlayer, InRemoteRole, Portal, Options, UniqueId, ErrorMessage);

    if (!PC)
    {
        ErrorMessage = TEXT("Failed to create player controller");
        return nullptr;
    }

    // 设置网络角色
    PC->SetReplicates(true);

    UE_LOG(LogTemp, Log, TEXT("Player logged in: %s"), *PC->GetName());

    return PC;
}

void AMyGameMode::PostLogin(APlayerController* NewPlayer)
{
    Super::PostLogin(NewPlayer);

    // 初始化玩家状态
    if (AMyPlayerState* PS = NewPlayer->GetPlayerState<AMyPlayerState>())
    {
        // 设置玩家名称
        if (PS->GetPlayerName().IsEmpty())
        {
            PS->SetPlayerName(FString::Printf(TEXT("Player_%d"), GetNumPlayers()));
        }

        // 分配队伍
        int32 TeamId = AssignPlayerToTeam(NewPlayer);
        PS->SetTeam(TeamId);

        // 发送欢迎消息
        NewPlayer->ClientMessage(FString::Printf(TEXT("Welcome to %s!"), *ServerName));
    }

    // 生成初始Pawn
    if (NewPlayer->GetPawn() == nullptr)
    {
        RestartPlayer(NewPlayer);
    }

    // 广播玩家加入事件
    Multicast_PlayerJoined(NewPlayer);

    // 更新会话信息
    UpdateSessionInfo();

    UE_LOG(LogTemp, Log, TEXT("PostLogin complete for %s, Total players: %d"),
        *NewPlayer->GetName(), GetNumPlayers());
}

void AMyGameMode::Logout(AController* Exiting)
{
    APlayerController* PC = Cast<APlayerController>(Exiting);
    if (!PC)
    {
        return;
    }

    FString PlayerName = PC->GetName();
    int32 TeamId = -1;

    // 保存玩家数据
    if (AMyPlayerState* PS = PC->GetPlayerState<AMyPlayerState>())
    {
        PlayerName = PS->GetPlayerName();
        TeamId = PS->TeamId;

        // 保存统计数据
        SavePlayerStats(PS);
    }

    // 广播玩家离开事件
    Multicast_PlayerLeft(PC, PlayerName);

    // 更新会话信息
    UpdateSessionInfo();

    Super::Logout(Exiting);

    UE_LOG(LogTemp, Log, TEXT("Player %s logged out, Remaining players: %d"),
        *PlayerName, GetNumPlayers() - 1);
}

// 队伍分配
int32 AMyGameMode::AssignPlayerToTeam(APlayerController* NewPlayer)
{
    if (!bTeamGame)
    {
        return -1;
    }

    // 计算各队伍人数
    TArray<int32> TeamCounts;
    TeamCounts.Init(0, NumTeams);

    for (FConstPlayerControllerIterator It = GetWorld()->GetPlayerControllerIterator(); It; ++It)
    {
        APlayerController* PC = It->Get();
        if (PC && PC != NewPlayer)
        {
            if (AMyPlayerState* PS = PC->GetPlayerState<AMyPlayerState>())
            {
                if (PS->TeamId >= 0 && PS->TeamId < NumTeams)
                {
                    TeamCounts[PS->TeamId]++;
                }
            }
        }
    }

    // 找人最少的队伍
    int32 MinTeam = 0;
    int32 MinCount = TeamCounts[0];
    for (int32 i = 1; i < NumTeams; i++)
    {
        if (TeamCounts[i] < MinCount)
        {
            MinCount = TeamCounts[i];
            MinTeam = i;
        }
    }

    return MinTeam;
}

// 广播函数
void AMyGameMode::Multicast_PlayerJoined_Implementation(APlayerController* NewPlayer)
{
    if (AMyPlayerState* PS = NewPlayer->GetPlayerState<AMyPlayerState>())
    {
        FString Message = FString::Printf(TEXT("%s joined the game!"), *PS->GetPlayerName());
        BroadcastMessage(Message);
    }
}

void AMyGameMode::Multicast_PlayerLeft_Implementation(APlayerController* LeavingPlayer, const FString& PlayerName)
{
    FString Message = FString::Printf(TEXT("%s left the game."), *PlayerName);
    BroadcastMessage(Message);
}

void AMyGameMode::BroadcastMessage(const FString& Message)
{
    for (FConstPlayerControllerIterator It = GetWorld()->GetPlayerControllerIterator(); It; ++It)
    {
        if (APlayerController* PC = It->Get())
        {
            PC->ClientMessage(Message);
        }
    }
}
```

---

## 四、会话状态机设计

### 4.1 会话状态定义

```cpp
// MySessionStateMachine.h
#pragma once

#include "CoreMinimal.h"
#include "MySessionStateMachine.generated.h"

// 会话状态
UENUM(BlueprintType)
enum class ESessionState : uint8
{
    None,               // 无会话
    Creating,           // 创建中
    Hosting,            // 作为主机
    Searching,          // 搜索中
    Joining,            // 加入中
    Connected,          // 已连接
    Disconnecting,      // 断开中
    Failed              // 失败
};

// 状态转换事件
UENUM(BlueprintType)
enum class ESessionEvent : uint8
{
    CreateRequest,      // 请求创建
    CreateSuccess,      // 创建成功
    CreateFailed,       // 创建失败
    SearchRequest,      // 请求搜索
    SearchComplete,     // 搜索完成
    JoinRequest,        // 请求加入
    JoinSuccess,        // 加入成功
    JoinFailed,         // 加入失败
    Disconnect,         // 断开连接
    ConnectionLost      // 连接丢失
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnStateChanged, ESessionState, OldState, ESessionState, NewState);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnEventTriggered, ESessionEvent, Event);

UCLASS()
class UMySessionStateMachine : public UObject
{
    GENERATED_BODY()

public:
    UMySessionStateMachine();

    // 状态查询
    UFUNCTION(BlueprintPure, Category = "Session")
    ESessionState GetCurrentState() const { return CurrentState; }

    UFUNCTION(BlueprintPure, Category = "Session")
    bool CanCreateSession() const;

    UFUNCTION(BlueprintPure, Category = "Session")
    bool CanJoinSession() const;

    UFUNCTION(BlueprintPure, Category = "Session")
    bool CanDisconnect() const;

    // 状态转换
    void ProcessEvent(ESessionEvent Event);

    // 委托
    UPROPERTY(BlueprintAssignable)
    FOnStateChanged OnStateChanged;

    UPROPERTY(BlueprintAssignable)
    FOnEventTriggered OnEventTriggered;

private:
    ESessionState CurrentState;

    // 状态转换表
    TMap<TPair<ESessionState, ESessionEvent>, ESessionState> TransitionTable;

    void InitializeTransitionTable();
    ESessionState GetNextState(ESessionState State, ESessionEvent Event) const;
};

// MySessionStateMachine.cpp
#include "MySessionStateMachine.h"

UMySessionStateMachine::UMySessionStateMachine()
{
    CurrentState = ESessionState::None;
    InitializeTransitionTable();
}

void UMySessionStateMachine::InitializeTransitionTable()
{
    // 状态转换表定义
    TransitionTable.Add({ESessionState::None, ESessionEvent::CreateRequest}, ESessionState::Creating);
    TransitionTable.Add({ESessionState::None, ESessionEvent::SearchRequest}, ESessionState::Searching);
    TransitionTable.Add({ESessionState::None, ESessionEvent::JoinRequest}, ESessionState::Joining);

    TransitionTable.Add({ESessionState::Creating, ESessionEvent::CreateSuccess}, ESessionState::Hosting);
    TransitionTable.Add({ESessionState::Creating, ESessionEvent::CreateFailed}, ESessionState::Failed);

    TransitionTable.Add({ESessionState::Searching, ESessionEvent::SearchComplete}, ESessionState::None);
    TransitionTable.Add({ESessionState::Searching, ESessionEvent::JoinRequest}, ESessionState::Joining);

    TransitionTable.Add({ESessionState::Joining, ESessionEvent::JoinSuccess}, ESessionState::Connected);
    TransitionTable.Add({ESessionState::Joining, ESessionEvent::JoinFailed}, ESessionState::Failed);

    TransitionTable.Add({ESessionState::Hosting, ESessionEvent::Disconnect}, ESessionState::Disconnecting);
    TransitionTable.Add({ESessionState::Connected, ESessionEvent::Disconnect}, ESessionState::Disconnecting);
    TransitionTable.Add({ESessionState::Connected, ESessionEvent::ConnectionLost}, ESessionState::Failed);

    TransitionTable.Add({ESessionState::Disconnecting, ESessionEvent::Disconnect}, ESessionState::None);
    TransitionTable.Add({ESessionState::Failed, ESessionEvent::Disconnect}, ESessionState::None);
}

void UMySessionStateMachine::ProcessEvent(ESessionEvent Event)
{
    ESessionState OldState = CurrentState;
    ESessionState NewState = GetNextState(CurrentState, Event);

    if (NewState != CurrentState)
    {
        CurrentState = NewState;
        OnStateChanged.Broadcast(OldState, NewState);

        UE_LOG(LogTemp, Log, TEXT("Session state: %d -> %d (event: %d)"),
            static_cast<int32>(OldState), static_cast<int32>(NewState), static_cast<int32>(Event));
    }

    OnEventTriggered.Broadcast(Event);
}

ESessionState UMySessionStateMachine::GetNextState(ESessionState State, ESessionEvent Event) const
{
    TPair<ESessionState, ESessionEvent> Key(State, Event);
    if (TransitionTable.Contains(Key))
    {
        return TransitionTable[Key];
    }
    return State; // 无效转换，保持当前状态
}

bool UMySessionStateMachine::CanCreateSession() const
{
    return CurrentState == ESessionState::None;
}

bool UMySessionStateMachine::CanJoinSession() const
{
    return CurrentState == ESessionState::None || CurrentState == ESessionState::Searching;
}

bool UMySessionStateMachine::CanDisconnect() const
{
    return CurrentState == ESessionState::Hosting || CurrentState == ESessionState::Connected;
}
```

---

## 五、实践任务

### 任务：实现完整的多人游戏大厅系统

1. 创建会话管理器
2. 实现会话搜索和过滤
3. 设计玩家加入流程
4. 实现状态机管理

---

## 六、总结

本课我们学习了：

1. **OnlineSubsystem**：架构和接口使用
2. **会话管理**：创建、搜索、加入、销毁
3. **玩家流程**：PreLogin、Login、PostLogin、Logout
4. **状态机**：会话状态转换管理

---

## 七、下节预告

**第9课：服务器架构模式**

将深入学习：
- 单一服务器架构
- 分区服务器架构
- 匹配服务器设计
- 服务器集群架构

---

*课程版本：1.0*
*最后更新：2026-04-10*