# 第4课：变量复制与回调

> 本课深入讲解UE5的属性复制机制和OnRep回调系统，这是实现高效网络同步的关键技术。

---

## 课程目标

- 深入理解OnRep回调函数的工作机制
- 掌握条件复制的各种模式
- 学会处理复杂数据类型的复制
- 掌握RepNotify的最佳实践
- 了解属性复制的性能优化策略

---

## 一、OnRep回调机制

### 1.1 OnRep工作原理

```
服务器端                                     客户端
   │                                           │
   │  修改属性值                                │
   │  Health = 80                              │
   │       │                                   │
   │       ▼                                   │
   │  收集变化的属性                            │
   │       │                                   │
   │       ▼                                   │
   │  序列化并发送                              │
   │       │                                   │
   │       └──────────────────────►            │
   │                               接收属性更新  │
   │                                     │      │
   │                                     ▼      │
   │                              Health = 80   │
   │                                     │      │
   │                                     ▼      │
   │                           触发 OnRep_Health │
   │                                     │      │
   │                                     ▼      │
   │                              更新UI/特效    │
   │                                           │
```

### 1.2 OnRep函数声明与实现

```cpp
// MyCharacter.h
UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    // 基本OnRep声明
    UPROPERTY(ReplicatedUsing = OnRep_Health)
    float Health;

    // OnRep回调 - 带旧值参数
    UFUNCTION()
    void OnRep_Health(float OldHealth);

    // OnRep回调 - 不带参数
    UPROPERTY(ReplicatedUsing = OnRep_IsDead)
    bool bIsDead;

    UFUNCTION()
    void OnRep_IsDead();
};

// MyCharacter.cpp
#include "Net/UnrealNetwork.h"

void AMyCharacter::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(AMyCharacter, Health);
    DOREPLIFETIME(AMyCharacter, bIsDead);
}

void AMyCharacter::OnRep_Health(float OldHealth)
{
    // 只在客户端执行
    // 注意：服务器修改属性时不会触发OnRep，需要手动调用

    UE_LOG(LogTemp, Log, TEXT("Health changed from %.1f to %.1f"), OldHealth, Health);

    // 处理生命值变化
    if (Health < OldHealth)
    {
        // 受到伤害
        PlayDamageEffect();
        ShowDamageNumber(OldHealth - Health);
    }
    else if (Health > OldHealth)
    {
        // 恢复生命
        PlayHealEffect();
    }

    // 更新UI
    UpdateHealthBar();
}

void AMyCharacter::OnRep_IsDead()
{
    if (bIsDead)
    {
        // 进入死亡状态
        PlayDeathAnimation();
        DisableInput();
    }
    else
    {
        // 复活
        PlayRespawnAnimation();
        EnableInput();
    }
}
```

### 1.3 服务器端触发OnRep

```cpp
// 重要：服务器修改属性时不会自动触发OnRep
// 需要手动调用或使用特殊方法

void AMyCharacter::ApplyDamage(float Damage)
{
    if (!HasAuthority())
    {
        return;
    }

    float OldHealth = Health;
    Health = FMath::Clamp(Health - Damage, 0.0f, MaxHealth);

    // 方法1：手动调用OnRep
    OnRep_Health(OldHealth);

    // 方法2：使用 FORCE OnRep（UE5推荐）
    // 在属性变化后立即在服务器触发OnRep
    // 这会在服务器本地也执行OnRep逻辑
}

// 使用REPNOTIFY_Always让OnRep始终触发
void AMyCharacter::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // REPNOTIFY_OnChanged - 只在值改变时触发OnRep（默认）
    DOREPLIFETIME_CONDITION_NOTIFY(AMyCharacter, Health, COND_None, REPNOTIFY_OnChanged);

    // REPNOTIFY_Always - 即使值相同也会触发OnRep
    DOREPLIFETIME_CONDITION_NOTIFY(AMyCharacter, bIsDead, COND_None, REPNOTIFY_Always);
}
```

### 1.4 OnRep执行时机

```cpp
// OnRep的执行时机

// 1. 属性初次复制时
// - Actor初次被客户端看到时，所有复制属性会被同步
// - OnRep会在属性设置后立即触发

// 2. 属性值变化时
// - 服务器修改属性后，下一次网络更新会发送变化
// - 客户端收到后触发OnRep

// 3. 网络延迟影响
// - OnRep触发时间取决于网络延迟
// - 高延迟可能导致OnRep延迟触发

// 监控OnRep执行
void AMyCharacter::OnRep_Health(float OldHealth)
{
    float CurrentTime = GetWorld()->GetTimeSeconds();
    UE_LOG(LogTemp, Log, TEXT("OnRep_Health at time %.2f: %.1f -> %.1f"),
        CurrentTime, OldHealth, Health);
}
```

