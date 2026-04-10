# 第6课：Dedicated Server开发

> 本课全面讲解UE5专用服务器（Dedicated Server）的开发、编译、部署和配置。

---

## 课程目标

- 掌握Dedicated Server的编译方法
- 理解服务器启动参数的配置
- 学会GameMode与GameState的设计
- 掌握PlayerController的服务器逻辑
- 理解服务器与客户端的角色分工

---

## 一、Dedicated Server概述

### 1.1 什么是Dedicated Server

```
┌─────────────────────────────────────────────────────────────┐
│                    Dedicated Server架构                      │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Dedicated Server                        │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │    │
│  │  │  无渲染     │  │  无本地玩家 │  │  纯逻辑计算 │  │    │
│  │  │  无音频     │  │  服务器权威 │  │  高性能运行 │  │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│            ┌─────────────┼─────────────┐                    │
│            │             │             │                    │
│            ▼             ▼             ▼                    │
│      ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│      │ Client 1 │  │ Client 2 │  │ Client N │              │
│      │ 有渲染   │  │ 有渲染   │  │ 有渲染   │              │
│      │ 有音频   │  │ 有音频   │  │ 有音频   │              │
│      └──────────┘  └──────────┘  └──────────┘              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Dedicated Server vs Listen Server

| 特性 | Dedicated Server | Listen Server |
|------|-----------------|---------------|
| 渲染 | 无 | 有（Host玩家） |
| 本地玩家 | 无 | 有（Host玩家） |
| 性能开销 | 低 | 较高 |
| 公平性 | 完全公平 | Host有优势 |
| 部署复杂度 | 需要独立服务器 | 简单 |
| 适用场景 | 竞技游戏、MMO | 合作游戏、LAN |

---

## 二、编译Dedicated Server

### 2.1 项目配置

```cpp
// 在项目的Build.cs中确保正确配置

// MyProject.Build.cs
public class MyProject : ModuleRules
{
    public MyProject(ReadOnlyTargetRules Target) : base(Target)
    {
        PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;

        PublicDependencyModuleNames.AddRange(new string[]
        {
            "Core",
            "CoreUObject",
            "Engine",
            "InputCore",
            "OnlineSubsystem",
            "OnlineSubsystemUtils"
        });

        // 服务器专用代码
        if (Target.Type == TargetType.Server)
        {
            // 服务器特有的依赖
            PublicDependencyModuleNames.Add("GameLiftServerSDK");
        }

        // 客户端专用代码
        if (Target.Type == TargetType.Client)
        {
            PublicDependencyModuleNames.Add("SteamVR");
        }
    }
}
```

### 2.2 编译命令

```bash
# Windows - 编译开发版服务器
UnrealBuildTool MyProject Win64 Development "G:\MyProject\MyProject.uproject" -serverconfig=true

# Windows - 编译发布版服务器
UnrealBuildTool MyProject Win64 Shipping "G:\MyProject\MyProject.uproject" -serverconfig=true

# Linux - 交叉编译
UnrealBuildTool MyProject Linux Shipping "G:\MyProject\MyProject.uproject" -serverconfig=true

# 使用UBT包装器
RunUAT BuildCookRun -project="G:\MyProject\MyProject.uproject" -targetplatform=Win64 -server -cook -build
```

### 2.3 打包服务器

```bash
# 使用Unreal Automation Tool打包
RunUAT BuildCookRun \
  -project="G:\MyProject\MyProject.uproject" \
  -noclient \
  -targetplatform=Win64 \
  -server \
  -serverplatform=Win64 \
  -cook \
  -build \
  -stage \
  -pak \
  -archive \
  -archivedirectory="G:\MyProject\PackagedServer"

# Linux服务器打包
RunUAT BuildCookRun \
  -project="G:\MyProject\MyProject.uproject" \
  -noclient \
  -targetplatform=Win64 \
  -server \
  -serverplatform=Linux \
  -cook \
  -build \
  -stage \
  -pak \
  -archive \
  -archivedirectory="G:\MyProject\PackagedServerLinux"
```

### 2.4 服务器Target配置

```csharp
// MyProjectServer.Target.cs
using UnrealBuildTool;
using System.Collections.Generic;

