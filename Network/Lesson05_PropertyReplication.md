# 第五课：属性复制详解

> **学习目标**: 深入理解属性复制的底层实现机制
> **时长**: 约60分钟
> **前置知识**: 第四课内容

---

## 一、属性复制架构

### 1.1 核心类关系

```
┌─────────────────────────────────────────────────────────────────────┐
│                      属性复制架构                                    │
└─────────────────────────────────────────────────────────────────────┘

                    ┌──────────────────┐
                    │   FRepLayout     │ 复制布局
                    │  (属性序列化方案) │
                    └────────┬─────────┘
                             │ 创建
                             ▼
                    ┌──────────────────┐
                    │ FObjectReplicator│ 对象复制器
                    │  (执行复制逻辑)   │
                    └────────┬─────────┘
                             │ 管理
                             ▼
                    ┌──────────────────┐
                    │    FRepState     │ 复制状态
                    │ (跟踪属性变化)    │
                    └──────────────────┘
```

### 1.2 FObjectReplicator

**位置**: `Engine/Public/Net/DataReplication.h:73`

FObjectReplicator是属性复制的核心执行者：

```cpp
/**
 * 表示一个正在被复制或处理RPC的对象
 *
 * Bunch结构:
 * |----------------|
 * | NetGUID ObjRef |
 * |----------------|
 * |                |
 * | Properties...  |
 * |                |
 * | RPCs...        |
 * |                |
 * |----------------|
 * | </End Tag>     |
 * |----------------|
 */
class FObjectReplicator
{
public:
    // 初始化
    void InitWithObject(UObject* InObject, UNetConnection* InConnection, bool bUseDefaultState = true);

    // 复制属性
    bool ReplicateProperties(FOutBunch& Bunch, FReplicationFlags RepFlags);

    // 接收属性
    bool ReceivedBunch(FNetBitReader& Bunch, const FReplicationFlags& RepFlags, const bool bHasRepLayout, bool& bOutHasUnmapped);

    // 接收RPC
    bool ReceivedRPC(FNetBitReader& Reader, TSet<FNetworkGUID>& OutUnmappedGuids, bool& bOutSkippedRpcExec, const FReplicationFlags& RepFlags, const FFieldNetCache* FieldCache, ESkipRpcBehavior SkipRpcBehavior);

    // 处理NAK
    void ReceivedNak(int32 NakPacketId);

    // 清理
    void CleanUp();
};
```

### 1.3 FRepLayout (复制布局)

**位置**: `Engine/Public/Net/RepLayout.h`

FRepLayout定义了如何序列化一个对象的所有复制属性：

```cpp
/**
 * 复制布局 - 定义属性如何被序列化
 */
class FRepLayout
{
public:
    // 属性列表
    TArray<FRepParentCmd> Parents;

    // 创建复制布局
    static TSharedPtr<FRepLayout> CreateFromObjectProperties(
        UObject* Object,
        UClass* ObjectClass,
        FProperty* RoleProperty,
        FProperty* RemoteRoleProperty
    );

    // 序列化属性
    void SerializeProperties(
        FNetBitWriter& Writer,
        FRepState* RepState,
        UObject* Object,
        FReplicationFlags RepFlags
    );
};
```

---

## 二、属性复制流程

### 2.1 服务器端复制流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                   服务器端属性复制流程                                │
└─────────────────────────────────────────────────────────────────────┘

TickFlush()
    │
    ▼
ServerReplicateActors()
    │
    ├── 遍历所有客户端连接
    │
    ▼
对每个Actor:
    │
    ├── PreReplication()          // 复制前回调
    │
    ▼
UActorChannel::ReplicateActor()
    │
    ▼
FObjectReplicator::ReplicateProperties()
    │
    ├── 比较属性变化
    │   └── FRepState::UpdateChangeList()
    │
    ├── 序列化脏属性
    │   └── FRepLayout::SerializeProperties()
    │
    ▼
SendBunch()
    │
    ▼
