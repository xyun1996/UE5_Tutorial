# 第十五课：专用服务器进阶

本课深入讲解玩家连接流程、登录验证、性能优化和问题排查。

---

## 目录

1. [玩家连接流程](#1-玩家连接流程)
2. [登录验证机制](#2-登录验证机制)
3. [服务器Tick机制](#3-服务器tick机制)
4. [性能优化](#4-性能优化)
5. [问题排查](#5-问题排查)

---

## 1. 玩家连接流程

### 1.1 完整连接流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        玩家连接完整流程                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  客户端                               服务器                              │
│    │                                    │                                 │
│    │  1. ClientConnect                  │                                 │
│    │ ─────────────────────────────────► │                                 │
│    │                                    │ 创建UNetConnection             │
│    │                                    │ 状态: USOCK_Pending            │
│    │                                    │                                 │
│    │  2. Stateless Handshake            │                                 │
│    │ ◄────────────────────────────────► │ 握手验证                       │
│    │                                    │                                │
│    │  3. NMT_Hello                      │                                 │
│    │ ─────────────────────────────────► │ 控制通道消息                   │
│    │                                    │                                │
│    │  4. NMT_Welcome                    │                                 │
│    │ ◄───────────────────────────────── │ 服务器欢迎                     │
│    │                                    │ 分配PlayerID                   │
│    │                                    │                                │
│    │  5. NMT_Login                      │                                 │
│    │ ─────────────────────────────────► │ PreLogin() 验证                │
│    │                                    │ Login() 创建PC                 │
│    │                                    │ PostLogin()                    │
│    │                                    │                                │
│    │  6. NMT_Netspeed                   │                                 │
│    │ ─────────────────────────────────► │ 设置网络速度                   │
│    │                                    │                                │
│    │  7. Join                           │                                 │
│    │ ◄───────────────────────────────── │ 发送当前地图                   │
│    │                                    │ 开始同步Actor                  │
│    │                                    │                                │
│    │  8. NMT_Join                       │                                 │
│    │ ─────────────────────────────────► │ 客户端确认加入                 │
│    │                                    │ 创建Pawn                       │
│    │                                    │                                │
│    │  9. Game Started                   │                                 │
│    │ ◄────────────────────────────────► │ 正常游戏循环                   │
│    │                                    │                                │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 连接状态机

```cpp
// Engine/Source/Runtime/Engine/Classes/Engine/NetConnection.h

enum EConnectionState
{
    USOCK_Invalid   = 0,  // 连接无效，可能未初始化
    USOCK_Closed    = 1,  // 连接已关闭
    USOCK_Pending   = 2,  // 等待连接
    USOCK_Open      = 3,  // 连接已打开
    USOCK_Closing   = 4,  // 正在关闭，等待可靠数据确认
};
```

### 1.3 关键源码：连接处理

```cpp
// NetDriver.cpp: 服务器接收新连接

void UNetDriver::TickDispatch(float DeltaSeconds)
{
    // 接受新连接
    while (UNetConnection* NewConnection = GetNewConnection())
    {
        // 创建控制通道
        UChannel* ControlChannel = NewConnection->CreateChannelByName(NAME_Control, CHANNEL_Open);
        ControlChannel->Open();

        // 触发连接事件
        GEngine->BroadcastNetworkEvent(NewConnection, ENetworkEvent::ClientJoined);
    }
}
```

### 1.4 消息类型（NMT）

```cpp
// Engine/Source/Runtime/Engine/Classes/Engine/Net.h

// 控制通道消息类型
enum EEngineNetControlMessage
{
    NMT_Hello,              // 客户端发起连接
    NMT_Welcome,            // 服务器欢迎
    NMT_Login,              // 客户端登录
    NMT_Failure,            // 连接失败
    NMT_ActorChannelFailure,// Actor通道失败
    NMT_Netspeed,           // 网络速度设置
    NMT_NextURL,            // 下一个URL（旅行）
    NMT_Join,               // 加入游戏
    NMT_Upgrade,            // 协议升级
    // ... 更多消息类型
};
```

---

## 2. 登录验证机制

### 2.1 登录流程详解

```cpp
// GameModeBase.cpp

// Step 1: PreLogin - 预登录验证
void AGameModeBase::PreLogin(const FString& Options, const FString& Address,
                             const FUniqueNetIdRepl& UniqueId, FString& ErrorMessage)
{
    // 验证唯一ID
    const bool bUniqueIdCheckOk = (!UniqueId.IsValid() ||
                                   UOnlineEngineInterface::Get()->IsCompatibleUniqueNetId(UniqueId));
    if (!bUniqueIdCheckOk)
    {
        ErrorMessage = TEXT("incompatible_unique_net_id");
        return;
    }

    // 让GameSession验证
    ErrorMessage = GameSession->ApproveLogin(Options);

    // 广播预登录事件
    FGameModeEvents::GameModePreLoginEvent.Broadcast(this, UniqueId, ErrorMessage);
}

// Step 2: Login - 创建PlayerController
APlayerController* AGameModeBase::Login(UPlayer* NewPlayer, ENetRole InRemoteRole,
                                        const FString& Portal, const FString& Options,
                                        const FUniqueNetIdRepl& UniqueId, FString& ErrorMessage)
{
    // 再次验证
    ErrorMessage = GameSession->ApproveLogin(Options);
    if (!ErrorMessage.IsEmpty())
    {
        return nullptr;
    }

    // 生成PlayerController
    APlayerController* NewPC = SpawnPlayerControllerCommon(InRemoteRole, FVector::ZeroVector,
                                                           FRotator::ZeroRotator, PlayerControllerClass);

    // 初始化玩家信息
    InitNewPlayer(NewPC, UniqueId, Options, Portal);

    return NewPC;
}

// Step 3: PostLogin - 登录后处理
void AGameModeBase::PostLogin(APlayerController* NewPlayer)
{
    // 更新玩家计数
    NumPlayers++;

    // 初始化玩家HUD
    NewPlayer->ClientSetHUD(HUDClass);

    // 让GameSession处理
    GameSession->PostLogin(NewPlayer);

    // 广播事件
    FGameModeEvents::GameModePostLoginEvent.Broadcast(this, NewPlayer);
}
```

### 2.2 GameSession验证

```cpp
// GameSession.cpp

FString AGameSession::ApproveLogin(const FString& Options)
{
    AGameModeBase* GameMode = GetWorld()->GetAuthGameMode();

    // 检查是否为观察者
    int32 SpectatorOnly = UGameplayStatics::GetIntOption(Options, TEXT("SpectatorOnly"), 0);

    // 检查服务器容量
    if (AtCapacity(SpectatorOnly == 1))
    {
        return TEXT("Server full.");
    }

    // 检查分屏玩家数量
    int32 SplitscreenCount = UGameplayStatics::GetIntOption(Options, TEXT("SplitscreenCount"), 0);
    if (SplitscreenCount > MaxSplitscreensPerConnection)
    {
        return TEXT("Maximum splitscreen players");
    }

    // 可以添加自定义验证逻辑
    // 例如：白名单、黑名单、密码验证等

    return TEXT("");  // 空字符串表示允许
}

bool AGameSession::AtCapacity(bool bSpectator)
{
    AGameModeBase* GameMode = GetWorld()->GetAuthGameMode();

    if (bSpectator)
    {
        // 检查观察者容量
        return (GameMode->GetNumSpectators() >= MaxSpectators);
    }
    else
    {
        // 检查玩家容量
        const int32 MaxPlayersToUse = CVarMaxPlayersOverride.GetValueOnGameThread() > 0 ?
                                       CVarMaxPlayersOverride.GetValueOnGameThread() : MaxPlayers;
        return ((MaxPlayersToUse > 0) && (GameMode->GetNumPlayers() >= MaxPlayersToUse));
    }
}
```

### 2.3 自定义登录验证

```cpp
// MyGameSession.h
UCLASS()
class AMyGameSession : public AGameSession
{
    GENERATED_BODY()

public:
    virtual FString ApproveLogin(const FString& Options) override;
    virtual void PostLogin(APlayerController* NewPlayer) override;

private:
    // 服务器密码
    UPROPERTY(Config)
    FString ServerPassword;

    // 玩家白名单
    TArray<FString> Whitelist;
};

// MyGameSession.cpp
FString AMyGameSession::ApproveLogin(const FString& Options)
{
    // 先调用父类验证
    FString ErrorMsg = Super::ApproveLogin(Options);
    if (!ErrorMsg.IsEmpty())
    {
        return ErrorMsg;
    }

    // 自定义：密码验证
    FString Password = UGameplayStatics::ParseOption(Options, TEXT("Password"));
    if (!ServerPassword.IsEmpty() && Password != ServerPassword)
    {
        return TEXT("Invalid password");
    }

    // 自定义：白名单验证
    FString PlayerId = UGameplayStatics::ParseOption(Options, TEXT("PlayerId"));
    if (Whitelist.Num() > 0 && !Whitelist.Contains(PlayerId))
    {
        return TEXT("Not in whitelist");
    }

    return TEXT("");
}
```

---

## 3. 服务器Tick机制

### 3.1 服务器主循环

```
┌─────────────────────────────────────────────────────────────────┐
│                        服务器Tick流程                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  FEngineLoop::Tick()                                            │
│  │                                                               │
│  ├── 1. TickDispatch (接收数据)                                 │
│  │   └── NetDriver::TickDispatch()                              │
│  │       ├── 接收所有Packet                                      │
│  │       ├── 解析Bunch                                           │
│  │       └── 处理RPC调用                                         │
│  │                                                               │
│  ├── 2. World Tick (游戏逻辑)                                   │
│  │   ├── GameMode::Tick()                                        │
│  │   ├── Actor::Tick()                                           │
│  │   └── 物理模拟                                                │
│  │                                                               │
│  ├── 3. TickFlush (发送数据)                                    │
│  │   └── NetDriver::TickFlush()                                  │
│  │       ├── 复制所有Actor                                       │
│  │       ├── 发送所有Bunch                                       │
│  │       └── 发送所有Packet                                      │
│  │                                                               │
│  └── 4. PostTickFlush                                           │
│      └── 清理工作                                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 TickFlush详解

```cpp
// NetDriver.cpp

void UNetDriver::TickFlush(float DeltaSeconds)
{
    if (!IsServer())
        return;

    // 1. 更新网络统计
    UpdateNetStats(DeltaSeconds);

    // 2. 复制所有需要同步的Actor
    ServerReplicateActors(DeltaSeconds);

    // 3. 发送所有连接的数据
    for (UNetConnection* Connection : ClientConnections)
    {
        if (Connection->IsNetReady(false))
        {
            Connection->FlushNet();
        }
    }
}

void UNetDriver::ServerReplicateActors(float DeltaSeconds)
{
    // 获取所有需要复制的Actor
    FNetworkObjectList& ObjectList = GetNetworkObjectList();

    // 对每个连接进行复制
    for (UNetConnection* Connection : ClientConnections)
    {
        // 按优先级排序Actor
        TArray<FNetworkObjectInfo*> PrioritizedActors;
        PrioritizeActors(Connection, ObjectList, PrioritizedActors);

        // 复制Actor到该连接
        for (FNetworkObjectInfo* ActorInfo : PrioritizedActors)
        {
            if (ActorInfo->Actor->NetDormancy <= DORM_DormantAll)
            {
                ReplicateActor(ActorInfo->Actor, Connection);
            }
        }
    }
}
```

### 3.3 服务器帧率控制

```cpp
// GameNetworkManager.cpp

// 服务器最大帧率
static TAutoConsoleVariable<int32> CVarNetServerMaxTickRate(
    TEXT("net.NetServerMaxTickRate"),
    30,
    TEXT("Maximum server tick rate")
);

// 帧时间计算
float GetNetTickTime()
{
    int32 MaxTickRate = CVarNetServerMaxTickRate.GetValueOnGameThread();
    if (MaxTickRate > 0)
    {
        return 1.0f / MaxTickRate;  // 例如：30fps = 33.3ms
    }
    return 0.0f;  // 不限制
}
```

### 3.4 带宽控制

```cpp
// 每个连接的带宽限制

// 最大更新速率（每秒发送次数）
MaxClientUpdateRate = 100  // 默认值

// 带宽限制
MaxInternetClientRate = 10000  // 10KB/s
MinInternetClientRate = 5000   // 5KB/s

// 动态调整
void UNetConnection::UpdateRateController()
{
    // 根据网络状况动态调整
    if (CurrentNetSpeed > TargetNetSpeed)
    {
        // 降低更新频率
        UpdateRate = FMath::Max(UpdateRate - 1, MinClientUpdateRate);
    }
    else if (CurrentNetSpeed < TargetNetSpeed * 0.8f)
    {
        // 提高更新频率
        UpdateRate = FMath::Min(UpdateRate + 1, MaxClientUpdateRate);
    }
}
```

---

## 4. 性能优化

### 4.1 网络性能配置

```ini
; DefaultEngine.ini

[/Script/Engine.GameNetDriver]
; 服务器帧率（越高越平滑，但CPU开销越大）
NetServerMaxTickRate=30

; 客户端更新率
MaxClientUpdateRate=60

; 带宽限制
MaxInternetClientRate=50000
MinInternetClientRate=5000

; 连接超时
InitialConnectTimeout=120.0
ConnectionTimeout=120.0

; 休眠优化
bOnlyRelevantToOwner=false
```

### 4.2 Actor网络优化

```cpp
// Actor网络属性优化

UCLASS()
class AMyActor : public AActor
{
    GENERATED_BODY()

public:
    // 1. 合理设置NetUpdateFrequency
    // 高频更新（如玩家）
    UPROPERTY(EditDefaultsOnly)
    float NetUpdateFrequency = 100.0f;

    // 低频更新（如静态物体）
    UPROPERTY(EditDefaultsOnly)
    float NetUpdateFrequency = 10.0f;

    // 2. 使用NetPriority排序
    // 重要Actor优先级高
    UPROPERTY(EditDefaultsOnly)
    float NetPriority = 3.0f;  // 默认1.0

    // 3. 使用NetDormancy减少更新
    virtual void BeginPlay() override
    {
        Super::BeginPlay();

        // 不需要频繁更新的Actor设为休眠
        if (!bNeedsFrequentUpdate)
        {
            SetNetDormancy(DORM_DormantAll);
        }
    }

    // 4. 使用bOnlyRelevantToOwner
    // 只对拥有者相关的Actor
    UPROPERTY(EditDefaultsOnly)
    uint32 bOnlyRelevantToOwner : 1;

    // 5. 使用NetUseOwnerRelevancy
    // 跟随拥有者的相关性
    UPROPERTY(EditDefaultsOnly)
    uint32 NetUseOwnerRelevancy : 1;
};
```

### 4.3 属性复制优化

```cpp
// 属性复制优化

UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    // 1. 使用条件复制
    UPROPERTY(ReplicatedUsing=OnRep_Health, VisibleAnywhere)
    float Health;

    UFUNCTION()
    void OnRep_Health(float OldHealth);

    // 2. 只在需要时复制
    UPROPERTY(Replicated, VisibleAnywhere, Category="Combat")
    float LastDamageTime;

    // 3. 使用RepNotify减少RPC
    // 不推荐
    UFUNCTION(Reliable, Server)
    void ServerSetHealth(float NewHealth);

    // 推荐
    UPROPERTY(ReplicatedUsing=OnRep_Health)
    float Health;

    // 4. 避免复制大数组
    // 使用FastArraySerializer代替
    UPROPERTY(Replicated)
    FMyFastArray MyArray;
};
```

### 4.4 性能监控

```cpp
// 服务器性能统计

// 控制台命令
// stat net - 显示网络统计
// stat fps - 显示帧率
// stat unit - 显示各线程耗时

// 日志输出
UE_LOG(LogNet, Log, TEXT("Server Stats:"));
UE_LOG(LogNet, Log, TEXT("  Tick Rate: %f"), 1.0f / DeltaSeconds);
UE_LOG(LogNet, Log, TEXT("  Players: %d"), NumPlayers);
UE_LOG(LogNet, Log, TEXT("  Actors: %d"), GetNetworkObjectList().GetActiveObjects().Num());
UE_LOG(LogNet, Log, TEXT("  Out Rate: %f KB/s"), OutRate / 1024.0f);
UE_LOG(LogNet, Log, TEXT("  In Rate: %f KB/s"), InRate / 1024.0f);

// CSV性能分析
// 开启: csvprofile start
// 停止: csvprofile stop
// 自动记录关键性能指标
```

---

## 5. 问题排查

### 5.1 常见问题与解决

#### 问题1：玩家无法连接

```
症状：客户端连接超时或被拒绝

排查步骤：
1. 检查端口是否开放
   - netstat -an | findstr 7777

2. 检查防火墙设置
   - Windows防火墙允许端口
   - 云服务器安全组配置

3. 检查服务器日志
   - 查看 -log 输出
   - 搜索 "Connection" 相关日志

4. 检查网络配置
   - -multihome 参数是否正确
   - IP地址是否正确

解决方案：
MyGameServer.exe -log -port=7777 -multihome=0.0.0.0
```

#### 问题2：玩家掉线

```
症状：玩家频繁断开连接

排查步骤：
1. 检查超时设置
   - ConnectionTimeout
   - KeepAliveTime

2. 检查网络质量
   - ping测试
   - 丢包率检查

3. 检查服务器负载
   - CPU使用率
   - 内存使用率

解决方案：
[/Script/Engine.GameNetDriver]
ConnectionTimeout=300.0
KeepAliveTime=0.2
```

#### 问题3：服务器卡顿

```
症状：服务器帧率低，游戏卡顿

排查步骤：
1. 使用stat命令分析
   stat unit - 各线程耗时
   stat game - 游戏逻辑耗时
   stat net - 网络耗时

2. 分析Actor数量
   - 检查Actor数量是否过多
   - 检查复制频率

3. 检查物理模拟
   - 物理计算是否过重

解决方案：
- 减少NetUpdateFrequency
- 使用NetDormancy
- 优化物理设置
- 增加服务器硬件配置
```

### 5.2 调试技巧

```cpp
// 1. 网络日志级别
// 命令行参数
-log -LogCmds="LogNet Verbose"

// 2. 可靠传输调试
net.Reliable.Debug=1

// 3. 网络数据包记录
net.PacketRecording.Enabled=1

// 4. 网络模拟（延迟/丢包）
net.PktLag=100        // 100ms延迟
net.PktLagVariance=20 // 20ms抖动
net.PktLoss=5         // 5%丢包

// 5. 自定义日志
UE_LOG(LogNet, Warning, TEXT("Connection %s: %s"),
       *Connection->GetName(), *Message);
```

### 5.3 性能分析工具

```
┌─────────────────────────────────────────────────────────────────┐
│                        性能分析工具                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 控制台命令                                                   │
│     stat net      - 网络统计                                    │
│     stat fps      - 帧率统计                                    │
│     stat unit     - 线程耗时                                    │
│     stat game     - 游戏逻辑统计                                │
│     dumpticks     - 输出所有Tick的Actor                         │
│                                                                  │
│  2. CSV Profiler                                                │
│     csvprofile start   - 开始记录                               │
│     csvprofile stop    - 停止并保存                             │
│                                                                  │
│  3. Session Frontend                                            │
│     Editor -> Tools -> Session Frontend                         │
│     - 实时性能监控                                               │
│     - 网络统计                                                   │
│                                                                  │
│  4. Unreal Insights                                             │
│     Trace插件                                                    │
│     - 帧时间分析                                                 │
│     - 网络追踪                                                   │
│                                                                  │
│  5. 外部工具                                                     │
│     Wireshark - 网络数据包分析                                  │
│     Process Monitor - 进程监控                                  │
│     PerfMon - Windows性能监控                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 课程总结

本课深入讲解了DS服务器的核心机制：

| 主题 | 要点 |
|------|------|
| 连接流程 | Hello → Welcome → Login → PostLogin → Join |
| 登录验证 | PreLogin → Login → PostLogin 三阶段验证 |
| Tick机制 | TickDispatch → WorldTick → TickFlush |
| 性能优化 | 帧率控制、带宽管理、Actor优化 |
| 问题排查 | 连接问题、掉线问题、卡顿问题 |

---

## 实践建议

1. **搭建测试环境**
   - 本地启动DS测试基本功能
   - 使用多台机器测试网络连接

2. **监控工具**
   - 熟练使用stat命令
   - 学会分析日志文件

3. **性能调优**
   - 根据游戏类型调整帧率
   - 合理设置Actor复制频率

4. **日志规范**
   - 关键操作记录日志
   - 便于问题排查

---

## 参考资料

- Engine/Source/Runtime/Engine/Private/GameModeBase.cpp
- Engine/Source/Runtime/Engine/Private/GameSession.cpp
- Engine/Source/Runtime/Engine/Private/NetDriver.cpp
- Engine/Source/Runtime/Engine/Private/DataChannel.cpp
- Engine/Source/Runtime/Engine/Private/LevelTick.cpp