public class MyProjectServerTarget : TargetRules
{
    public MyProjectServerTarget(TargetInfo Target) : base(Target)
    {
        Type = TargetType.Server;
        DefaultBuildSettings = BuildSettingsVersion.V5;
        ExtraModuleNames.Add("MyProject");

        // 服务器优化设置
        bUseLoggingInShipping = false;  // 发布版禁用日志
        bUseChecksInShipping = false;   // 发布版禁用检查
        bUseAssertsInShipping = false;

        // 减少包体大小
        bStripSymbols = true;
    }
}
```

---

## 三、服务器启动参数

### 3.1 基本启动命令

```bash
# Windows
MyProjectServer.exe MyMap?Listen -log

# Linux
./MyProjectServer MyMap?Listen -log

# 完整参数示例
MyProjectServer.exe MyMap?Listen?MaxPlayers=16?Port=7777 -log -MultiHome=192.168.1.100
```

### 3.2 URL参数详解

```cpp
// 地图参数
// MyMap?Listen - 启动MyMap作为监听服务器
// MyMap?Game=MyGameMode - 指定GameMode类
// MyMap?MaxPlayers=16 - 设置最大玩家数
// MyMap?Port=7777 - 设置端口
// MyMap?Name=ServerName - 设置服务器名称

// 组合参数示例
// MyMap?Listen?Game=/Script/MyProject.MyGameMode?MaxPlayers=32?Port=7777
```

### 3.3 命令行参数

```bash
# 常用命令行参数

# 日志相关
-log                    # 启用日志输出
-logfile=Server.log     # 指定日志文件
-Verbose                # 详细日志

# 网络相关
-Port=7777              # 游戏端口
-QueryPort=27015        # Steam查询端口
-MultiHome=192.168.1.1  # 绑定特定IP

# 性能相关
-deterministic          # 确定性物理
-usefixedtimestep       # 固定时间步
-FixedTimestep=0.016667 # 固定步长（60fps）

# 其他
-RenderOffscreen        # 离屏渲染（某些服务器需要）
-Unattended             # 无交互模式
-NoSteam                # 禁用Steam
```

### 3.4 配置文件

```ini
; DefaultEngine.ini - 服务器配置

[/Script/Engine.Engine]
; 网络驱动配置
NetDriverDefinitions=(DefName="GameNetDriver",DriverClassName="IpNetDriver",DriverClassNameFallback="IpNetDriver")

[/Script/IpDrv.IpNetDriver]
; 网络设置
MaxClientRate=15000
MaxInternetClientRate=10000
KeepAliveTime=0.2
InitialConnectTimeout=30.0
ConnectionTimeout=180.0
NetServerMaxTickRate=30
MaxNetTickRate=120

[/Script/Engine.GameEngine]
; 服务器信息
ServerDefaultMap=/Game/Maps/EntryMap
LocalMapOptions=?Listen

[/Script/Engine.GameSession]
; 会话设置
MaxPlayers=32
MaxSpectators=4

[/Script/OnlineSubsystemUtils.IpNetDriver]
; 高级网络设置
NetConnectionClassName=IpConnection
AllowDownloads=False
```

### 3.5 启动脚本示例

```bash
#!/bin/bash
# start_server.sh - Linux服务器启动脚本

PROJECT_NAME="MyProject"
MAP_NAME="Lobby"
PORT=7777
QUERY_PORT=27015
MAX_PLAYERS=32

./MyProjectServer $MAP_NAME \
  ?Listen \
  ?MaxPlayers=$MAX_PLAYERS \
  ?Port=$PORT \
  -log \
  -log=$PROJECT_NAME.log \
  -Port=$PORT \
  -QueryPort=$QUERY_PORT \
  -unattended \
  -RenderOffscreen &

echo "Server started on port $PORT"
```

```bat
@echo off
REM start_server.bat - Windows服务器启动脚本

set PROJECT_NAME=MyProject
set MAP_NAME=Lobby
set PORT=7777
set MAX_PLAYERS=32

MyProjectServer.exe %MAP_NAME% ^
  ?Listen ^
  ?MaxPlayers=%MAX_PLAYERS% ^
  ?Port=%PORT% ^
  -log ^
  -Port=%PORT% ^
  -unattended

pause
```

---

## 四、GameMode与GameState

### 4.1 GameMode - 服务器权威

```cpp
// MyGameMode.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/GameModeBase.h"
#include "MyGameMode.generated.h"

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnPlayerJoinedDelegate, APlayerController*, Player, const FString&, PlayerName);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnPlayerLeftDelegate, APlayerController*, Player);

UCLASS()
class AMyGameMode : public AGameModeBase
{
    GENERATED_BODY()

public:
    AMyGameMode();

