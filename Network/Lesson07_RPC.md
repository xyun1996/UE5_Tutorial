# 第七课：RPC远程过程调用

> **学习目标**: 深入理解RPC机制和使用规范
> **时长**: 约60分钟
> **前置知识**: 第六课内容

---

## 一、RPC概述

### 1.1 什么是RPC?

RPC (Remote Procedure Call) 是**在远程机器上执行函数**的机制。允许代码像调用本地函数一样调用远程函数。

```
┌─────────────────────────────────────────────────────────────────────┐
│                      RPC调用示意图                                   │
└─────────────────────────────────────────────────────────────────────┘

客户端                                服务器
  │                                     │
  │  调用 ServerFire()                  │
  │    ↓                                │
  │  参数序列化                          │
  │    ↓                                │
  │  ─────── 网络传输 ─────────────────→│
  │                                     │  执行 ServerFire_Implementation()
  │                                     │
  │                                     │  调用 ClientShowDamage()
  │                                     │    ↓
  │                                     │  参数序列化
  │                                     │    ↓
  │ ◄─────── 网络传输 ──────────────────│
  │                                     │
  │  执行 ClientShowDamage_Implementation()
  │                                     │
```

### 1.2 RPC类型

```cpp
// 服务器RPC - 客户端调用，服务器执行
UFUNCTION(Server, Reliable)
void ServerFire();

// 客户端RPC - 服务器调用，特定客户端执行
UFUNCTION(Client, Reliable)
void ClientShowDamage(float Damage);

// 多播RPC - 服务器调用，所有客户端执行
UFUNCTION(NetMulticast, Unreliable)
void MulticastPlayEffect(FVector Location);
```

---

## 二、RPC类型详解

### 2.1 Server RPC

**用途**: 客户端请求服务器执行操作

```cpp
// 声明
UFUNCTION(Server, Reliable, WithValidation)
void ServerFire();

UFUNCTION(Server, Reliable, WithValidation)
void ServerMoveTo(FVector TargetLocation);

// 实现
void AMyCharacter::ServerFire_Implementation()
{
    // 在服务器上执行
    if (CurrentWeapon)
    {
        CurrentWeapon->Fire();
    }
}

bool AMyCharacter::ServerFire_Validate()
{
    // 验证参数合法性
    return true;
}
```

**执行规则**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Server RPC 执行规则                               │
└─────────────────────────────────────────────────────────────────────┘

调用位置          │ 执行结果
─────────────────┼────────────────────
客户端           │ 发送到服务器执行
服务器           │ 直接本地执行
无NetOwner       │ 被忽略
```

### 2.2 Client RPC

**用途**: 服务器通知特定客户端

```cpp
// 声明
UFUNCTION(Client, Reliable)
void ClientShowScore(int32 NewScore);

UFUNCTION(Client, Unreliable)
void ClientUpdatePosition(FVector Location);

// 实现
void AMyPlayerController::ClientShowScore_Implementation(int32 NewScore)
{
    // 在客户端上执行
    UpdateScoreUI(NewScore);
}
```

**执行规则**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Client RPC 执行规则                               │
└─────────────────────────────────────────────────────────────────────┘

调用位置          │ 执行结果
─────────────────┼────────────────────
服务器           │ 发送到目标客户端执行
客户端           │ 被忽略
```

### 2.3 NetMulticast RPC

**用途**: 服务器通知所有客户端

```cpp
// 声明
UFUNCTION(NetMulticast, Unreliable)
void MulticastPlayExplosion(FVector Location);

UFUNCTION(NetMulticast, Reliable)
void MulticastGameStart();

// 实现
void AMyActor::MulticastPlayExplosion_Implementation(FVector Location)
{
    // 在所有客户端上执行
    UGameplayStatics::SpawnEmitterAtLocation(
        GetWorld(),
        ExplosionEffect,
        Location
    );
}
```

**执行规则**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                  NetMulticast RPC 执行规则                           │
└─────────────────────────────────────────────────────────────────────┘

