# 第2课：Actor复制系统

> 本课深入讲解UE5的Actor复制机制，这是多人游戏网络同步的核心技术。

---

## 课程目标

- 理解Actor复制的基本原理
- 掌握Role与RemoteRole的工作机制
- 熟练使用UProperty网络修饰符
- 了解ReplicationGraph的基本概念
- 掌握网络优先级与更新频率的配置

---

## 一、Actor复制原理

### 1.1 什么是Actor复制

Actor复制是指服务器将Actor的状态变化自动同步给所有相关客户端的过程。

```
┌─────────────────────────────────────────────────────────────┐
│                        服务器                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                     Actor                            │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │   │
│  │  │ Position │  │ Rotation │  │ Health   │          │   │
│  │  │ (复制)   │  │ (复制)   │  │ (复制)   │          │   │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘          │   │
│  └───────┼─────────────┼─────────────┼─────────────────┘   │
│          │             │             │                      │
│          └─────────────┴─────────────┘                      │
│                        │                                     │
│              复制系统收集变化                                 │
│                        │                                     │
└────────────────────────┼─────────────────────────────────────┘
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
    ┌──────────┐   ┌──────────┐   ┌──────────┐
    │ Client 1 │   │ Client 2 │   │ Client 3 │
    │ 更新位置  │   │ 更新位置  │   │ 更新位置  │
    │ 更新旋转  │   │ 更新旋转  │   │ 更新旋转  │
    │ 更新血量  │   │ 更新血量  │   │ 更新血量  │
    └──────────┘   └──────────┘   └──────────┘
```

### 1.2 复制条件概述

```cpp
// 复制属性的三种基本条件

// 1. Replicated - 自动复制，客户端被动接收
UPROPERTY(Replicated)
float Health;

// 2. ReplicatedUsing - 复制并触发回调
UPROPERTY(ReplicatedUsing = OnRep_Health)
float Health;

// 3. NotReplicated - 不复制（默认行为）
UPROPERTY()
float LocalOnlyValue;
```

---

## 二、Role与RemoteRole

### 2.1 ENetRole枚举详解

```cpp
// Engine/Source/Runtime/Engine/Classes/Engine/EngineTypes.h

enum ENetRole
{
    ROLE_None,              // 无网络角色，不参与复制
    ROLE_SimulatedProxy,    // 模拟代理，客户端被动接收更新
    ROLE_AutonomousProxy,   // 自主代理，客户端可以进行本地预测
    ROLE_Authority,         // 权威，拥有对该Actor的完全控制权
    ROLE_MAX,
};
```

**角色权限对比：**

| 角色 | 可以执行RPC | 可以修改复制属性 | 典型用途 |
|------|------------|-----------------|---------|
| **None** | 否 | 否 | 本地装饰物、UI元素 |
| **SimulatedProxy** | 只能调用Multicast | 否 | 其他玩家的角色、AI敌人 |
| **AutonomousProxy** | Server RPC + Multicast | 否 | 本地玩家控制的角色 |
| **Authority** | 所有RPC | 是 | 服务器上的所有Actor |

### 2.2 角色关系图解

```
服务器视角:
┌─────────────────────────────────────────────────────────┐
│                      服务器 World                         │
│                                                          │
│  本地玩家Character (服务器拥有)                            │
│  LocalRole = Authority                                   │
│  RemoteRole = AutonomousProxy  ───────────────┐          │
│                                                │          │
│  其他玩家Character (服务器拥有)                 │          │
│  LocalRole = Authority                        │          │
│  RemoteRole = SimulatedProxy ─────────────────┐│          │
│                                               ││          │
└───────────────────────────────────────────────┼┼──────────┘
                                                ││
                    网络同步                      ││
                                                ││
┌───────────────────────────────────────────────┼┼──────────┐
│                      客户端 World              ││          │
│                                               ││          │
│  本地玩家Character ◄──────────────────────────┘│          │
│  LocalRole = AutonomousProxy                   │          │
│  RemoteRole = Authority                        │          │
│                                                │          │
│  其他玩家Character ◄───────────────────────────┘          │
│  LocalRole = SimulatedProxy                               │
│  RemoteRole = Authority                                   │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

### 2.3 设置Actor的网络角色

```cpp
// 在Actor构造函数中设置
AMyCharacter::AMyCharacter()
{
    // 启用网络复制
    bReplicates = true;

    // 设置远程角色（服务器端设置）
    SetRemoteRoleForBackwardsCompat(ROLE_AutonomousProxy);

    // 其他网络设置
    NetUpdateFrequency = 100.0f;    // 每秒更新100次
    NetCullDistanceSquared = 15000.0f * 15000.0f;  // 网络剔除距离
    NetPriority = 1.0f;             // 网络优先级
}

