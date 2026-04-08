# 第十课：网络性能优化

> **学习目标**: 掌握UE网络性能优化的各种技术
> **时长**: 约60分钟
> **前置知识**: 第九课内容

---

## 一、网络性能概述

### 1.1 性能指标

```
┌─────────────────────────────────────────────────────────────────────┐
│                    网络性能关键指标                                   │
└─────────────────────────────────────────────────────────────────────┘

带宽 (Bandwidth)
    - 入站/出站速率 (Bytes/sec)
    - 目标: < 64KB/s (互联网), < 1MB/s (局域网)

延迟 (Latency)
    - 往返时间 (RTT)
    - 目标: < 100ms (互联网), < 20ms (局域网)

丢包率 (Packet Loss)
    - 丢失包的比例
    - 目标: < 1%

Tick率 (Tick Rate)
    - 服务器每秒更新次数
    - 目标: 30-60 Hz
```

### 1.2 性能瓶颈

```
┌─────────────────────────────────────────────────────────────────────┐
│                    常见性能瓶颈                                      │
└─────────────────────────────────────────────────────────────────────┘

CPU瓶颈:
    - 属性比较开销
    - 过多Actor复制
    - 复杂的相关性计算

带宽瓶颈:
    - 复制属性过多
    - 高频RPC
    - 大量Actor同时更新

网络瓶颈:
    - 延迟过高
    - 丢包严重
    - 带宽不足
```

---

## 二、Push Model优化

### 2.1 传统方式 vs Push Model

```
传统方式 (Pull Model):
    每帧比较所有复制属性
    │
    ├── 遍历所有属性
    ├── 内存比较
    └── 生成脏列表
    → CPU开销大

Push Model:
    属性变化时主动标记
    │
    ├── 修改属性时标记脏
    └── 直接生成脏列表
    → CPU开销小
```

### 2.2 启用Push Model

```cpp
// 在GetLifetimeReplicatedProps中使用
void AMyActor::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 使用Push Model的属性
    DOREPLIFETIME_WITH_PARAMS(AMyActor, Score, FDoRepLifetimeParams(
        COND_None,          // Condition
        REPNOTIFY_OnChanged, // RepNotifyCondition
        true                 // bIsPushBased = true
    ));
}
```

### 2.3 手动标记脏属性

```cpp
// MyActor.h
UCLASS()
class AMyActor : public AActor
{
    GENERATED_BODY()

protected:
    UPROPERTY(Replicated)
    int32 Score;

    UPROPERTY(Replicated)
    float Health;

public:
    void AddScore(int32 Points);
    void TakeDamage(float Damage);
};
```

```cpp
// MyActor.cpp
#include "Net/Core/PushModel/PushModel.h"

void AMyActor::AddScore(int32 Points)
{
    if (HasAuthority())
    {
        Score += Points;

        // 手动标记属性脏
        MARK_PROPERTY_DIRTY(AMyActor, Score);
    }
}

void AMyActor::TakeDamage(float Damage)
{
    if (HasAuthority())
    {
        Health -= Damage;

        // 手动标记属性脏
        MARK_PROPERTY_DIRTY(AMyActor, Health);
    }
}
```

### 2.4 Push Model宏

```cpp
// 标记单个属性
MARK_PROPERTY_DIRTY(ClassName, PropertyName)

// 标记多个属性
MARK_PROPERTIES_DIRTY_BEGIN(ClassName)
    MARK_PROPERTY_DIRTY(ClassName, Property1)
    MARK_PROPERTY_DIRTY(ClassName, Property2)
MARK_PROPERTIES_DIRTY_END()
```

---

## 三、NetDormancy休眠优化

### 3.1 休眠机制

```
┌─────────────────────────────────────────────────────────────────────┐
│                    NetDormancy工作原理                               │
└─────────────────────────────────────────────────────────────────────┘

活跃状态 (Awake):
    - 每帧检查复制
    - 消耗CPU和带宽
    │
    │ Actor失去相关性/调用SetNetDormancy
    ▼
休眠状态 (Dormant):
    - 停止属性检查
    - 不发送更新
    - 保持客户端Actor存在
    │
    │ Actor重新相关/调用FlushNetDormancy
    ▼
活跃状态 (Awake)
```

### 3.2 休眠类型

```cpp
enum ENetDormancy
{
    // 永不休眠
    DORM_Never,

    // 可休眠，当前清醒
    DORM_Awake,

    // 对所有连接休眠
    DORM_DormantAll,

    // 对部分连接休眠（需要自定义逻辑）
    DORM_DormantPartial,

    // 初始休眠（地图中的Actor）
    DORM_Initial,
};
```

### 3.3 使用示例

