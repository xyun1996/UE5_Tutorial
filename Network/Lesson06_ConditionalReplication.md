# 第六课：条件复制与RepNotify

> **学习目标**: 深入掌握复制条件和属性通知的高级用法
> **时长**: 约60分钟
> **前置知识**: 第五课内容

---

## 一、复制条件详解

### 1.1 所有复制条件一览

```cpp
enum ELifetimeCondition
{
    COND_None                = 0,   // 无条件，复制给所有客户端
    COND_InitialOnly         = 1,   // 仅初始同步
    COND_OwnerOnly           = 2,   // 仅复制给所有者
    COND_SkipOwner           = 3,   // 复制给除所有者外的客户端
    COND_SimulatedOnly       = 4,   // 仅复制给SimulatedProxy
    COND_AutonomousOnly      = 5,   // 仅复制给AutonomousProxy
    COND_SimulatedOrPhysics  = 6,   // SimulatedProxy或物理模拟
    COND_InitialOrOwner      = 7,   // 初始同步或所有者
    COND_ReplayOrOwner       = 8,   // 回放或所有者
    COND_ReplayOnly          = 9,   // 仅回放
    COND_SimulatedOnlyNoReplay    = 10,  // SimulatedProxy不含回放
    COND_SimulatedOrPhysicsNoReplay = 11, // SimulatedProxy或物理不含回放
    COND_Max                 = 12,
};
```

### 1.2 条件详解与使用场景

```
┌─────────────────────────────────────────────────────────────────────┐
│                      复制条件详解                                    │
└─────────────────────────────────────────────────────────────────────┘

┌──────────────────┬──────────────────────────────────────────────────┐
│      条件         │              详细说明与使用场景                   │
├──────────────────┼──────────────────────────────────────────────────┤
│ COND_None        │ 默认行为，复制给所有客户端                        │
│                  │ 适用：公共游戏状态、分数、全局事件                 │
│                  │                                                  │
│ 示例: DOREPLIFETIME(AActor, Score)               │
├──────────────────┼──────────────────────────────────────────────────┤
│ COND_InitialOnly │ 只在Actor首次复制到客户端时同步                   │
│                  │ 之后属性变化不会再复制                            │
│                  │ 适用：出生点、初始配置、角色类型                  │
│                  │                                                  │
│ 示例: DOREPLIFETIME_CONDITION(AActor, SpawnPoint, COND_InitialOnly)│
├──────────────────┼──────────────────────────────────────────────────┤
│ COND_OwnerOnly   │ 只有拥有该Actor的客户端才会收到                  │
│                  │ 需要Actor有NetOwner                              │
│                  │ 适用：私有状态、本地UI数据、弹药数                │
│                  │                                                  │
│ 示例: DOREPLIFETIME_CONDITION(AActor, AmmoCount, COND_OwnerOnly)   │
├──────────────────┼──────────────────────────────────────────────────┤
│ COND_SkipOwner   │ 所有客户端都能收到，除了所有者                   │
│                  │ 适用：其他玩家看到的效果、外观                    │
│                  │                                                  │
│ 示例: DOREPLIFETIME_CONDITION(AActor, bIsCrouching, COND_SkipOwner)│
├──────────────────┼──────────────────────────────────────────────────┤
│ COND_Simulated   │ 只复制给Role为SimulatedProxy的客户端              │
│ Only             │ 不复制给AutonomousProxy                          │
│                  │ 适用：其他玩家的位置更新                          │
│                  │                                                  │
│ 示例: DOREPLIFETIME_CONDITION(AActor, Location, COND_SimulatedOnly)│
├──────────────────┼──────────────────────────────────────────────────┤
│ COND_Autonomous  │ 只复制给Role为AutonomousProxy的客户端             │
│ Only             │ 通常是玩家控制的Pawn                             │
│                  │ 适用：玩家自己的预测状态                          │
│                  │                                                  │
│ 示例: DOREPLIFETIME_CONDITION(AActor, PredictedPos, COND_AutonomousOnly)│
└──────────────────┴──────────────────────────────────────────────────┘
```

---

## 二、条件复制实战

### 2.1 玩家角色示例