// 动态修改角色
void AMyActor::SetNetworkRole(ENetRole NewRole)
{
    if (HasAuthority())
    {
        SetReplicates(true);
        SetRemoteRoleForBackwardsCompat(NewRole);
    }
}

// 条件性设置角色
void AMyGameMode::SpawnActorWithRole(TSubclassOf<AMyActor> ActorClass, ENetRole DesiredRole)
{
    FActorSpawnParameters SpawnParams;
    SpawnParams.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;

    AMyActor* NewActor = GetWorld()->SpawnActor<AMyActor>(ActorClass, SpawnParams);

    if (NewActor)
    {
        NewActor->SetReplicates(true);
        NewActor->SetRemoteRoleForBackwardsCompat(DesiredRole);
    }
}
```

### 2.4 判断当前角色

```cpp
// 常用判断方法

// 1. 检查是否为权威
if (HasAuthority())
{
    // 服务器端逻辑
}

// 2. 检查本地角色
if (GetLocalRole() == ROLE_Authority)
{
    // 当前端拥有权威
}

// 3. 检查远程角色
if (GetRemoteRole() == ROLE_AutonomousProxy)
{
    // 对端是自主代理（通常是PlayerController）
}

// 4. 检查是否为模拟代理
if (GetLocalRole() == ROLE_SimulatedProxy)
{
    // 当前是模拟代理，只能被动接收更新
}

// 5. 完整的角色检查示例
void AMyActor::DebugNetworkRole()
{
    FString RoleString;

    switch (GetLocalRole())
    {
        case ROLE_None:
            RoleString = TEXT("None");
            break;
        case ROLE_SimulatedProxy:
            RoleString = TEXT("SimulatedProxy");
            break;
        case ROLE_AutonomousProxy:
            RoleString = TEXT("AutonomousProxy");
            break;
        case ROLE_Authority:
            RoleString = TEXT("Authority");
            break;
        default:
            RoleString = TEXT("Unknown");
            break;
    }

    UE_LOG(LogTemp, Log, TEXT("Actor %s LocalRole: %s"), *GetName(), *RoleString);
}
```

---

## 三、UProperty网络修饰符

### 3.1 属性复制修饰符详解

```cpp
// MyCharacter.h
UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    // ============ 基本复制修饰符 ============

    // Replicated - 简单复制，无回调
    UPROPERTY(Replicated)
    float MaxHealth;

    // ReplicatedUsing - 复制并触发回调函数
    UPROPERTY(ReplicatedUsing = OnRep_CurrentHealth)
    float CurrentHealth;

    // OnRep回调函数声明
    UFUNCTION()
    void OnRep_CurrentHealth(float OldValue);

    // ============ 高级复制修饰符 ============

    // ReplicatedUsing + 条件复制
    UPROPERTY(ReplicatedUsing = OnRep_TeamId, EditAnywhere, Category = "Team")
    int32 TeamId;

    UFUNCTION()
    void OnRep_TeamId(int32 OldTeamId);

    // transient - 不参与序列化和复制
    UPROPERTY(Transient)
    float TempDamageMultiplier;

    // NotReplicated - 明确标记不复制（在结构体内有用）
    UPROPERTY(NotReplicated)
    FString LocalDebugString;
};
```

### 3.2 GetLifetimeReplicatedProps实现

```cpp
// MyCharacter.cpp

#include "Net/UnrealNetwork.h"  // 必须包含此头文件