    virtual void InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage) override;
    virtual void PreLogin(const FString& Options, const FString& Address, const FUniqueNetIdRepl& UniqueId, FString& ErrorMessage) override;
    virtual APlayerController* Login(UPlayer* NewPlayer, ENetRole InRemoteRole, const FString& Portal, const FString& Options, const FUniqueNetIdRepl& UniqueId, FString& ErrorMessage) override;
    virtual void PostLogin(APlayerController* NewPlayer) override;
    virtual void Logout(AController* Exiting) override;
    virtual bool ReadyToStartMatch_Implementation() override;
    virtual void StartMatch() override;
    virtual void EndMatch() override;

    // 游戏规则
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Game Rules")
    int32 MaxPlayers = 32;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Game Rules")
    float MatchDuration = 600.0f; // 10分钟

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Game Rules")
    float WarmupDuration = 30.0f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Game Rules")
    int32 ScoreToWin = 100;

    // 委托
    UPROPERTY(BlueprintAssignable, Category = "Events")
    FOnPlayerJoinedDelegate OnPlayerJoined;

    UPROPERTY(BlueprintAssignable, Category = "Events")
    FOnPlayerLeftDelegate OnPlayerLeft;

    // 公共函数
    UFUNCTION(BlueprintCallable, Category = "Game")
    void StartNewMatch();

    UFUNCTION(BlueprintCallable, Category = "Game")
    void RestartMatch();

    UFUNCTION(BlueprintCallable, Category = "Game")
    void EndCurrentMatch();

    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Game")
    bool IsMatchInProgress() const;

protected:
    // 生成点管理
    virtual AActor* ChoosePlayerStart_Implementation(AController* Player) override;
    virtual bool ShouldSpawnAtStartSpot(AController* Player) override;

    // 游戏状态追踪
    FTimerHandle MatchTimerHandle;
    float MatchTimeRemaining;
    bool bMatchStarted;

private:
    void UpdateMatchTimer();
    void CheckWinCondition();
    void BroadcastMatchStart();
    void BroadcastMatchEnd();
};

// MyGameMode.cpp
#include "MyGameMode.h"
#include "MyGameState.h"
#include "MyPlayerState.h"
#include "MyPlayerController.h"
#include "Kismet/GameplayStatics.h"

AMyGameMode::AMyGameMode()
{
    // 设置默认类
    GameStateClass = AMyGameState::StaticClass();
    PlayerStateClass = AMyPlayerState::StaticClass();
    PlayerControllerClass = AMyPlayerController::StaticClass();

    // 延迟玩家自动生成
    bDelayedStart = true;

    bMatchStarted = false;
}

void AMyGameMode::InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage)
{
    Super::InitGame(MapName, Options, ErrorMessage);

    UE_LOG(LogTemp, Log, TEXT("Game initialized on map: %s"), *MapName);

    // 解析命令行参数
    FString MaxPlayersStr = UGameplayStatics::ParseOption(Options, TEXT("MaxPlayers"));
    if (!MaxPlayersStr.IsEmpty())
    {
        MaxPlayers = FCString::Atoi(*MaxPlayersStr);
    }

    // 初始化游戏状态
    if (AMyGameState* GS = Cast<AMyGameState>(GameState))
    {
        GS->InitializeGameState();
    }
}

void AMyGameMode::PreLogin(const FString& Options, const FString& Address, const FUniqueNetIdRepl& UniqueId, FString& ErrorMessage)
{
    Super::PreLogin(Options, Address, UniqueId, ErrorMessage);

    // 检查服务器是否已满
    if (GetNumPlayers() >= MaxPlayers)
    {
        ErrorMessage = TEXT("Server is full");
        return;
    }

    // 检查是否被封禁
    // if (IsPlayerBanned(UniqueId))
    // {
    //     ErrorMessage = TEXT("You are banned from this server");
    //     return;
    // }

    UE_LOG(LogTemp, Log, TEXT("Player attempting to join from: %s"), *Address);
}

APlayerController* AMyGameMode::Login(UPlayer* NewPlayer, ENetRole InRemoteRole, const FString& Portal, const FString& Options, const FUniqueNetIdRepl& UniqueId, FString& ErrorMessage)
{
    APlayerController* PC = Super::Login(NewPlayer, InRemoteRole, Portal, Options, UniqueId, ErrorMessage);

    if (PC)
    {
        UE_LOG(LogTemp, Log, TEXT("Player logged in: %s"), *PC->GetName());
    }

    return PC;
}

