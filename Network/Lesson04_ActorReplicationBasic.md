# 第四课：Actor网络复制基础

> **学习目标**: 掌握Actor网络复制的基本原理和配置方法
> **时长**: 约60分钟
> **前置知识**: 第一至第三课内容

---

## 一、Actor网络复制概述

### 1.1 什么是Actor复制?

Actor复制是指**服务器将Actor的状态同步到客户端**的过程。这是UE多人游戏的核心机制。

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Actor复制示意图                                  │
└─────────────────────────────────────────────────────────────────────┘

                    服务器 (Authority)
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
   客户端 A          客户端 B          客户端 C
        │                │                │
        ▼                ▼                ▼
   复制Actor         复制Actor         复制Actor
   (属性+RPC)        (属性+RPC)        (属性+RPC)
```

### 1.2 复制的三要素

1. **属性复制 (Property Replication)**: 同步Actor的成员变量
2. **RPC (Remote Procedure Call)**: 远程函数调用
3. **子对象复制 (Subobject Replication)**: 同步Actor的组件和子对象

---

## 二、网络角色 (ENetRole)

### 2.1 ENetRole 枚举

**位置**: `Engine/Classes/Engine/EngineTypes.h:3345`

```cpp
UENUM(BlueprintType)
enum ENetRole : int
{
    /** 无角色 */
    ROLE_None,

    /** 模拟代理 - 完全由服务器控制，客户端仅接收状态 */
    ROLE_SimulatedProxy,

    /** 自治代理 - 客户端可以本地执行部分操作 */
    ROLE_AutonomousProxy,

    /** 权威 - 拥有完全控制权，通常是服务器 */
    ROLE_Authority,

    ROLE_MAX,
};
```

### 2.2 角色详解

```
┌─────────────────────────────────────────────────────────────────────┐
│                      网络角色详解                                    │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────┬──────────────────────────────────────────────────┐
│      角色        │                    说明                          │
├─────────────────┼──────────────────────────────────────────────────┤
│ ROLE_None       │ 不参与网络复制                                    │
├─────────────────┼──────────────────────────────────────────────────┤
│ ROLE_Simulated  │ 由服务器完全控制                                  │
│ Proxy           │ 客户端只能接收更新                                 │
│                 │ 常见：其他玩家、环境物体                            │
│                 │ 可通过插值实现平滑表现                              │
├─────────────────┼──────────────────────────────────────────────────┤
│ ROLE_Autonomous │ 客户端有部分控制权                                 │
│ Proxy           │ 可以本地执行移动预测                               │
│                 │ 常见：玩家控制的Pawn                               │
│                 │ 可以调用Server RPC                                │
├─────────────────┼──────────────────────────────────────────────────┤
│ ROLE_Authority  │ 完全控制权                                        │
│                 │ 服务器上所有Actor默认角色                          │
│                 │ 负责同步状态给客户端                               │
└─────────────────┴──────────────────────────────────────────────────┘
```

### 2.3 Role与RemoteRole

每个Actor有两个角色属性：

```cpp
class AActor
{
    // 本地角色 - 本地机器上的角色
    TEnumAsByte<enum ENetRole> Role;

    // 远程角色 - 远程机器上看到的角色
    TEnumAsByte<enum ENetRole> RemoteRole;
};
```

**服务器与客户端的角色对应**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                   服务器与客户端角色对应                              │
└─────────────────────────────────────────────────────────────────────┘

服务器上的Actor                    客户端上的Actor
┌───────────────────┐            ┌───────────────────┐
│ Role = Authority  │            │ Role = Simulated  │
│ RemoteRole =      │◄──────────►│ Proxy             │
│   SimulatedProxy  │            │ RemoteRole =      │
│                   │            │   Authority       │
└───────────────────┘            └───────────────────┘


玩家控制的Pawn:
┌───────────────────┐            ┌───────────────────┐
│ Role = Authority  │            │ Role = Autonomous │
│ RemoteRole =      │◄──────────►│ Proxy             │
│   AutonomousProxy │            │ RemoteRole =      │
│                   │            │   Authority       │
└───────────────────┘            └───────────────────┘
```

### 2.4 获取角色信息

```cpp
// 获取本地角色
ENetRole GetLocalRole() const { return Role; }

// 获取远程角色
ENetRole GetRemoteRole() const;

// 检查是否是权威
bool HasAuthority() const { return GetLocalRole() == ROLE_Authority; }

// 检查网络模式
ENetMode GetNetMode() const;
```