void AMyCharacter::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 基本复制
    DOREPLIFETIME(AMyCharacter, MaxHealth);

    // 条件复制 - 只对所有者可见
    DOREPLIFETIME_CONDITION(AMyCharacter, CurrentHealth, COND_OwnerOnly);

    // 条件复制 - 只在初始同步时复制
    DOREPLIFETIME_CONDITION(AMyCharacter, TeamId, COND_InitialOnly);

    // 条件复制 - 自定义条件
    DOREPLIFETIME_CONDITION_NOTIFY(AMyCharacter, SomeProperty, COND_None, REPNOTIFY_OnChanged);
}
```

### 3.3 复制条件（Lifetime Conditions）

```cpp
// 可用的复制条件枚举

enum ELifetimeCondition
{
    COND_None,                    // 无条件，始终复制
    COND_InitialOnly,             // 仅初始同步时复制
    COND_OwnerOnly,               // 仅复制给所有者
    COND_SkipOwner,               // 复制给除所有者外的所有人
    COND_SimulatedOnly,           // 仅复制给SimulatedProxy
    COND_SimulatedOnlyNoReplay,   // 仅复制给SimulatedProxy，不录制回放
    COND_AutonomousOnly,          // 仅复制给AutonomousProxy
    COND_SimulatedOrPhysics,      // 复制给SimulatedProxy或物理Actor
    COND_InitialOrOwner,          // 初始同步或所有者
    COND_ReplayOrOwner,           // 回放或所有者
    COND_ReplayOnly,              // 仅回放
    COND_SkipReplay,              // 跳过回放
    COND_Custom,                  // 自定义条件
    COND_Max,
};
```

**条件使用示例：**

```cpp
void AMyCharacter::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 所有人都看到的属性
    DOREPLIFETIME(AMyCharacter, bIsAlive);

    // 只有所有者知道自己的弹药量
    DOREPLIFETIME_CONDITION(AMyCharacter, AmmoCount, COND_OwnerOnly);

    // 其他玩家看到的位置（不包括所有者）
    DOREPLIFETIME_CONDITION(AMyCharacter, bIsCrouching, COND_SkipOwner);

    // 只在初始化时同步的属性（之后不再更新）
    DOREPLIFETIME_CONDITION(AMyCharacter, CharacterClass, COND_InitialOnly);

    // 只在模拟代理上显示的效果
    DOREPLIFETIME_CONDITION(AMyCharacter, CosmeticEffectIndex, COND_SimulatedOnly);

    // 物理Actor的位置同步
    DOREPLIFETIME_CONDITION(AMyCharacter, PhysicsVelocity, COND_SimulatedOrPhysics);
}
```

### 3.4 OnRep回调详解

```cpp
// OnRep回调的特性：
// 1. 在属性值从服务器复制到客户端后自动调用
// 2. 只在客户端执行
// 3. 可以获取旧值（可选参数）

// 声明OnRep回调
UFUNCTION()
void OnRep_CurrentHealth(float OldValue);

// 实现OnRep回调
void AMyCharacter::OnRep_CurrentHealth(float OldValue)
{
    // 检查值是否真的改变了
    if (OldValue != CurrentHealth)
    {
        // 处理生命值变化
        if (CurrentHealth < OldValue)
        {
            // 受到伤害
            PlayDamageEffect();
        }
        else
        {
            // 恢复生命
            PlayHealEffect();
        }

        // 更新UI
        UpdateHealthBar();

        // 广播事件
        OnHealthChanged.Broadcast(OldValue, CurrentHealth);
    }
}

// 带参数vs不带参数的OnRep

// 方式1：带旧值参数
UPROPERTY(ReplicatedUsing = OnRep_Health)
float Health;
UFUNCTION()
void OnRep_Health(float OldHealth);  // 可以获取旧值

// 方式2：不带参数
UPROPERTY(ReplicatedUsing = OnRep_Ammo)
int32 Ammo;
UFUNCTION()
void OnRep_Ammo();  // 无法获取旧值

// 方式3：使用REPNOTIFY_Always
// 在GetLifetimeReplicatedProps中使用
DOREPLIFETIME_CONDITION_NOTIFY(AMyCharacter, Health, COND_None, REPNOTIFY_Always);
// 这意味着即使值相同也会触发OnRep
```

### 3.5 复杂数据类型的复制

```cpp
// 结构体复制
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

    UPROPERTY(NotReplicated)  // 结构体内的NotReplicated属性
    FString LocalNickname;
};

// 在Actor中使用
UPROPERTY(ReplicatedUsing = OnRep_Stats)
FPlayerStats PlayerStats;

