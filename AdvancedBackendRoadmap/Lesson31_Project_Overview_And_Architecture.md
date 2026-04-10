# 第31课：实战项目概述与架构设计

> 学习目标：了解实战项目的整体架构，设计多人在线竞技游戏的技术方案。

---

## 31.1 项目概述

### 31.1.1 项目目标

**项目名称**：Arena Champions（竞技冠军）

**项目描述**：开发一个支持多人在线的竞技游戏Demo，包含完整的后端服务系统。

**核心功能**：
- 用户注册与登录
- 匹配系统（1v1, 2v2, 3v3）
- 游戏房间管理
- 实时对战
- 排行榜系统
- 成就系统
- 好友系统
- 聊天系统

### 31.1.2 技术栈

| 层级 | 技术选型 |
|------|----------|
| 游戏引擎 | Unreal Engine 5.3+ |
| 编程语言 | C++ / Blueprints |
| 服务器 | Dedicated Server |
| 数据库 | PostgreSQL / Redis |
| 后端服务 | Node.js / Go |
| 云服务 | AWS / PlayFab |
| 容器化 | Docker / Kubernetes |
| 监控 | Prometheus / Grafana |

---

## 31.2 架构设计

### 31.2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                           客户端层                                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                  │
│  │  PC Client  │  │ Console     │  │  Mobile     │                  │
│  └─────────────┘  └─────────────┘  └─────────────┘                  │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         接入层 (API Gateway)                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                  │
│  │   Kong API  │  │   Load      │  │    CDN      │                  │
│  │   Gateway   │  │   Balancer  │  │             │                  │
│  └─────────────┘  └─────────────┘  └─────────────┘                  │
└─────────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  Auth Service │   │ Match Service │   │ Game Servers  │
│   认证服务     │   │   匹配服务     │   │   游戏服务器   │
└───────────────┘   └───────────────┘   └───────────────┘
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ User Database │   │ Match Redis   │   │ Game State    │
│   用户数据库   │   │   匹配缓存     │   │   游戏状态     │
└───────────────┘   └───────────────┘   └───────────────┘

┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│Social Service │   │ Leaderboard   │   │  Analytics    │
│   社交服务     │   │   排行榜       │   │   分析服务     │
└───────────────┘   └───────────────┘   └───────────────┘
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ Social Database│  │ TimescaleDB   │   │ Elasticsearch │
│   社交数据库   │   │   时序数据库   │   │   搜索引擎     │
└───────────────┘   └───────────────┘   └───────────────┘
```

### 31.2.2 数据流设计

```
用户登录流程:
1. 客户端 -> API Gateway -> Auth Service
2. Auth Service -> 验证凭证 -> 生成JWT Token
3. Auth Service -> 加载用户数据 -> 返回Token和用户信息
4. 客户端存储Token，用于后续请求

匹配流程:
1. 客户端 -> 发起匹配请求 -> Match Service
2. Match Service -> 加入匹配队列 -> Redis
3. Match Service -> 定期匹配 -> 创建匹配
4. Match Service -> 分配游戏服务器 -> 返回连接信息
5. 客户端 -> 连接游戏服务器 -> 开始游戏