**ENetMode 枚举**:

```cpp
enum ENetMode
{
    NM_Standalone,      // 单机游戏
    NM_DedicatedServer, // 专用服务器
    NM_ListenServer,    // 监听服务器（玩家同时也是服务器）
    NM_Client,          // 客户端
};
```

---

## 三、Actor网络属性

### 3.1 核心网络属性

```cpp
class AActor
{
public:
    // ========== 基础复制设置 ==========

    // 是否启用复制
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Replication)
    uint8 bReplicates:1;

    // 远程角色
    TEnumAsByte<enum ENetRole> RemoteRole;

    // 网络休眠状态
    TEnumAsByte<enum ENetDormancy> NetDormancy;


    // ========== 更新频率 ==========

    // 网络更新频率 (次/秒)
    UPROPERTY(Category=Replication, EditDefaultsOnly)
    float NetUpdateFrequency;

    // 最小网络更新频率
    UPROPERTY(Category=Replication, EditDefaultsOnly)
    float MinNetUpdateFrequency;


    // ========== 优先级 ==========

    // 网络优先级
    UPROPERTY(Category=Replication, EditDefaultsOnly)
    float NetPriority;


    // ========== 其他设置 ==========

    // 客户端地图加载时加载
    UPROPERTY(Category=Replication, EditAnywhere)
    uint8 bNetLoadOnClient:1;

    // 使用Owner的相关性
    UPROPERTY(Category=Replication, EditDefaultsOnly)
    uint8 bNetUseOwnerRelevancy:1;
};
```

### 3.2 属性详解

#### bReplicates

```cpp
// 是否启用网络复制
// true: Actor会被复制到客户端
// false: Actor仅在本地存在

// 设置方法
void SetReplicates(bool bInReplicates);

// 示例
AMyActor::AMyActor()
{
    bReplicates = true;
}
```

#### NetUpdateFrequency

```cpp
// 每秒复制次数
// 高频率 (如 100): 实时性高，带宽消耗大
// 低频率 (如 1-10): 节省带宽，适合不常变化的对象

// 动态调整
void SetNetUpdateFrequency(float Frequency);
float GetNetUpdateFrequency() const;

// 示例
AMyActor::AMyActor()
{
    NetUpdateFrequency = 30.0f;  // 每秒30次
}
```

#### NetPriority

```cpp
// 复制优先级 (默认 1.0)
// 高优先级 (>1.0): 带宽紧张时优先复制
// 低优先级 (<1.0): 带宽紧张时可能被跳过

// 动态获取优先级
virtual float GetNetPriority(
    const FVector& ViewPos,
    const FVector& ViewDir,
    AActor* Viewer,
    AActor* ViewTarget,
    UActorChannel* InChannel,
    float Time,
    bool bLowBandwidth
);
```

### 3.3 NetDormancy (网络休眠)

**位置**: `Engine/Classes/Engine/EngineTypes.h:3360`

```cpp
enum ENetDormancy : int
{
    /** 永不休眠 */
    DORM_Never,

    /** 可休眠，当前清醒 */
    DORM_Awake,

    /** 对所有连接休眠 */
    DORM_DormantAll,

    /** 对部分连接休眠 */
    DORM_DormantPartial,

    /** 初始休眠 (放置在地图中的Actor) */
    DORM_Initial,

    DORM_MAX,
};
```

**休眠机制图解**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                      网络休眠状态转换                                │
└─────────────────────────────────────────────────────────────────────┘

                    ┌──────────────┐
                    │ DORM_Awake   │
                    │ (活跃复制)    │
                    └──────────────┘
                           │
            ┌──────────────┴──────────────┐
            │ Actor失去相关性              │
            │ 或调用SetNetDormancy()       │
            ▼                              │
    ┌──────────────┐                      │
    │DORM_DormantAll│                     │
    │ (休眠)        │                     │
    │ 不复制属性     │                     │
    └──────────────┘                      │
            │                              │
            │ Actor重新相关                │
            │ 或FlushNetDormancy()         │
            └──────────────────────────────┘

休眠的好处:
    - 减少CPU开销（不比较属性）
    - 节省带宽（不发送更新）
    - 保持客户端Actor存在
```

---

## 四、启用Actor复制

### 4.1 基本设置

```cpp
// MyActor.h
UCLASS()
class AMyActor : public AActor
{
    GENERATED_BODY()

public:
    AMyActor();
};

