# 第1课：UE5网络架构总览

> 本课将全面介绍虚幻引擎5的网络架构基础，帮助开发者理解UE5如何实现多人游戏的底层机制。

---

## 课程目标

- 理解客户端-服务器模型与对等网络模型的区别
- 掌握UE5网络架构的核心组件及其职责
- 深入理解四种NetMode的工作原理
- 了解网络连接的完整生命周期
- 熟悉网络驱动架构的设计理念

---

## 一、网络模型基础

### 1.1 客户端-服务器模型（Client-Server Model）

客户端-服务器模型是UE5多人游戏的核心架构模式。

```
┌─────────────────────────────────────────────────────────┐
│                     Dedicated Server                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │   GameMode  │  │  GameState  │  │   World     │      │
│  │  (权威逻辑)  │  │ (全局状态)   │  │  (世界数据)  │      │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
│                         ▲                                 │
│                         │ 网络连接                        │
│         ┌───────────────┼───────────────┐                │
│         ▼               ▼               ▼                │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐           │
│  │ Client 1 │    │ Client 2 │    │ Client 3 │           │
│  │ PlayerCtl│    │ PlayerCtl│    │ PlayerCtl│           │
│  │ (本地预测)│    │ (本地预测)│    │ (本地预测)│           │
│  └──────────┘    └──────────┘    └──────────┘           │
└─────────────────────────────────────────────────────────┘
```

**核心特点：**

| 特性 | 说明 |
|------|------|
| **服务器权威** | 所有游戏状态的最终决定权在服务器 |
| **状态同步** | 服务器将状态变化同步给所有客户端 |
| **客户端预测** | 客户端预测本地操作结果，等待服务器确认 |
| **安全性** | 客户端无法直接修改游戏状态，防止作弊 |

**优势：**
- 安全性高，防止客户端作弊
- 状态一致性好，服务器是唯一真相来源
- 易于实现反作弊机制

**劣势：**
- 服务器成本较高
- 对服务器网络质量要求高
- 单点故障风险

### 1.2 对等网络模型（Peer-to-Peer Model）

P2P模型中，每个客户端同时扮演客户端和服务器的角色。

```
┌──────────┐         ┌──────────┐
│  Peer A  │◄───────►│  Peer B  │
│ (Host)   │         │ (Client) │
└────┬─────┘         └────┬─────┘
     │                    │
     │    ┌──────────┐    │
     └───►│  Peer C  │◄───┘
          │ (Client) │
          └──────────┘
```

**UE5中的P2P实现：**

UE5通过Listen Server实现类似P2P的模式，其中一名玩家同时作为服务器。

```cpp
// Listen Server 配置示例
// 在命令行启动时指定
// UE5Editor.exe MyProject MyMap?Listen -log

// 或在代码中开启服务器监听
void AMyGameInstance::StartListenServer()
{
    // 创建监听服务器
    UWorld* World = GetWorld();
    if (World)
    {
        World->Listen(true);
    }
}
```

**适用场景：**
- 小型合作游戏（2-4人）
- 局域网游戏
- 快速原型开发

**不推荐场景：**
- 竞技类游戏（安全性不足）
- 大型多人游戏（性能瓶颈）

---

## 二、UE5网络架构核心组件

### 2.1 架构总览图

```
┌────────────────────────────────────────────────────────────────┐
│                        Game Instance                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    Engine Core                            │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │  │
│  │  │  UEngine    │  │ UWorld      │  │ UNetDriver  │      │  │
│  │  │  (引擎核心)  │  │ (世界管理)   │  │ (网络驱动)   │      │  │
│  │  └─────────────┘  └─────────────┘  └──────┬──────┘      │  │
│  │                                          │               │  │
│  │  ┌───────────────────────────────────────┴────────────┐  │  │
│  │  │                Network Layer                        │  │  │
│  │  │  ┌──────────────┐  ┌──────────────┐               │  │  │
│  │  │  │ UNetConnection│  │ UPackageMap  │               │  │  │
│  │  │  │ (连接管理)     │  │ (对象映射)    │               │  │  │
│  │  │  └──────────────┘  └──────────────┘               │  │  │
│  │  │  ┌──────────────┐  ┌──────────────┐               │  │  │
│  │  │  │ UActorChannel│  │ FObjectReplicator│            │  │  │
│  │  │  │ (Actor通道)   │  │ (复制器)       │               │  │  │
│  │  │  └──────────────┘  └──────────────┘               │  │  │
│  │  └───────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

### 2.2 核心组件详解

#### UNetDriver（网络驱动）

网络驱动是UE5网络层的核心，负责管理所有网络连接和数据传输。

```cpp
// Engine/Source/Runtime/Engine/Classes/Engine/NetDriver.h