void AMyGameMode::PostLogin(APlayerController* NewPlayer)
{
    Super::PostLogin(NewPlayer);

    UE_LOG(LogTemp, Log, TEXT("Player joined: %s"), *NewPlayer->GetName());

    // 初始化玩家状态
    if (AMyPlayerState* PS = NewPlayer->GetPlayerState<AMyPlayerState>())
    {
        PS->SetPlayerName(NewPlayer->GetName());
        PS->SetTeam(AssignPlayerTeam(NewPlayer));
    }

    // 广播玩家加入
    OnPlayerJoined.Broadcast(NewPlayer, NewPlayer->GetName());

    // 检查是否可以开始比赛
    if (ReadyToStartMatch())
    {
        StartMatch();
    }
}

void AMyGameMode::Logout(AController* Exiting)
{
    Super::Logout(Exiting);

    if (APlayerController* PC = Cast<APlayerController>(Exiting))
    {
        UE_LOG(LogTemp, Log, TEXT("Player left: %s"), *PC->GetName());
        OnPlayerLeft.Broadcast(PC);
    }
}

bool AMyGameMode::ReadyToStartMatch_Implementation()
{
    // 至少需要2名玩家
    if (GetNumPlayers() < 2)
    {
        return false;
    }

    return !bMatchStarted;
}

void AMyGameMode::StartMatch()
{
    Super::StartMatch();

    bMatchStarted = true;
    MatchTimeRemaining = MatchDuration;

    // 启动比赛计时器
    GetWorld()->GetTimerManager().SetTimer(
        MatchTimerHandle,
        this,
        &AMyGameMode::UpdateMatchTimer,
        1.0f,
        true
    );

    BroadcastMatchStart();
    UE_LOG(LogTemp, Log, TEXT("Match started"));
}

void AMyGameMode::EndMatch()
{
    Super::EndMatch();

    bMatchStarted = false;
    GetWorld()->GetTimerManager().ClearTimer(MatchTimerHandle);

    BroadcastMatchEnd();
    UE_LOG(LogTemp, Log, TEXT("Match ended"));
}

void AMyGameMode::UpdateMatchTimer()
{
    MatchTimeRemaining -= 1.0f;

    // 更新GameState
    if (AMyGameState* GS = Cast<AMyGameState>(GameState))
    {
        GS->SetMatchTimeRemaining(MatchTimeRemaining);
    }

    // 检查是否超时
    if (MatchTimeRemaining <= 0.0f)
    {
        EndMatch();
        return;
    }

    // 定期检查胜利条件
    CheckWinCondition();
}

void AMyGameMode::CheckWinCondition()
{
    // 检查是否有玩家达到获胜分数
    for (FConstPlayerControllerIterator It = GetWorld()->GetPlayerControllerIterator(); It; ++It)
    {
        APlayerController* PC = It->Get();
        if (AMyPlayerState* PS = PC->GetPlayerState<AMyPlayerState>())
        {
            if (PS->GetScore() >= ScoreToWin)
            {
                EndMatch();
                return;
            }
        }
    }
}

void AMyGameMode::StartNewMatch()
{
    // 重置游戏状态
    if (AMyGameState* GS = Cast<AMyGameState>(GameState))
    {
        GS->ResetMatchState();
    }

    // 重置所有玩家
    for (FConstPlayerControllerIterator It = GetWorld()->GetPlayerControllerIterator(); It; ++It)
    {
        if (APlayerController* PC = It->Get())
        {
            RestartPlayer(PC);

            if (AMyPlayerState* PS = PC->GetPlayerState<AMyPlayerState>())
            {
                PS->ResetStats();
            }
        }
    }

    StartMatch();
}

void AMyGameMode::RestartMatch()
{
    // 重新加载地图
    FString MapName = GetWorld()->GetName();
    GetWorld()->ServerTravel(MapName + "?Listen");
}

AActor* AMyGameMode::ChoosePlayerStart_Implementation(AController* Player)
{
    // 根据队伍选择生成点
    TArray<AActor*> PlayerStarts;
    UGameplayStatics::GetAllActorsOfClass(this, APlayerStart::StaticClass(), PlayerStarts);

    if (PlayerStarts.Num() == 0)
    {
        return nullptr;
    }

    // 简单随机选择
    int32 Index = FMath::Rand() % PlayerStarts.Num();
    return PlayerStarts[Index];
}
```

### 4.2 GameState - 复制游戏状态

```cpp
// MyGameState.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/GameStateBase.h"
#include "MyGameState.generated.h"