// MyActor.cpp
AMyActor::AMyActor()
{
    // 启用复制
    bReplicates = true;

    // 设置远程角色
    RemoteRole = ROLE_SimulatedProxy;

    // 设置更新频率
    NetUpdateFrequency = 30.0f;

    // 设置优先级
    NetPriority = 1.0f;
}
```

### 4.2 运行时设置

```cpp
// 动态启用/禁用复制
void AMyActor::ToggleReplication(bool bEnable)
{
    SetReplicates(bEnable);
}

// 动态设置休眠
void AMyActor::PutToSleep()
{
    SetNetDormancy(DORM_DormantAll);
}

// 唤醒
void AMyActor::WakeUp()
{
    FlushNetDormancy();
}
```

### 4.3 条件复制

```cpp
// 根据条件启用复制
bool AMyActor::IsRelevantForClient(AActor* RealViewer, AActor* ViewTarget, const FVector& SrcLocation)
{
    // 只对距离近的客户端复制
    float Distance = FVector::Dist(RealViewer->GetActorLocation(), SrcLocation);
    return Distance < RelevantDistance;
}
```

---

## 五、属性复制基础

### 5.1 声明复制属性

```cpp
// MyActor.h
UCLASS()
class AMyActor : public AActor
{
    GENERATED_BODY()

public:
    // 简单属性复制
    UPROPERTY(Replicated)
    int32 Score;

    // 带RepNotify的属性
    UPROPERTY(ReplicatedUsing=OnRep_Health)
    float Health;

    UFUNCTION()
    void OnRep_Health(float OldValue);

    // 条件复制
    UPROPERTY(Replicated)
    bool bIsVisible;

protected:
    // 重写GetLifetimeReplicatedProps
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
};
```

### 5.2 实现GetLifetimeReplicatedProps

```cpp
// MyActor.cpp
#include "Net/UnrealNetwork.h"

void AMyActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 基础复制
    DOREPLIFETIME(AMyActor, Score);

    // 带RepNotify的复制
    DOREPLIFETIME_CONDITION_NOTIFY(AMyActor, Health, COND_None, REPNOTIFY_OnChanged);

    // 条件复制 - 仅所有者
    DOREPLIFETIME_CONDITION(AMyActor, bIsVisible, COND_OwnerOnly);
}
```

### 5.3 复制宏详解

```cpp
// 基础复制
DOREPLIFETIME(ClassName, PropertyName)
// 无条件复制给所有人

// 条件复制
DOREPLIFETIME_CONDITION(ClassName, PropertyName, Condition)
// 按条件复制

// 带RepNotify
DOREPLIFETIME_NOTIFY(ClassName, PropertyName, NotifyFlags)
// 无条件复制，带通知

// 完整形式
DOREPLIFETIME_CONDITION_NOTIFY(ClassName, PropertyName, Condition, NotifyFlags)
// 条件复制 + RepNotify
```

---

## 六、复制条件 (ELifetimeCondition)

### 6.1 条件枚举

```cpp
enum ELifetimeCondition
{
    COND_None               = 0,   // 无条件
    COND_InitialOnly        = 1,   // 仅初始同步
    COND_OwnerOnly          = 2,   // 仅所有者
    COND_SkipOwner          = 3,   // 跳过所有者
    COND_SimulatedOnly      = 4,   // 仅模拟代理
    COND_AutonomousOnly     = 5,   // 仅自治代理
    COND_SimulatedOrPhysics = 6,   // 模拟或物理
    COND_InitialOrOwner     = 7,   // 初始或所有者
    COND_ReplayOrOwner      = 8,   // 回放或所有者
    COND_ReplayOnly         = 9,   // 仅回放
    COND_SimulatedOnlyNoReplay = 10, // 仅模拟，不含回放
    COND_SimulatedOrPhysicsNoReplay = 11, // 模拟或物理，不含回放
    COND_Max                = 12,
};
```

### 6.2 条件使用场景

```
┌─────────────────────────────────────────────────────────────────────┐
│                      复制条件使用场景                                │
└─────────────────────────────────────────────────────────────────────┘