class ENGINE_API UNetDriver : public UObject
{
    // 关键成员变量
protected:
    /** 网络连接列表 */
    TArray<class UNetConnection*> ClientConnections;

    /** 服务器连接（客户端使用） */
    class UNetConnection* ServerConnection;

    /** 是否为服务器 */
    bool bIsServer;

    /** 世界指针 */
    UWorld* World;

    /** Package Map - 用于序列化对象引用 */
    class UPackageMap* PackageMap;

public:
    // 关键方法

    /** 初始化网络驱动 */
    virtual bool InitConnect(FNetworkNotify* InNotify,
                            const FURL& ConnectURL,
                            FString& Error);

    /** 初始化服务器监听 */
    virtual bool InitListen(FNetworkNotify* InNotify,
                           const FURL& ListenURL,
                           FString& Error);

    /** 处理所有连接的Tick */
    virtual void TickDispatch(float DeltaSeconds);

    /** 接收网络数据 */
    virtual void ProcessRemoteFunction(class AActor* Actor,
                                       UFunction* Function,
                                       void* Parameters,
                                       FOutParmRec* OutParms,
                                       FFrame* Stack,
                                       class UObject* SubObject);
};
```

**派生类：**

| 类名 | 用途 | 默认端口 |
|------|------|----------|
| UIpNetDriver | 基于TCP/UDP的IP网络驱动 | 7777 |
| UWebSocketNetDriver | WebSocket网络驱动（Web支持） | 80/443 |
| USteamNetDriver | Steam P2P网络驱动 | Steam协议 |
| UEpicNetDriver | Epic Online Services驱动 | EOS协议 |

#### UNetConnection（网络连接）

每个客户端与服务器之间的连接由UNetConnection管理。

```cpp
// Engine/Source/Runtime/Engine/Classes/Engine/NetConnection.h

class ENGINE_API UNetConnection : public UObject
{
protected:
    /** 连接状态 */
    enum EConnectionState State;

    /** 关联的NetDriver */
    UNetDriver* Driver;

    /** Actor通道列表 */
    TMap<int32, class UActorChannel*> ActorChannels;

    /** 远程地址 */
    FUniqueNetIdRepl PlayerId;

    /** 连接URL */
    FURL URL;

public:
    /** 发送RPC */
    virtual void SendRpc(AActor* Actor,
                        UFunction* Function,
                        void* Parameters);

    /** 关闭连接 */
    virtual void Close();

    /** 获取连接延迟 */
    float GetAvgLag() const;
};
```

#### UActorChannel（Actor通道）

Actor通道负责单个Actor的所有网络复制通信。

```cpp
// Engine/Source/Runtime/Engine/Classes/Engine/ActorChannel.h

class ENGINE_API UActorChannel : public UChannel
{
protected:
    /** 关联的Actor */
    AActor* Actor;

    /** 复制器 */
    class FObjectReplicator* Replicator;

    /** 通道索引 */
    int32 ChannelIndex;

public:
    /** 复制Actor */
    virtual bool ReplicateActor();