// 游戏阶段
UENUM(BlueprintType)
enum class EMatchPhase : uint8
{
    WaitingForPlayers,
    Warmup,
    InProgress,
    PostMatch
};

UCLASS()
class AMyGameState : public AGameStateBase
{
    GENERATED_BODY()

public:
    AMyGameState();

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    // 游戏阶段
    UPROPERTY(ReplicatedUsing = OnRep_MatchPhase, BlueprintReadOnly, Category = "Game State")
    EMatchPhase MatchPhase;

    // 比赛时间
    UPROPERTY(ReplicatedUsing = OnRep_MatchTimeRemaining, BlueprintReadOnly, Category = "Game State")
    float MatchTimeRemaining;

    // 获胜者
    UPROPERTY(Replicated, BlueprintReadOnly, Category = "Game State")
    APlayerState* Winner;

    // 服务器信息
    UPROPERTY(Replicated, BlueprintReadOnly, Category = "Server")
    FString ServerName;

    UPROPERTY(Replicated, BlueprintReadOnly, Category = "Server")
    FString GameVersion;

    // 公共函数
    UFUNCTION(BlueprintCallable, Category = "Game State")
    void SetMatchPhase(EMatchPhase NewPhase);

    UFUNCTION(BlueprintCallable, Category = "Game State")
    void SetMatchTimeRemaining(float Time);

    UFUNCTION(BlueprintCallable, Category = "Game State")
    void SetWinner(APlayerState* PlayerState);

    UFUNCTION(BlueprintCallable, BlueprintPure, Category = "Game State")
    FString GetMatchPhaseString() const;

    void InitializeGameState();
    void ResetMatchState();

protected:
    UFUNCTION()
    void OnRep_MatchPhase();

    UFUNCTION()
    void OnRep_MatchTimeRemaining();

    // 用于广播的委托
    DECLARE_MULTICAST_DELEGATE_OneParam(FOnMatchPhaseChangedDelegate, EMatchPhase);
    FOnMatchPhaseChangedDelegate OnMatchPhaseChangedEvent;

public:
    FOnMatchPhaseChangedDelegate& OnMatchPhaseChanged() { return OnMatchPhaseChangedEvent; }
};

// MyGameState.cpp
#include "MyGameState.h"
#include "Net/UnrealNetwork.h"

AMyGameState::AMyGameState()
{
    MatchPhase = EMatchPhase::WaitingForPlayers;
    MatchTimeRemaining = 0.0f;
    ServerName = TEXT("My Server");
    GameVersion = TEXT("1.0.0");
}

void AMyGameState::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(AMyGameState, MatchPhase);
    DOREPLIFETIME(AMyGameState, MatchTimeRemaining);
    DOREPLIFETIME(AMyGameState, Winner);
    DOREPLIFETIME(AMyGameState, ServerName);
    DOREPLIFETIME(AMyGameState, GameVersion);
}

void AMyGameState::SetMatchPhase(EMatchPhase NewPhase)
{
    if (HasAuthority())
    {
        MatchPhase = NewPhase;
        OnRep_MatchPhase();
    }
}

void AMyGameState::SetMatchTimeRemaining(float Time)
{
    if (HasAuthority())
    {
        MatchTimeRemaining = Time;
    }
}

void AMyGameState::SetWinner(APlayerState* PlayerState)
{
    if (HasAuthority())
    {
        Winner = PlayerState;
        SetMatchPhase(EMatchPhase::PostMatch);
    }
}

FString AMyGameState::GetMatchPhaseString() const
{
    switch (MatchPhase)
    {
        case EMatchPhase::WaitingForPlayers:
            return TEXT("Waiting For Players");
        case EMatchPhase::Warmup:
            return TEXT("Warmup");
        case EMatchPhase::InProgress:
            return TEXT("In Progress");
        case EMatchPhase::PostMatch:
            return TEXT("Post Match");
        default:
            return TEXT("Unknown");
    }
}

void AMyGameState::OnRep_MatchPhase()
{
    OnMatchPhaseChangedEvent.Broadcast(MatchPhase);
    UE_LOG(LogTemp, Log, TEXT("Match phase changed to: %s"), *GetMatchPhaseString());
}

void AMyGameState::OnRep_MatchTimeRemaining()
{
    // 客户端可以在这里更新UI
}