UFUNCTION()
void OnRep_Stats(const FPlayerStats& OldStats)
{
    if (PlayerStats.Health != OldStats.Health)
    {
        OnHealthChanged();
    }
    if (PlayerStats.Kills != OldStats.Kills)
    {
        OnKillCountChanged();
    }
}

// TArray复制
UPROPERTY(ReplicatedUsing = OnRep_Inventory)
TArray<AInventoryItem*> Inventory;

UFUNCTION()
void OnRep_Inventory(const TArray<AInventoryItem*>& OldInventory)
{
    // 比较新旧数组来确定变化
    for (auto* Item : Inventory)
    {
        if (!OldInventory.Contains(Item))
        {
            OnItemAdded(Item);
        }
    }

    for (auto* OldItem : OldInventory)
    {
        if (!Inventory.Contains(OldItem))
        {
            OnItemRemoved(OldItem);
        }
    }
}

// TMap复制
UPROPERTY(Replicated)
TMap<FName, float> AttributeValues;

// TSet复制
UPROPERTY(Replicated)
TSet<AActor*> KnownActors;
```

---

## 四、Actor复制流程

### 4.1 复制流程图

```
┌─────────────────────────────────────────────────────────────┐
│                        服务器端                               │
│                                                              │
│  1. 游戏Tick更新                                             │
│     │                                                        │
│     ▼                                                        │
│  2. 收集需要复制的Actor（ReplicationGraph）                   │
│     │                                                        │
│     ▼                                                        │
│  3. 按优先级排序Actor列表                                     │
│     │                                                        │
│     ▼                                                        │
│  4. 对每个Actor：                                            │
│     ├── 检查NetCullDistance（距离剔除）                       │
│     ├── 检查NetUpdateFrequency（更新频率）                    │
│     ├── 收集变化的属性                                       │
│     └── 序列化属性数据                                       │
│     │                                                        │
│     ▼                                                        │
│  5. 发送数据包给相关客户端                                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
                           │
                           │ 网络传输
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                        客户端                                 │
│                                                              │
│  6. 接收网络数据包                                           │
│     │                                                        │
│     ▼                                                        │
│  7. 反序列化属性数据                                         │
│     │                                                        │
│     ▼                                                        │
│  8. 更新属性值                                               │
│     │                                                        │
│     ▼                                                        │
│  9. 触发OnRep回调（如有）                                    │
│     │                                                        │
│     ▼                                                        │
│  10. 更新游戏状态                                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 复制优先级

```cpp
// 设置网络优先级
AMyCharacter::AMyCharacter()
{
    // 优先级越高，越优先复制
    // 默认值为1.0
    NetPriority = 2.0f;  // 高优先级

    // 玩家角色通常需要高优先级
    // 远处的物体可以降低优先级
}

// 动态调整优先级
float AMyActor::GetNetPriority(const FVector& ViewPos,
                                const FVector& ViewDir,
                                AActor* Viewer,
                                class UNetConnection* ViewerConnection,
                                UActorChannel* InChannel,
                                float Time,
                                bool bLowBandwidth)
{
    // 基础优先级
    float Priority = NetPriority;

    // 距离调整
    float DistanceSq = (GetActorLocation() - ViewPos).SizeSquared();
    if (DistanceSq > 10000.0f * 10000.0f)
    {
        Priority *= 0.5f;  // 远距离降低优先级
    }

    // 视线方向调整
    FVector ToActor = (GetActorLocation() - ViewPos).GetSafeNormal();
    float DotProduct = FVector::DotProduct(ViewDir, ToActor);
    if (DotProduct > 0.8f)
    {
        Priority *= 1.5f;  // 视线内提高优先级
    }

    return Priority;
}
```

### 4.3 网络更新频率