    /** 接收属性更新 */
    virtual void ReceivedBunch(FInBunch& Bunch);
};
```

### 2.3 数据流转过程

```
┌─────────────────────────────────────────────────────────────┐
│                        服务器端                               │
│                                                              │
│  1. 游戏逻辑更新 Actor 属性                                   │
│     │                                                        │
│     ▼                                                        │
│  2. ReplicationGraph 确定需要复制的 Actor                     │
│     │                                                        │
│     ▼                                                        │
│  3. FObjectReplicator 收集属性变化                           │
│     │                                                        │
│     ▼                                                        │
│  4. UNetDriver 序列化并发送数据包                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
                           │
                           │ 网络传输
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                        客户端                                 │
│                                                              │
│  5. UNetDriver 接收数据包                                    │
│     │                                                        │
│     ▼                                                        │
│  6. UActorChannel 反序列化数据                               │
│     │                                                        │
│     ▼                                                        │
│  7. 应用属性更新，触发 OnRep 回调                             │
│     │                                                        │
│     ▼                                                        │
│  8. 游戏逻辑响应状态变化                                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 三、NetMode详解

### 3.1 四种NetMode

UE5定义了四种网络模式，决定了游戏如何处理网络通信。

```cpp
// Engine/Source/Runtime/Engine/Classes/Engine/EngineTypes.h

namespace ENetMode
{
    enum Type
    {
        NM_Standalone,      // 单机模式，无网络
        NM_DedicatedServer, // 专用服务器，无本地玩家
        NM_ListenServer,    // 监听服务器，有本地玩家
        NM_Client,          // 纯客户端
        NM_MAX,
    };
}
```

### 3.2 详细对比

| NetMode | 本地玩家 | 渲染 | 执行服务器逻辑 | 典型应用 |
|---------|---------|------|--------------|---------|
| **Standalone** | 有 | 有 | 执行所有逻辑 | 单机游戏、离线模式 |
| **DedicatedServer** | 无 | 无 | 执行所有逻辑 | 竞技游戏服务器 |
| **ListenServer** | 有（作为Host） | 有 | 执行所有逻辑 | 合作游戏、LAN Party |
| **Client** | 有 | 有 | 仅客户端逻辑 | 联机游戏客户端 |

### 3.3 代码示例：检测当前NetMode

```cpp
// AActor.h 中提供的方法
ENetMode AActor::GetNetMode() const;

// 实用宏定义
#define IS_STANDALONE(World) ((World) && (World)->IsStandalone())
#define IS_DEDICATED_SERVER(World) ((World) && (World)->IsNetMode(NM_DedicatedServer))
#define IS_LISTEN_SERVER(World) ((World) && (World)->IsNetMode(NM_ListenServer))
#define IS_CLIENT(World) ((World) && (World)->IsNetMode(NM_Client))

// 在Actor中使用
void AMyCharacter::DebugNetMode()
{
    switch (GetNetMode())
    {
        case NM_Standalone:
            UE_LOG(LogTemp, Log, TEXT("Running in Standalone mode"));
            break;
        case NM_DedicatedServer:
            UE_LOG(LogTemp, Log, TEXT("Running as Dedicated Server"));
            break;
        case NM_ListenServer:
            UE_LOG(LogTemp, Log, TEXT("Running as Listen Server"));
            break;
        case NM_Client:
            UE_LOG(LogTemp, Log, TEXT("Running as Client"));
            break;
        default:
            break;
    }
}
```

### 3.4 角色与远程角色

每个Actor在网络中有两个关键属性，决定了它的复制行为：

```cpp
// Engine/Source/Runtime/Engine/Classes/GameFramework/Actor.h

class AActor : public UObject
{
protected:
    /** 本地的角色 */
    ENetRole LocalRole;

    /** 远程的角色 */
    ENetRole RemoteRole;

public:
    // 角色枚举
    // ROLE_None - 无网络角色
    // ROLE_SimulatedProxy - 模拟代理（客户端被动接收）
    // ROLE_AutonomousProxy - 自主代理（客户端可预测）
    // ROLE_Authority - 权威（拥有状态控制权）
};
```

**角色关系表：**