调用位置          │ 执行结果
─────────────────┼────────────────────
服务器           │ 发送到所有客户端执行
客户端           │ 被忽略
```

---

## 三、RPC标识符

### 3.1 可靠性标识符

```cpp
// 可靠RPC - 保证送达，有顺序保证
UFUNCTION(Server, Reliable)
void ServerSetHealth(float NewHealth);

// 不可靠RPC - 不保证送达，可能丢失
UFUNCTION(NetMulticast, Unreliable)
void MulticastPlaySound(FVector Location);
```

**选择指南**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    可靠性选择指南                                    │
└─────────────────────────────────────────────────────────────────────┘

Reliable (可靠):
    - 必须送达的数据
    - 游戏状态改变
    - 重要操作
    - 例：购买物品、技能释放、状态同步

Unreliable (不可靠):
    - 可以丢失的数据
    - 高频更新
    - 视觉/音效
    - 例：粒子效果、位置预测、语音数据
```

### 3.2 WithValidation标识符

```cpp
// 服务器RPC必须有WithValidation
UFUNCTION(Server, Reliable, WithValidation)
void ServerPurchaseItem(int32 ItemId);

// 验证函数
bool AMyCharacter::ServerPurchaseItem_Validate(int32 ItemId)
{
    // 检查参数合法性
    if (ItemId < 0 || ItemId >= MaxItems)
    {
        return false;  // 拒绝执行，可能踢出玩家
    }

    return true;  // 允许执行
}

// 实现函数
void AMyCharacter::ServerPurchaseItem_Implementation(int32 ItemId)
{
    // 实际购买逻辑
    // ...
}
```

### 3.3 其他标识符

```cpp
// BlueprintCallable - 蓝图可调用
UFUNCTION(Server, Reliable, WithValidation, BlueprintCallable)
void ServerRespawn();

// BlueprintNativeEvent - 蓝图可重写
UFUNCTION(NetMulticast, Unreliable, BlueprintNativeEvent)
void MulticastOnDeath();

// Sealed - 不可在子类重写
UFUNCTION(Server, Reliable, WithValidation, Sealed)
void ServerSetTeam(int32 TeamId);
```

---

## 四、RPC执行条件

### 4.1 NetOwner要求

RPC需要Actor有有效的NetOwner才能正确执行：

```cpp
// Server RPC 需要条件
// - Actor必须有NetOwner
// - 调用者必须是Owner或有权限

bool AActor::HasNetOwner() const
{
    // 检查Owner链是否包含PlayerController
    return GetNetConnection() != nullptr;
}

bool AActor::HasLocalNetOwner() const
{
    // 检查是否有本地控制的Owner
    // ...
}
```

### 4.2 执行规则详解

```
┌─────────────────────────────────────────────────────────────────────┐
│                    RPC完整执行规则                                   │
└─────────────────────────────────────────────────────────────────────┘

Server RPC:
┌────────────────────┬──────────────────────────────────────┐
│     调用位置        │              执行结果                 │
├────────────────────┼──────────────────────────────────────┤
│ 客户端 + 有Owner    │ 发送到服务器执行                      │
│ 客户端 + 无Owner    │ 被忽略，不执行                        │
│ 服务器             │ 直接本地执行                          │
│ 无权限             │ 被忽略                               │
└────────────────────┴──────────────────────────────────────┘

Client RPC:
┌────────────────────┬──────────────────────────────────────┐
│     调用位置        │              执行结果                 │
├────────────────────┼──────────────────────────────────────┤
│ 服务器             │ 发送到目标客户端执行                   │
│ 客户端             │ 被忽略                               │
└────────────────────┴──────────────────────────────────────┘

Multicast RPC:
┌────────────────────┬──────────────────────────────────────┐
│     调用位置        │              执行结果                 │
├────────────────────┼──────────────────────────────────────┤
│ 服务器             │ 发送到所有相关客户端执行               │
│ 客户端             │ 被忽略                               │
└────────────────────┴──────────────────────────────────────┘
```