```cpp
// MyCharacter.h
UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    virtual void GetLifetimeReplicatedProps(
        TArray<FLifetimeProperty>& OutLifetimeProps) const override;

protected:
    // 所有人都需要的位置
    UPROPERTY(Replicated)
    FVector ReplicatedLocation;

    // 其他玩家看到的状态（不给自己）
    UPROPERTY(ReplicatedUsing=OnRep_Crouching)
    bool bIsCrouching;

    // 只有所有者能看到
    UPROPERTY(Replicated)
    int32 AmmoCount;

    // 只有玩家自己能预测
    UPROPERTY(Replicated)
    FVector PredictedVelocity;

    // 初始配置
    UPROPERTY(Replicated)
    FString CharacterName;

    UFUNCTION()
    void OnRep_Crouching();
};
```

```cpp
// MyCharacter.cpp
void AMyCharacter::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 无条件 - 所有人
    DOREPLIFETIME(AMyCharacter, ReplicatedLocation);

    // 跳过所有者 - 其他玩家看到
    DOREPLIFETIME_CONDITION_NOTIFY(
        AMyCharacter, bIsCrouching,
        COND_SkipOwner,
        REPNOTIFY_OnChanged
    );

    // 仅所有者 - 私有数据
    DOREPLIFETIME_CONDITION(AMyCharacter, AmmoCount, COND_OwnerOnly);

    // 仅自治代理 - 玩家预测
    DOREPLIFETIME_CONDITION(AMyCharacter, PredictedVelocity, COND_AutonomousOnly);

    // 仅初始 - 角色配置
    DOREPLIFETIME_CONDITION(AMyCharacter, CharacterName, COND_InitialOnly);
}
```

### 2.2 武器系统示例

```cpp
// MyWeapon.h
UCLASS()
class AMyWeapon : public AActor
{
    GENERATED_BODY()

public:
    virtual void GetLifetimeReplicatedProps(
        TArray<FLifetimeProperty>& OutLifetimeProps) const override;

protected:
    // 武器外观 - 所有人
    UPROPERTY(ReplicatedUsing=OnRep_WeaponSkin)
    int32 WeaponSkinId;

    // 弹药 - 仅所有者
    UPROPERTY(Replicated)
    int32 CurrentAmmo;

    // 准心偏移 - 仅所有者
    UPROPERTY(Replicated)
    float CrosshairSpread;

    // 换弹状态 - 除所有者外
    UPROPERTY(ReplicatedUsing=OnReloading)
    bool bIsReloading;

    // 武器配置 - 仅初始
    UPROPERTY(Replicated)
    float BaseDamage;

    UFUNCTION()
    void OnRep_WeaponSkin();

    UFUNCTION()
    void OnReloading();
};
```

```cpp
// MyWeapon.cpp
void AMyWeapon::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 武器外观给所有人
    DOREPLIFETIME_CONDITION_NOTIFY(
        AMyWeapon, WeaponSkinId,
        COND_None,
        REPNOTIFY_OnChanged
    );

    // 弹药只有所有者关心
    DOREPLIFETIME_CONDITION(AMyWeapon, CurrentAmmo, COND_OwnerOnly);

    // 准心只有所有者需要
    DOREPLIFETIME_CONDITION(AMyWeapon, CrosshairSpread, COND_OwnerOnly);

    // 换弹动画给其他人看
    DOREPLIFETIME_CONDITION_NOTIFY(
        AMyWeapon, bIsReloading,
        COND_SkipOwner,
        REPNOTIFY_OnChanged
    );

    // 基础伤害不变
    DOREPLIFETIME_CONDITION(AMyWeapon, BaseDamage, COND_InitialOnly);
}
```

### 2.3 游戏状态示例

```cpp
// MyGameState.h
UCLASS()
class AMyGameState : public AGameStateBase
{
    GENERATED_BODY()

public:
    virtual void GetLifetimeReplicatedProps(
        TArray<FLifetimeProperty>& OutLifetimeProps) const override;

protected:
    // 游戏时间 - 所有人
    UPROPERTY(Replicated)
    float RemainingTime;

    // 游戏阶段 - 所有人
    UPROPERTY(ReplicatedUsing=OnRep_MatchState)
    FName MatchState;

    // 服务器私有数据
    UPROPERTY(Replicated)
    TArray<FString> ConnectedPlayerNames;

    UFUNCTION()
    void OnRep_MatchState();
};
```