---

## 二、条件复制详解

### 2.1 所有条件类型

```cpp
// Engine/Source/Runtime/Engine/Classes/Engine/EngineTypes.h

enum ELifetimeCondition
{
    COND_None,                    // 无条件，始终复制
    COND_InitialOnly,             // 仅初始同步时复制
    COND_OwnerOnly,               // 仅复制给所有者
    COND_SkipOwner,               // 复制给除所有者外的所有人
    COND_SimulatedOnly,           // 仅复制给SimulatedProxy
    COND_SimulatedOnlyNoReplay,   // 仅SimulatedProxy，不录回放
    COND_AutonomousOnly,          // 仅复制给AutonomousProxy
    COND_SimulatedOrPhysics,      // SimulatedProxy或物理Actor
    COND_InitialOrOwner,          // 初始同步或所有者
    COND_ReplayOrOwner,           // 回放或所有者
    COND_ReplayOnly,              // 仅回放
    COND_SkipReplay,              // 跳过回放
    COND_Custom,                  // 自定义条件
};
```

### 2.2 条件复制使用场景

```cpp
// MyCharacter.h
UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    // ===== COND_None - 所有人可见 =====
    UPROPERTY(Replicated)
    FVector Location;  // 位置所有人都要看

    // ===== COND_InitialOnly - 只同步一次 =====
    UPROPERTY(Replicated)
    int32 CharacterClassId;  // 角色职业不会变

    // ===== COND_OwnerOnly - 只有所有者可见 =====
    UPROPERTY(Replicated)
    float CurrentStamina;  // 只有自己知道耐力

    // ===== COND_SkipOwner - 其他玩家可见 =====
    UPROPERTY(Replicated)
    bool bIsInvisible;  // 其他玩家看到隐身状态

    // ===== COND_SimulatedOnly - 只给模拟代理 =====
    UPROPERTY(Replicated)
    float CosmeticValue;  // 模拟代理用于显示效果

    // ===== COND_AutonomousOnly - 只给自主代理 =====
    UPROPERTY(Replicated)
    float InputPredictionValue;  // 预测用数据
};

// MyCharacter.cpp
void AMyCharacter::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 无条件复制
    DOREPLIFETIME(AMyCharacter, Location);

    // 只在初始化时复制
    DOREPLIFETIME_CONDITION(AMyCharacter, CharacterClassId, COND_InitialOnly);

    // 只有所有者可见
    DOREPLIFETIME_CONDITION(AMyCharacter, CurrentStamina, COND_OwnerOnly);

    // 跳过所有者（其他玩家可见）
    DOREPLIFETIME_CONDITION(AMyCharacter, bIsInvisible, COND_SkipOwner);

    // 只给模拟代理
    DOREPLIFETIME_CONDITION(AMyCharacter, CosmeticValue, COND_SimulatedOnly);

    // 只给自主代理
    DOREPLIFETIME_CONDITION(AMyCharacter, InputPredictionValue, COND_AutonomousOnly);
}
```

### 2.3 条件复制流程图

```
┌─────────────────────────────────────────────────────────────┐
│                    服务器准备复制属性                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  COND_None?     │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │ Yes                         │ No
              ▼                             ▼
        发送给所有客户端            ┌─────────────────┐
                                   │ COND_InitialOnly?│
                                   └────────┬────────┘
                                            │
                             ┌──────────────┴──────────────┐
                             │ Yes                         │ No
                             ▼                             ▼
                    只发送一次              ┌─────────────────┐
                                            │ COND_OwnerOnly? │
                                            └────────┬────────┘
                                                     │
                                      ┌──────────────┴──────────────┐
                                      │ Yes                         │ No
                                      ▼                             ▼
                               只发给所有者            ┌─────────────────┐
                                                       │ COND_SkipOwner? │
                                                       └────────┬────────┘
                                                                │
                                                 ┌──────────────┴──────────────┐
                                                 │ Yes                         │ No
                                                 ▼                             ▼
                                          发给除所有者外             ┌─────────────────┐
                                                                   │  其他条件...    │
                                                                   └─────────────────┘
```