┌────────────────────┬────────────────────────────────────────────────┐
│       条件          │                  使用场景                       │
├────────────────────┼────────────────────────────────────────────────┤
│ COND_None          │ 默认值，所有客户端都需要                         │
│                    │ 例：公共游戏状态、分数                          │
├────────────────────┼────────────────────────────────────────────────┤
│ COND_InitialOnly   │ 只在Actor首次复制时同步                         │
│                    │ 例：初始配置、出生点                            │
├────────────────────┼────────────────────────────────────────────────┤
│ COND_OwnerOnly     │ 只有拥有该Actor的客户端能看到                   │
│                    │ 例：私有状态、本地UI数据                        │
├────────────────────┼────────────────────────────────────────────────┤
│ COND_SkipOwner     │ 所有客户端除拥有者外                            │
│                    │ 例：其他玩家看到的效果                          │
├────────────────────┼────────────────────────────────────────────────┤
│ COND_SimulatedOnly │ 只同步给SimulatedProxy角色                      │
│                    │ 例：其他玩家的位置                              │
├────────────────────┼────────────────────────────────────────────────┤
│ COND_AutonomousOnly│ 只同步给AutonomousProxy角色                     │
│                    │ 例：玩家自己的预测状态                          │
└────────────────────┴────────────────────────────────────────────────┘
```

### 6.3 条件复制示例

```cpp
void AMyCharacter::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 所有人都需要的位置
    DOREPLIFETIME(AMyCharacter, Location);

    // 只同步给其他玩家（不给自己）
    DOREPLIFETIME_CONDITION(AMyCharacter, bIsCrouching, COND_SkipOwner);

    // 只有所有者能看到自己的弹药
    DOREPLIFETIME_CONDITION(AMyCharacter, AmmoCount, COND_OwnerOnly);

    // 只在初始同步时发送
    DOREPLIFETIME_CONDITION(AMyCharacter, CharacterName, COND_InitialOnly);
}
```

---

## 七、RepNotify (属性通知)

### 7.1 概念

RepNotify是在属性复制到客户端后**自动调用的回调函数**，用于响应属性变化。

### 7.2 使用方法

```cpp
// MyActor.h
UCLASS()
class AMyActor : public AActor
{
    GENERATED_BODY()

public:
    // 声明带RepNotify的属性
    UPROPERTY(ReplicatedUsing=OnRep_Health)
    float Health;

    // 回调函数
    UFUNCTION()
    void OnRep_Health(float OldValue);

    // 无参数版本
    UPROPERTY(ReplicatedUsing=OnRep_IsAlive)
    bool bIsAlive;

    UFUNCTION()
    void OnRep_IsAlive();
};
```

```cpp
// MyActor.cpp
void AMyActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 使用DOREPLIFETIME_CONDITION_NOTIFY
    DOREPLIFETIME_CONDITION_NOTIFY(AMyActor, Health, COND_None, REPNOTIFY_OnChanged);
}

void AMyActor::OnRep_Health(float OldValue)
{
    // 在客户端执行
    UE_LOG(LogTemp, Log, TEXT("Health changed: %f -> %f"), OldValue, Health);

    // 更新UI
    UpdateHealthBar();

    // 播放受伤效果
    if (Health < OldValue)
    {
        PlayDamageEffect();
    }
}
```

### 7.3 RepNotify标志

```cpp
enum ERepNotifyFlags
{
    // 只在值改变时触发
    REPNOTIFY_OnChanged = 0,

    // 每次同步都触发（即使值未变）
    REPNOTIFY_Always = 1,
};
```

### 7.4 RepNotify执行时机

```
服务器修改属性
        │
        ▼
属性序列化到Bunch
        │
        ▼
发送到客户端
        │
        ▼
客户端接收并更新属性
        │
        ▼
调用OnRep_XXX()回调 ◄── 在客户端执行!
        │
        ▼
更新UI/特效/逻辑
```

---

## 八、复制生命周期函数

### 8.1 PreReplication

```cpp
/**
 * 在每次复制前调用（服务器端）
 * 用于动态修改复制行为
 */
virtual void PreReplication(IRepChangedPropertyTracker& ChangedPropertyTracker);
```

**使用场景**:
- 动态关闭某些属性的复制
- 根据状态调整复制内容

```cpp
void AMyActor::PreReplication(IRepChangedPropertyTracker& ChangedPropertyTracker)
{
    Super::PreReplication(ChangedPropertyTracker);

    // 如果在安全区，不复制位置
    if (bInSafeZone)
    {
        DOREPLIFETIME_ACTIVE_OVERRIDE(AMyActor, Location, false);
    }
}
```

### 8.2 ReplicateSubobjects

```cpp
/**
 * 复制子对象（服务器端）
 * 用于复制ActorComponent和其他UObject
 */
