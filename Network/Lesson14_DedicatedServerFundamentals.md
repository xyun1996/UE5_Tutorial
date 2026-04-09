# 第十四课：专用服务器基础

本课深入讲解UE Dedicated Server（DS）的核心概念、架构和启动流程。

---

## 目录

1. [DS服务器概述](#1-ds服务器概述)
2. [服务器架构](#2-服务器架构)
3. [启动流程](#3-启动流程)
4. [服务器配置](#4-服务器配置)
5. [网络模式](#5-网络模式)

---

## 1. DS服务器概述

### 1.1 什么是Dedicated Server

Dedicated Server（专用服务器）是一个**没有本地玩家**的纯服务器进程，专注于：
- 处理玩家连接
- 运行游戏逻辑
- 同步游戏状态
- 管理网络复制

### 1.2 DS vs Listen Server vs Client

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        网络模式对比                                       │
├──────────────┬──────────────────┬───────────────────┬───────────────────┤
│    特性       │  Dedicated Server │   Listen Server   │      Client       │
├──────────────┼──────────────────┼───────────────────┼───────────────────┤
│ 本地玩家      │       无          │        有          │        有         │
│ 渲染         │       无          │        有          │        有         │
│ 物理模拟      │     完整          │       完整         │      部分         │
│ 游戏逻辑      │     权威          │       权威         │      本地         │
│ 性能开销      │      低           │       高           │      中           │
│ 作弊防护      │      强           │       弱           │       -           │
│ 适用场景      │    正式运营        │      开发测试       │     玩家端        │
└──────────────┴──────────────────┴───────────────────┴───────────────────┘
```

### 1.3 编译宏定义

```cpp
// UEBuildTarget.cs 中的编译配置
// DS编译时的宏定义
UE_SERVER = 1           // 表示这是服务器构建
WITH_SERVER_CODE = 1    // 包含服务器代码
WITH_CLIENT_CODE = 0    // 不包含客户端代码（DS）

// 运行时检测
bool IsRunningDedicatedServer();  // 检测是否运行DS
```

### 1.4 源码关键点

```cpp
// 判断是否为DS的标准方法
// Engine/Source/Runtime/Core/Public/Misc/App.h
namespace FApp
{
    ENGINE_API bool IsRunningDedicatedServer();
    ENGINE_API bool IsServerOnly();
}

// 示例：条件编译
#if UE_SERVER
    // 服务器专用代码
    ServerOnlyFunction();
#endif

// 示例：运行时检测
if (IsRunningDedicatedServer())
{
    // DS专用逻辑
    // 例如：不加载UI资源、不播放音效等
}
```

---

## 2. 服务器架构

### 2.1 核心组件层次

```
┌─────────────────────────────────────────────────────────────────┐
│                        GameInstance                              │
│  (游戏实例，管理整个游戏的生命周期)                               │
├─────────────────────────────────────────────────────────────────┤
│                           World                                  │
│  (游戏世界，包含所有Actor和关卡)                                  │
├─────────────────────────────────────────────────────────────────┤
│                    GameMode / GameSession                        │
│  (游戏规则、玩家管理)                     │
├─────────────────────────────────────────────────────────────────┤
│                         NetDriver                                │
│  (网络驱动，管理所有连接)                                         │
├─────────────────────────────────────────────────────────────────┤
│                      NetConnection(s)                            │
│  (玩家连接，每个玩家一个)                                         │
├─────────────────────────────────────────────────────────────────┤
│                        Channel(s)                                │
│  (数据通道，ActorChannel/ControlChannel/VoiceChannel)           │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 关键类职责

#### GameMode - 游戏规则

```cpp
// Engine/Source/Runtime/Engine/Private/GameModeBase.cpp

// 初始化游戏
void AGameModeBase::InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage);

// 玩家登录流程
void AGameModeBase::PreLogin(const FString& Options, const FString& Address,
                             const FUniqueNetIdRepl& UniqueId, FString& ErrorMessage);
APlayerController* AGameModeBase::Login(UPlayer* NewPlayer, ENetRole InRemoteRole,
                                        const FString& Portal, const FString& Options,
                                        const FUniqueNetIdRepl& UniqueId, FString& ErrorMessage);
void AGameModeBase::PostLogin(APlayerController* NewPlayer);
```

#### GameSession - 会话管理

```cpp
// Engine/Source/Runtime/Engine/Private/GameSession.cpp

// 登录验证
FString AGameSession::ApproveLogin(const FString& Options)
{
    // 检查服务器是否已满
    if (AtCapacity(SpectatorOnly == 1))
    {
        return TEXT("Server full.");
    }

    // 检查分屏玩家数量
    if (SplitscreenCount > MaxSplitscreensPerConnection)
    {
        return TEXT("Maximum splitscreen players");
    }

    return TEXT("");  // 空字符串表示允许登录
}

// 踢出玩家
bool AGameSession::KickPlayer(APlayerController* KickedPlayer, const FText& KickReason);

// 封禁玩家
bool AGameSession::BanPlayer(APlayerController* BannedPlayer, const FText& BanReason);
```

#### NetDriver - 网络驱动

```cpp
// Engine/Source/Runtime/Engine/Classes/Engine/NetDriver.h

class UNetDriver : public UObject
{
    // 连接列表
    TArray<class UNetConnection*> ClientConnections;

    // 服务器相关
    bool IsServer() const { return bIsServer; }
    FString LowLevelGetNetworkNumber();

    // Tick处理
    virtual void TickDispatch(float DeltaSeconds);
    virtual void TickFlush(float DeltaSeconds);
};
```

### 2.3 Actor角色分布

```
┌─────────────────────────────────────────────────────────────────┐
│                        服务器端                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    GameMode (权威)                        │   │
│  │  - 游戏规则执行                                           │   │
│  │  - 玩家认证                                               │   │
│  │  - 胜负判定                                               │   │
│  └─────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    GameState (复制)                       │   │
│  │  - 游戏状态                                               │   │
│  │  - 分数信息                                               │   │
│  │  - 所有客户端可见                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  PlayerState (复制)                       │   │
│  │  - 玩家数据                                               │   │
│  │  - 所有人可见                                             │   │
│  └─────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                PlayerController (每个玩家)                │   │
│  │  - LocalRole = ROLE_Authority (服务器)                    │   │
│  │  - RemoteRole = ROLE_AutonomousProxy (客户端拥有)         │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. 启动流程

### 3.1 DS启动命令行

```bash
# 基本启动命令
MyGameServer.exe MyMap?Game=MyGameMode -log

# 常用参数
MyGameServer.exe MyMap?Game=MyGameMode?MaxPlayers=16 -log -port=7777

# 完整示例
MyGameServer.exe /Game/Maps/Downtown?Game=/Script/MyGame.TDMGameMode?MaxPlayers=32 -log -port=7777 -QueryPort=27015 -SteamServerName="My Server"
```

### 3.2 启动参数详解

| 参数 | 说明 | 示例 |
|------|------|------|
| `MapName` | 地图名称 | `/Game/Maps/Downtown` |
| `?Game=` | GameMode类 | `?Game=/Script/MyGame.TDMGameMode` |
| `?MaxPlayers=` | 最大玩家数 | `?MaxPlayers=16` |
| `-log` | 启用日志 | `-log` |
| `-port=` | 游戏端口 | `-port=7777` |
| `-QueryPort=` | 查询端口 | `-QueryPort=27015` |
| `-multihome=` | 多网卡绑定 | `-multihome=192.168.1.100` |
| `-nullrhi` | 无渲染 | `-nullrhi`（DS默认） |
| `-nographics` | 无图形 | `-nographics` |

### 3.3 启动流程源码

```
┌─────────────────────────────────────────────────────────────────┐
│                        DS启动流程                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 引擎初始化                                                   │
│     LaunchEngineLoop.cpp: FEngineLoop::Init()                   │
│     ├── 解析命令行参数                                           │
│     ├── 初始化模块管理器                                         │
│     └── 创建GameInstance                                         │
│                                                                  │
│  2. 世界初始化                                                   │
│     World.cpp: UWorld::InitializeNewWorld()                     │
│     ├── 创建GameMode                                             │
│     ├── 创建GameState                                            │
│     └── 调用GameMode::InitGame()                                 │
│                                                                  │
│  3. 网络初始化                                                   │
│     NetDriver.cpp: UNetDriver::InitListen()                     │
│     ├── 创建Socket                                               │
│     ├── 绑定端口                                                  │
│     └── 初始化Connection列表                                     │
│                                                                  │
│  4. 服务器Actor生成                                              │
│     UnrealEngine.cpp: UEngine::SpawnServerActors()              │
│     └── 生成服务器专用Actor                                      │
│                                                                  │
│  5. 主循环                                                       │
│     LaunchEngineLoop.cpp: FEngineLoop::Tick()                   │
│     ├── World->Tick()                                            │
│     ├── NetDriver->TickDispatch()                                │
│     ├── GameMode逻辑                                             │
│     └── NetDriver->TickFlush()                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.4 GameMode初始化流程

```cpp
// GameModeBase.cpp
void AGameModeBase::InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage)
{
    // 1. 创建GameSession
    GameSession = NewObject<AGameSession>(this, GameSessionClass);

    // 2. 初始化GameSession选项
    GameSession->InitOptions(Options);

    // 3. 绑定委托
    FGameDelegates::Get().GetPendingConnectionLostDelegate().AddUObject(this, &AGameModeBase::NotifyPendingConnectionLost);
}
```

---

## 4. 服务器配置

### 4.1 DefaultEngine.ini配置

```ini
[/Script/Engine.GameEngine]
; 启用网络休眠
+NetDriverDefinitions=(DefName="GameNetDriver",DriverClassName="Engine.GameNetDriver",DriverClassNameFallback="Engine.GameNetDriver")

[/Script/Engine.GameNetDriver]
; 网络配置
NetConnectionClassName="/Script/Engine.IpConnection"
InitialConnectTimeout=120.0
ConnectionTimeout=120.0
KeepAliveTime=0.2
MaxClientUpdateRate=100
NetServerMaxTickRate=30

[/Script/Engine.GameSession]
; 玩家限制
MaxPlayers=16
MaxSpectators=2
MaxSplitscreensPerConnection=4

[/Script/Engine.NetworkSettings]
; 网络设置
NetServerMaxTickRate=30
MaxClientUpdateRate=100
MaxInternetClientRate=10000
MinInternetClientRate=5000
```

### 4.2 命令行配置覆盖

```cpp
// GameSession.cpp: InitOptions()

// 通过URL参数配置
void AGameSession::InitOptions(const FString& Options)
{
    // 解析MaxPlayers参数
    if (UGameplayStatics::HasOption(Options, TEXT("MaxPlayers")))
    {
        MaxPlayers = UGameplayStatics::GetIntOption(Options, TEXT("MaxPlayers"), MaxPlayers);
    }

    // 解析MaxSpectators参数
    if (UGameplayStatics::HasOption(Options, TEXT("MaxSpectators")))
    {
        MaxSpectators = UGameplayStatics::GetIntOption(Options, TEXT("MaxSpectators"), MaxSpectators);
    }
}
```

### 4.3 运行时CVar配置

```cpp
// 控制台变量，可在运行时修改

// 最大玩家数覆盖
net.MaxPlayersOverride=0  // 0表示使用默认值

// 网络帧率
net.NetServerMaxTickRate=30

// 带宽限制
net.MaxClientRate=10000
net.MinClientRate=5000

// 调试相关
net.Reliable.Debug=0
net.PacketRecording.Enabled=0
```

---

## 5. 网络模式

### 5.1 ENetMode枚举

```cpp
// Engine/Source/Runtime/Engine/Classes/Engine/EngineTypes.h

enum ENetMode
{
    NM_Standalone,        // 单机模式，无网络
    NM_DedicatedServer,   // 专用服务器
    NM_ListenServer,      // 监听服务器（有本地玩家）
    NM_Client,            // 客户端
    NM_MAX
};
```

### 5.2 获取网络模式

```cpp
// 获取World的网络模式
ENetMode GetNetMode() const;

// 快捷判断
bool IsServer() const;  // NM_DedicatedServer 或 NM_ListenServer
bool IsClient() const;  // NM_Client

// Actor中的判断
if (GetNetMode() == NM_DedicatedServer)
{
    // DS专用逻辑
}

// 全局判断
if (IsRunningDedicatedServer())
{
    // DS专用逻辑
}
```

### 5.3 角色与网络模式

```cpp
// Actor的角色（Role）和远程角色（RemoteRole）在不同网络模式下不同

// Dedicated Server上的PlayerController
LocalRole = ROLE_Authority          // 服务器拥有权威
RemoteRole = ROLE_AutonomousProxy   // 客户端可以自主移动

// Dedicated Server上的非玩家Actor
LocalRole = ROLE_Authority          // 服务器权威
RemoteRole = ROLE_SimulatedProxy    // 客户端只是模拟

// Client上的PlayerController
LocalRole = ROLE_AutonomousProxy    // 有一定自主权
RemoteRole = ROLE_Authority         // 服务器是权威
```

### 5.4 DS特有优化

```cpp
// DS不需要的功能会被条件编译排除

#if !UE_SERVER
    // 渲染相关
    InitializeRenderSystem();
    CreateGameViewport();

    // 音频相关
    InitializeAudioSystem();

    // UI相关
    CreateUIWidgets();
#endif

// 运行时跳过
if (IsRunningDedicatedServer())
{
    // 不播放音效
    return;
}

// 不加载客户端资源
if (IsRunningDedicatedServer())
{
    // 只加载服务器需要的资源
    // 不加载材质、纹理、UI等
}
```

---

## 课程总结

本课介绍了DS服务器的基础知识：

| 主题 | 要点 |
|------|------|
| DS概述 | 无本地玩家、无渲染、纯服务器逻辑 |
| 架构 | GameMode → GameSession → NetDriver → Connection → Channel |
| 启动流程 | 引擎初始化 → 世界创建 → 网络初始化 → 主循环 |
| 配置 | DefaultEngine.ini + 命令行参数 + CVar |
| 网络模式 | NM_DedicatedServer vs NM_ListenServer vs NM_Client |

---

## 下一课预告

第十五课将深入讲解：
- 玩家连接流程
- 登录验证机制
- 服务器性能优化
- 常见问题排查

---

## 参考资料

- Engine/Source/Runtime/Engine/Private/GameModeBase.cpp
- Engine/Source/Runtime/Engine/Private/GameSession.cpp
- Engine/Source/Runtime/Engine/Private/NetDriver.cpp
- Engine/Source/Runtime/Launch/Private/LaunchEngineLoop.cpp