数据发送到客户端
```

### 2.2 客户端接收流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                   客户端属性接收流程                                  │
└─────────────────────────────────────────────────────────────────────┘

TickDispatch()
    │
    ▼
接收Packet
    │
    ▼
UNetConnection::ReceivedRawPacket()
    │
    ▼
拆解为Bunch
    │
    ▼
UChannel::ReceivedRawBunch()
    │
    ▼
UActorChannel::ReceivedBunch()
    │
    ▼
FObjectReplicator::ReceivedBunch()
    │
    ├── 反序列化属性
    │   └── FRepLayout::ReceiveProperties()
    │
    ├── 更新对象属性
    │
    ▼
触发RepNotify回调
```

---

## 三、支持的属性类型

### 3.1 基础类型

```cpp
// 完全支持的基础类型
int8, int16, int32, int64
uint8, uint16, uint32, uint64
float, double
bool
FString
FName
FText
```

### 3.2 数学类型

```cpp
// 向量和旋转
FVector      // 3D向量
FVector2D    // 2D向量
FVector4     // 4D向量
FRotator     // 旋转器
FQuat        // 四元数
FTransform   // 变换
FColor       // 颜色
FLinearColor // 线性颜色
```

### 3.3 容器类型

```cpp
// 动态数组
UPROPERTY(Replicated)
TArray<int32> MyArray;

// 映射
UPROPERTY(Replicated)
TMap<FName, int32> MyMap;

// 集合
UPROPERTY(Replicated)
TSet<int32> MySet;
```

### 3.4 对象引用

```cpp
// Actor引用
UPROPERTY(Replicated)
AActor* MyActor;

// UObject引用
UPROPERTY(Replicated)
UObject* MyObject;

// 子对象引用
UPROPERTY(Replicated)
UActorComponent* MyComponent;

// 软引用
UPROPERTY(Replicated)
TSoftObjectPtr<AActor> SoftActorRef;
```

### 3.5 自定义结构体

```cpp
USTRUCT(BlueprintType)
struct FMyStruct
{
    GENERATED_BODY()

    UPROPERTY()
    int32 Value;

    UPROPERTY()
    FVector Location;

    // 需要支持网络序列化
    bool NetSerialize(FArchive& Ar, class UPackageMap* Map, bool& bOutSuccess);
};
```

---

## 四、自定义结构体序列化

### 4.1 NetSerialize 方法

```cpp
// MyStruct.h
USTRUCT(BlueprintType)
struct FMyStruct
{
    GENERATED_BODY()

    UPROPERTY()
    int32 Value;

    UPROPERTY()
    FVector Location;

    // 网络序列化方法
    bool NetSerialize(FArchive& Ar, class UPackageMap* Map, bool& bOutSuccess);
};

// MyStruct.cpp
bool FMyStruct::NetSerialize(FArchive& Ar, UPackageMap* Map, bool& bOutSuccess)
{
    Ar << Value;
    Ar << Location;

    bOutSuccess = true;
    return true;
}
```

### 4.2 优化版本 - 条件序列化

```cpp
bool FMyStruct::NetSerialize(FArchive& Ar, UPackageMap* Map, bool& bOutSuccess)
{
    // 序列化值
    Ar << Value;

    // 条件序列化位置
    bool bHasLocation = (Location != FVector::ZeroVector);
    Ar << bHasLocation;

    if (bHasLocation)
    {
        // 量化位置以减少带宽
        FVector QuantizedLocation = Location;
        if (Ar.IsSaving())
        {
            // 发送时量化
            QuantizedLocation = Location.GridSnap(0.1f);
        }
        Ar << QuantizedLocation;
        if (Ar.IsLoading())
        {
            Location = QuantizedLocation;
        }
    }

    bOutSuccess = true;
    return true;
}
```

### 4.3 结构体复制示例

```cpp
USTRUCT(BlueprintType)
struct FPlayerState
{
    GENERATED_BODY()

    UPROPERTY()
    int32 Health;

    UPROPERTY()
    int32 MaxHealth;

    UPROPERTY()
    FVector Position;

    UPROPERTY()
    FRotator Rotation;

    bool NetSerialize(FArchive& Ar, UPackageMap* Map, bool& bOutSuccess)
    {
        // 使用压缩方式序列化位置
        Ar << Health;
        Ar << MaxHealth;

        // 压缩向量 (使用量化)
        SerializePackedVector<100, 30>(Ar, Position);

        // 压缩旋转
        Rotation.SerializeCompressed(Ar);

        bOutSuccess = true;
        return true;
    }
};

// 在Actor中使用
UPROPERTY(Replicated)
FPlayerState CurrentState;
```