| 场景 | LocalRole (服务器) | RemoteRole (服务器) | LocalRole (客户端) | RemoteRole (客户端) |
|------|-------------------|--------------------|--------------------|--------------------|
| 服务器Actor | Authority | SimulatedProxy | - | - |
| 客户端PlayerController | SimulatedProxy | AutonomousProxy | AutonomousProxy | Authority |
| 复制Actor | Authority | SimulatedProxy | SimulatedProxy | Authority |

### 3.5 实践：根据NetMode编写逻辑

```cpp
// MyCharacter.h
UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    virtual void BeginPlay() override;

    // 只在服务器执行的函数
    UFUNCTION(BlueprintCallable, Category = "Gameplay")
    void ServerOnlyFunction();

    // 只在客户端执行的函数
    UFUNCTION(BlueprintCallable, Category = "Gameplay")
    void ClientOnlyFunction();

    // 在所有端执行但行为不同的函数
    UFUNCTION(BlueprintCallable, Category = "Gameplay")
    void CrossPlatformFunction();

protected:
    UPROPERTY(ReplicatedUsing = OnRep_Health)
    float Health;

    UFUNCTION()
    void OnRep_Health(float OldHealth);
};

// MyCharacter.cpp
void AMyCharacter::BeginPlay()
{
    Super::BeginPlay();

    // 根据NetMode执行不同逻辑
    if (GetNetMode() == NM_DedicatedServer)
    {
        // 服务器专用初始化
        InitializeServerLogic();
    }
    else if (GetNetMode() == NM_Client)
    {
        // 客户端专用初始化
        InitializeClientLogic();
    }
}

void AMyCharacter::ServerOnlyFunction()
{
    if (!HasAuthority())
    {
        UE_LOG(LogTemp, Warning, TEXT("ServerOnlyFunction called on client - ignoring"));
        return;
    }

    // 服务器逻辑
    ProcessGameplayLogic();
}

void AMyCharacter::ClientOnlyFunction()
{
    if (GetNetMode() == NM_DedicatedServer)
    {
        return; // 专用服务器不执行客户端逻辑
    }

    // 客户端逻辑（包括Listen Server的本地客户端）
    UpdateLocalEffects();
}

void AMyCharacter::CrossPlatformFunction()
{
    if (HasAuthority())
    {
        // 服务器端逻辑
        ApplyGameplayEffect();
    }
    else
    {
        // 客户端预测逻辑
        PredictEffectLocally();
    }
}

// 复制注册
void AMyCharacter::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(AMyCharacter, Health);
}

void AMyCharacter::OnRep_Health(float OldHealth)
{
    // 客户端响应生命值变化
    if (Health < OldHealth)
    {
        PlayDamageEffect();
    }

    UpdateHealthUI();
}
```

---

## 四、网络连接生命周期

### 4.1 连接状态机

```
┌─────────────┐
│    US_Open   │ ──连接请求──► ┌─────────────┐
│   (未连接)   │                │  US_Init    │
└─────────────┘                │  (初始化)   │
                               └──────┬──────┘
                                      │
                    握手成功/失败      │
                    ┌─────────────────┴─────────────────┐
                    ▼                                   ▼
            ┌─────────────┐                    ┌─────────────┐
            │  US_Open    │                    │  US_Closed  │
            │  (已连接)   │                    │   (已关闭)  │
            └──────┬──────┘                    └─────────────┘
                   │
         正常游戏进行
                   │
                   ▼
            ┌─────────────┐
            │ US_Pending  │
            │ (等待关闭)  │
            └──────┬──────┘
                   │
                   ▼
            ┌─────────────┐
            │  US_Closed  │
            │   (已关闭)  │
            └─────────────┘
```

### 4.2 客户端连接流程