### 2.4 自定义复制条件

```cpp
// 实现自定义复制条件

UCLASS()
class AMyActor : public AActor
{
    GENERATED_BODY()

public:
    // 使用COND_Custom条件
    UPROPERTY(Replicated)
    int32 TeamVisibleData;

    // 自定义条件判断函数
    virtual bool IsTeamVisibleToConnection(UNetConnection* Connection) const;

protected:
    // 存储哪些队伍可见
    TArray<int32> VisibleToTeams;
};

// 在ReplicationGraph或自定义逻辑中使用
void AMyActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    FDoRepLifetimeParams Params;
    Params.bIsPushBased = true;
    Params.Condition = COND_Custom;

    DOREPLIFETIME_WITH_PARAMS(AMyActor, TeamVisibleData, Params);
}

// 在ReplicationGraph节点中实现自定义逻辑
class UMyTeamVisibilityNode : public UReplicationGraphNode
{
public:
    virtual void GatherActorListsForConnection(
        const FConnectionGatherActorListParameters& Params) override
    {
        for (FActorRepListType Actor : ActorsToList)
        {
            AMyActor* MyActor = Cast<AMyActor>(Actor);
            if (MyActor)
            {
                // 检查队伍可见性
                if (MyActor->IsTeamVisibleToConnection(Params.Connection))
                {
                    Params.OutGatheredReplicationLists.AddActor(Actor);
                }
            }
        }
    }
};
```

---

## 三、复杂数据类型复制

### 3.1 结构体复制

```cpp
// 定义可复制的结构体
USTRUCT(BlueprintType)
struct FPlayerStats
{
    GENERATED_BODY()

    UPROPERTY(Replicated)
    float Health = 100.0f;

    UPROPERTY(Replicated)
    float Stamina = 100.0f;

    UPROPERTY(Replicated)
    int32 Kills = 0;

    UPROPERTY(Replicated)
    int32 Deaths = 0;

    // 不复制的属性
    UPROPERTY(NotReplicated)
    FString LocalName;

    // 计算属性（不复制）
    float GetKD() const
    {
        return Deaths > 0 ? static_cast<float>(Kills) / Deaths : Kills;
    }
};

// 在Actor中使用
UCLASS()
class AMyPlayerState : public APlayerState
{
    GENERATED_BODY()

public:
    UPROPERTY(ReplicatedUsing = OnRep_PlayerStats)
    FPlayerStats PlayerStats;

    UFUNCTION()
    void OnRep_PlayerStats(const FPlayerStats& OldStats);
};

// 实现
void AMyPlayerState::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(AMyPlayerState, PlayerStats);
}

void AMyPlayerState::OnRep_PlayerStats(const FPlayerStats& OldStats)
{
    // 检测具体哪个字段变化了
    if (PlayerStats.Health != OldStats.Health)
    {
        OnHealthChanged(OldStats.Health, PlayerStats.Health);
    }

    if (PlayerStats.Kills != OldStats.Kills)
    {
        OnKillsChanged(OldStats.Kills, PlayerStats.Kills);
    }

    // 更新UI
    UpdateStatsUI();
}
```

### 3.2 TArray复制

```cpp
UCLASS()
class AMyInventory : public AActor
{
    GENERATED_BODY()

public:
    // 数组复制
    UPROPERTY(ReplicatedUsing = OnRep_Items)
    TArray<AInventoryItem*> Items;

    UPROPERTY(ReplicatedUsing = OnRep_ItemIds)
    TArray<int32> ItemIds;

    UFUNCTION()
    void OnRep_Items(const TArray<AInventoryItem*>& OldItems);

    UFUNCTION()
    void OnRep_ItemIds();
};

// 注意：数组复制可能开销较大
// 对于频繁变化的大型数组，考虑其他方案

void AMyInventory::OnRep_Items(const TArray<AInventoryItem*>& OldItems)
{
    // 找出新添加的物品
    for (AInventoryItem* Item : Items)
    {
        if (!OldItems.Contains(Item))
        {
            OnItemAdded(Item);
        }
    }

    // 找出被移除的物品
    for (AInventoryItem* OldItem : OldItems)
    {
        if (!Items.Contains(OldItem))
        {
            OnItemRemoved(OldItem);
        }
    }
}

// 优化：使用增量更新
UCLASS()
class AMyInventory : public AActor
{
    // 不直接复制整个数组，而是通过RPC发送变化
    TArray<AInventoryItem*> Items;

    UFUNCTION(Server, Reliable)
    void Server_AddItem(AInventoryItem* Item);

    UFUNCTION(Server, Reliable)
    void Server_RemoveItem(int32 Index);

    UFUNCTION(NetMulticast, Reliable)
    void Multicast_AddItem(AInventoryItem* Item);

    UFUNCTION(NetMulticast, Reliable)
    void Multicast_RemoveItem(int32 Index);
};
```