```cpp
// MyGameState.cpp
void AMyGameState::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 游戏状态给所有人
    DOREPLIFETIME(AMyGameState, RemainingTime);
    DOREPLIFETIME_CONDITION_NOTIFY(
        AMyGameState, MatchState,
        COND_None,
        REPNOTIFY_OnChanged
    );

    // 服务器私有数据使用条件
    // 或者干脆不复制
}
```

---

## 三、RepNotify高级用法

### 3.1 RepNotify标志详解

```cpp
enum ERepNotifyFlags
{
    // 只在值改变时触发回调
    REPNOTIFY_OnChanged = 0,

    // 每次同步都触发回调（即使值未变）
    REPNOTIFY_Always = 1,
};
```

**使用场景**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                   RepNotify标志选择                                  │
└─────────────────────────────────────────────────────────────────────┘

REPNOTIFY_OnChanged:
    - 大多数情况使用
    - 性能更好
    - 示例：血量变化、状态切换

REPNOTIFY_Always:
    - 特殊需求时使用
    - 如：每次同步都需要执行逻辑
    - 示例：时间同步、定时器重置
```

### 3.2 带参数的RepNotify

```cpp
// 声明带旧值的回调
UFUNCTION()
void OnRep_Health(float OldValue);

// 注册
DOREPLIFETIME_CONDITION_NOTIFY(
    AMyActor, Health,
    COND_None,
    REPNOTIFY_OnChanged
);

// 实现
void AMyActor::OnRep_Health(float OldValue)
{
    // 比较新旧值
    float Damage = OldValue - Health;
    if (Damage > 0)
    {
        // 受伤了
        PlayDamageEffect(Damage);
    }

    // 更新UI
    UpdateHealthBar(Health);
}
```

### 3.3 无参数RepNotify

```cpp
// 无参数版本
UFUNCTION()
void OnRep_IsAlive();

// 实现
void AMyActor::OnRep_IsAlive()
{
    if (bIsAlive)
    {
        // 复活
        PlayRespawnEffect();
    }
    else
    {
        // 死亡
        PlayDeathEffect();
    }
}
```

### 3.4 多属性联动

```cpp
// 场景：位置和旋转需要一起响应
USTRUCT(BlueprintType)
struct FActorTransform
{
    GENERATED_BODY()

    UPROPERTY()
    FVector Location;

    UPROPERTY()
    FRotator Rotation;

    UPROPERTY()
    FVector Scale;
};

// 使用结构体而不是分开的属性
UPROPERTY(ReplicatedUsing=OnRep_Transform)
FActorTransform ActorTransform;

UFUNCTION()
void OnRep_Transform()
{
    SetActorLocation(ActorTransform.Location);
    SetActorRotation(ActorTransform.Rotation);
    SetActorScale3D(ActorTransform.Scale);
}
```

---

## 四、条件组合技巧

### 4.1 动态条件控制

```cpp
void AMyActor::PreReplication(IRepChangedPropertyTracker& ChangedPropertyTracker)
{
    Super::PreReplication(ChangedPropertyTracker);

    // 根据游戏状态动态控制
    if (bIsStealthMode)
    {
        // 隐身模式下不复制位置
        DOREPLIFETIME_ACTIVE_OVERRIDE(AMyActor, ReplicatedLocation, false);
    }

    // 根据距离控制
    if (bIsDistant)
    {
        // 远距离降低更新频率
        NetUpdateFrequency = 1.0f;
    }
}
```

### 4.2 属性依赖

```cpp
// 属性A变化时触发属性B的更新
void AMyActor::OnRep_Health(float OldValue)
{
    // 血量为0时更新存活状态
    if (Health <= 0.0f && bIsAlive)
    {
        bIsAlive = false;
        // 服务器设置后会自动复制
    }
}