```cpp
// NetUpdateFrequency - 每秒最大更新次数
AMyCharacter::AMyCharacter()
{
    // 高频更新（玩家角色）
    NetUpdateFrequency = 100.0f;  // 每秒最多100次

    // 低频更新（背景物体）
    // NetUpdateFrequency = 10.0f;

    // 非常低频（装饰物）
    // NetUpdateFrequency = 1.0f;
}

// MinNetUpdateFrequency - 每秒最小更新次数
// 确保即使Actor静止也会定期同步
AMyActor::AMyActor()
{
    MinNetUpdateFrequency = 2.0f;  // 每秒至少2次
}

// 自定义更新频率逻辑
float AMyActor::GetNetUpdateFrequency()
{
    // 根据游戏状态动态调整
    if (bIsInCombat)
    {
        return 100.0f;  // 战斗中高频更新
    }
    else if (bIsMoving)
    {
        return 50.0f;   // 移动时中等频率
    }
    else
    {
        return 10.0f;   // 静止时低频更新
    }
}
```

### 4.4 网络剔除距离

```cpp
// NetCullDistanceSquared - 超出此距离的Actor不会被复制
AMyCharacter::AMyCharacter()
{
    // 15km内的玩家会收到此Actor的更新
    NetCullDistanceSquared = 15000.0f * 15000.0f;
}

// 动态调整剔除距离
void AMyActor::AdjustCullDistance(float NewDistance)
{
    NetCullDistanceSquared = NewDistance * NewDistance;

    // 强制立即重新计算可见性
    if (UNetDriver* NetDriver = GetWorld()->GetNetDriver())
    {
        NetDriver->ForceActorRelevant(this);
    }
}

// 根据重要性动态调整
void AMyImportantActor::UpdateCullDistanceBasedOnImportance()
{
    if (bIsObjectiveTarget)
    {
        // 任务目标始终在远处可见
        NetCullDistanceSquared = 50000.0f * 50000.0f;
    }
    else
    {
        // 普通物体正常剔除
        NetCullDistanceSquared = 10000.0f * 10000.0f;
    }
}
```

---

## 五、ReplicationGraph基础

### 5.1 ReplicationGraph概述

ReplicationGraph是UE5的高性能复制系统，用于优化大量Actor的网络复制。

```cpp
// 启用ReplicationGraph（DefaultEngine.ini）
[/Script/Engine.Engine]
NetDriverDefinitions=(DefName="GameNetDriver",DriverClassName="/Script/Engine.ReplicationGraphDriver",DriverClassNameFallback="/Script/Engine.ReplicationGraphDriver")

[/Script/Engine.ReplicationGraph]
bUseReplicationGraph=true
```

### 5.2 ReplicationGraph节点类型

```
┌─────────────────────────────────────────────────────────────┐
│                    ReplicationGraph                          │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              GridSpatializationNode                  │    │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐               │    │
│  │  │Cell(0,0)│ │Cell(1,0)│ │Cell(2,0)│  ...          │    │
│  │  │玩家可见  │ │玩家可见  │ │玩家可见  │               │    │
│  │  └─────────┘ └─────────┘ └─────────┘               │    │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐               │    │
│  │  │Cell(0,1)│ │Cell(1,1)│ │Cell(2,1)│  ...          │    │
│  │  └─────────┘ └─────────┘ └─────────┘               │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │               AlwaysRelevantNode                     │    │
│  │  所有玩家都需要的Actor（GameState, GameMode等）       │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              PlayerStateNode                         │    │
│  │  只对特定玩家相关的PlayerState                        │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 自定义ReplicationGraph

```cpp
// MyReplicationGraph.h
#pragma once

#include "ReplicationGraph.h"
#include "MyReplicationGraph.generated.h"

UCLASS()
class UMyReplicationGraph : public UReplicationGraph
{
    GENERATED_BODY()

public:
    virtual void InitGlobalGraphNodes() override;
    virtual void InitConnectionGraphNodes(UNetReplicationGraphConnection* Connection) override;

private:
    // 空间化节点
    UReplicationGraphNode_GridSpatialization2D* GridNode;

    // 始终相关节点
    UReplicationGraphNode_AlwaysRelevant_AlwaysRelevantNode* AlwaysRelevantNode;

    // 玩家状态节点
    UReplicationGraphNode_PlayerStateFrequencyLimiter* PlayerStateNode;
};

// MyReplicationGraph.cpp
#include "MyReplicationGraph.h"