### 3.3 TMap和TSet复制

```cpp
UCLASS()
class AMyDataManager : public AActor
{
    GENERATED_BODY()

public:
    // TMap复制
    UPROPERTY(Replicated)
    TMap<int32, FString> StringDataMap;

    UPROPERTY(Replicated)
    TMap<FName, float> AttributeMap;

    // TSet复制
    UPROPERTY(Replicated)
    TSet<AActor*> KnownActors;

    UPROPERTY(Replicated)
    TSet<FName> UnlockedAbilities;
};

// 注意：TMap和TSet的复制开销较大
// 服务器会发送完整的容器数据

// 优化建议：
// 1. 对于频繁变化的数据，避免使用TMap/TSet
// 2. 使用RPC进行增量更新
// 3. 将数据拆分为多个小属性

// 替代方案
UCLASS()
class AMyDataManager : public AActor
{
    // 不复制整个Map
    // TMap<FName, float> AttributeMap;

    // 改用单独属性
    UPROPERTY(ReplicatedUsing = OnRep_Health)
    float Health;

    UPROPERTY(ReplicatedUsing = OnRep_Stamina)
    float Stamina;

    UPROPERTY(ReplicatedUsing = OnRep_Mana)
    float Mana;

    // 或使用结构体数组
    UPROPERTY(Replicated)
    TArray<FAttributeData> AttributeList;
};
```

### 3.4 对象引用复制

```cpp
UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    // Actor引用
    UPROPERTY(Replicated)
    AActor* TargetActor;

    // 组件引用（需要特殊处理）
    UPROPERTY(Replicated)
    UStaticMeshComponent* WeaponMesh;

    // UObject引用
    UPROPERTY(Replicated)
    UTexture2D* PortraitTexture;

    // TSubclassOf引用
    UPROPERTY(Replicated)
    TSubclassOf<AWeapon> WeaponClass;
};

// 对象引用复制的注意事项：
// 1. 被引用的对象必须也是可复制的
// 2. 或者是已存在的资源（如Texture, Blueprint Class）
// 3. 组件引用需要组件本身支持复制

// 组件复制
void AMyCharacter::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 组件引用复制需要组件设置bReplicates = true
    DOREPLIFETIME(AMyCharacter, WeaponMesh);
}

// 在组件构造函数中
UMyWeaponComponent::UMyWeaponComponent()
{
    SetIsReplicatedByDefault(true);  // 组件可复制
}
```

---

## 四、RepNotify最佳实践

### 4.1 何时使用OnRep vs RPC

```cpp
// 使用OnRep的场景：
// 1. 状态数据需要持续同步
// 2. 新玩家加入时需要获取当前值
// 3. 需要知道旧值和新值

// 使用RPC的场景：
// 1. 一次性事件
// 2. 不需要保存状态
// 3. 瞬时效果

// 示例：生命值系统

// 正确：使用OnRep
UPROPERTY(ReplicatedUsing = OnRep_Health)
float Health;  // 状态数据，需要持续同步

UFUNCTION()
void OnRep_Health(float OldHealth)
{
    // 更新血条UI
    // 新玩家加入时自动获得当前Health值
}

// 正确：使用Multicast
UFUNCTION(NetMulticast, Unreliable)
void Multicast_PlayDamageEffect();  // 一次性效果

// 组合使用
void AMyCharacter::TakeDamage(float Damage, AActor* DamageCauser)
{
    Health -= Damage;  // 触发OnRep_Health

    // 同时广播伤害效果
    Multicast_PlayDamageEffect();
    Multicast_ShowDamageNumber(Damage);
}
```

### 4.2 避免重复逻辑