---

## 五、RPC参数传递

### 5.1 支持的参数类型

```cpp
// 基础类型
UFUNCTION(Server, Reliable, WithValidation)
void ServerSetValues(int32 IntValue, float FloatValue, bool BoolValue);

// 字符串
UFUNCTION(Server, Reliable, WithValidation)
void ServerSetName(const FString& Name);

// 向量和旋转
UFUNCTION(Server, Reliable, WithValidation)
void ServerMoveTo(FVector Location, FRotator Rotation);

// 枚举
UFUNCTION(Server, Reliable, WithValidation)
void ServerSetState(EPlayerState NewState);

// 对象引用
UFUNCTION(Server, Reliable, WithValidation)
void ServerEquipWeapon(AWeapon* Weapon);

// 结构体
UFUNCTION(Server, Reliable, WithValidation)
void ServerUpdateStats(const FPlayerStats& Stats);

// 数组
UFUNCTION(Server, Reliable, WithValidation)
void ServerSetInventory(const TArray<FItemData>& Items);
```

### 5.2 参数限制

```cpp
// 不要使用过多参数
// 不好
UFUNCTION(Server, Reliable, WithValidation)
void ServerUpdateAll(int32 A, int32 B, int32 C, int32 D, int32 E, ...);

// 使用结构体
// 好
USTRUCT()
struct FUpdateData
{
    GENERATED_BODY()

    UPROPERTY()
    int32 A, B, C, D, E;
};

UFUNCTION(Server, Reliable, WithValidation)
void ServerUpdateData(const FUpdateData& Data);
```

### 5.3 参数验证

```cpp
// 验证所有输入参数
UFUNCTION(Server, Reliable, WithValidation)
void ServerMove(FVector Location);

bool AMyCharacter::ServerMove_Validate(FVector Location)
{
    // 检查位置是否合理
    if (!FMath::IsFinite(Location.X) ||
        !FMath::IsFinite(Location.Y) ||
        !FMath::IsFinite(Location.Z))
    {
        return false;  // 无效位置，拒绝
    }

    // 检查移动距离是否合理
    FVector LastLocation = GetActorLocation();
    float Distance = FVector::Dist(LastLocation, Location);
    if (Distance > MaxMoveDistance)
    {
        return false;  // 移动距离过大，可能是作弊
    }

    return true;
}
```

---

## 六、RPC最佳实践

### 6.1 安全验证

```cpp
// 服务器RPC必须验证
UFUNCTION(Server, Reliable, WithValidation)
void ServerPurchaseItem(int32 ItemId, int32 Quantity);

bool AMyCharacter::ServerPurchaseItem_Validate(int32 ItemId, int32 Quantity)
{
    // 基本参数验证
    if (ItemId < 0 || ItemId >= MaxItems)
    {
        return false;
    }

    if (Quantity <= 0 || Quantity > 100)
    {
        return false;
    }

    // 业务逻辑验证
    AShopItem* Item = Shop->GetItem(ItemId);
    if (!Item || !Item->IsAvailable())
    {
        return false;
    }

    // 资源验证
    if (PlayerGold < Item->Price * Quantity)
    {
        return false;
    }

    return true;
}

void AMyCharacter::ServerPurchaseItem_Implementation(int32 ItemId, int32 Quantity)
{
    // 安全执行购买逻辑
    Shop->PurchaseItem(this, ItemId, Quantity);
}
```

### 6.2 带宽优化

```cpp
// 使用不可靠RPC减少带宽
UFUNCTION(NetMulticast, Unreliable)
void MulticastPlayFootstepSound(FVector Location);

// 批量发送
USTRUCT()
struct FMovementBatch
{
    GENERATED_BODY()

    UPROPERTY()
    TArray<FVector> Locations;

    UPROPERTY()
    TArray<FRotator> Rotations;
};

UFUNCTION(Server, Reliable, WithValidation)
void ServerSendMovementBatch(const FMovementBatch& Batch);
```