void AMyGameState::InitializeGameState()
{
    MatchPhase = EMatchPhase::WaitingForPlayers;
    UE_LOG(LogTemp, Log, TEXT("Game state initialized"));
}

void AMyGameState::ResetMatchState()
{
    MatchPhase = EMatchPhase::WaitingForPlayers;
    MatchTimeRemaining = 0.0f;
    Winner = nullptr;
}
```

---

## 五、PlayerController服务器逻辑

### 5.1 服务器端PlayerController

```cpp
// MyPlayerController.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/PlayerController.h"
#include "MyPlayerController.generated.h"

UCLASS()
class AMyPlayerController : public APlayerController
{
    GENERATED_BODY()

public:
    AMyPlayerController();

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
    virtual void BeginPlay() override;
    virtual void Tick(float DeltaTime) override;

    // 输入处理
    virtual void SetupInputComponent() override;

    // 服务器RPC
    UFUNCTION(Server, Reliable, WithValidation)
    void Server_RequestRespawn();

    UFUNCTION(Server, Reliable, WithValidation)
    void Server_RequestTeamChange(int32 NewTeamId);

    UFUNCTION(Server, Reliable, WithValidation)
    void Server_SendChatMessage(const FString& Message);

    UFUNCTION(Server, Reliable, WithValidation)
    void Server_SetReady(bool bReady);

    // 客户端RPC
    UFUNCTION(Client, Reliable)
    void Client_ShowMessage(const FString& Title, const FString& Message);

    UFUNCTION(Client, Reliable)
    void Client_UpdateMatchInfo(float TimeRemaining, int32 PlayerCount);

    UFUNCTION(Client, Reliable)
    void Client_OnMatchEnded(APlayerState* Winner);

    UFUNCTION(Client, Reliable)
    void Client_KickFromServer(const FString& Reason);

    // 公共函数
    UFUNCTION(BlueprintCallable, Category = "Game")
    void RespawnPlayer();

    UFUNCTION(BlueprintCallable, Category = "Game")
    void SendMessage(const FString& Message);

    // 查询函数
    UFUNCTION(BlueprintPure, Category = "Game")
    bool IsLocalPlayer() const;

protected:
    // 输入绑定
    void MoveForward(float Value);
    void MoveRight(float Value);
    void LookUp(float Value);
    void Turn(float Value);
    void Jump();
    void StopJumping();
    void Fire();
    void Reload();

    // 服务器端处理
    void HandlePlayerDeath();
    void HandlePlayerRespawn();

    // 复制属性
    UPROPERTY(Replicated)
    bool bIsReady;

    UPROPERTY(ReplicatedUsing = OnRep_TeamId)
    int32 TeamId;

    UFUNCTION()
    void OnRep_TeamId();

private:
    FTimerHandle RespawnTimerHandle;
    float RespawnDelay = 5.0f;
};

// MyPlayerController.cpp
#include "MyPlayerController.h"
#include "MyCharacter.h"
#include "MyGameMode.h"
#include "MyGameState.h"
#include "Net/UnrealNetwork.h"

AMyPlayerController::AMyPlayerController()
{
    bIsReady = false;
    TeamId = -1;
}

void AMyPlayerController::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(AMyPlayerController, bIsReady);
    DOREPLIFETIME(AMyPlayerController, TeamId);
}

void AMyPlayerController::BeginPlay()
{
    Super::BeginPlay();

    if (IsLocalController())
    {
        // 客户端初始化
        UE_LOG(LogTemp, Log, TEXT("Local PlayerController initialized"));
    }
}

void AMyPlayerController::SetupInputComponent()
{
    Super::SetupInputComponent();

    // 移动输入
    InputComponent->BindAxis("MoveForward", this, &AMyPlayerController::MoveForward);
    InputComponent->BindAxis("MoveRight", this, &AMyPlayerController::MoveRight);
    InputComponent->BindAxis("LookUp", this, &AMyPlayerController::LookUp);
    InputComponent->BindAxis("Turn", this, &AMyPlayerController::Turn);

    // 动作输入
    InputComponent->BindAction("Jump", IE_Pressed, this, &AMyPlayerController::Jump);
    InputComponent->BindAction("Jump", IE_Released, this, &AMyPlayerController::StopJumping);
    InputComponent->BindAction("Fire", IE_Pressed, this, &AMyPlayerController::Fire);
    InputComponent->BindAction("Reload", IE_Pressed, this, &AMyPlayerController::Reload);
}