```cpp
// 客户端连接完整流程

// 1. 发起连接请求
void AMyGameInstance::ConnectToServer(const FString& ServerAddress)
{
    // 构建连接URL
    FString URL = FString::Printf(
        TEXT("%s?Name=%s"),
        *ServerAddress,
        *GetPlayerName()
    );

    // 发起连接
    JoinSession(URL);
}

// 2. GameInstance处理连接
void UGameInstance::JoinSession(const FString& URL)
{
    // 获取当前World
    UWorld* World = GetWorld();

    // 确保断开现有连接
    if (World && World->GetNetDriver())
    {
        World->GetNetDriver()->Close();
    }

    // 客户端旅行到服务器
    GetEngine()->Browse(*WorldContextObject, FURL(NULL, *URL, TRAVEL_Absolute));
}
```

### 4.3 服务器端连接处理

```cpp
// GameMode中处理玩家连接

// 1. 预登录 - 验证玩家是否可以加入
void AGameModeBase::PreLogin(const FString& Options,
                             const FString& Address,
                             const FUniqueNetIdRepl& UniqueId,
                             FString& ErrorMessage)
{
    // 检查服务器是否已满
    if (GetNumPlayers() >= MaxPlayers)
    {
        ErrorMessage = TEXT("Server is full");
        return;
    }

    // 检查是否被封禁
    if (IsPlayerBanned(UniqueId))
    {
        ErrorMessage = TEXT("You are banned from this server");
        return;
    }

    // 可以在这里进行其他验证
    // ...
}

// 2. 登录 - 创建PlayerController
APlayerController* AGameModeBase::Login(UPlayer* NewPlayer,
                                        ENetRole InRemoteRole,
                                        const FString& Portal,
                                        const FString& Options,
                                        const FUniqueNetIdRepl& UniqueId,
                                        FString& ErrorMessage)
{
    // 创建PlayerController
    FActorSpawnParameters SpawnParams;
    SpawnParams.Owner = NewPlayer;
    SpawnParams.Instigator = nullptr;

    APlayerController* PC = SpawnPlayerController(InRemoteRole, SpawnLocation, SpawnRotation);

    // 初始化PlayerController
    if (PC)
    {
        PC->SetPlayer(NewPlayer);
        NewPlayer->PlayerController = PC;
    }

    return PC;
}

// 3. 后登录 - 玩家加入游戏
void AGameModeBase::PostLogin(APlayerController* NewPlayer)
{
    // 增加玩家计数
    ++NumPlayers;

    // 初始化玩家状态
    if (NewPlayer->PlayerState)
    {
        NewPlayer->PlayerState->SetPlayerName(GetPlayerNameFromController(NewPlayer));
    }

    // 通知其他玩家有新玩家加入
    for (FConstPlayerControllerIterator It = GetWorld()->GetPlayerControllerIterator(); It; ++It)
    {
        if (It->Get() != NewPlayer)
        {
            // 广播新玩家加入事件
        }
    }

    // 生成玩家的Pawn
    RestartPlayer(NewPlayer);
}

// 4. 玩家断开连接
void AGameModeBase::Logout(AController* Exiting)
{
    // 减少玩家计数
    --NumPlayers;

    // 清理玩家状态
    // ...

    // 通知其他玩家
    // ...
}
```

### 4.4 网络通知接口

```cpp
// 实现网络通知接口
class AMyGameMode : public AGameModeBase, public FNetworkNotify
{
public:
    // 接受新连接
    virtual EAcceptConnection::Type NotifyAcceptingConnection() override
    {
        if (bServerFull)
        {
            return EAcceptConnection::Reject;
        }
        return EAcceptConnection::Accept;
    }

    // 连接已建立
    virtual void NotifyAcceptedConnection(class UNetConnection* Connection) override
    {
        UE_LOG(LogTemp, Log, TEXT("New connection from: %s"), *Connection->LowLevelGetRemoteAddress());
    }

    // 接收RPC调用
    virtual bool NotifyAcceptingChannel(class UChannel* Channel) override
    {
        return true;
    }
};
```

---

## 五、网络驱动架构

### 5.1 IPNetDriver详解

IPNetDriver是UE5默认的网络驱动，基于UDP协议实现。