---

## 五、Delta序列化

### 5.1 什么是Delta序列化?

Delta序列化只发送**变化的部分**，而不是整个数据结构，用于优化大型容器和结构体的复制。

### 5.2 支持Delta序列化的类型

```cpp
// 内置支持Delta序列化
TArray<T>
TMap<K, V>

// 自定义Delta序列化
// 在结构体中实现NetDeltaSerialize
```

### 5.3 自定义Delta序列化

```cpp
// 使用FastArraySerializer优化数组复制
USTRUCT()
struct FMyArrayItem
{
    GENERATED_BODY()

    UPROPERTY()
    int32 Id;

    UPROPERTY()
    FString Name;

    bool NetDeltaSerialize(FNetDeltaSerializeInfo& DeltaParams)
    {
        // 实现Delta序列化逻辑
        return true;
    }
};

template<>
struct TStructOpsTypeTraits<FMyArrayItem> : public TStructOpsTypeTraitsBase2<FMyArrayItem>
{
    enum
    {
        WithNetDeltaSerialize = true,
    };
};
```

### 5.4 FastArraySerializer

```cpp
// 使用FFastArraySerializer进行高效的数组复制
USTRUCT(BlueprintType)
struct FMyItemArray : public FFastArraySerializer
{
    GENERATED_BODY()

    UPROPERTY()
    TArray<FMyArrayItem> Items;

    bool NetDeltaSerialize(FNetDeltaSerializeInfo& DeltaParams)
    {
        return FFastArraySerializer::FastArrayDeltaSerialize<FMyArrayItem, FMyItemArray>(
            Items, DeltaParams, *this);
    }
};

template<>
struct TStructOpsTypeTraits<FMyItemArray> : public TStructOpsTypeTraitsBase2<FMyItemArray>
{
    enum
    {
        WithNetDeltaSerialize = true,
    };
};

// 在Actor中使用
UPROPERTY(Replicated)
FMyItemArray MyItems;

// 修改数组后标记脏
void AMyActor::AddItem(const FMyArrayItem& Item)
{
    MyItems.Items.Add(Item);
    MyItems.MarkItemDirty(MyItems.Items.Last());
    MyItems.MarkArrayDirty();
}
```

---

## 六、属性复制回调

### 6.1 RepNotify详解

```cpp
// 基本形式 - 无参数
UFUNCTION()
void OnRep_PropertyName();

// 带旧值参数
UFUNCTION()
void OnRep_PropertyName(FPropertyName OldValue);
```

**RepNotify执行条件**:

```cpp
enum ERepNotifyFlags
{
    // 只在值改变时触发
    REPNOTIFY_OnChanged = 0,

    // 每次同步都触发（即使值未变）
    REPNOTIFY_Always = 1,
};
```

### 6.2 多属性RepNotify

```cpp
// 当多个属性需要一起响应时
void AMyActor::OnRep_Position()
{
    // 位置和旋转一起更新
    UpdateTransform();
}

void AMyActor::OnRep_Rotation()
{
    // 位置和旋转一起更新
    UpdateTransform();
}

// 更好的做法：使用结构体
USTRUCT(BlueprintType)
struct FTransformState
{
    GENERATED_BODY()

    UPROPERTY()
    FVector Position;

    UPROPERTY()
    FRotator Rotation;
};

UPROPERTY(ReplicatedUsing=OnRep_TransformState)
FTransformState TransformState;

UFUNCTION()
void OnRep_TransformState()
{
    SetActorLocation(TransformState.Position);
    SetActorRotation(TransformState.Rotation);
}
```

---

## 七、复制条件动态控制

### 7.1 PreReplication中使用

```cpp
void AMyActor::PreReplication(IRepChangedPropertyTracker& ChangedPropertyTracker)
{
    Super::PreReplication(ChangedPropertyTracker);

    // 动态禁用某些属性的复制
    if (bIsInvisible)
    {
        // 禁用位置复制
        DOREPLIFETIME_ACTIVE_OVERRIDE(AMyActor, Location, false);
    }

    // 根据游戏状态动态控制
    if (CurrentGameMode == EGameMode::Spectator)
    {
        DOREPLIFETIME_ACTIVE_OVERRIDE(AMyActor, bIsVisible, false);
    }
}
```