void AMyPlayerController::MoveForward(float Value)
{
    if (GetPawn())
    {
        GetPawn()->AddMovementInput(GetPawn()->GetActorForwardVector(), Value);
    }
}

void AMyPlayerController::MoveRight(float Value)
{
    if (GetPawn())
    {
        GetPawn()->AddMovementInput(GetPawn()->GetActorRightVector(), Value);
    }
}

void AMyPlayerController::LookUp(float Value)
{
    AddPitchInput(Value);
}

void AMyPlayerController::Turn(float Value)
{
    AddYawInput(Value);
}

void AMyPlayerController::Jump()
{
    if (GetPawn())
    {
        GetPawn()->Jump();
    }
}

void AMyPlayerController::StopJumping()
{
    if (GetPawn())
    {
        GetPawn()->StopJumping();
    }
}

void AMyPlayerController::Fire()
{
    if (AMyCharacter* Character = Cast<AMyCharacter>(GetPawn()))
    {
        Character->Fire();
    }
}

void AMyPlayerController::Reload()
{
    if (AMyCharacter* Character = Cast<AMyCharacter>(GetPawn()))
    {
        Character->Reload();
    }
}

// ========== 服务器RPC实现 ==========

bool AMyPlayerController::Server_RequestRespawn_Validate()
{
    return true;
}

void AMyPlayerController::Server_RequestRespawn_Implementation()
{
    RespawnPlayer();
}

bool AMyPlayerController::Server_RequestTeamChange_Validate(int32 NewTeamId)
{
    return NewTeamId >= 0 && NewTeamId < 4; // 假设最多4个队伍
}

void AMyPlayerController::Server_RequestTeamChange_Implementation(int32 NewTeamId)
{
    TeamId = NewTeamId;

    // 通知GameMode
    if (AMyGameMode* GM = Cast<AMyGameMode>(GetWorld()->GetAuthGameMode()))
    {
        // GM->OnPlayerTeamChanged(this, NewTeamId);
    }
}

bool AMyPlayerController::Server_SendChatMessage_Validate(const FString& Message)
{
    return Message.Len() > 0 && Message.Len() < 256;
}

void AMyPlayerController::Server_SendChatMessage_Implementation(const FString& Message)
{
    // 广播聊天消息给所有玩家
    for (FConstPlayerControllerIterator It = GetWorld()->GetPlayerControllerIterator(); It; ++It)
    {
        if (APlayerController* PC = It->Get())
        {
            // PC->Client_ReceiveChatMessage(PlayerState->GetPlayerName(), Message);
        }
    }
}

bool AMyPlayerController::Server_SetReady_Validate(bool bReady)
{
    return true;
}

void AMyPlayerController::Server_SetReady_Implementation(bool bReady)
{
    bIsReady = bReady;
}

// ========== 客户端RPC实现 ==========

void AMyPlayerController::Client_ShowMessage_Implementation(const FString& Title, const FString& Message)
{
    // 显示消息给玩家
    UE_LOG(LogTemp, Log, TEXT("%s: %s"), *Title, *Message);

    // 更新UI
    if (IsLocalController())
    {
        // UMyHUD* HUD = Cast<UMyHUD>(GetHUD());
        // HUD->ShowMessage(Title, Message);
    }
}

void AMyPlayerController::Client_UpdateMatchInfo_Implementation(float TimeRemaining, int32 PlayerCount)
{
    if (IsLocalController())
    {
        // 更新UI显示
        UE_LOG(LogTemp, Log, TEXT("Match time: %.0f, Players: %d"), TimeRemaining, PlayerCount);
    }
}

void AMyPlayerController::Client_OnMatchEnded_Implementation(APlayerState* WinnerState)
{
    if (IsLocalController())
    {
        FString WinnerName = WinnerState ? WinnerState->GetPlayerName() : TEXT("None");
        Client_ShowMessage(TEXT("Match Ended"), FString::Printf(TEXT("Winner: %s"), *WinnerName));
    }
}

void AMyPlayerController::Client_KickFromServer_Implementation(const FString& Reason)
{
    Client_ShowMessage(TEXT("Kicked"), Reason);

    // 延迟返回主菜单
    FTimerHandle KickTimer;
    GetWorld()->GetTimerManager().SetTimer(KickTimer, [this]()
    {
        ClientReturnToMainMenuWithTextReason(FText::FromString("Kicked from server"));
    }, 2.0f, false);
}

// ========== 公共函数 ==========

