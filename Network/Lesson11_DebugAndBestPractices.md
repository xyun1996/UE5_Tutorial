# 第十一课：网络调试与最佳实践

> **学习目标**: 掌握网络调试工具和开发最佳实践
> **时长**: 约60分钟
> **前置知识**: 第十课内容

---

## 一、网络调试工具

### 1.1 控制台命令汇总

```
┌─────────────────────────────────────────────────────────────────────┐
│                    常用网络调试命令                                   │
└─────────────────────────────────────────────────────────────────────┘

统计命令:
    stat net              - 显示网络统计信息
    stat game             - 显示游戏统计
    stat fps              - 显示帧率
    stat unit             - 显示各线程时间

网络命令:
    net.Objects           - 显示网络对象
    net.Channels          - 显示通道信息
    net.Demo              - 回放相关
    net.ListClients       - 列出所有客户端

调试命令:
    net.PktDraw           - 绘制数据包
    net.PktOrder          - 数据包排序
    net.DebugProperty     - 调试属性复制
    net.DumpRPC           - 转储RPC调用

连接命令:
    net.KickBan <id>      - 踢出并封禁客户端
    net.Kick <id>         - 踢出客户端
    net.MaxClientRate     - 设置最大客户端速率
```

### 1.2 网络模拟命令

```
; 模拟延迟
net.PktLag=100           - 100ms延迟
net.PktLagVariance=20    - 20ms延迟抖动

; 模拟丢包
net.PktLoss=5            - 5%丢包率
net.PktLossMaxSize=0     - 最大丢包大小
net.PktLossMinSize=0     - 最小丢包大小

; 模拟乱序
net.PktOrder=1           - 启用乱序

; 模拟重复
net.PktDup=2             - 2%重复包
```

---

## 二、统计信息解读

### 2.1 stat net 输出

```
Net
├── InBunches          接收的Bunch数量
├── OutBunches         发送的Bunch数量
├── InPackets          接收的包数量
├── OutPackets         发送的包数量
├── InBytes            接收字节数/秒
├── OutBytes           发送字节数/秒
├── InPacketsLost      丢包数(入)
├── OutPacketsLost     丢包数(出)
├── Actors             活跃Actor数量
├── ReplicatedActors   复制的Actor数量
└── NetUpdateTime      网络更新耗时
```

### 2.2 关键指标分析

```cpp
// 正常范围参考
struct FNetworkMetrics
{
    // 带宽 (互联网游戏)
    static constexpr int32 MaxOutBytesPerSecond = 64000;  // 64KB/s
    static constexpr int32 MaxInBytesPerSecond = 64000;

    // 延迟
    static constexpr float MaxAcceptablePing = 0.1f;  // 100ms
    static constexpr float MaxJitter = 0.02f;         // 20ms

    // 丢包率
    static constexpr float MaxPacketLoss = 0.01f;     // 1%

    // 复制Actor数量
    static constexpr int32 MaxReplicatedActors = 500;
};
```

### 2.3 性能分析工具

```cpp
// 使用网络追踪
#if UE_NET_TRACE_ENABLED
// 在代码中添加追踪点
UE_NET_TRACE_BEGIN(NamedEvent);
// ... 代码
UE_NET_TRACE_END(NamedEvent);
#endif
```

---

## 三、日志系统

### 3.1 网络日志类别

```cpp
// 常用日志类别
DECLARE_LOG_CATEGORY_EXTERN(LogNet, Log, All);
DECLARE_LOG_CATEGORY_EXTERN(LogNetTraffic, Log, All);
DECLARE_LOG_CATEGORY_EXTERN(LogNetPlayerManagement, Log, All);
DECLARE_LOG_CATEGORY_EXTERN(LogNetRpc, Log, All);
```

### 3.2 启用详细日志

```
; 在命令行或配置文件中设置
LogNet Verbose
LogNetTraffic Verbose
LogNetPlayerManagement Verbose
LogNetRpc Verbose
```

### 3.3 自定义日志

```cpp
// 在代码中添加日志
void AMyActor::OnRep_Health(float OldValue)
{
    UE_LOG(LogNet, Log, TEXT("Health changed: %f -> %f on %s"),
        OldValue, Health, *GetName());

    if (Health <= 0.0f)
    {
        UE_LOG(LogNet, Warning, TEXT("Actor %s died"), *GetName());
    }
}

// RPC日志
void AMyActor::ServerFire_Implementation()
{
    UE_LOG(LogNetRpc, Verbose, TEXT("ServerFire called by %s"),
        *GetNetConnection()->PlayerId.ToString());
}
```

---

## 四、常见问题排查