```cpp
// 配置IPNetDriver（DefaultEngine.ini）

[/Script/Engine.Engine]
!NetDriverDefinitions=ClearArray
+NetDriverDefinitions=(DefName="GameNetDriver",DriverClassName="IpNetDriver",DriverClassNameFallback="IpNetDriver")

[/Script/IpDrv.IpNetDriver]
NetConnectionClassName=IpConnection
AllowDownloads=True
MaxClientRate=15000
MaxInternetClientRate=10000
KeepAliveTime=0.2
InitialConnectTimeout=30.0
ConnectionTimeout=180.0
```

### 5.2 WebSocketNetDriver

用于Web平台和需要HTTP穿透的场景。

```cpp
// 配置WebSocketNetDriver（DefaultEngine.ini）

[/Script/Engine.Engine]
+NetDriverDefinitions=(DefName="GameNetDriver",DriverClassName="WebSocketNetDriver",DriverClassNameFallback="WebSocketNetDriver")

[/Script/WebSocketNetDriver.WebSocketNetDriver]
NetConnectionClassName=WebSocketConnection
```

```cpp
// 使用WebSocket连接
void ConnectViaWebSocket(const FString& ServerURL)
{
    // WebSocket URL格式
    // ws://server:port/ 或者 wss://server:port/ (加密)

    FString WSUrl = FString::Printf(TEXT("ws://%s:7777"), *ServerURL);

    // 连接逻辑
    APlayerController* PC = GetWorld()->GetFirstPlayerController();
    PC->ClientTravel(WSUrl, ETravelType::TRAVEL_Absolute);
}
```

### 5.3 自定义网络驱动

```cpp
// 创建自定义网络驱动
UCLASS()
class UMyCustomNetDriver : public UIpNetDriver
{
    GENERATED_BODY()

public:
    virtual bool InitConnect(FNetworkNotify* InNotify,
                            const FURL& ConnectURL,
                            FString& Error) override
    {
        // 自定义连接初始化
        bool bSuccess = Super::InitConnect(InNotify, ConnectURL, Error);

        if (bSuccess)
        {
            // 添加自定义握手逻辑
            PerformCustomHandshake();
        }

        return bSuccess;
    }

    virtual void TickDispatch(float DeltaSeconds) override
    {
        Super::TickDispatch(DeltaSeconds);

        // 自定义网络Tick逻辑
        ProcessCustomPackets();
    }

private:
    void PerformCustomHandshake()
    {
        // 实现自定义握手协议
    }

    void ProcessCustomPackets()
    {
        // 处理自定义数据包
    }
};
```

### 5.4 网络统计与监控

```cpp
// 获取网络统计信息
void AMyGameInstance::LogNetworkStats()
{
    UNetDriver* NetDriver = GetWorld()->GetNetDriver();
    if (!NetDriver)
    {
        return;
    }

    // 基本统计
    UE_LOG(LogTemp, Log, TEXT("=== Network Statistics ==="));
    UE_LOG(LogTemp, Log, TEXT("Is Server: %s"), NetDriver->IsServer() ? TEXT("True") : TEXT("False"));

    if (NetDriver->IsServer())
    {
        // 服务器统计
        int32 ClientCount = NetDriver->ClientConnections.Num();
        UE_LOG(LogTemp, Log, TEXT("Connected Clients: %d"), ClientCount);

        for (UNetConnection* Conn : NetDriver->ClientConnections)
        {
            if (Conn)
            {
                UE_LOG(LogTemp, Log, TEXT("  Client: %s, Lag: %.2fms"),
                    *Conn->PlayerId.ToString(),
                    Conn->GetAvgLag() * 1000.0f);
            }
        }
    }
    else
    {
        // 客户端统计
        if (UNetConnection* ServerConn = NetDriver->ServerConnection)
        {
            UE_LOG(LogTemp, Log, TEXT("Server Connection: %s"),
                *ServerConn->LowLevelGetRemoteAddress());
            UE_LOG(LogTemp, Log, TEXT("Average Lag: %.2fms"),
                ServerConn->GetAvgLag() * 1000.0f);
        }
    }
}

// 使用控制台命令查看网络状态
// 在游戏中按 ~ 键打开控制台，输入：
// stat net - 显示网络统计
// net.ToggleNetDriver - 切换网络驱动调试信息
// net.Profile - 网络性能分析
```