```cpp
// 设置初始休眠
AMyActor::AMyActor()
{
    bReplicates = true;

    // 地图中的Actor初始休眠
    NetDormancy = DORM_Initial;
}

// 进入休眠
void AMyActor::GoDormant()
{
    SetNetDormancy(DORM_DormantAll);
}

// 唤醒
void AMyActor::WakeUp()
{
    FlushNetDormancy();
}

// 条件性休眠
void AMyActor::UpdateDormancy()
{
    if (IsRelevantToAnyClient())
    {
        FlushNetDormancy();
    }
    else
    {
        SetNetDormancy(DORM_DormantAll);
    }
}
```

### 3.4 自定义休眠逻辑

```cpp
// 使用DORM_DormantPartial
void AMyActor::GetNetDormancy(
    const FVector& ViewPos,
    const FVector& ViewDir,
    AActor* Viewer,
    AActor* ViewTarget,
    UActorChannel* InChannel,
    float Time,
    bool bLowBandwidth)
{
    // 自定义休眠判断
    float Distance = FVector::Dist(GetActorLocation(), ViewPos);

    if (Distance > DormancyDistance)
    {
        return DORM_DormantAll;
    }

    return DORM_Awake;
}
```

---

## 四、更新频率优化

### 4.1 NetUpdateFrequency

```cpp
// 控制复制频率
AMyActor::AMyActor()
{
    bReplicates = true;

    // 高频更新 - 实时性要求高
    NetUpdateFrequency = 100.0f;  // 每秒100次

    // 中频更新 - 一般对象
    NetUpdateFrequency = 30.0f;   // 每秒30次

    // 低频更新 - 不常变化
    NetUpdateFrequency = 5.0f;    // 每秒5次

    // 最低频率
    MinNetUpdateFrequency = 1.0f; // 最少每秒1次
}
```

### 4.2 动态调整频率

```cpp
void AMyActor::UpdateFrequency()
{
    if (bInCombat)
    {
        // 战斗中高频更新
        NetUpdateFrequency = 60.0f;
    }
    else if (bIsMoving)
    {
        // 移动时中频更新
        NetUpdateFrequency = 20.0f;
    }
    else
    {
        // 静止时低频更新
        NetUpdateFrequency = 5.0f;
    }
}
```

### 4.3 按优先级分类

```cpp
// 不同类型Actor的推荐设置
struct FNetworkSettings
{
    // 玩家角色 - 高优先级
    static constexpr float PlayerFrequency = 100.0f;
    static constexpr float PlayerPriority = 3.0f;

    // AI角色 - 中优先级
    static constexpr float AIFrequency = 30.0f;
    static constexpr float AIPriority = 1.5f;

    // 投射物 - 高优先级但不持久
    static constexpr float ProjectileFrequency = 100.0f;
    static constexpr float ProjectilePriority = 2.0f;

    // 道具 - 低优先级
    static constexpr float ItemFrequency = 10.0f;
    static constexpr float ItemPriority = 0.5f;

    // 背景物体 - 最低优先级
    static constexpr float BackgroundFrequency = 1.0f;
    static constexpr float BackgroundPriority = 0.1f;
};
```

---

## 五、相关性优化

### 5.1 IsNetRelevantFor

```cpp
/**
 * 判断Actor对特定连接是否相关
 * 返回false则不复制
 */
virtual bool IsNetRelevantFor(
    AActor* RealViewer,
    AActor* ViewTarget,
    const FVector& SrcLocation
);
```

### 5.2 自定义相关性

```cpp
bool AMyActor::IsNetRelevantFor(
    AActor* RealViewer,
    AActor* ViewTarget,
    const FVector& SrcLocation)
{
    // 距离检查
    float Distance = FVector::Dist(GetActorLocation(), SrcLocation);
    if (Distance > NetCullDistanceSquared)
    {
        return false;
    }

    // 视野检查
    FVector DirToActor = (GetActorLocation() - SrcLocation).GetSafeNormal();
    FVector ViewDirection = RealViewer->GetActorForwardVector();
    float DotProduct = FVector::DotProduct(ViewDirection, DirToActor);

    // 只复制视野内的Actor
    if (DotProduct < 0.0f)
    {
        return false;
    }

    // 队伍检查
    AMyCharacter* ViewerChar = Cast<AMyCharacter>(RealViewer);
    if (ViewerChar && ViewerChar->GetTeamId() != TeamId)
    {
        return false;
    }

    return true;
}
```

### 5.3 相关性超时

```cpp
// Actor.h中的设置
// 失去相关性后通道保持的时间
UPROPERTY(Config)
float RelevantTimeout = 5.0f;
```

---

## 六、带宽优化

### 6.1 属性压缩