### 4.1 属性不同步

```
问题: 客户端属性未更新

排查步骤:
    1. 检查是否注册到GetLifetimeReplicatedProps
    2. 检查DOREPLIFETIME宏是否正确
    3. 检查属性是否有UPROPERTY(Replicated)标记
    4. 检查Actor是否bReplicates=true
    5. 检查连接状态是否正常

调试命令:
    net.DebugProperty 1
    LogNet Property Verbose
```

### 4.2 RPC不执行

```
问题: RPC调用但未执行

排查步骤:
    1. 检查调用位置是否正确
       - Server RPC: 客户端调用
       - Client RPC: 服务器调用
       - Multicast: 服务器调用

    2. 检查NetOwner
       - Server RPC需要Actor有NetOwner
       - 使用HasNetOwner()检查

    3. 检查连接状态
       - 确保连接未断开

调试命令:
    net.DumpRPC
    LogNetRpc Verbose
```

### 4.3 高延迟/卡顿

```
问题: 网络延迟高，游戏卡顿

排查步骤:
    1. 检查带宽使用 (stat net)
    2. 检查复制Actor数量
    3. 检查更新频率设置
    4. 检查是否有大量可靠RPC堆积
    5. 检查服务器Tick率

优化建议:
    - 使用Push Model
    - 使用NetDormancy
    - 降低NetUpdateFrequency
    - 减少复制属性
```

### 4.4 连接断开

```
问题: 客户端频繁断开

排查步骤:
    1. 检查超时设置
       - InitialConnectTimeout
       - ConnectionTimeout

    2. 检查网络稳定性
       - 使用net.PktLoss模拟
       - 检查丢包率

    3. 检查服务器负载
       - CPU/内存使用
       - 连接数

    4. 检查错误日志
       - LogNet
       - LogNetPlayerManagement
```

---

## 五、调试技巧

### 5.1 断点调试

```cpp
// 关键断点位置
// 服务器端:
UActorChannel::ReplicateActor()       // Actor复制入口
FObjectReplicator::ReplicateProperties() // 属性复制
UNetDriver::TickFlush()               // 发送数据

// 客户端:
UActorChannel::ReceivedBunch()        // 接收Bunch
FObjectReplicator::ReceivedBunch()    // 接收属性
UChannel::ReceivedRawBunch()          // 原始数据
```

### 5.2 数据包捕获

```cpp
// 启用数据包记录
net.RecordPackets 1

// 分析数据包
net.PlayDemo <filename>

// 导出数据包
net.ExportPackets <filename>
```

### 5.3 自定义调试输出

```cpp
// 创建调试Actor
UCLASS()
class ANetworkDebugActor : public AActor
{
    GENERATED_BODY()

public:
    virtual void Tick(float DeltaTime) override
    {
        Super::Tick(DeltaTime);

        if (!HasAuthority())
            return;

        // 显示连接状态
        UNetDriver* NetDriver = GetWorld()->GetNetDriver();
        if (NetDriver)
        {
            for (UNetConnection* Conn : NetDriver->ClientConnections)
            {
                FString DebugInfo = FString::Printf(
                    TEXT("Client: %s, Ping: %.0fms, Out: %d B/s"),
                    *Conn->PlayerId.ToString(),
                    Conn->AvgLag * 1000.0f,
                    Conn->OutBytesPerSecond
                );

                DrawDebugString(
                    GetWorld(),
                    GetActorLocation(),
                    DebugInfo,
                    nullptr,
                    FColor::White,
                    0.0f,
                    true
                );
            }
        }
    }
};
```

---

## 六、最佳实践

### 6.1 服务器权威原则

```cpp
// 好的做法: 服务器验证所有操作
void AMyCharacter::ServerPurchaseItem_Implementation(int32 ItemId)
{
    // 服务器端验证
    if (!CanAffordItem(ItemId))
    {
        return;  // 拒绝非法请求
    }

    // 服务器端执行
    ProcessPurchase(ItemId);
}

// 不好的做法: 信任客户端
void AMyCharacter::ServerPurchaseItem_Implementation(int32 ItemId)
{
    // 直接执行，没有验证
    ProcessPurchase(ItemId);  // 危险！
}
```

### 6.2 最小化复制

```cpp
// 好的做法: 只复制必要的属性
UPROPERTY(Replicated)
int32 Score;          // 必要

UPROPERTY(Replicated)
FVector Location;     // 必要

UPROPERTY()
FString DebugString;  // 不复制

// 使用条件复制
DOREPLIFETIME_CONDITION(AMyActor, PrivateData, COND_OwnerOnly);
```

### 6.3 RPC使用原则