游戏流程:
1. 客户端连接 -> Game Server验证
2. 加载关卡 -> 同步状态
3. 游戏进行 -> 实时复制
4. 游戏结束 -> 上报结果 -> 更新排行榜
```

---

## 31.3 项目结构

### 31.3.1 游戏项目结构

```
ArenaChampions/
├── Source/
│   ├── ArenaChampions/
│   │   ├── Core/
│   │   │   ├── ArenaGameInstance.h/cpp
│   │   │   ├── ArenaGameMode.h/cpp
│   │   │   ├── ArenaGameState.h/cpp
│   │   │   └── ArenaPlayerController.h/cpp
│   │   ├── Player/
│   │   │   ├── ArenaPlayerState.h/cpp
│   │   │   ├── ArenaCharacter.h/cpp
│   │   │   └── ArenaPawn.h/cpp
│   │   ├── Matchmaking/
│   │   │   ├── MatchmakingClient.h/cpp
│   │   │   ├── MatchmakingTypes.h
│   │   │   └── MatchmakingQueue.h/cpp
│   │   ├── GameModes/
│   │   │   ├── ArenaGameMode_Deathmatch.h/cpp
│   │   │   ├── ArenaGameMode_TeamBattle.h/cpp
│   │   │   └── ArenaGameMode_Ranked.h/cpp
│   │   ├── Weapons/
│   │   │   ├── ArenaWeapon.h/cpp
│   │   │   ├── ArenaProjectile.h/cpp
│   │   │   └── ArenaDamageType.h/cpp
│   │   ├── UI/
│   │   │   ├── ArenaHUD.h/cpp
│   │   │   ├── ArenaMainMenu.h/cpp
│   │   │   └── ArenaScoreboard.h/cpp
│   │   ├── Network/
│   │   │   ├── NetworkManager.h/cpp
│   │   │   ├── ReplicationManager.h/cpp
│   │   │   └── LatencyCompensation.h/cpp
│   │   └── Subsystems/
│   │       ├── AuthSubsystem.h/cpp
│   │       ├── SocialSubsystem.h/cpp
│   │       └── AnalyticsSubsystem.h/cpp
│   └── ArenaChampionsServer/
│       ├── ServerGameMode.h/cpp
│       ├── ServerManager.h/cpp
│       └── ServerMetrics.h/cpp
├── Content/
│   ├── Maps/
│   │   ├── MainMenu.umap
│   │   ├── Arena_01.umap
│   │   ├── Arena_02.umap
│   │   └── Lobby.umap
│   ├── Characters/
│   ├── Weapons/
│   ├── UI/
│   ├── Audio/
│   └── Effects/
├── Plugins/
│   ├── OnlineSubsystem/
│   ├── PlayFab/
│   └── CustomNetworking/
└── Config/
    ├── DefaultEngine.ini
    ├── DefaultGame.ini
    └── DefaultServer.ini
```

### 31.3.2 后端服务结构

```
ArenaBackend/
├── services/
│   ├── auth-service/
│   │   ├── src/
│   │   │   ├── controllers/
│   │   │   ├── services/
│   │   │   ├── models/
│   │   │   └── middleware/
│   │   ├── Dockerfile
│   │   └── package.json
│   ├── match-service/
│   │   ├── src/
│   │   │   ├── queue/
│   │   │   ├── matcher/
│   │   │   └── allocator/
│   │   ├── Dockerfile
│   │   └── package.json
│   ├── social-service/
│   │   ├── src/
│   │   │   ├── friends/
│   │   │   ├── chat/
│   │   │   └── presence/
│   │   └── Dockerfile
│   ├── leaderboard-service/
│   │   ├── src/
│   │   │   ├── rankings/
│   │   │   ├── seasons/
│   │   │   └── rewards/
│   │   └── Dockerfile
│   └── gateway/
│       ├── kong.yml
│       └── docker-compose.yml
├── infrastructure/
│   ├── kubernetes/
│   │   ├── deployments/
│   │   ├── services/
│   │   └── configmaps/
│   ├── terraform/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── docker/
│       ├── docker-compose.yml
│       └── Dockerfile.server
├── shared/
│   ├── proto/
│   │   └── arena.proto
│   └── libs/
└── scripts/
    ├── deploy.sh
    ├── backup.sh
    └── monitor.sh
```

---

## 31.4 核心类设计

### 31.4.1 游戏实例类

```csharp
// ArenaGameInstance.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/GameInstance.h"
#include "Subsystems/SubsystemCollection.h"

#include "ArenaGameInstance.generated.h"

class UAuthSubsystem;
class UMatchmakingSubsystem;
class USocialSubsystem;
class UAnalyticsSubsystem;

UCLASS()
class ARENACHAMPIONS_API UArenaGameInstance : public UGameInstance
{
    GENERATED_BODY()

public:
    virtual void Init() override;
    virtual void Shutdown() override;

    // 获取子系统
    UFUNCTION(BlueprintPure, Category = "Arena")
    UAuthSubsystem* GetAuthSubsystem() const;