void UMyReplicationGraph::InitGlobalGraphNodes()
{
    // 创建空间化节点
    GridNode = CreateNewNode<UReplicationGraphNode_GridSpatialization2D>();
    GridNode->CellSize = 10000.0f;  // 10km格子
    GridNode->SpatialBias = FVector2D(-500000.0f, -500000.0f);
    AddGlobalGraphNode(GridNode);

    // 创建始终相关节点
    AlwaysRelevantNode = CreateNewNode<UReplicationGraphNode_AlwaysRelevant_AlwaysRelevantNode>();
    AddGlobalGraphNode(AlwaysRelevantNode);

    // 创建玩家状态节点
    PlayerStateNode = CreateNewNode<UReplicationGraphNode_PlayerStateFrequencyLimiter>();
    PlayerStateNode->AddPlayerStateDependency();  // 添加依赖关系
    AddGlobalGraphNode(PlayerStateNode);
}

void UMyReplicationGraph::InitConnectionGraphNodes(UNetReplicationGraphConnection* Connection)
{
    // 为每个连接创建专属节点
    UReplicationGraphNode_AlwaysRelevant_ForConnection* AlwaysRelevantForConnection =
        CreateNewNode<UReplicationGraphNode_AlwaysRelevant_ForConnection>();
    AddConnectionGraphNode(AlwaysRelevantForConnection, Connection);
}
```

### 5.4 注册Actor到ReplicationGraph

```cpp
// 方式1：通过类设置（推荐）
// 在Actor的构造函数中
AMyActor::AMyActor()
{
    // 使用空间化节点
    SetReplicates(true);

    // 指定ReplicationGraph类
    ReplicationGraphSettings.bUseSpatialization = true;
}

// 方式2：手动注册到特定节点
void AMyGameMode::RegisterActorToReplicationGraph(AActor* Actor)
{
    if (UMyReplicationGraph* Graph = Cast<UMyReplicationGraph>(GetWorld()->GetReplicationGraph()))
    {
        // 根据Actor类型决定放入哪个节点
        if (Actor->IsA<APlayerState>())
        {
            // 玩家状态放专用节点
            Graph->PlayerStateNode->AddActor(Actor);
        }
        else if (Actor->bAlwaysRelevant)
        {
            // 始终相关的Actor
            Graph->AlwaysRelevantNode->AddActor(Actor);
        }
        else
        {
            // 空间化Actor
            Graph->GridNode->AddActor(Actor);
        }
    }
}
```

---

## 六、实践任务

### 任务1：创建一个完整的网络Actor

```cpp
// MyNetworkActor.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyNetworkActor.generated.h"

// 状态变化委托
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnStateChangedDelegate, float, OldValue, float, NewValue);

UCLASS(BlueprintType)
class AMyNetworkActor : public AActor
{
    GENERATED_BODY()

public:
    AMyNetworkActor();

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    // 网络属性
    UPROPERTY(ReplicatedUsing = OnRep_ActiveState, BlueprintReadOnly, Category = "State")
    bool bIsActive;

    UPROPERTY(ReplicatedUsing = OnRep_PowerLevel, BlueprintReadOnly, Category = "State")
    float PowerLevel;

    UPROPERTY(Replicated, BlueprintReadOnly, Category = "State")
    FName ObjectName;

    // 只有所有者可见
    UPROPERTY(Replicated, BlueprintReadOnly, Category = "Owner", meta = (AllowPrivateAccess = "true"))
    APlayerController* OwnerController;

    // 委托
    UPROPERTY(BlueprintAssignable, Category = "Events")
    FOnStateChangedDelegate OnActiveStateChanged;

    UPROPERTY(BlueprintAssignable, Category = "Events")
    FOnStateChangedDelegate OnPowerLevelChanged;

    // 服务器函数
    UFUNCTION(BlueprintCallable, Category = "State", Server, Reliable)
    void Server_SetActive(bool bNewActive);

    UFUNCTION(BlueprintCallable, Category = "State", Server, Reliable)
    void Server_SetPowerLevel(float NewPowerLevel);

protected:
    virtual void BeginPlay() override;

    // OnRep回调
    UFUNCTION()
    void OnRep_ActiveState(bool bOldValue);

    UFUNCTION()
    void OnRep_PowerLevel(float OldValue);

    // 辅助函数
    void HandleActiveStateChanged();
    void HandlePowerLevelChanged(float OldValue);
};

// MyNetworkActor.cpp
#include "MyNetworkActor.h"
#include "Net/UnrealNetwork.h"