```cpp
// 原则:
// 1. Server RPC用于客户端请求
// 2. Client RPC用于服务器通知
// 3. Multicast用于广播效果

// 好的做法
UFUNCTION(Server, Reliable, WithValidation)
void ServerRequestAction(int32 ActionId);

UFUNCTION(Client, Reliable)
void ClientNotifyResult(bool bSuccess);

UFUNCTION(NetMulticast, Unreliable)
void MulticastPlayEffect(FVector Location);

// 避免过度使用Multicast
// UFUNCTION(NetMulticast, Reliable) // 慎用
// void MulticastUpdateAll();
```

### 6.4 预测与回滚

```cpp
// 客户端预测
void AMyCharacter::MoveForward(float Value)
{
    // 客户端立即执行
    AddMovementInput(GetActorForwardVector(), Value);

    // 通知服务器
    ServerMove(GetActorLocation(), GetActorRotation());
}

// 服务器校正
void AMyCharacter::ServerMove_Validate(FVector Location, FRotator Rotation)
{
    // 验证移动是否合法
    return true;
}

void AMyCharacter::ServerMove_Implementation(FVector Location, FRotator Rotation)
{
    // 服务器执行
    SetActorLocation(Location);
    SetActorRotation(Rotation);

    // 如果需要校正，通知客户端
    if (FVector::Dist(Location, GetActorLocation()) > CorrectionThreshold)
    {
        ClientCorrectPosition(GetActorLocation(), GetActorRotation());
    }
}
```

---

## 七、安全考虑

### 7.1 输入验证

```cpp
// 验证所有RPC参数
bool AMyCharacter::ServerFire_Validate(FVector TargetLocation)
{
    // 基本验证
    if (!TargetLocation.IsValid())
        return false;

    // 业务验证
    float Distance = FVector::Dist(GetActorLocation(), TargetLocation);
    if (Distance > MaxFireRange)
        return false;

    return true;
}
```

### 7.2 防作弊

```cpp
// 服务器验证关键操作
void AMyCharacter::ServerPurchaseItem_Implementation(int32 ItemId)
{
    // 检查金币是否足够
    if (PlayerGold < ItemCosts[ItemId])
    {
        // 记录可疑行为
        UE_LOG(LogNet, Warning, TEXT("Possible cheat: Player %s attempted purchase without gold"),
            *PlayerId.ToString());
        return;
    }

    // 执行购买
    PlayerGold -= ItemCosts[ItemId];
    GiveItem(ItemId);
}
```

---

## 八、代码模板

### 8.1 可复制Actor模板

```cpp
/**
 * 标准可复制Actor模板
 */
UCLASS()
class AReplicableActor : public AActor
{
    GENERATED_BODY()

public:
    AReplicableActor()
    {
        bReplicates = true;
        NetUpdateFrequency = 30.0f;
        NetDormancy = DORM_Awake;
    }

    virtual void GetLifetimeReplicatedProps(
        TArray<FLifetimeProperty>& OutLifetimeProps) const override
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);

        // 添加复制属性
    }

    virtual bool ReplicateSubobjects(
        UActorChannel* Channel,
        FOutBunch* Bunch,
        FReplicationFlags* RepFlags) override
    {
        bool bWrote = Super::ReplicateSubobjects(Channel, Bunch, RepFlags);

        // 复制子对象

        return bWrote;
    }
};
```

### 8.2 安全RPC模板

```cpp
/**
 * 安全RPC模板
 */
UFUNCTION(Server, Reliable, WithValidation)
void ServerAction(int32 Param1, FVector Param2);

bool AMyActor::ServerAction_Validate(int32 Param1, FVector Param2)
{
    // 参数验证
    if (Param1 < 0)
        return false;
    if (!Param2.IsValid())
        return false;

    // 业务验证
    if (!CanPerformAction())
        return false;

    return true;
}

void AMyActor::ServerAction_Implementation(int32 Param1, FVector Param2)
{
    // 执行逻辑
    PerformAction(Param1, Param2);
}
```

---

## 九、总结

### 9.1 核心要点

1. **调试工具**: stat net、日志、断点
2. **问题排查**: 检查连接、验证注册、查看日志
3. **最佳实践**: 服务器权威、最小复制、安全验证
4. **安全考虑**: 输入验证、防作弊

### 9.2 下一课预告

**第十二课：Iris新一代网络系统**

将深入讲解：
- Iris架构概述
- 与传统系统对比
- 迁移指南

---

*上一课: [第十课：网络性能优化](./Lesson10_NetworkOptimization.md)*
*下一课: [第十二课：Iris新一代网络系统](./Lesson12_Iris.md)*