```cpp
// 常见错误：OnRep和服务器逻辑重复

// 错误示例
void AMyCharacter::SetHealth(float NewHealth)
{
    if (HasAuthority())
    {
        Health = NewHealth;
        // 服务器端更新UI
        UpdateHealthBar();  // 这里更新一次
    }
}

void AMyCharacter::OnRep_Health(float OldHealth)
{
    UpdateHealthBar();  // 客户端也更新，正确
}

// 问题：服务器只更新一次，客户端也只更新一次
// 但如果想在服务器也触发OnRep逻辑呢？

// 正确示例：统一使用OnRep
void AMyCharacter::SetHealth(float NewHealth)
{
    if (HasAuthority())
    {
        float OldHealth = Health;
        Health = NewHealth;
        // 服务器也调用OnRep
        OnRep_Health(OldHealth);
    }
}

void AMyCharacter::OnRep_Health(float OldHealth)
{
    // 服务器和客户端都执行这里的逻辑
    UpdateHealthBar();
    CheckDeathCondition();
}

// 更好的方式：使用辅助函数
void AMyCharacter::SetHealth(float NewHealth)
{
    if (HasAuthority())
    {
        Health = NewHealth;
        // OnRep会自动在客户端触发
        // 服务器需要手动触发
        HandleHealthChanged();
    }
}

void AMyCharacter::OnRep_Health(float OldHealth)
{
    HandleHealthChanged();
}

void AMyCharacter::HandleHealthChanged()
{
    // 通用逻辑，服务器和客户端都可用
    UpdateHealthBar();

    if (Health <= 0.0f && !bIsDead)
    {
        HandleDeath();
    }
}
```

### 4.3 OnRep性能优化

```cpp
// 1. 避免在OnRep中执行重型操作

// 不好
void AMyCharacter::OnRep_Position(FVector OldPosition)
{
    // 每次位置更新都重新计算路径
    RecalculatePath();  // 开销大
    UpdateNavigation();  // 开销大
}

// 更好
void AMyCharacter::OnRep_Position(FVector OldPosition)
{
    // 只更新视觉表现
    UpdateVisualPosition();

    // 延迟计算路径
    if (!PathUpdateTimer.IsValid())
    {
        GetWorld()->GetTimerManager().SetTimer(
            PathUpdateTimer,
            this,
            &AMyCharacter::RecalculatePath,
            0.1f,
            false
        );
    }
}

// 2. 使用条件判断避免不必要的更新

void AMyCharacter::OnRep_TeamId(int32 OldTeamId)
{
    // 只有本地玩家关心队伍变化
    APlayerController* PC = Cast<APlayerController>(GetController());
    if (PC && PC->IsLocalController())
    {
        UpdateTeamUI();
    }
}

// 3. 批量处理多个属性变化

void AMyCharacter::OnRep_PlayerStats(const FPlayerStats& OldStats)
{
    // 批量更新，减少UI刷新次数
    bool bNeedsUIUpdate = false;

    if (PlayerStats.Health != OldStats.Health)
    {
        bNeedsUIUpdate = true;
    }

    if (PlayerStats.Stamina != OldStats.Stamina)
    {
        bNeedsUIUpdate = true;
    }

    if (bNeedsUIUpdate)
    {
        // 只刷新一次UI
        RefreshAllStatsUI();
    }
}
```

### 4.4 调试OnRep

```cpp
// 调试工具

// 1. 记录OnRep调用
void AMyCharacter::OnRep_Health(float OldHealth)
{
    UE_LOG(LogTemp, Log, TEXT("OnRep_Health: %.1f -> %.1f on %s"),
        OldHealth, Health,
        HasAuthority() ? TEXT("Server") : TEXT("Client"));

    // 使用Visual Studio断点调试
    // 或使用UE_LOG
}

// 2. 检查属性是否正确注册
void AMyCharacter::DebugReplicatedProperties()
{
    // 获取所有复制属性
    TArray<FLifetimeProperty> Props;
    GetLifetimeReplicatedProps(Props);

    for (const FLifetimeProperty& Prop : Props)
    {
        UE_LOG(LogTemp, Log, TEXT("Replicated Property: Index=%d, Condition=%d"),
            Prop.GetRepIndex(),
            Prop.GetCondition());
    }
}

// 3. 使用控制台命令
// net.Replicate.ExactProperty - 显示属性复制信息
// stat net - 网络统计

// 4. 使用Unreal Insights
// 追踪属性复制的详细信息
```

---

## 五、高级复制技巧

### 5.1 FastArray序列化