AMyNetworkActor::AMyNetworkActor()
{
    PrimaryActorTick.bCanEverTick = false;

    // 网络设置
    bReplicates = true;
    NetUpdateFrequency = 30.0f;
    NetPriority = 1.0f;
    NetCullDistanceSquared = 15000.0f * 15000.0f;

    // 默认值
    bIsActive = false;
    PowerLevel = 100.0f;
}

void AMyNetworkActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 所有客户端可见
    DOREPLIFETIME(AMyNetworkActor, bIsActive);
    DOREPLIFETIME(AMyNetworkActor, PowerLevel);
    DOREPLIFETIME(AMyNetworkActor, ObjectName);

    // 只有所有者可见
    DOREPLIFETIME_CONDITION(AMyNetworkActor, OwnerController, COND_OwnerOnly);
}

void AMyNetworkActor::BeginPlay()
{
    Super::BeginPlay();

    if (HasAuthority())
    {
        // 服务器初始化
        ObjectName = FName(*FString::Printf(TEXT("Object_%d"), FMath::Rand()));
    }
}

void AMyNetworkActor::Server_SetActive_Implementation(bool bNewActive)
{
    if (bIsActive != bNewActive)
    {
        bool bOldValue = bIsActive;
        bIsActive = bNewActive;

        // 服务器端直接处理
        HandleActiveStateChanged();

        UE_LOG(LogTemp, Log, TEXT("Object %s active state changed to: %s"),
            *ObjectName.ToString(), bIsActive ? TEXT("true") : TEXT("false"));
    }
}

void AMyNetworkActor::Server_SetPowerLevel_Implementation(float NewPowerLevel)
{
    NewPowerLevel = FMath::Clamp(NewPowerLevel, 0.0f, 100.0f);

    if (!FMath::IsNearlyEqual(PowerLevel, NewPowerLevel))
    {
        float OldValue = PowerLevel;
        PowerLevel = NewPowerLevel;

        // 服务器端处理
        HandlePowerLevelChanged(OldValue);
    }
}

void AMyNetworkActor::OnRep_ActiveState(bool bOldValue)
{
    // 客户端回调
    OnActiveStateChanged.Broadcast(bOldValue ? 1.0f : 0.0f, bIsActive ? 1.0f : 0.0f);
    HandleActiveStateChanged();
}

void AMyNetworkActor::OnRep_PowerLevel(float OldValue)
{
    // 客户端回调
    OnPowerLevelChanged.Broadcast(OldValue, PowerLevel);
    HandlePowerLevelChanged(OldValue);
}

void AMyNetworkActor::HandleActiveStateChanged()
{
    // 通用逻辑（服务器和客户端都执行）
    if (bIsActive)
    {
        // 激活效果
        PlayActivateEffect();
    }
    else
    {
        // 停用效果
        PlayDeactivateEffect();
    }
}

void AMyNetworkActor::HandlePowerLevelChanged(float OldValue)
{
    // 更新视觉效果
    UpdatePowerVisuals(PowerLevel);
}
```

### 任务2：实现条件复制系统

```cpp
// MyConditionalReplicationActor.h
UCLASS()
class AMyConditionalReplicationActor : public AActor
{
    GENERATED_BODY()

public:
    AMyConditionalReplicationActor();

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    // 使用自定义条件复制
    virtual bool ReplicateSubobjects(UActorChannel* Channel,
                                     class FOutBunch* Bunch,
                                     FReplicationFlags* RepFlags) override;

protected:
    // 自定义条件判断
    bool ShouldReplicateToClient(AActor* Viewer) const;

    // 条件属性
    UPROPERTY(ReplicatedUsing = OnRep_TeamData, BlueprintReadOnly)
    int32 TeamId;

    UPROPERTY(ReplicatedUsing = OnRep_VisibleToTeams, BlueprintReadOnly)
    TArray<int32> VisibleToTeams;

    UPROPERTY(Replicated, BlueprintReadOnly)
    bool bGlobalVisibility;

    UFUNCTION()
    void OnRep_TeamData(int32 OldTeamId);

    UFUNCTION()
    void OnRep_VisibleToTeams();
};

// MyConditionalReplicationActor.cpp
#include "MyConditionalReplicationActor.h"
#include "Net/UnrealNetwork.h"