virtual bool ReplicateSubobjects(
    UActorChannel* Channel,
    FOutBunch* Bunch,
    FReplicationFlags* RepFlags
);
```

```cpp
bool AMyActor::ReplicateSubobjects(
    UActorChannel* Channel,
    FOutBunch* Bunch,
    FReplicationFlags* RepFlags
)
{
    bool bWrote = Super::ReplicateSubobjects(Channel, Bunch, RepFlags);

    // 复制自定义子对象
    for (UObject* Subobj : ReplicatedSubobjects)
    {
        bWrote |= Channel->ReplicateSubobject(Subobj, *Bunch, *RepFlags);
    }

    return bWrote;
}
```

### 8.3 OnRep_Owner

```cpp
/**
 * Owner改变时的回调
 */
UFUNCTION()
virtual void OnRep_Owner();
```

```cpp
void AMyActor::OnRep_Owner()
{
    // 处理Owner变化
    if (Owner)
    {
        // 绑定到新Owner
    }
    else
    {
        // 清理
    }
}
```

---

## 九、完整示例

### 9.1 简单可复制Actor

```cpp
// ReplicatedActor.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "ReplicatedActor.generated.h"

UCLASS()
class MYGAME_API AReplicatedActor : public AActor
{
    GENERATED_BODY()

public:
    AReplicatedActor();

protected:
    virtual void GetLifetimeReplicatedProps(
        TArray<FLifetimeProperty>& OutLifetimeProps
    ) const override;

    // ========== 复制属性 ==========

    // 公共分数
    UPROPERTY(Replicated)
    int32 Score;

    // 血量（带通知）
    UPROPERTY(ReplicatedUsing=OnRep_Health)
    float Health;

    // 是否可见（仅所有者）
    UPROPERTY(Replicated)
    bool bIsVisible;

    // 当前状态（仅初始）
    UPROPERTY(Replicated)
    FString CurrentState;

    // ========== RepNotify回调 ==========

    UFUNCTION()
    void OnRep_Health(float OldValue);

public:
    // 服务器端修改函数
    UFUNCTION(BlueprintCallable, Category="Game")
    void SetHealth(float NewHealth);

    UFUNCTION(BlueprintCallable, Category="Game")
    void AddScore(int32 Points);
};
```

```cpp
// ReplicatedActor.cpp
#include "ReplicatedActor.h"
#include "Net/UnrealNetwork.h"

AReplicatedActor::AReplicatedActor()
{
    // 启用复制
    bReplicates = true;

    // 设置默认值
    Score = 0;
    Health = 100.0f;
    bIsVisible = true;
    CurrentState = TEXT("Idle");

    // 网络设置
    NetUpdateFrequency = 30.0f;
    NetPriority = 1.0f;
}

void AReplicatedActor::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps
) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 基础复制
    DOREPLIFETIME(AReplicatedActor, Score);

    // 带RepNotify
    DOREPLIFETIME_CONDITION_NOTIFY(
        AReplicatedActor, Health,
        COND_None,
        REPNOTIFY_OnChanged
    );

    // 条件复制
    DOREPLIFETIME_CONDITION(
        AReplicatedActor, bIsVisible,
        COND_OwnerOnly
    );

    DOREPLIFETIME_CONDITION(
        AReplicatedActor, CurrentState,
        COND_InitialOnly
    );
}

void AReplicatedActor::OnRep_Health(float OldValue)
{
    UE_LOG(LogTemp, Log, TEXT("Health: %f -> %f"), OldValue, Health);

    // 更新UI
    // PlayEffects...
}

void AReplicatedActor::SetHealth(float NewHealth)
{
    // 只在服务器端执行
    if (HasAuthority())
    {
        Health = FMath::Clamp(NewHealth, 0.0f, 100.0f);
    }
}

void AReplicatedActor::AddScore(int32 Points)
{
    if (HasAuthority())
    {
        Score += Points;
    }
}
```

### 9.2 玩家角色示例

```cpp
// MyCharacter.h
UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    AMyCharacter();

    virtual void GetLifetimeReplicatedProps(
        TArray<FLifetimeProperty>& OutLifetimeProps
    ) const override;

    // 移动输入
    void MoveForward(float Value);
    void MoveRight(float Value);

    // 服务器RPC
    UFUNCTION(Server, Reliable, WithValidation)
    void ServerFire();

    // 多播RPC
    UFUNCTION(NetMulticast, Unreliable)
    void MulticastPlayFireEffect(FVector Location);

