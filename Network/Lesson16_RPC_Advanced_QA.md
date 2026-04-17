# UE5 RPC 高级问题与解决方案

本文档整理了UE5网络编程中RPC相关的常见问题与解决方案。

---

## 目录

1. [RPC时序问题](#1-rpc时序问题)
2. [RPC时序不一致的设计考虑](#2-rpc时序不一致的设计考虑)
3. [RPC依赖属性同步的时序问题](#3-rpc依赖属性同步的时序问题)
4. [RPC执行失败的回调机制](#4-rpc执行失败的回调机制)
5. [可靠RPC堆积检测](#5-可靠rpc堆积检测)

---

## 1. RPC时序问题

### 问题本质

在UE5中，RPC **不保证时序**。这是UE网络系统的关键特性之一。

#### 不保证顺序的原因

- RPC通过UDP协议传输，本身就是无连接、不保证顺序的
- 网络延迟、丢包重传等因素会导致到达顺序与发送顺序不同
- 不同客户端到服务器的路由路径可能不同

#### 实际表现

```cpp
// 客户端A发送顺序：
ClientRPC_A();  // 第1个发出
ClientRPC_B();  // 第2个发出
ClientRPC_C();  // 第3个发出

// 服务器接收顺序可能是：
// RPC_B → RPC_A → RPC_C（顺序被打乱）
```

### Reliable vs Unreliable

| 类型 | 特性 |
|------|------|
| **Reliable RPC** | 保证最终送达，但**仍不保证顺序** |
| **Unreliable RPC** | 可能丢失，也可能乱序 |

### 同一个Actor的RPC

即使是同一个Actor上的多个Reliable RPC，也不保证按发送顺序到达。但UE会对**同一个RPC函数的多次调用**进行排队处理（Reliable情况下）。

### 解决方案

#### 方法一：使用可靠的Replication代替RPC

```cpp
// 如果需要严格时序，用属性复制
UPROPERTY(Replicated)
int32 ActionSequence;  // 通过OnRep触发，保证最终一致性
```

#### 方法二：在RPC中携带序列号

```cpp
UFUNCTION(Server, Reliable)
void ServerDoAction(int32 SequenceNumber, FActionData Data);
// 服务器端按SequenceNumber排序处理
```

#### 方法三：使用Gameplay Ability System (GAS)

- GAS内置了预测和同步机制
- 自动处理客户端预测和服务器校正

---

## 2. RPC时序不一致的设计考虑

### 需要考虑时序的场景

#### 状态依赖型操作

```cpp
// 危险设计：假设顺序执行
UFUNCTION(Server, Reliable)
void ServerEquipWeapon(AWeapon* Weapon);

UFUNCTION(Server, Reliable)
void ServerSetAmmoCount(int32 Count);

// 问题：如果SetAmmoCount先到达，武器还未装备
```

#### 连续动作序列

```cpp
// 组合技：A → B → C
// 如果顺序错乱，整个连招逻辑会崩
UFUNCTION(Server, Reliable)
void ServerPerformComboStep(int32 StepIndex);
```

#### 资源交易/消耗

```cpp
// 购买物品 → 扣金币 → 添加物品
// 顺序错乱可能导致状态不一致
```

### 不需要考虑时序的场景

#### 独立的、幂等的状态更新

```cpp
// 每次都是完整状态，不依赖前一次
UFUNCTION(Server, Reliable)
void ServerUpdatePosition(FVector NewPos);

// 最新的位置覆盖旧的，顺序无关
```

#### 一次性事件触发

```cpp
// 独立事件，互不影响
UFUNCTION(Server, Reliable)
void ServerPlayerDeath();

UFUNCTION(NetMulticast, Reliable)
void MulticastPlaySound(USoundBase* Sound);
```

#### 高频且可丢失的数据

```cpp
// Unreliable本身就是允许丢包的，时序更不重要
UFUNCTION(Server, Unreliable)
void ServerUpdateAimDirection(FRotator Rotation);
```

### 设计策略

#### 策略一：合并为单一RPC

```cpp
// 不好的设计
ServerSetHealth(100);
ServerSetMaxHealth(150);
ServerSetArmor(50);

// 好的设计：一次性发送完整状态
UFUNCTION(Server, Reliable)
void ServerUpdateCharacterStats(const FCharacterStats& Stats);
```

#### 策略二：服务器作为权威

```cpp
// 客户端只发送意图，服务器决定执行顺序
UFUNCTION(Server, Reliable)
void ServerRequestAction(EActionType Action);

// 服务器本地有Action队列，按自己的逻辑处理
void AMyCharacter::ServerRequestAction_Implementation(EActionType Action)
{
    ActionQueue.Enqueue(Action);
    ProcessActionQueue();
}
```

#### 策略三：序列号 + 服务器重排序

```cpp
// 客户端
void PerformCombo()
{
    CurrentSequence++;
    ServerComboAction(CurrentSequence, ActionData);
}

UFUNCTION(Server, Reliable)
void ServerComboAction(int32 Sequence, FActionData Data);

// 服务器
TMap<APlayerController*, int32> LastProcessedSequence;

void ServerComboAction_Implementation(int32 Sequence, FActionData Data)
{
    // 忽略旧的或重复的包
    if (Sequence <= LastProcessedSequence[PlayerController])
        return;

    // 缓存乱序到达的包
    PendingActions.Add(Sequence, Data);

    // 按顺序处理
    while (PendingActions.Contains(ExpectedSequence))
    {
        ProcessAction(PendingActions[ExpectedSequence]);
        PendingActions.Remove(ExpectedSequence);
        ExpectedSequence++;
    }
}
```

#### 策略四：改用属性复制

```cpp
// 对于需要严格同步的状态
UPROPERTY(ReplicatedUsing=OnRep_CurrentWeapon)
AWeapon* CurrentWeapon;

UPROPERTY(ReplicatedUsing=OnRep_AmmoCount)
int32 AmmoCount;

// 属性复制保证最终一致性，且带宽更优
```

### 场景决策表

| 场景 | 是否需要考虑时序 | 推荐方案 |
|------|-----------------|---------|
| 状态更新（位置、朝向） | ❌ | 直接覆盖 |
| 完整状态同步 | ❌ | 合并RPC或用Replication |
| 连续动作序列 | ✅ | 序列号/服务器队列 |
| 资源消耗/交易 | ✅ | 服务器权威验证 |
| 独立事件（死亡、得分） | ❌ | 无需处理 |
| 高频输入（移动、瞄准） | ❌ | Unreliable + 最新值覆盖 |

### 核心原则

1. **能用状态同步解决的就不用RPC**
2. **必须用RPC时让服务器做权威决策**

---

## 3. RPC依赖属性同步的时序问题

### 问题场景

```
T1：客户端通过属性同步状态A给服务器
T2：客户端调用RPC B，B依赖状态A

如果此时A还没到服务器，B执行时可能使用旧值
```

### 问题本质

```
客户端时间线:
T1: 属性A复制 → 服务器
T2: RPC B(A依赖) → 服务器

服务器可能收到:
情况1: A → B ✓ 正常
情况2: B → A ✗ B执行时A还是旧值
```

属性复制和RPC走的是不同的网络通道，无法保证到达顺序。

### 解决方案

#### 方案一：RPC携带完整依赖数据（推荐）

```cpp
// 原来的设计（有问题）
UPROPERTY(Replicated)
FWeaponState WeaponState;  // A

UFUNCTION(Server, Reliable)
void ServerFire();  // B，依赖WeaponState

// 改进后的设计
UFUNCTION(Server, Reliable)
void ServerFire(const FWeaponState& InWeaponState);  // 把依赖数据打包带走
```

```cpp
void AMyCharacter::Fire()
{
    // 客户端直接传当前状态
    ServerFire(WeaponState);
}

void AMyCharacter::ServerFire_Implementation(const FWeaponState& InWeaponState)
{
    // 服务器使用客户端传来的数据
    // 或者用服务器的权威数据做验证
    if (ValidateWeaponState(InWeaponState))
    {
        ProcessFire(InWeaponState);
    }
}
```

#### 方案二：序列号同步

```cpp
// 客户端
UPROPERTY(Replicated)
FWeaponState WeaponState;

int32 StateSequence = 0;

UFUNCTION(Server, Reliable)
void ServerFire(int32 Sequence);

// 属性复制时带上序列号
void OnRep_WeaponState()
{
    StateSequence++;
    WeaponState.Sequence = StateSequence;
}

void Fire()
{
    ServerFire(StateSequence);
}
```

```cpp
// 服务器端
TMap<APlayerController*, int32> LastSyncedSequence;

void ServerFire_Implementation(int32 Sequence)
{
    int32 LastSynced = LastSyncedSequence[PlayerController];

    if (Sequence > LastSynced)
    {
        // 状态还没到，缓存这个RPC
        PendingRPCs.Add(Sequence, FPendingRPC{Sequence, CurrentTime});

        // 设置超时检查
        GetWorldTimerManager().SetTimer(TimeoutHandle, this,
            &AMyCharacter::CheckPendingRPCs, 0.1f, true);
    }
    else
    {
        // 状态已同步，直接执行
        ProcessFire();
    }
}
```

#### 方案三：服务器权威计算（最佳实践）

```cpp
// 客户端只发"意图"，不传状态
UFUNCTION(Server, Reliable)
void ServerFire();

void AMyCharacter::ServerFire_Implementation()
{
    // 服务器直接用自己维护的权威状态
    FWeaponState& ServerWeaponState = GetWeaponState();

    ProcessFire(ServerWeaponState);
}
```

#### 方案四：客户端等待确认

```cpp
UPROPERTY(ReplicatedUsing=OnRep_WeaponState)
FWeaponState WeaponState;

UFUNCTION(Server, Reliable)
void ServerSetWeaponState(const FWeaponState& NewState);

UFUNCTION(Client, Reliable)
void ClientAckWeaponState(int32 Sequence);

// 客户端
void SetWeaponState(const FWeaponState& NewState)
{
    PendingSequence++;
    ServerSetWeaponState(PendingSequence, NewState);
    // 不立即调用Fire，等待确认
}

void ClientAckWeaponState_Implementation(int32 Sequence)
{
    if (Sequence == PendingSequence)
    {
        // 状态已同步，现在可以安全调用Fire
        ServerFire();
    }
}
```

#### 方案五：属性+RPC合并发送

```cpp
// 不再单独复制属性，改用RPC一次性发送
UFUNCTION(Server, Reliable)
void ServerFireWithState(const FWeaponState& NewState);

void AMyCharacter::Fire()
{
    ServerFireWithState(CurrentWeaponState);
}

void AMyCharacter::ServerFireWithState_Implementation(const FWeaponState& NewState)
{
    // 服务器更新状态
    WeaponState = NewState;

    // 然后执行逻辑
    ProcessFire();

    // 如果其他客户端需要同步，服务器再广播
    MulticastWeaponStateChanged(WeaponState);
}
```

### 方案对比

| 方案 | 复杂度 | 带宽消耗 | 延迟 | 适用场景 |
|------|--------|---------|------|---------|
| RPC携带数据 | 低 | 中 | 最低 | 通用方案 |
| 序列号等待 | 高 | 低 | 中 | 强顺序要求 |
| 服务器权威 | 最低 | 最低 | 低 | 服务器可计算状态 |
| 等待确认 | 中 | 低 | 高 | 严格一致性 |
| 合并发送 | 低 | 中 | 低 | 状态变化不频繁 |

### 推荐选择

**第一选择**：服务器权威
```cpp
// 如果服务器能自己维护状态，这是最优解
UFUNCTION(Server, Reliable)
void ServerFire();  // 服务器用自己计算的状态
```

**第二选择**：RPC携带数据
```cpp
// 状态无法在服务器计算时，把依赖数据打包传递
UFUNCTION(Server, Reliable)
void ServerFire(const FWeaponState& State);
```

**第三选择**：合并发送
```cpp
// 状态变化和操作总是一起发生时
UFUNCTION(Server, Reliable)
void ServerFireWithState(const FWeaponState& State);
```

### 核心原则

1. **服务器权威优先**：能用服务器状态就用服务器状态
2. **数据跟随操作**：依赖数据随RPC一起发送
3. **避免隐式依赖**：不要假设RPC执行时属性已同步
4. **减少网络往返**：不要为了同步增加额外的确认步骤

---

## 4. RPC执行失败的回调机制

### 问题本质

RPC是单向的，没有返回值机制：

```cpp
// 客户端调用
bool Result = ServerDoSomething();  // ❌ RPC不支持返回值

// RPC定义
UFUNCTION(Server, Reliable)
void ServerDoSomething();  // 返回值只能是void
```

### 解决方案

#### 方案一：显式回调RPC（推荐）

```cpp
// 客户端→服务器
UFUNCTION(Server, Reliable)
void ServerDoSomething(int32 RequestID);

// 服务器→客户端（回调）
UFUNCTION(Client, Reliable)
void ClientOnDoSomethingResult(int32 RequestID, bool bSuccess, const FString& ErrorMsg);
```

```cpp
// 客户端
int32 NextRequestID = 0;
TMap<int32, TFunction<void(bool, const FString&)>> PendingRequests;

void DoSomething()
{
    int32 RequestID = ++NextRequestID;

    // 注册回调
    PendingRequests.Add(RequestID, [](bool bSuccess, const FString& Error) {
        if (bSuccess)
        {
            UE_LOG(LogTemp, Log, TEXT("操作成功"));
        }
        else
        {
            UE_LOG(LogTemp, Warning, TEXT("操作失败: %s"), *Error);
        }
    });

    ServerDoSomething(RequestID);
}

void ClientOnDoSomethingResult_Implementation(int32 RequestID, bool bSuccess, const FString& ErrorMsg)
{
    if (auto* Callback = PendingRequests.Find(RequestID))
    {
        (*Callback)(bSuccess, ErrorMsg);
        PendingRequests.Remove(RequestID);
    }
}
```

#### 方案二：Future/Promise模式

```cpp
// RPCRequestManager.h
UCLASS()
class URPCRequestManager : public UObject
{
    GENERATED_BODY()

public:
    // 创建一个带超时的请求
    TFuture<FRPCResult> CreateRequest(
        const FString& RPCName,
        float TimeoutSeconds = 5.0f
    );

    // 服务器调用此方法响应请求
    void HandleResponse(int32 RequestID, const FRPCResult& Result);

private:
    struct FPendingRequest
    {
        TPromise<FRPCResult> Promise;
        float TimeoutTime;
        FString RPCName;
    };

    TMap<int32, FPendingRequest> PendingRequests;
    int32 NextRequestID = 0;
};

USTRUCT()
struct FRPCResult
{
    GENERATED_BODY()

    UPROPERTY()
    bool bSuccess = false;

    UPROPERTY()
    FString ErrorMessage;

    UPROPERTY()
    TArray<uint8> Payload;
};
```

```cpp
// 使用方式
void AMyCharacter::PurchaseItem(int32 ItemID)
{
    RPCManager->CreateRequest("PurchaseItem", 5.0f)
        .Then([this, ItemID](FRPCResult Result)
        {
            if (Result.bSuccess)
            {
                ShowMessage(TEXT("购买成功"));
            }
            else
            {
                ShowErrorMessage(Result.ErrorMessage);
            }
        });

    ServerPurchaseItem(ItemID, RPCManager->GetCurrentRequestID());
}
```

#### 方案三：状态同步 + 客户端监听

```cpp
// 服务器维护的操作结果
UPROPERTY(ReplicatedUsing=OnRep_LastOperationResult)
FOperationResult LastOperationResult;

USTRUCT()
struct FOperationResult
{
    GENERATED_BODY()

    UPROPERTY()
    int32 RequestID = -1;

    UPROPERTY()
    bool bSuccess = false;

    UPROPERTY()
    FString ErrorMessage;
};

// 客户端监听变化
UFUNCTION()
void OnRep_LastOperationResult()
{
    if (LastOperationResult.RequestID == MyPendingRequestID)
    {
        if (LastOperationResult.bSuccess)
        {
            // 处理成功
        }
        else
        {
            // 处理失败
        }
        MyPendingRequestID = -1;
    }
}
```

#### 方案四：基于事件的系统

```cpp
// 定义事件
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnOperationComplete, int32, RequestID, bool, bSuccess);

UCLASS()
class AMyGameState : public AGameStateBase
{
    GENERATED_BODY()

public:
    UPROPERTY(BlueprintAssignable, Category = "Events")
    FOnOperationComplete OnOperationComplete;

    UFUNCTION(Client, Reliable)
    void ClientBroadcastOperationResult(int32 RequestID, bool bSuccess);
};
```

#### 方案五：统一错误码设计

```cpp
UENUM(BlueprintType)
enum class ERPCErrorCode : uint8
{
    None = 0,
    InvalidRequest = 1,
    NotAuthorized = 2,
    InsufficientResources = 3,
    CooldownActive = 4,
    TargetInvalid = 5,
    ServerError = 99
};

USTRUCT(BlueprintType)
struct FRPCError
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadWrite)
    ERPCErrorCode Code = ERPCErrorCode::None;

    UPROPERTY(BlueprintReadWrite)
    FString Message;
};
```

### 方案对比

| 方案 | 复杂度 | 适用场景 | 优点 | 缺点 |
|------|--------|---------|------|------|
| 回调RPC | 低 | 简单操作 | 直接明了 | 每个RPC都要写回调 |
| Future/Promise | 中 | 复杂业务 | 代码优雅 | 需要封装 |
| 状态同步 | 低 | 低频操作 | 利用现有机制 | 不够实时 |
| 事件系统 | 中 | 多监听者 | 解耦 | 需要GameState |

### 必须处理的场景

```cpp
// 资源消耗型操作（购买、升级、消耗）
ServerPurchaseItem(ItemID);  // 必须知道是否成功

// 状态改变型操作
ServerEquipWeapon(WeaponID);  // 可能失败（背包满、等级不足等）

// 需要授权的操作
ServerJoinGuild(GuildID);  // 可能被拒绝
```

### 可以忽略的场景

```cpp
// 幂等且非关键的操作
MulticastPlaySound(Sound);  // 失败也无所谓

// 高频状态更新
ServerUpdatePosition(Pos);  // 下次会覆盖
```

### 核心原则

**永远不要假设RPC执行成功**，必须要有反馈机制。

---

## 5. 可靠RPC堆积检测

### 堆积原理

```
客户端                                    服务器
  │                                         │
  │──── RPC 1 (Reliable) ──────────────────>│
  │──── RPC 2 (Reliable) ──────────────────>│
  │──── RPC 3 (Reliable) ────┐              │
  │──── RPC 4 (Reliable) ────┤ 发送队列堆积  │
  │──── RPC 5 (Reliable) ────┤              │
  │──── RPC 6 (Reliable) ────┘              │
```

可靠RPC需要ACK确认，如果网络拥塞或服务器处理慢，会在发送队列中堆积。

### 内置检测工具

#### 控制台命令

```
stat net              // 显示网络统计
stat netprofile       // 网络性能分析
netprofile            // 启用网络性能分析
show net              // 显示网络调试信息
net.ListClientActors  // 列出客户端Actor
```

#### 控制台变量

```ini
; DefaultEngine.ini
net.ClientNetSpeedUpdateFrequency=...
net.MaxClientUpdateRate=...
net.ServerMaxTickRate=...

; 调试相关
net.VerifyPeerConnections=1
net.TestPacketSimulationLoss=0.1  ; 模拟10%丢包
net.TestPacketSimulationLag=100   ; 模拟100ms延迟
```

### 代码检测方法

#### 方案一：监控UNetConnection

```cpp
UCLASS()
class UNetworkMonitorComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    virtual void TickComponent(float DeltaTime, ELevelTick TickType,
                               FActorComponentTickFunction* ThisTickFunction) override
    {
        if (APlayerController* PC = Cast<APlayerController>(GetOwner()))
        {
            if (UNetConnection* NetConn = PC->GetNetConnection())
            {
                CheckReliableBuffer(NetConn);
            }
        }
    }

private:
    void CheckReliableBuffer(UNetConnection* Connection)
    {
        int32 ReliableBufferCount = 0;

        for (UChannel* Channel : Connection->Channels)
        {
            if (UActorChannel* ActorChannel = Cast<UActorChannel>(Channel))
            {
                ReliableBufferCount += ActorChannel->ReliableBuffers.Num();
            }
        }

        const int32 WarningThreshold = 50;
        const int32 CriticalThreshold = 100;

        if (ReliableBufferCount > CriticalThreshold)
        {
            UE_LOG(LogNet, Error, TEXT("Critical: Reliable buffer overflow! Count: %d"), ReliableBufferCount);
            OnReliableBufferCritical.Broadcast(ReliableBufferCount);
        }
        else if (ReliableBufferCount > WarningThreshold)
        {
            UE_LOG(LogNet, Warning, TEXT("Warning: Reliable buffer high: %d"), ReliableBufferCount);
            OnReliableBufferWarning.Broadcast(ReliableBufferCount);
        }
    }

    UPROPERTY(BlueprintAssignable)
    FOnReliableBufferEvent OnReliableBufferWarning;

    UPROPERTY(BlueprintAssignable)
    FOnReliableBufferEvent OnReliableBufferCritical;
};
```

#### 方案二：完整监控系统

```cpp
USTRUCT(BlueprintType)
struct FNetworkHealthMetrics
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    int32 ReliableBufferDepth = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 UnackedPacketCount = 0;

    UPROPERTY(BlueprintReadOnly)
    float AverageLatencyMs = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    float PacketLossRate = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    bool bIsHealthy = true;

    UPROPERTY(BlueprintReadOnly)
    FString HealthWarning;
};

UCLASS(BlueprintType)
class UNetworkHealthMonitor : public UObject
{
    GENERATED_BODY()

public:
    UFUNCTION(BlueprintCallable, Category = "Network")
    FNetworkHealthMetrics GetNetworkHealthMetrics() const;

    UFUNCTION(BlueprintCallable, Category = "Network")
    bool IsNetworkHealthy() const;

    // 阈值配置
    UPROPERTY(EditDefaultsOnly, Category = "Configuration")
    int32 ReliableBufferWarningThreshold = 30;

    UPROPERTY(EditDefaultsOnly, Category = "Configuration")
    int32 ReliableBufferCriticalThreshold = 80;

    UPROPERTY(EditDefaultsOnly, Category = "Configuration")
    float MaxAcceptableLatencyMs = 200.0f;

    UPROPERTY(EditDefaultsOnly, Category = "Configuration")
    float MaxAcceptablePacketLoss = 0.05f;  // 5%

private:
    void CollectMetrics();
    void EvaluateHealth();

    FNetworkHealthMetrics CurrentMetrics;
};
```

#### 方案三：控制台命令集成

```cpp
UCLASS()
class ANetworkDebugManager : public AActor
{
    GENERATED_BODY()

public:
    UFUNCTION(Exec, Category = "Network Debug")
    void NetDebugQueue()
    {
        DumpReliableQueueStats();
    }

    UFUNCTION(Exec, Category = "Network Debug")
    void NetDebugMonitor(bool bEnable)
    {
        bMonitoringEnabled = bEnable;
    }

private:
    void DumpReliableQueueStats()
    {
        UE_LOG(LogNet, Log, TEXT("=== Reliable RPC Queue Statistics ==="));

        UNetDriver* NetDriver = GWorld->GetNetDriver();
        if (!NetDriver) return;

        int32 TotalQueued = 0;

        for (UNetConnection* Connection : NetDriver->ClientConnections)
        {
            if (!Connection) continue;

            for (UChannel* Channel : Connection->Channels)
            {
                if (UActorChannel* ActorChannel = Cast<UActorChannel>(Channel))
                {
                    int32 ChannelDepth = ActorChannel->ReliableBuffers.Num();
                    if (ChannelDepth > 0)
                    {
                        AActor* Actor = ActorChannel->GetActor();
                        FString ActorName = Actor ? Actor->GetName() : TEXT("Unknown");

                        UE_LOG(LogNet, Log, TEXT("  Actor %s: %d queued RPCs"),
                            *ActorName, ChannelDepth);
                    }
                }
            }
        }

        UE_LOG(LogNet, Log, TEXT("Total queued reliable RPCs: %d"), TotalQueued);
    }
};
```

### 解决RPC堆积的方法

#### 1. 控制RPC发送频率

```cpp
UCLASS()
class AMyCharacter : public ACharacter
{
    float LastFireTime = 0.0f;
    float FireRPCRateLimit = 0.1f;  // 最小间隔100ms

    void Fire()
    {
        float CurrentTime = GetWorld()->GetTimeSeconds();
        if (CurrentTime - LastFireTime < FireRPCRateLimit)
        {
            return;  // 速率限制
        }

        LastFireTime = CurrentTime;
        ServerFire();
    }
};
```

#### 2. 批量发送RPC

```cpp
// 不好的做法
for (int32 i = 0; i < Items.Num(); i++)
{
    ServerPickupItem(Items[i]);  // 10个RPC
}

// 好的做法
UFUNCTION(Server, Reliable)
void ServerPickupItems(const TArray<AItem*>& Items);

ServerPickupItems(Items);  // 1个RPC
```

#### 3. 使用Unreliable替代

```cpp
// 高频、可丢失的操作用Unreliable
UFUNCTION(Server, Unreliable)
void ServerUpdateAimDirection(const FRotator& Rotation);

// 只有真正需要保证送达的才用Reliable
UFUNCTION(Server, Reliable)
void ServerPurchaseItem(int32 ItemID);
```

### 关键指标

| 指标 | 警告阈值 | 严重阈值 |
|------|---------|---------|
| ReliableBufferDepth | > 30 | > 80 |
| AvgLatency | > 200ms | - |
| PacketLoss | > 5% | - |

### 检测方式对比

| 检测方式 | 用途 | 复杂度 |
|---------|------|--------|
| `stat net` | 快速查看 | 低 |
| UNetConnection监控 | 实时检测 | 中 |
| 自定义Channel | 精细控制 | 高 |
| 完整监控系统 | 生产环境 | 高 |

---

## 总结

| 问题 | 核心要点 | 推荐解决方案 |
|------|---------|-------------|
| RPC时序 | 不保证顺序，Reliable也不保证 | 服务器权威 / 携带序列号 |
| 时序设计 | 根据场景决定是否考虑时序 | 合并RPC / 服务器队列 |
| 属性依赖 | 属性复制和RPC走不同通道 | RPC携带依赖数据 / 服务器权威 |
| 失败回调 | RPC无返回值 | 显式回调RPC / Future模式 |
| 堆积检测 | 监控ReliableBufferDepth | 控制台命令 / 自定义监控组件 |

### 核心设计原则

1. **服务器权威优先**：能用服务器状态就用服务器状态
2. **数据跟随操作**：依赖数据随RPC一起发送
3. **不要假设顺序**：永远不假设RPC执行顺序
4. **必须有反馈**：关键操作必须有成功/失败回调
5. **监控网络健康**：实时检测可靠RPC堆积情况