void AMyPlayerController::RespawnPlayer()
{
    if (HasAuthority())
    {
        // 销毁当前Pawn
        if (GetPawn())
        {
            GetPawn()->Destroy();
        }

        // 请求GameMode生成新Pawn
        if (AMyGameMode* GM = Cast<AMyGameMode>(GetWorld()->GetAuthGameMode()))
        {
            GM->RestartPlayer(this);
        }
    }
    else
    {
        Server_RequestRespawn();
    }
}

void AMyPlayerController::SendMessage(const FString& Message)
{
    Server_SendChatMessage(Message);
}

bool AMyPlayerController::IsLocalPlayer() const
{
    return IsLocalController();
}

void AMyPlayerController::OnRep_TeamId()
{
    // 客户端响应队伍变化
    UE_LOG(LogTemp, Log, TEXT("Team changed to: %d"), TeamId);
}

// ========== 死亡处理 ==========

void AMyPlayerController::HandlePlayerDeath()
{
    if (!HasAuthority())
    {
        return;
    }

    // 开始重生计时
    GetWorld()->GetTimerManager().SetTimer(
        RespawnTimerHandle,
        this,
        &AMyPlayerController::HandlePlayerRespawn,
        RespawnDelay,
        false
    );

    // 通知客户端
    Client_ShowMessage(TEXT("You Died"), FString::Printf(TEXT("Respawning in %.0f seconds..."), RespawnDelay));
}

void AMyPlayerController::HandlePlayerRespawn()
{
    RespawnPlayer();
}
```

---

## 六、服务器与客户端角色分工

### 6.1 角色分工表

| 功能 | 服务器 | 客户端 |
|------|--------|--------|
| 游戏规则判断 | ✓ 执行 | ✗ |
| 伤害计算 | ✓ 计算 | ✗ |
| AI决策 | ✓ 执行 | ✗ |
| 物理模拟 | ✓ 主模拟 | 预测/渲染 |
| 移动 | 验证 | 预测+请求 |
| 动画 | 状态同步 | 播放 |
| 音效 | 广播触发 | 播放 |
| UI | ✗ | ✓ 更新 |
| 输入处理 | 接收验证 | 本地处理 |

### 6.2 代码分工模式

```cpp
// 角色移动示例 - 展示服务器和客户端分工

void AMyCharacter::Move(const FVector& Direction, float Value)
{
    if (Value != 0.0f)
    {
        if (HasAuthority())
        {
            // 服务器：执行并验证移动
            FVector NewLocation = GetActorLocation() + Direction * Value * MoveSpeed * DeltaTime;

            // 验证新位置是否合法
            if (IsValidLocation(NewLocation))
            {
                SetActorLocation(NewLocation);
            }
        }
        else
        {
            // 客户端：预测移动
            AddMovementInput(Direction, Value);

            // 发送输入到服务器
            Server_Move(Direction, Value);
        }
    }
}

// 射击示例
void AMyCharacter::Fire()
{
    if (HasAuthority())
    {
        // 服务器：执行射击逻辑
        PerformFire();
    }
    else
    {
        // 客户端：播放本地效果并请求服务器
        PlayFireAnimation();
        PlayFireSound();
        Server_Fire();
    }
}

void AMyCharacter::Server_Fire_Implementation()
{
    // 服务器验证并执行射击
    if (CanFire())
    {
        PerformFire();

        // 广播给所有客户端播放效果
        Multicast_PlayFireEffects();
    }
}

void AMyCharacter::Multicast_PlayFireEffects_Implementation()
{
    // 所有客户端播放效果
    if (!IsLocallyControlled())
    {
        PlayFireAnimation();
        PlayFireSound();
    }
}
```

---

## 七、实践任务

### 任务：搭建完整的Dedicated Server项目

1. 创建服务器Target配置
2. 实现自定义GameMode和GameState
3. 配置服务器启动参数
4. 测试多客户端连接

---

## 八、总结

本课我们学习了：

1. **编译部署**：Dedicated Server的编译和打包方法
2. **启动参数**：URL参数和命令行参数配置
3. **GameMode**：服务器权威的游戏规则管理
4. **GameState**：复制游戏状态给所有客户端
5. **PlayerController**：服务器端逻辑处理

---

## 九、下节预告

**第7课：服务器框架设计**

将深入学习：
- GameInstance持久化管理
- PlayerState生命周期
- 断线重连机制
- 服务器配置管理

---

*课程版本：1.0*
*最后更新：2026-04-10*