    UFUNCTION(BlueprintPure, Category = "Arena")
    UMatchmakingSubsystem* GetMatchmakingSubsystem() const;

    UFUNCTION(BlueprintPure, Category = "Arena")
    USocialSubsystem* GetSocialSubsystem() const;

    // 游戏状态
    UFUNCTION(BlueprintPure, Category = "Arena")
    bool IsLoggedIn() const;

    UFUNCTION(BlueprintPure, Category = "Arena")
    FString GetCurrentUserId() const;

    // 配置
    UPROPERTY(EditDefaultsOnly, Category = "Arena")
    FString ApiBaseUrl;

    UPROPERTY(EditDefaultsOnly, Category = "Arena")
    FString WebSocketUrl;

    UPROPERTY(EditDefaultsOnly, Category = "Arena")
    FString GameVersion;

protected:
    // 子系统指针缓存
    UPROPERTY()
    UAuthSubsystem* AuthSubsystem;

    UPROPERTY()
    UMatchmakingSubsystem* MatchmakingSubsystem;

    UPROPERTY()
    USocialSubsystem* SocialSubsystem;

    UPROPERTY()
    UAnalyticsSubsystem* AnalyticsSubsystem;

private:
    void InitializeSubsystems();
    void LoadConfiguration();
};
```

### 31.4.2 游戏模式基类

```csharp
// ArenaGameMode.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/GameModeBase.h"

#include "ArenaGameMode.generated.h"

class AArenaGameState;
class AArenaPlayerState;
class AArenaPlayerController;

UENUM(BlueprintType)
enum class EMatchState : uint8
{
    WaitingForPlayers,
    Warmup,
    InProgress,
    RoundEnd,
    MatchEnd,
    PostMatch
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnMatchStateChanged, EMatchState, NewState);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnPlayerJoined, AArenaPlayerController*, Player);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnPlayerKilled, APlayerState*, Killer, APlayerState*, Victim);

UCLASS()
class ARENACHAMPIONS_API AArenaGameMode : public AGameModeBase
{
    GENERATED_BODY()

public:
    AArenaGameMode();

    virtual void BeginPlay() override;
    virtual void Tick(float DeltaTime) override;
    virtual void PostLogin(APlayerController* NewPlayer) override;
    virtual void Logout(AController* Exiting) override;

    // 匹配状态管理
    UFUNCTION(BlueprintCallable, Category = "Match")
    void SetMatchState(EMatchState NewState);

    UFUNCTION(BlueprintPure, Category = "Match")
    EMatchState GetMatchState() const { return CurrentMatchState; }

    // 玩家管理
    UFUNCTION(BlueprintCallable, Category = "Match")
    void PlayerReady(AArenaPlayerController* Player);

    UFUNCTION(BlueprintPure, Category = "Match")
    int32 GetConnectedPlayerCount() const;

    UFUNCTION(BlueprintPure, Category = "Match")
    int32 GetReadyPlayerCount() const;

    // 游戏逻辑
    UFUNCTION(BlueprintCallable, Category = "Match")
    void StartMatch();

    UFUNCTION(BlueprintCallable, Category = "Match")
    void EndMatch();

    UFUNCTION(BlueprintCallable, Category = "Match")
    void RespawnPlayer(AArenaPlayerController* Player);

    // 得分系统
    UFUNCTION(BlueprintCallable, Category = "Match")
    void AddScore(APlayerState* Player, float Score);

    UFUNCTION(BlueprintCallable, Category = "Match")
    void RecordKill(APlayerState* Killer, APlayerState* Victim);

    // 配置
    UPROPERTY(EditDefaultsOnly, Category = "Match")
    int32 MinPlayersToStart = 2;

    UPROPERTY(EditDefaultsOnly, Category = "Match")
    int32 MaxPlayers = 8;

    UPROPERTY(EditDefaultsOnly, Category = "Match")
    float WarmupDuration = 30.0f;

    UPROPERTY(EditDefaultsOnly, Category = "Match")
    float MatchDuration = 600.0f;  // 10分钟