```cpp
// 使用量化减少带宽
USTRUCT()
struct FCompressedLocation
{
    GENERATED_BODY()

    // 使用整数坐标（厘米精度）
    UPROPERTY()
    int32 X;

    UPROPERTY()
    int32 Y;

    UPROPERTY()
    int32 Z;

    // 从FVector转换
    void FromVector(const FVector& Vec)
    {
        X = FMath::RoundToInt(Vec.X * 100.0f);
        Y = FMath::RoundToInt(Vec.Y * 100.0f);
        Z = FMath::RoundToInt(Vec.Z * 100.0f);
    }

    FVector ToVector() const
    {
        return FVector(X / 100.0f, Y / 100.0f, Z / 100.0f);
    }
};
```

### 6.2 减少RPC频率

```cpp
// 使用定时器批量发送
void AMyCharacter::UpdateMovement()
{
    // 不好的做法：每帧发送RPC
    // ServerUpdatePosition(GetActorLocation());

    // 好的做法：批量发送
    AccumulatedPosition += GetActorLocation();

    if (GetWorld()->GetTimeSeconds() - LastUpdateTime > UpdateInterval)
    {
        ServerUpdatePosition(AccumulatedPosition / PositionCount);
        AccumulatedPosition = FVector::ZeroVector;
        PositionCount = 0;
        LastUpdateTime = GetWorld()->GetTimeSeconds();
    }
}
```

### 6.3 条件复制

```cpp
// 根据接收者选择复制内容
void AMyActor::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 私有数据只发给所有者
    DOREPLIFETIME_CONDITION(AMyActor, PrivateData, COND_OwnerOnly);

    // 公共效果跳过所有者
    DOREPLIFETIME_CONDITION(AMyActor, PublicEffect, COND_SkipOwner);

    // 只发一次
    DOREPLIFETIME_CONDITION(AMyActor, InitialConfig, COND_InitialOnly);
}
```

---

## 七、统计与调试

### 7.1 控制台命令

```
; 显示网络统计
stat net

; 显示详细包统计
stat packet

; 显示网络对象
net.Objects

; 显示带宽详情
net.DetailedNetTraffic

; 控制台变量
net.MaxClientUpdateRate 30
net.MaxClientRate 10000
```

### 7.2 性能分析

```cpp
// 使用stat net查看
/*
Net
├── InBunches          - 接收的Bunch数量
├── OutBunches         - 发送的Bunch数量
├── InPackets          - 接收的包数量
├── OutPackets         - 发送的包数量
├── InBytes            - 接收字节数
├── OutBytes           - 发送字节数
├── Actors             - 复制的Actor数量
└── ReplicatedProps    - 复制的属性数量
*/
```

### 7.3 性能监控代码

```cpp
void AMyGameMode::MonitorNetworkPerformance()
{
    UNetDriver* NetDriver = GetWorld()->GetNetDriver();
    if (!NetDriver)
        return;

    // 统计信息
    int32 TotalOutBytes = 0;
    int32 TotalActors = 0;

    for (UNetConnection* Conn : NetDriver->ClientConnections)
    {
        TotalOutBytes += Conn->OutBytesPerSecond;
        TotalActors += Conn->OpenChannels.Num();
    }

    UE_LOG(LogNet, Log,
        TEXT("Network: OutBytes=%d/s, Actors=%d"),
        TotalOutBytes, TotalActors
    );
}
```

---

## 八、优化清单

### 8.1 检查清单

```
┌─────────────────────────────────────────────────────────────────────┐
│                    网络优化检查清单                                   │
└─────────────────────────────────────────────────────────────────────┘

□ 属性优化
    ├─ 只复制必要的属性
    ├─ 使用条件复制
    ├─ 使用Push Model
    └─ 压缩数据结构

□ 频率优化
    ├─ 合理设置NetUpdateFrequency
    ├─ 动态调整更新频率
    └─ 使用NetDormancy

□ 相关性优化
    ├─ 实现IsNetRelevantFor
    ├─ 设置合适的NetCullDistance
    └─ 使用bNetUseOwnerRelevancy

□ RPC优化
    ├─ 减少RPC频率
    ├─ 使用Reliable/Unreliable正确
    └─ 批量发送数据

□ 带宽优化
    ├─ 减少不必要的复制
    ├─ 使用Delta序列化
    └─ 压缩大对象
```

---

## 九、总结

### 9.1 核心要点

1. **Push Model**: 主动标记脏属性，减少CPU开销
2. **NetDormancy**: 休眠不相关Actor，节省资源
3. **更新频率**: 按需设置，动态调整
4. **相关性**: 只复制相关Actor
5. **带宽**: 压缩、条件、批量

### 9.2 下一课预告

**第十一课：网络调试与最佳实践**

将深入讲解：

- 调试工具和命令
- 常见问题排查
- 开发最佳实践

---

_上一课: [第九课：子对象复制](./Lesson09_SubobjectReplication.md)_
_下一课: [第十一课：网络调试与最佳实践](./Lesson11_DebugAndBestPractices.md)_