### 7.2 条件函数

```cpp
// 自定义条件判断
bool AMyActor::ShouldReplicateProperty(FLifetimeProperty& Property)
{
    // 根据属性名决定
    if (Property.RepIndex == GetReplicatedPropertyIndex("AmmoCount"))
    {
        return HasLocalNetOwner();
    }

    return true;
}
```

---

## 八、性能优化

### 8.1 减少复制频率

```cpp
AMyActor::AMyActor()
{
    // 不常变化的对象使用低频率
    NetUpdateFrequency = 5.0f;  // 每秒5次

    // 最小频率限制
    MinNetUpdateFrequency = 1.0f;  // 最少每秒1次
}
```

### 8.2 条件复制优化

```cpp
void AMyActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 使用条件减少不必要的复制
    DOREPLIFETIME_CONDITION(AMyActor, PrivateData, COND_OwnerOnly);
    DOREPLIFETIME_CONDITION(AMyActor, bCrouching, COND_SkipOwner);
    DOREPLIFETIME_CONDITION(AMyActor, InitialConfig, COND_InitialOnly);
}
```

### 8.3 使用Push Model

```cpp
// 传统方式 - 每帧比较所有属性
UPROPERTY(Replicated)
int32 Score;

// Push Model - 手动标记脏属性
UPROPERTY(ReplicatedUsing=OnRep_Score)
int32 Score;

void AMyActor::AddScore(int32 Amount)
{
    Score += Amount;
    // 标记属性脏，触发复制
    MARK_PROPERTY_DIRTY(AMyActor, Score);
}
```

---

## 九、调试技巧

### 9.1 控制台命令

```
; 显示属性复制详情
net.Replication.DebugProperty 1

; 显示复制统计
stat net

; 详细日志
LogNetPropertyTraffic Verbose
```

### 9.2 日志输出

```cpp
// 在RepNotify中添加日志
void AMyActor::OnRep_Health(float OldValue)
{
    UE_LOG(LogNet, Log, TEXT("Health changed: %f -> %f"), OldValue, Health);
}

// 追踪属性复制
#if !UE_BUILD_SHIPPING
void AMyActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    UE_LOG(LogNet, Log, TEXT("Registering replicated properties for %s"), *GetName());
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    // ...
}
#endif
```

---

## 十、完整示例

### 10.1 复杂属性复制示例

```cpp
// PlayerStateActor.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "PlayerStateActor.generated.h"

USTRUCT(BlueprintType)
struct FWeaponState
{
    GENERATED_BODY()

    UPROPERTY()
    int32 AmmoCount;

    UPROPERTY()
    int32 MaxAmmo;

    UPROPERTY()
    FString WeaponName;

    bool NetSerialize(FArchive& Ar, UPackageMap* Map, bool& bOutSuccess);
};

UCLASS()
class APlayerStateActor : public AActor
{
    GENERATED_BODY()

public:
    APlayerStateActor();

    virtual void GetLifetimeReplicatedProps(
        TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    virtual void PreReplication(IRepChangedPropertyTracker& ChangedPropertyTracker) override;

    // 基础属性
    UPROPERTY(ReplicatedUsing=OnRep_Health)
    float Health;

    UPROPERTY(Replicated)
    int32 Score;

    // 结构体属性
    UPROPERTY(ReplicatedUsing=OnRep_WeaponState)
    FWeaponState CurrentWeapon;

    // 条件属性
    UPROPERTY(Replicated)
    bool bIsInvisible;

    UPROPERTY(Replicated)
    FVector CurrentLocation;

    // 仅所有者
    UPROPERTY(Replicated)
    int32 SecretCode;

    // RepNotify
    UFUNCTION()
    void OnRep_Health(float OldValue);

    UFUNCTION()
    void OnRep_WeaponState();

    // 服务器函数
    UFUNCTION(BlueprintCallable, Category="Game")
    void TakeDamage(float Damage);

    UFUNCTION(BlueprintCallable, Category="Game")
    void AddScore(int32 Points);

    UFUNCTION(BlueprintCallable, Category="Game")
    void ReloadWeapon();
};
```