protected:
    // 弹药（仅所有者）
    UPROPERTY(Replicated)
    int32 AmmoCount;

    // 是否蹲下（跳过所有者）
    UPROPERTY(ReplicatedUsing=OnRep_Crouching)
    bool bIsCrouching;

    UFUNCTION()
    void OnRep_Crouching();
};
```

```cpp
// MyCharacter.cpp
AMyCharacter::AMyCharacter()
{
    bReplicates = true;

    // 玩家控制的Pawn需要AutonomousProxy
    RemoteRole = ROLE_AutonomousProxy;

    AmmoCount = 30;
    bIsCrouching = false;
}

void AMyCharacter::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps
) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME_CONDITION(AMyCharacter, AmmoCount, COND_OwnerOnly);
    DOREPLIFETIME_CONDITION_NOTIFY(AMyCharacter, bIsCrouching, COND_SkipOwner, REPNOTIFY_OnChanged);
}

void AMyCharacter::MoveForward(float Value)
{
    if (Value != 0.0f)
    {
        AddMovementInput(GetActorForwardVector(), Value);
    }
}

void AMyCharacter::ServerFire_Validate()
{
    // 验证参数合法性
}

void AMyCharacter::ServerFire_Implementation()
{
    if (AmmoCount > 0)
    {
        AmmoCount--;

        // 通知所有客户端播放效果
        MulticastPlayFireEffect(GetActorLocation());
    }
}

void AMyCharacter::MulticastPlayFireEffect_Implementation(FVector Location)
{
    // 所有客户端执行
    UGameplayStatics::SpawnEmitterAtLocation(
        GetWorld(),
        FireEffect,
        Location
    );
}

void AMyCharacter::OnRep_Crouching()
{
    // 其他客户端看到玩家蹲下
    // 更新动画状态
}
```

---

## 十、调试与最佳实践

### 10.1 调试命令

```
; 网络统计
stat net

; 显示Actor复制信息
net.Objects

; 显示特定Actor的复制状态
net.DetailedActorInfo <ActorName>

; 控制台变量
net.Replication.DebugProperty 1  ; 调试属性复制
```

### 10.2 最佳实践

| 实践 | 说明 |
|-----|------|
| 最小化复制属性 | 只复制必要的属性 |
| 使用条件复制 | 根据需要使用COND_* |
| 合理设置更新频率 | 不常变化的对象降低频率 |
| 使用RepNotify | 客户端响应属性变化 |
| 验证服务器RPC | 防止作弊 |
| 避免复制大型对象 | 使用引用或索引 |

### 10.3 常见问题

| 问题 | 原因 | 解决方案 |
|-----|------|---------|
| 属性不同步 | 未注册到GetLifetimeReplicatedProps | 添加DOREPLIFETIME |
| RepNotify不触发 | 未使用正确的宏 | 使用DOREPLIFETIME_CONDITION_NOTIFY |
| 客户端RPC无效 | 无NetOwner | 确保有Owner链 |
| 高带宽消耗 | 复制过多属性 | 减少复制属性，使用条件 |

---

## 十一、总结

### 11.1 核心要点

1. **网络角色**: Role和RemoteRole决定Actor的控制方式
2. **启用复制**: 设置bReplicates = true
3. **属性复制**: 使用GetLifetimeReplicatedProps注册
4. **复制条件**: COND_*控制复制目标
5. **RepNotify**: 客户端响应属性变化

### 11.2 下一课预告

**第五课：属性复制详解**

将深入讲解：
- FObjectReplicator工作原理
- FRepLayout复制布局
- 支持的属性类型
- 容器复制

---

## 十二、课后练习

### 练习1：创建复制Actor

创建一个可复制的Actor，包含：
- 生命值（带RepNotify）
- 弹药数（仅所有者）
- 得分（所有人）

### 练习2：实现简单游戏逻辑

实现一个简单的得分系统：
- 服务器增加分数
- 所有客户端看到更新
- 触发UI更新

### 练习3：调试复制

使用控制台命令观察Actor的复制状态，记录属性同步过程。

---

## 参考资料

1. **源码文件**:
   - `Engine/Classes/GameFramework/Actor.h`
   - `Engine/Classes/Engine/EngineTypes.h`
   - `Engine/Public/Net/UnrealNetwork.h`

2. **官方文档**:
   - [Actor Replication](https://docs.unrealengine.com/5.0/en-US/actor-replication/)

---

*上一课: [第三课：通道系统](./Lesson03_ChannelSystem.md)*
*下一课: [第五课：属性复制详解](./Lesson05_PropertyReplication.md)*