```cpp
// FastArraySerialization用于高效复制大型数组

USTRUCT()
struct FMyItemEntry : public FFastArraySerializerItem
{
    GENERATED_BODY()

    UPROPERTY()
    int32 ItemId;

    UPROPERTY()
    int32 Count;
};

USTRUCT()
struct FMyInventoryArray : public FFastArraySerializer
{
    GENERATED_BODY()

    UPROPERTY()
    TArray<FMyItemEntry> Items;

    // 可选：OnRep
    void PostReplicatedAdd(const TArrayView<int32>& AddedIndices, int32 FinalSize);
    void PostReplicatedRemove(const TArrayView<int32>& RemovedIndices, int32 FinalSize);
};

// 在Actor中使用
UCLASS()
class AMyPlayerState : public APlayerState
{
    GENERATED_BODY()

    UPROPERTY(Replicated)
    FMyInventoryArray Inventory;
};

// 实现FastArray复制
void AMyPlayerState::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // FastArray需要特殊注册
    FDoRepLifetimeParams Params;
    Params.bIsPushBased = true;
    DOREPLIFETIME_WITH_PARAMS_FAST(AMyPlayerState, Inventory, Params);
}

// FastArray回调
void FMyInventoryArray::PostReplicatedAdd(const TArrayView<int32>& AddedIndices, int32 FinalSize)
{
    for (int32 Index : AddedIndices)
    {
        // 处理新添加的项目
        UE_LOG(LogTemp, Log, TEXT("Item added at index %d"), Index);
    }
}

void FMyInventoryArray::PostReplicatedRemove(const TArrayView<int32>& RemovedIndices, int32 FinalSize)
{
    for (int32 Index : RemovedIndices)
    {
        // 处理被移除的项目
        UE_LOG(LogTemp, Log, TEXT("Item removed at index %d"), Index);
    }
}
```

### 5.2 条件属性组

```cpp
// 根据条件切换整组属性的复制

UCLASS()
class AMyVehicle : public AActor
{
    GENERATED_BODY()

public:
    // 驾驶模式
    UPROPERTY(ReplicatedUsing = OnRep_bIsBeingDriven)
    bool bIsBeingDriven;

    // 驾驶相关属性 - 只在驾驶时复制
    UPROPERTY(Replicated)
    float Fuel;  // 只在驾驶时同步

    UPROPERTY(Replicated)
    float Speed;  // 只在驾驶时同步

protected:
    UFUNCTION()
    void OnRep_bIsBeingDriven();
};

// 自定义复制条件
void AMyVehicle::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(AMyVehicle, bIsBeingDriven);

    // 根据驾驶状态决定是否复制
    // 这需要自定义条件逻辑
    // 或者使用COND_Custom
}

// 动态启用/禁用复制
void AMyVehicle::SetBeingDriven(bool bDriven)
{
    bIsBeingDriven = bDriven;

    if (bDriven)
    {
        // 启用驾驶相关属性的复制
        // 可以通过修改NetUpdateFrequency实现
        NetUpdateFrequency = 100.0f;
    }
    else
    {
        // 降低更新频率
        NetUpdateFrequency = 1.0f;
    }
}
```

### 5.3 属性复制与对象池

```cpp
// 当使用对象池时，需要注意属性复制的重置

UCLASS()
class AMyPooledActor : public AActor
{
    GENERATED_BODY()

public:
    UPROPERTY(ReplicatedUsing = OnRep_Active)
    bool bActive;

    UPROPERTY(Replicated)
    FVector PooledLocation;

    // 从池中取出
    void Activate(const FVector& Location)
    {
        bActive = true;
        PooledLocation = Location;
        SetActorLocation(Location);
        SetActorHiddenInGame(false);

        // 强制立即同步状态
        ForceNetUpdate();
    }

    // 放回池中
    void Deactivate()
    {
        bActive = false;
        SetActorHiddenInGame(true);
        ForceNetUpdate();
    }

protected:
    UFUNCTION()
    void OnRep_Active()
    {
        SetActorHiddenInGame(!bActive);
        if (bActive)
        {
            SetActorLocation(PooledLocation);
        }
    }
};

// 注意：从池中取出时，确保所有复制属性都被正确初始化
```

---

## 六、实践任务

### 任务：实现完整的玩家状态系统