```cpp
// PlayerStateActor.cpp
#include "PlayerStateActor.h"
#include "Net/UnrealNetwork.h"

// FWeaponState NetSerialize
bool FWeaponState::NetSerialize(FArchive& Ar, UPackageMap* Map, bool& bOutSuccess)
{
    Ar << AmmoCount;
    Ar << MaxAmmo;
    Ar << WeaponName;
    bOutSuccess = true;
    return true;
}

APlayerStateActor::APlayerStateActor()
{
    bReplicates = true;
    NetUpdateFrequency = 30.0f;

    Health = 100.0f;
    Score = 0;
    bIsInvisible = false;
    SecretCode = 0;

    CurrentWeapon.AmmoCount = 30;
    CurrentWeapon.MaxAmmo = 30;
    CurrentWeapon.WeaponName = TEXT("Rifle");
}

void APlayerStateActor::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 带RepNotify
    DOREPLIFETIME_CONDITION_NOTIFY(APlayerStateActor, Health, COND_None, REPNOTIFY_OnChanged);

    // 基础复制
    DOREPLIFETIME(APlayerStateActor, Score);

    // 结构体复制
    DOREPLIFETIME_CONDITION_NOTIFY(APlayerStateActor, CurrentWeapon, COND_None, REPNOTIFY_OnChanged);

    // 条件复制
    DOREPLIFETIME(APlayerStateActor, bIsInvisible);
    DOREPLIFETIME(APlayerStateActor, CurrentLocation);

    // 仅所有者
    DOREPLIFETIME_CONDITION(APlayerStateActor, SecretCode, COND_OwnerOnly);
}

void APlayerStateActor::PreReplication(IRepChangedPropertyTracker& ChangedPropertyTracker)
{
    Super::PreReplication(ChangedPropertyTracker);

    // 隐身时不复制位置
    if (bIsInvisible)
    {
        DOREPLIFETIME_ACTIVE_OVERRIDE(APlayerStateActor, CurrentLocation, false);
    }
}

void APlayerStateActor::OnRep_Health(float OldValue)
{
    UE_LOG(LogTemp, Log, TEXT("Health: %f -> %f"), OldValue, Health);
    // 更新UI
    // 播放受伤效果
}

void APlayerStateActor::OnRep_WeaponState()
{
    UE_LOG(LogTemp, Log, TEXT("Weapon changed: %s, Ammo: %d"),
        *CurrentWeapon.WeaponName, CurrentWeapon.AmmoCount);
    // 更新武器UI
}

void APlayerStateActor::TakeDamage(float Damage)
{
    if (HasAuthority())
    {
        Health = FMath::Max(0.0f, Health - Damage);
    }
}

void APlayerStateActor::AddScore(int32 Points)
{
    if (HasAuthority())
    {
        Score += Points;
    }
}

void APlayerStateActor::ReloadWeapon()
{
    if (HasAuthority())
    {
        CurrentWeapon.AmmoCount = CurrentWeapon.MaxAmmo;
    }
}
```

---

## 十一、总结

### 11.1 核心要点

1. **FObjectReplicator**: 属性复制的核心执行者
2. **FRepLayout**: 定义属性如何序列化
3. **支持类型**: 基础类型、数学类型、容器、对象引用、自定义结构体
4. **Delta序列化**: 只发送变化部分
5. **RepNotify**: 客户端响应属性变化

### 11.2 下一课预告

**第六课：条件复制与RepNotify**

将深入讲解：
- 所有复制条件的详细用法
- RepNotify高级用法
- 条件组合技巧

---

## 参考资料

1. **源码文件**:
   - `Engine/Public/Net/DataReplication.h`
   - `Engine/Public/Net/RepLayout.h`
   - `Engine/Public/Net/UnrealNetwork.h`

---

*上一课: [第四课：Actor网络复制基础](./Lesson04_ActorReplicationBasic.md)*
*下一课: [第六课：条件复制与RepNotify](./Lesson06_ConditionalReplication.md)*