---

## 六、实践任务

### 任务1：创建网络调试工具

```cpp
// 创建一个网络调试Actor
// MyNetworkDebugger.h
UCLASS()
class AMyNetworkDebugger : public AActor
{
    GENERATED_BODY()

public:
    AMyNetworkDebugger();

    virtual void Tick(float DeltaTime) override;

    UFUNCTION(BlueprintCallable, Category = "Network")
    FString GetCurrentNetModeString() const;

    UFUNCTION(BlueprintCallable, Category = "Network")
    float GetPing() const;

    UFUNCTION(BlueprintCallable, Category = "Network")
    int32 GetConnectedPlayers() const;

    UFUNCTION(BlueprintCallable, Category = "Network")
    void PrintNetworkInfo();

protected:
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Network")
    bool bShowDebugOnScreen = true;
};

// MyNetworkDebugger.cpp
AMyNetworkDebugger::AMyNetworkDebugger()
{
    PrimaryActorTick.bCanEverTick = true;
}

void AMyNetworkDebugger::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    if (bShowDebugOnScreen)
    {
        PrintNetworkInfo();
    }
}

FString AMyNetworkDebugger::GetCurrentNetModeString() const
{
    switch (GetNetMode())
    {
        case NM_Standalone:
            return TEXT("Standalone");
        case NM_DedicatedServer:
            return TEXT("Dedicated Server");
        case NM_ListenServer:
            return TEXT("Listen Server");
        case NM_Client:
            return TEXT("Client");
        default:
            return TEXT("Unknown");
    }
}

float AMyNetworkDebugger::GetPing() const
{
    UNetDriver* NetDriver = GetWorld()->GetNetDriver();
    if (!NetDriver)
    {
        return -1.0f;
    }

    if (NetDriver->IsServer())
    {
        return 0.0f; // 服务器本身无延迟
    }

    if (UNetConnection* ServerConn = NetDriver->ServerConnection)
    {
        return ServerConn->GetAvgLag() * 1000.0f; // 转换为毫秒
    }

    return -1.0f;
}

int32 AMyNetworkDebugger::GetConnectedPlayers() const
{
    UNetDriver* NetDriver = GetWorld()->GetNetDriver();
    if (!NetDriver || !NetDriver->IsServer())
    {
        return 0;
    }

    return NetDriver->ClientConnections.Num();
}

void AMyNetworkDebugger::PrintNetworkInfo()
{
    if (!GEngine)
    {
        return;
    }

    // 屏幕打印网络信息
    FString Info = FString::Printf(
        TEXT("NetMode: %s\nPing: %.1fms\nPlayers: %d"),
        *GetCurrentNetModeString(),
        GetPing(),
        GetConnectedPlayers()
    );

    GEngine->AddOnScreenDebugMessage(
        -1,
        0.0f,
        FColor::Green,
        Info,
        false,
        FVector2D(1.0f, 1.0f)
    );
}
```

### 任务2：实现连接状态监听