// 更好的做法是在服务器端处理
void AMyActor::TakeDamage(float Damage)
{
    if (HasAuthority())
    {
        Health -= Damage;
        if (Health <= 0.0f)
        {
            bIsAlive = false;
        }
    }
}
```

### 4.3 条件判断函数

```cpp
// 自定义条件判断
bool AMyActor::ShouldReplicateToConnection(
    UNetConnection* Connection,
    const FProperty* Property)
{
    // 检查特定属性
    if (Property->GetFName() == GET_MEMBER_NAME_CHECKED(AMyActor, SecretData))
    {
        // 只有管理员能看到
        APlayerController* PC = Cast<APlayerController>(Connection->GetPlayerController(0));
        return PC && PC->IsAdmin();
    }

    return true;
}
```

---

## 五、常见模式

### 5.1 状态机模式

```cpp
UENUM(BlueprintType)
enum class ECharacterState : uint8
{
    Idle,
    Walking,
    Running,
    Jumping,
    Dead
};

UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

protected:
    UPROPERTY(ReplicatedUsing=OnRep_CharacterState)
    ECharacterState CurrentState;

    UPROPERTY(ReplicatedUsing=OnRep_PrevState)
    ECharacterState PreviousState;

    UFUNCTION()
    void OnRep_CharacterState()
    {
        // 状态切换动画
        PlayStateAnimation(PreviousState, CurrentState);
        PreviousState = CurrentState;
    }

    void SetState(ECharacterState NewState)
    {
        if (HasAuthority())
        {
            PreviousState = CurrentState;
            CurrentState = NewState;
        }
    }
};
```

### 5.2 计时器模式

```cpp
UCLASS()
class AMyGameMode : public AGameModeBase
{
    GENERATED_BODY()

protected:
    // 使用REPNOTIFY_Always确保每次同步都更新
    UPROPERTY(ReplicatedUsing=OnRep_ServerTime)
    float ServerTime;

    UFUNCTION()
    void OnRep_ServerTime()
    {
        // 同步客户端时间
        float TimeDiff = ServerTime - GetWorld()->GetTimeSeconds();
        AdjustClientTime(TimeDiff);
    }
};

// 注册
DOREPLIFETIME_CONDITION_NOTIFY(
    AMyGameMode, ServerTime,
    COND_None,
    REPNOTIFY_Always  // 每次都触发
);
```

### 5.3 集合同步模式

```cpp
UCLASS()
class ATeamActor : public AActor
{
    GENERATED_BODY()

protected:
    // 使用Delta序列化优化数组
    UPROPERTY(ReplicatedUsing=OnRep_TeamMembers)
    TArray<APlayerState*> TeamMembers;

    UPROPERTY(Replicated)
    int32 TeamScore;

    UFUNCTION()
    void OnRep_TeamMembers()
    {
        // 更新团队UI
        UpdateTeamUI();
    }

    void AddTeamMember(APlayerState* NewMember)
    {
        if (HasAuthority() && !TeamMembers.Contains(NewMember))
        {
            TeamMembers.Add(NewMember);
        }
    }
};
```

---

## 六、性能优化

### 6.1 条件选择优化

```cpp
// 根据实际需求选择条件
void AMyActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 好：根据角色类型选择条件
    DOREPLIFETIME_CONDITION(AMyActor, PlayerSpecificData, COND_OwnerOnly);
    DOREPLIFETIME_CONDITION(AMyActor, OtherPlayerData, COND_SkipOwner);

    // 避免：不必要的COND_None
    // DOREPLIFETIME(AMyActor, PrivateData); // 私有数据不应该给所有人
}
```

### 6.2 RepNotify优化

```cpp
// 好：只在变化时触发
DOREPLIFETIME_CONDITION_NOTIFY(
    AMyActor, Health,
    COND_None,
    REPNOTIFY_OnChanged
);

// 避免：不必要的Always
// DOREPLIFETIME_CONDITION_NOTIFY(AMyActor, Health, COND_None, REPNOTIFY_Always);
```

### 6.3 批量更新

```cpp
// 使用结构体批量更新相关属性
USTRUCT(BlueprintType)
struct FCharacterStats
{
    GENERATED_BODY()

    UPROPERTY()
    float Health;