AMyConditionalReplicationActor::AMyConditionalReplicationActor()
{
    bReplicates = true;
}

void AMyConditionalReplicationActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 使用COND_Custom条件
    FDoRepLifetimeParams Params;
    Params.bIsPushBased = true;
    Params.Condition = COND_Custom;  // 自定义条件

    DOREPLIFETIME_WITH_PARAMS(AMyConditionalReplicationActor, TeamId, Params);
    DOREPLIFETIME(AMyConditionalReplicationActor, bGlobalVisibility);
}

bool AMyConditionalReplicationActor::ShouldReplicateToClient(AActor* Viewer) const
{
    // 如果全局可见，复制给所有人
    if (bGlobalVisibility)
    {
        return true;
    }

    // 检查查看者的队伍是否在可见列表中
    // 注意：这里需要根据你的队伍系统调整
    // int32 ViewerTeam = GetTeamForActor(Viewer);
    // return VisibleToTeams.Contains(ViewerTeam);

    return false;
}

void AMyConditionalReplicationActor::OnRep_TeamData(int32 OldTeamId)
{
    UE_LOG(LogTemp, Log, TEXT("Team changed from %d to %d"), OldTeamId, TeamId);
}

void AMyConditionalReplicationActor::OnRep_VisibleToTeams()
{
    UE_LOG(LogTemp, Log, TEXT("Visible teams updated"));
}
```

---

## 七、常见问题

### Q1：属性复制不生效怎么办？

```cpp
// 检查清单：
// 1. 确保Actor的bReplicates = true
// 2. 确保在GetLifetimeReplicatedProps中注册了属性
// 3. 确保包含了 #include "Net/UnrealNetwork.h"
// 4. 确保属性有UPROPERTY(Replicated)或UPROPERTY(ReplicatedUsing)修饰符
// 5. 确保在服务器端修改属性值

// 调试代码
void AMyActor::DebugReplication()
{
    if (HasAuthority())
    {
        UE_LOG(LogTemp, Warning, TEXT("Authority: Property should replicate"));
        // 修改属性测试
        TestProperty += 1.0f;
        ForceNetUpdate();  // 强制立即发送更新
    }
}
```

### Q2：OnRep回调没有触发？

```cpp
// 可能原因：

// 1. OnRep函数没有标记为UFUNCTION()
// 错误：
void OnRep_Health();  // 缺少UFUNCTION

// 正确：
UFUNCTION()
void OnRep_Health();

// 2. 属性没有使用ReplicatedUsing
// 错误：
UPROPERTY(Replicated)
float Health;  // 不会触发OnRep

// 正确：
UPROPERTY(ReplicatedUsing = OnRep_Health)
float Health;

// 3. 值没有实际改变
// 使用REPNOTIFY_Always强制触发
DOREPLIFETIME_CONDITION_NOTIFY(AMyActor, Health, COND_None, REPNOTIFY_Always);
```

### Q3：如何优化网络带宽？

```cpp
// 1. 降低更新频率
NetUpdateFrequency = 10.0f;  // 从100降到10

// 2. 使用合适的复制条件
DOREPLIFETIME_CONDITION(AMyActor, SecretData, COND_OwnerOnly);  // 只有所有者需要

// 3. 减少复制属性数量
// 不需要同步的属性使用NotReplicated或Transient

// 4. 使用ReplicationGraph进行空间分区
// 只有附近的玩家才会收到更新

// 5. 使用压缩
// 对于Vector，可以使用更小的精度
UPROPERTY(Replicated)
FVector_NetQuantize NormalizedPosition;  // 压缩位置
```

---

## 八、总结

本课我们学习了：

1. **复制原理**：理解Actor复制的基本机制
2. **角色系统**：掌握Role与RemoteRole的工作方式
3. **属性修饰符**：熟练使用各种复制修饰符和条件
4. **复制流程**：了解优先级、更新频率、剔除距离
5. **ReplicationGraph**：入门高性能复制系统

---

## 九、下节预告

**第3课：RPC深入解析**

将深入学习：
- Server RPC、Client RPC、Multicast详解
- RPC可靠性保证
- RPC参数序列化
- RPC性能优化

---

*课程版本：1.0*
*最后更新：2026-04-09*