### 6.3 RPC与属性复制配合

```cpp
// 使用属性复制同步状态，RPC触发事件
UPROPERTY(ReplicatedUsing=OnRep_Health)
float Health;

UFUNCTION()
void OnRep_Health(float OldValue)
{
    // 属性变化时更新UI
    UpdateHealthUI();
}

UFUNCTION(Client, Reliable)
void ClientTakeDamage(float Damage, FVector DamageLocation);

void AMyCharacter::ClientTakeDamage_Implementation(float Damage, FVector DamageLocation)
{
    // 显示受伤效果（RPC触发）
    PlayDamageEffect(DamageLocation);

    // Health属性会通过复制同步
}
```

---

## 七、完整示例

### 7.1 武器系统RPC

```cpp
// MyWeapon.h
UCLASS()
class AMyWeapon : public AActor
{
    GENERATED_BODY()

public:
    // 开火
    UFUNCTION(Server, Reliable, WithValidation)
    void ServerFire(FVector TargetLocation);

    // 换弹
    UFUNCTION(Server, Reliable, WithValidation)
    void ServerReload();

    // 通知所有客户端开火效果
    UFUNCTION(NetMulticast, Unreliable)
    void MulticastPlayFireEffect(FVector Location, FRotator Rotation);

    // 通知所有者弹药变化
    UFUNCTION(Client, Reliable)
    void ClientUpdateAmmo(int32 NewAmmo);

protected:
    UPROPERTY(Replicated)
    int32 CurrentAmmo;

    UPROPERTY(Replicated)
    int32 MaxAmmo;

    UPROPERTY(ReplicatedUsing=OnReloading)
    bool bIsReloading;

    UFUNCTION()
    void OnReloading() { /* 更新UI */ }
};
```

```cpp
// MyWeapon.cpp
void AMyWeapon::ServerFire_Validate(FVector TargetLocation)
{
    // 验证位置有效
    return TargetLocation.IsValid();
}

void AMyWeapon::ServerFire_Implementation(FVector TargetLocation)
{
    if (bIsReloading || CurrentAmmo <= 0)
    {
        return;
    }

    CurrentAmmo--;

    // 执行开火逻辑
    PerformFire(TargetLocation);

    // 通知所有客户端播放效果
    MulticastPlayFireEffect(GetActorLocation(), GetActorRotation());

    // 通知所有者弹药更新
    ClientUpdateAmmo(CurrentAmmo);
}

void AMyWeapon::ServerReload_Validate()
{
    return true;
}

void AMyWeapon::ServerReload_Implementation()
{
    if (bIsReloading || CurrentAmmo >= MaxAmmo)
    {
        return;
    }

    bIsReloading = true;

    // 设置定时器完成换弹
    FTimerHandle ReloadTimer;
    GetWorldTimerManager().SetTimer(
        ReloadTimer,
        [this]()
        {
            CurrentAmmo = MaxAmmo;
            bIsReloading = false;
            ClientUpdateAmmo(CurrentAmmo);
        },
        ReloadTime,
        false
    );
}

void AMyWeapon::MulticastPlayFireEffect_Implementation(FVector Location, FRotator Rotation)
{
    // 所有客户端播放效果
    UGameplayStatics::SpawnEmitterAtLocation(
        GetWorld(),
        MuzzleFlash,
        Location,
        Rotation
    );

    UGameplayStatics::PlaySoundAtLocation(
        GetWorld(),
        FireSound,
        Location
    );
}

void AMyWeapon::ClientUpdateAmmo_Implementation(int32 NewAmmo)
{
    // 客户端更新弹药UI
    UpdateAmmoUI(NewAmmo);
}
```

### 7.2 玩家移动RPC