    UPROPERTY()
    float Stamina;

    UPROPERTY()
    float Mana;
};

UPROPERTY(ReplicatedUsing=OnRep_Stats)
FCharacterStats Stats;

// 一次性更新所有属性
void AMyCharacter::UpdateStats(const FCharacterStats& NewStats)
{
    if (HasAuthority())
    {
        Stats = NewStats;
    }
}
```

---

## 七、完整示例

### 7.1 完整的角色复制系统

```cpp
// MyPlayerCharacter.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Character.h"
#include "MyPlayerCharacter.generated.h"

UENUM(BlueprintType)
enum class EPlayerState : uint8
{
    Alive,
    Dead,
    Spectating
};

USTRUCT(BlueprintType)
struct FPlayerStats
{
    GENERATED_BODY()

    UPROPERTY()
    float Health;

    UPROPERTY()
    float MaxHealth;

    UPROPERTY()
    float Stamina;

    float GetHealthPercent() const { return Health / MaxHealth; }
};

UCLASS()
class AMyPlayerCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    AMyPlayerCharacter();

    virtual void GetLifetimeReplicatedProps(
        TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    virtual void PreReplication(IRepChangedPropertyTracker& ChangedPropertyTracker) override;

    // === 公共属性（所有人） ===

    UPROPERTY(ReplicatedUsing=OnRep_PlayerState)
    EPlayerState CurrentState;

    UPROPERTY(Replicated)
    int32 TeamId;

    UPROPERTY(ReplicatedUsing=OnRep_IsCrouching)
    bool bIsCrouching;

    UPROPERTY(ReplicatedUsing=OnRep_IsSprinting)
    bool bIsSprinting;

    // === 所有者属性 ===

    UPROPERTY(ReplicatedUsing=OnRep_Stats)
    FPlayerStats Stats;

    UPROPERTY(Replicated)
    int32 AmmoCount;

    UPROPERTY(Replicated)
    int32 GrenadeCount;

    // === 初始属性 ===

    UPROPERTY(Replicated)
    FString PlayerName;

    // === 排除所有者 ===

    UPROPERTY(ReplicatedUsing=OnRep_WeaponIndex)
    int32 CurrentWeaponIndex;

protected:
    // RepNotify回调
    UFUNCTION()
    void OnRep_PlayerState();

    UFUNCTION()
    void OnRep_IsCrouching();

    UFUNCTION()
    void OnRep_IsSprinting();

    UFUNCTION()
    void OnRep_Stats();

    UFUNCTION()
    void OnRep_WeaponIndex(int32 OldIndex);

    // 服务器函数
    UFUNCTION(Server, Reliable)
    void ServerSetCrouching(bool bCrouch);

    UFUNCTION(Server, Reliable)
    void ServerSetSprinting(bool bSprint);

    UFUNCTION(Server, Reliable)
    void ServerSwitchWeapon(int32 NewIndex);

public:
    // 游戏逻辑
    void TakeDamage(float Damage);
    void UseAmmo(int32 Amount);
    void SwitchWeapon(int32 Index);
};
```

```cpp
// MyPlayerCharacter.cpp
#include "MyPlayerCharacter.h"
#include "Net/UnrealNetwork.h"

AMyPlayerCharacter::AMyPlayerCharacter()
{
    bReplicates = true;
    RemoteRole = ROLE_AutonomousProxy;
    NetUpdateFrequency = 30.0f;

    CurrentState = EPlayerState::Alive;
    TeamId = 0;
    bIsCrouching = false;
    bIsSprinting = false;

    Stats.Health = 100.0f;
    Stats.MaxHealth = 100.0f;
    Stats.Stamina = 100.0f;

    AmmoCount = 30;
    GrenadeCount = 2;
    CurrentWeaponIndex = 0;
}