```cpp
// MyPlayerState.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/PlayerState.h"
#include "MyPlayerState.generated.h"

// 玩家统计数据
USTRUCT(BlueprintType)
struct FPlayerStatistics
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    int32 Kills = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 Deaths = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 Assists = 0;

    UPROPERTY(BlueprintReadOnly)
    float Score = 0.0f;

    float GetKDA() const
    {
        return Deaths > 0 ? static_cast<float>(Kills + Assists / 2) / Deaths : Kills;
    }
};

// 玩家设置
USTRUCT(BlueprintType)
struct FPlayerSettings
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FLinearColor PlayerColor = FLinearColor::White;

    UPROPERTY(BlueprintReadOnly)
    int32 SelectedCharacterId = 0;

    UPROPERTY(BlueprintReadOnly)
    bool bVoiceChatEnabled = true;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnStatisticsChangedDelegate, const FPlayerStatistics&, NewStats);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnSettingsChangedDelegate, const FPlayerSettings&, NewSettings);

UCLASS()
class AMyPlayerState : public APlayerState
{
    GENERATED_BODY()

public:
    AMyPlayerState();

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    // 统计数据
    UPROPERTY(ReplicatedUsing = OnRep_Statistics, BlueprintReadOnly, Category = "Stats")
    FPlayerStatistics Statistics;

    // 设置
    UPROPERTY(ReplicatedUsing = OnRep_Settings, BlueprintReadOnly, Category = "Settings")
    FPlayerSettings Settings;

    // 游戏内状态
    UPROPERTY(ReplicatedUsing = OnRep_bIsAlive, BlueprintReadOnly, Category = "Game")
    bool bIsAlive;

    UPROPERTY(ReplicatedUsing = OnRep_TeamId, BlueprintReadOnly, Category = "Game")
    int32 TeamId;

    // 委托
    UPROPERTY(BlueprintAssignable, Category = "Events")
    FOnStatisticsChangedDelegate OnStatisticsChanged;

    UPROPERTY(BlueprintAssignable, Category = "Events")
    FOnSettingsChangedDelegate OnSettingsChanged;

    // 服务器函数
    UFUNCTION(Server, Reliable, WithValidation)
    void Server_UpdateSettings(const FPlayerSettings& NewSettings);

    UFUNCTION(BlueprintCallable, Category = "Stats")
    void AddKill();

    UFUNCTION(BlueprintCallable, Category = "Stats")
    void AddDeath();

    UFUNCTION(BlueprintCallable, Category = "Stats")
    void AddAssist();

    UFUNCTION(BlueprintCallable, Category = "Stats")
    void AddScore(float Points);

    UFUNCTION(BlueprintCallable, Category = "Game")
    void SetAlive(bool bNewAlive);

    UFUNCTION(BlueprintCallable, Category = "Game")
    void SetTeam(int32 NewTeamId);

protected:
    UFUNCTION()
    void OnRep_Statistics(const FPlayerStatistics& OldStats);

    UFUNCTION()
    void OnRep_Settings(const FPlayerSettings& OldSettings);

    UFUNCTION()
    void OnRep_bIsAlive(bool bOldAlive);

    UFUNCTION()
    void OnRep_TeamId(int32 OldTeamId);

private:
    void BroadcastStatisticsChanged();
    void BroadcastSettingsChanged();
};

// MyPlayerState.cpp
#include "MyPlayerState.h"
#include "Net/UnrealNetwork.h"

AMyPlayerState::AMyPlayerState()
{
    bIsAlive = true;
    TeamId = -1;
}

void AMyPlayerState::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 统计数据 - 所有人可见
    DOREPLIFETIME(AMyPlayerState, Statistics);

    // 设置 - 只有所有者可见
    DOREPLIFETIME_CONDITION(AMyPlayerState, Settings, COND_OwnerOnly);

    // 存活状态 - 所有人可见
    DOREPLIFETIME(AMyPlayerState, bIsAlive);

    // 队伍ID - 所有人可见
    DOREPLIFETIME(AMyPlayerState, TeamId);
}

// ========== 服务器函数 ==========

bool AMyPlayerState::Server_UpdateSettings_Validate(const FPlayerSettings& NewSettings)
{
    // 验证设置合法性
    if (NewSettings.SelectedCharacterId < 0 || NewSettings.SelectedCharacterId > 100)
    {
        return false;
    }
    return true;
}

void AMyPlayerState::Server_UpdateSettings_Implementation(const FPlayerSettings& NewSettings)
{
    FPlayerSettings OldSettings = Settings;
    Settings = NewSettings;

    // 服务器端触发回调
    OnRep_Settings(OldSettings);
}

void AMyPlayerState::AddKill()
{
    if (HasAuthority())
    {
        Statistics.Kills++;
        Statistics.Score += 100.0f;
        BroadcastStatisticsChanged();
    }
}

void AMyPlayerState::AddDeath()
{
    if (HasAuthority())
    {
        Statistics.Deaths++;
        BroadcastStatisticsChanged();
    }
}

void AMyPlayerState::AddAssist()
{
    if (HasAuthority())
    {
        Statistics.Assists++;
        Statistics.Score += 50.0f;
        BroadcastStatisticsChanged();
    }
}

void AMyPlayerState::AddScore(float Points)
{
    if (HasAuthority())
    {
        Statistics.Score += Points;
        BroadcastStatisticsChanged();
    }
}

void AMyPlayerState::SetAlive(bool bNewAlive)
{
    if (HasAuthority())
    {
        bool bOldAlive = bIsAlive;
        bIsAlive = bNewAlive;
        OnRep_bIsAlive(bOldAlive);
    }
}

void AMyPlayerState::SetTeam(int32 NewTeamId)
{
    if (HasAuthority())
    {
        int32 OldTeamId = TeamId;
        TeamId = NewTeamId;
        OnRep_TeamId(OldTeamId);
    }
}

// ========== OnRep回调 ==========

void AMyPlayerState::OnRep_Statistics(const FPlayerStatistics& OldStats)
{
    // 客户端响应统计变化
    OnStatisticsChanged.Broadcast(Statistics);

    UE_LOG(LogTemp, Log, TEXT("Stats updated - K:%d D:%d A:%d Score:%.0f"),
        Statistics.Kills, Statistics.Deaths, Statistics.Assists, Statistics.Score);
}

void AMyPlayerState::OnRep_Settings(const FPlayerSettings& OldSettings)
{
    // 响应设置变化
    OnSettingsChanged.Broadcast(Settings);
}

void AMyPlayerState::OnRep_bIsAlive(bool bOldAlive)
{
    if (bIsAlive != bOldAlive)
    {
        UE_LOG(LogTemp, Log, TEXT("Player alive state changed to: %s"),
            bIsAlive ? TEXT("Alive") : TEXT("Dead"));
    }
}

void AMyPlayerState::OnRep_TeamId(int32 OldTeamId)
{
    if (TeamId != OldTeamId)
    {
        UE_LOG(LogTemp, Log, TEXT("Team changed from %d to %d"), OldTeamId, TeamId);
    }
}

// ========== 辅助函数 ==========

void AMyPlayerState::BroadcastStatisticsChanged()
{
    // 在服务器端广播变化
    OnStatisticsChanged.Broadcast(Statistics);
}
```