```cpp
// MyCharacter.h
UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    // 移动请求
    UFUNCTION(Server, Reliable, WithValidation)
    void ServerMove(FVector Location, FRotator Rotation, float Timestamp);

    // 快速位置校正
    UFUNCTION(Server, Unreliable, WithValidation)
    void ServerMoveFast(FVector Location, FRotator Rotation);

    // 跳跃
    UFUNCTION(Server, Reliable, WithValidation)
    void ServerJump();

    // 下蹲
    UFUNCTION(Server, Reliable, WithValidation)
    void ServerSetCrouching(bool bCrouch);

protected:
    UPROPERTY(Replicated)
    FVector ReplicatedLocation;

    UPROPERTY(Replicated)
    FRotator ReplicatedRotation;

    UPROPERTY(ReplicatedUsing=OnCrouching)
    bool bIsCrouching;
};
```

```cpp
// MyCharacter.cpp
void AMyCharacter::ServerMove_Validate(FVector Location, FRotator Rotation, float Timestamp)
{
    // 基本验证
    if (!Location.IsValid() || !Rotation.IsValid())
    {
        return false;
    }

    // 反作弊验证
    float Distance = FVector::Dist(GetActorLocation(), Location);
    float MaxDistance = MaxSpeed * (GetWorld()->GetTimeSeconds() - Timestamp);
    if (Distance > MaxDistance * 1.5f)
    {
        return false;  // 移动过快
    }

    return true;
}

void AMyCharacter::ServerMove_Implementation(FVector Location, FRotator Rotation, float Timestamp)
{
    // 设置位置
    SetActorLocation(Location);
    SetActorRotation(Rotation);

    ReplicatedLocation = Location;
    ReplicatedRotation = Rotation;
}

void AMyCharacter::ServerJump_Validate()
{
    return true;
}

void AMyCharacter::ServerJump_Implementation()
{
    Jump();
}

void AMyCharacter::ServerSetCrouching_Validate(bool bCrouch)
{
    return true;
}

void AMyCharacter::ServerSetCrouching_Implementation(bool bCrouch)
{
    if (bCrouch)
    {
        Crouch();
    }
    else
    {
        UnCrouch();
    }
    bIsCrouching = bCrouch;
}
```

---

## 八、调试与故障排除

### 8.1 调试命令

```
; 显示RPC统计
stat net

; RPC日志
LogNetRpc Verbose

; 追踪特定RPC
net.DumpRpc
```

### 8.2 常见问题

| 问题 | 原因 | 解决方案 |
|-----|------|---------|
| Server RPC不执行 | 无NetOwner | 确保Actor有Owner |
| Client RPC不执行 | 调用位置错误 | 只在服务器调用 |
| Multicast只在服务器执行 | 调用位置错误 | 只在服务器调用 |
| RPC丢失 | 使用Unreliable | 对重要数据使用Reliable |
| 验证失败返回false | 参数验证失败 | 检查Validate逻辑 |

### 8.3 RPC日志

```cpp
void AMyCharacter::ServerFire_Implementation()
{
    UE_LOG(LogNet, Log, TEXT("ServerFire called by %s"), *GetName());

    // ...

    UE_LOG(LogNet, Log, TEXT("ServerFire completed, Ammo: %d"), CurrentAmmo);
}
```

---

## 九、总结

### 9.1 核心要点

1. **Server RPC**: 客户端调用，服务器执行
2. **Client RPC**: 服务器调用，特定客户端执行
3. **Multicast RPC**: 服务器调用，所有客户端执行
4. **可靠性选择**: 重要操作用Reliable，效果用Unreliable
5. **安全验证**: 必须验证所有参数

### 9.2 下一课预告

**第八课：对象引用与NetGUID系统**

将深入讲解：
- NetGUID原理
- 对象引用序列化
- 动态对象导出/导入

---

*上一课: [第六课：条件复制与RepNotify](./Lesson06_ConditionalReplication.md)*
*下一课: [第八课：对象引用与NetGUID系统](./Lesson08_NetGUID.md)*