void AMyPlayerCharacter::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 公共属性
    DOREPLIFETIME_CONDITION_NOTIFY(
        AMyPlayerCharacter, CurrentState,
        COND_None, REPNOTIFY_OnChanged
    );
    DOREPLIFETIME(AMyPlayerCharacter, TeamId);
    DOREPLIFETIME_CONDITION_NOTIFY(
        AMyPlayerCharacter, bIsCrouching,
        COND_SkipOwner, REPNOTIFY_OnChanged
    );
    DOREPLIFETIME_CONDITION_NOTIFY(
        AMyPlayerCharacter, bIsSprinting,
        COND_SkipOwner, REPNOTIFY_OnChanged
    );

    // 所有者属性
    DOREPLIFETIME_CONDITION_NOTIFY(
        AMyPlayerCharacter, Stats,
        COND_OwnerOnly, REPNOTIFY_OnChanged
    );
    DOREPLIFETIME_CONDITION(AMyPlayerCharacter, AmmoCount, COND_OwnerOnly);
    DOREPLIFETIME_CONDITION(AMyPlayerCharacter, GrenadeCount, COND_OwnerOnly);

    // 初始属性
    DOREPLIFETIME_CONDITION(AMyPlayerCharacter, PlayerName, COND_InitialOnly);

    // 排除所有者
    DOREPLIFETIME_CONDITION_NOTIFY(
        AMyPlayerCharacter, CurrentWeaponIndex,
        COND_SkipOwner, REPNOTIFY_OnChanged
    );
}

void AMyPlayerCharacter::PreReplication(IRepChangedPropertyTracker& ChangedPropertyTracker)
{
    Super::PreReplication(ChangedPropertyTracker);

    // 死亡时不复制动作状态
    if (CurrentState == EPlayerState::Dead)
    {
        DOREPLIFETIME_ACTIVE_OVERRIDE(AMyPlayerCharacter, bIsCrouching, false);
        DOREPLIFETIME_ACTIVE_OVERRIDE(AMyPlayerCharacter, bIsSprinting, false);
    }
}

void AMyPlayerCharacter::OnRep_PlayerState()
{
    // 状态变化处理
    if (CurrentState == EPlayerState::Dead)
    {
        PlayDeathAnimation();
    }
}

void AMyPlayerCharacter::OnRep_IsCrouching()
{
    // 更新动画
    UpdateCharacterAnimation();
}

void AMyPlayerCharacter::OnRep_IsSprinting()
{
    // 更新动画
    UpdateCharacterAnimation();
}

void AMyPlayerCharacter::OnRep_Stats()
{
    // 更新UI
    UpdateHealthUI(Stats.GetHealthPercent());
    UpdateStaminaUI(Stats.Stamina / 100.0f);
}

void AMyPlayerCharacter::OnRep_WeaponIndex(int32 OldIndex)
{
    // 更新武器显示
    UpdateWeaponVisual(CurrentWeaponIndex);
}

void AMyPlayerCharacter::ServerSetCrouching_Implementation(bool bCrouch)
{
    bIsCrouching = bCrouch;
}

bool AMyPlayerCharacter::ServerSetCrouching_Validate(bool bCrouch)
{
    return true;
}

void AMyPlayerCharacter::TakeDamage(float Damage)
{
    if (HasAuthority() && CurrentState == EPlayerState::Alive)
    {
        Stats.Health = FMath::Max(0.0f, Stats.Health - Damage);
        if (Stats.Health <= 0.0f)
        {
            CurrentState = EPlayerState::Dead;
        }
    }
}

void AMyPlayerCharacter::SwitchWeapon(int32 Index)
{
    if (Index >= 0 && Index < 3)
    {
        ServerSwitchWeapon(Index);
    }
}

void AMyPlayerCharacter::ServerSwitchWeapon_Implementation(int32 NewIndex)
{
    CurrentWeaponIndex = NewIndex;
}
```

---

## 八、总结

### 8.1 核心要点

1. **条件选择**: 根据数据可见性选择正确的复制条件
2. **RepNotify**: 用于客户端响应属性变化
3. **条件组合**: PreReplication动态控制
4. **性能优化**: 减少不必要的复制

### 8.2 下一课预告

**第七课：RPC远程过程调用**

将深入讲解：
- RPC类型与使用规则
- 可靠性选择
- RPC验证与安全

---

*上一课: [第五课：属性复制详解](./Lesson05_PropertyReplication.md)*
*下一课: [第七课：RPC远程过程调用](./Lesson07_RPC.md)*