    UPROPERTY(EditDefaultsOnly, Category = "Match")
    int32 ScoreToWin = 25;

    UPROPERTY(EditDefaultsOnly, Category = "Match")
    float RespawnDelay = 5.0f;

    // 事件
    UPROPERTY(BlueprintAssignable, Category = "Match")
    FOnMatchStateChanged OnMatchStateChanged;

    UPROPERTY(BlueprintAssignable, Category = "Match")
    FOnPlayerJoined OnPlayerJoined;

    UPROPERTY(BlueprintAssignable, Category = "Match")
    FOnPlayerKilled OnPlayerKilled;

protected:
    EMatchState CurrentMatchState = EMatchState::WaitingForPlayers;
    float MatchTimer = 0.0f;
    FDateTime MatchStartTime;
    TMap<APlayerState*, float> PlayerScores;

    void HandleWaitingForPlayers();
    void HandleWarmup(float DeltaTime);
    void HandleInProgress(float DeltaTime);
    void HandleMatchEnd();
    void CheckWinCondition();
    void BroadcastMatchResults();
};
```

---

## 31.5 开发环境搭建

### 31.5.1 必要软件

```
开发环境:
- Unreal Engine 5.3+
- Visual Studio 2022 / Rider for Unreal
- Git
- Docker Desktop
- Node.js 18+
- PostgreSQL 15+
- Redis 7+

工具:
- Postman (API测试)
- DBeaver (数据库管理)
- RedisInsight (Redis管理)
- Wireshark (网络分析)
```

### 31.5.2 配置文件

```ini
; DefaultEngine.ini
[OnlineSubsystem]
DefaultPlatformService=Default

[OnlineSubsystemSteam]
bEnabled=true
SteamDevAppId=480
GameServerQueryPort=27015

[/Script/Engine.GameEngine]
+NetDriverDefinitions=(DefName="GameNetDriver",DriverClassName="OnlineSubsystemUtils.IpNetDriver",DriverClassNameFallback="OnlineSubsystemUtils.IpNetDriver")

[/Script/OnlineSubsystemUtils.IpNetDriver]
NetConnectionClassName="OnlineSubsystemUtils.IpConnection"
InitialConnectTimeout=120.0f
ConnectionTimeout=120.0f

[/Script/Engine.NetworkSettings]
NetServerMaxTickRate=60
MaxClientUpdateRate=60
MaxClientRate=15000
MaxInternetClientRate=15000

[/Script/ArenaChampions.ArenaGameInstance]
ApiBaseUrl=http://localhost:8080/api
WebSocketUrl=ws://localhost:8080/ws
GameVersion=1.0.0
```

---

## 31.6 开发计划

### 31.6.1 里程碑

| 阶段 | 内容 | 预计时间 |
|------|------|----------|
| 第31课 | 项目架构设计 | 本课 |
| 第32课 | 用户系统实现 | 下一课 |
| 第33课 | 匹配系统开发 | 待完成 |
| 第34课 | 游戏房间逻辑 | 待完成 |
| 第35课 | 结果统计与排行 | 待完成 |
| 第36课 | 微服务架构搭建 | 待完成 |
| 第37课 | 数据库集群部署 | 待完成 |
| 第38课 | 缓存层设计 | 待完成 |
| 第39课 | CI/CD流水线 | 待完成 |
| 第40课 | 生产环境部署 | 待完成 |

---

## 31.7 实践任务

### 任务1：创建项目

1. 创建新的UE5 C++项目
2. 配置项目结构
3. 设置版本控制

### 任务2：配置开发环境

1. 安装必要工具
2. 配置Docker环境
3. 搭建本地数据库

### 任务3：设计数据模型

设计以下数据模型：
- User（用户）
- Match（比赛）
- PlayerStats（玩家统计）
- Friend（好友关系）

---

## 31.8 总结

本课完成了：
- 项目整体规划
- 技术架构设计
- 项目结构搭建
- 核心类设计
- 开发环境配置
- 开发计划制定

**下一课预告**：用户系统实现 - 注册、登录、认证和用户数据管理。

---

*课程版本：1.0*
*最后更新：2026-04-10*