---

## 七、常见问题

### Q1：OnRep没有触发？

```cpp
// 检查清单：
// 1. 属性是否正确声明？
UPROPERTY(ReplicatedUsing = OnRep_Property)  // 注意是 ReplicatedUsing 不是 Replicated

// 2. 是否在GetLifetimeReplicatedProps中注册？
DOREPLIFETIME(AMyClass, Property);

// 3. 是否包含正确的头文件？
#include "Net/UnrealNetwork.h"

// 4. OnRep函数是否标记为UFUNCTION？
UFUNCTION()
void OnRep_Property();

// 5. Actor是否启用了复制？
bReplicates = true;
```

### Q2：如何调试属性复制？

```cpp
// 使用控制台命令
// stat net - 查看网络统计
// net.Replicate.Property - 属性复制详情

// 代码中调试
void AMyActor::DebugPropertyReplication()
{
    // 打印所有复制属性
    UClass* Class = GetClass();
    for (TFieldIterator<FProperty> It(Class); It; ++It)
    {
        FProperty* Property = *It;
        if (Property->HasAnyPropertyFlags(CPF_Net))
        {
            UE_LOG(LogTemp, Log, TEXT("Replicated: %s"), *Property->GetName());
        }
    }
}
```

---

## 八、总结

本课我们学习了：

1. **OnRep机制**：深入理解回调函数的工作原理
2. **条件复制**：掌握各种复制条件的使用场景
3. **复杂数据类型**：学会处理结构体、数组、Map的复制
4. **最佳实践**：优化性能、避免常见错误
5. **高级技巧**：FastArray、条件属性组

---

## 九、下节预告

**第5课：网络同步策略**

将深入学习：
- 状态同步 vs 输入同步
- 客户端预测机制
- 服务器权威设计
- 延迟补偿技术

---

*课程版本：1.0*
*最后更新：2026-04-10*