```cpp
// MyConnectionListener.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyConnectionListener.generated.h"

DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnConnectionEstablished);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnConnectionLost, const FString&, Reason);

UCLASS()
class AMyConnectionListener : public AActor
{
    GENERATED_BODY()

public:
    AMyConnectionListener();

    virtual void BeginPlay() override;

    UPROPERTY(BlueprintAssignable, Category = "Network")
    FOnConnectionEstablished OnConnectionEstablished;

    UPROPERTY(BlueprintAssignable, Category = "Network")
    FOnConnectionLost OnConnectionLost;

protected:
    UFUNCTION()
    void HandleConnectionEstablished();

    UFUNCTION()
    void HandleConnectionLost();

private:
    bool bWasConnected = false;
    FTimerHandle CheckTimerHandle;

    void CheckConnectionState();
};

// MyConnectionListener.cpp
#include "MyConnectionListener.h"
#include "Engine/NetDriver.h"

AMyConnectionListener::AMyConnectionListener()
{
    PrimaryActorTick.bCanEverTick = false;
}

void AMyConnectionListener::BeginPlay()
{
    Super::BeginPlay();

    // 定期检查连接状态
    GetWorld()->GetTimerManager().SetTimer(
        CheckTimerHandle,
        this,
        &AMyConnectionListener::CheckConnectionState,
        0.5f, // 每0.5秒检查一次
        true
    );
}

void AMyConnectionListener::CheckConnectionState()
{
    UNetDriver* NetDriver = GetWorld()->GetNetDriver();
    bool bIsConnected = (NetDriver != nullptr);

    if (NetDriver && NetDriver->IsServer())
    {
        bIsConnected = NetDriver->ClientConnections.Num() > 0;
    }
    else if (NetDriver && NetDriver->ServerConnection)
    {
        bIsConnected = NetDriver->ServerConnection->State == US_Open;
    }

    // 检测状态变化
    if (bIsConnected && !bWasConnected)
    {
        HandleConnectionEstablished();
    }
    else if (!bIsConnected && bWasConnected)
    {
        HandleConnectionLost();
    }

    bWasConnected = bIsConnected;
}

void AMyConnectionListener::HandleConnectionEstablished()
{
    UE_LOG(LogTemp, Log, TEXT("Connection established"));
    OnConnectionEstablished.Broadcast();
}

void AMyConnectionListener::HandleConnectionLost()
{
    UE_LOG(LogTemp, Warning, TEXT("Connection lost"));
    OnConnectionLost.Broadcast(TEXT("Connection lost"));
}
```

---

## 七、常见问题与解决方案

### Q1：如何判断当前是否为权威端？

```cpp
// 使用 HasAuthority() 方法
if (HasAuthority())
{
    // 当前是服务器端（Authority）
    ApplyDamageOnServer();
}

// 或检查 LocalRole
if (GetLocalRole() == ROLE_Authority)
{
    // 等同于 HasAuthority()
}
```

### Q2：Dedicated Server和Listen Server如何选择？

| 场景 | 推荐模式 |
|------|---------|
| 竞技游戏（FPS/MOBA） | Dedicated Server |
| 合作游戏（PVE） | Listen Server 可接受 |
| 大型多人在线 | Dedicated Server |
| 快速原型/测试 | Listen Server |
| LAN Party | Listen Server |
| 商业级产品 | Dedicated Server |

### Q3：如何在编辑器中测试多人游戏？

```
1. 打开编辑器
2. 点击工具栏右侧的下拉箭头
3. 选择 "Network Emulation" 或 "Number of Players"
4. 设置玩家数量为 2+
5. 勾选 "Use Net Emulation" 可模拟网络延迟
6. 点击 Play 开始多人测试
```

---

## 八、总结

本课我们学习了：

1. **网络模型**：理解了客户端-服务器模型与P2P模型的区别
2. **核心组件**：掌握了UNetDriver、UNetConnection、UActorChannel等核心组件
3. **NetMode**：深入理解四种网络模式的特点与应用场景
4. **连接生命周期**：了解了完整的连接建立、维护、断开流程
5. **网络驱动**：学习了IPNetDriver和WebSocketNetDriver的使用

---

## 九、扩展阅读

1. [Unreal Engine Networking Architecture](https://docs.unrealengine.com/5.0/en-US/networking-architecture-in-unreal-engine/)
2. [Replication Graph](https://docs.unrealengine.com/5.0/en-US/replication-graph-in-unreal-engine/)
3. UE5源码：`Engine/Source/Runtime/Engine/Private/NetDriver.cpp`
4. UE5源码：`Engine/Source/Runtime/Engine/Private/NetConnection.cpp`

---

## 十、下节预告

**第2课：Actor复制系统**

将深入学习：
- Actor复制原理与机制
- UProperty网络修饰符
- 复制条件与优先级
- ReplicationGraph基础

---

*课程版本：1.0*
*最后更新：2026-04-09*