# 第3课：RPC深入解析

> 本课深入讲解UE5的远程过程调用（RPC）机制，这是实现客户端与服务器通信的核心技术。

---

## 课程目标

- 理解Server RPC、Client RPC、Multicast的区别与用法
- 掌握RPC执行条件与可靠性保证
- 了解RPC参数限制与序列化机制
- 掌握RPC与复制系统的交互
- 学会RPC性能优化策略

---

## 一、RPC概述

### 1.1 什么是RPC

RPC（Remote Procedure Call）允许在一个端调用函数，而在另一个端执行。

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  客户端调用 Server RPC                                        │
│  ┌──────────┐                      ┌──────────┐             │
│  │ Client   │  ── Server RPC ───► │  Server  │             │
│  │          │                      │  执行函数 │             │
│  └──────────┘                      └──────────┘             │
│                                                              │
│  服务器调用 Client RPC                                        │
│  ┌──────────┐                      ┌──────────┐             │
│  │ Server   │  ── Client RPC ───► │  Client  │             │
│  │          │                      │  执行函数 │             │
│  └──────────┘                      └──────────┘             │
│                                                              │
│  服务器调用 Multicast RPC                                     │
│  ┌──────────┐        ┌──────────┐                           │
│  │ Server   │ ─────► │ Client 1 │                           │
│  │          │ ─────► │ Client 2 │                           │
│  │          │ ─────► │ Client 3 │                           │
│  └──────────┘        └──────────┘                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 RPC类型对比

| RPC类型 | 调用者 | 执行者 | 用途 |
|---------|-------|--------|------|
| **Server RPC** | 客户端 | 服务器 | 客户端请求服务器执行操作 |
| **Client RPC** | 服务器 | 特定客户端 | 服务器通知特定客户端 |
| **Multicast RPC** | 服务器 | 所有客户端 | 服务器广播给所有客户端 |

---

## 二、Server RPC详解

### 2.1 基本用法

```cpp
// MyCharacter.h
UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    // Server RPC声明
    UFUNCTION(Server, Reliable, WithValidation)
    void Server_FireWeapon(const FVector& TargetLocation);

    UFUNCTION(Server, Reliable, WithValidation)
    void Server_UseItem(int32 ItemId);

    UFUNCTION(Server, Reliable, WithValidation)
    void Server_SetPlayerName(const FString& NewName);
};

// MyCharacter.cpp
void AMyCharacter::Server_FireWeapon_Implementation(const FVector& TargetLocation)
{
    // 这个代码在服务器上执行
    if (!HasAuthority())
    {
        return; // 安全检查
    }

    // 执行射击逻辑
    FVector StartLocation = GetActorLocation();
    FVector Direction = (TargetLocation - StartLocation).GetSafeNormal();

    // 执行射线检测
    FHitResult HitResult;
    FCollisionQueryParams Params;
    Params.AddIgnoredActor(this);

    bool bHit = GetWorld()->LineTraceSingleByChannel(
        HitResult,
        StartLocation,
        TargetLocation,
        ECC_Pawn,
        Params
    );

    if (bHit && HitResult.GetActor())
    {
        // 处理命中
        ApplyDamageToTarget(HitResult.GetActor());
    }

    // 广播射击效果给所有客户端
    Multicast_PlayFireEffect(StartLocation, TargetLocation);
}

// 验证函数 - 用于防止作弊
bool AMyCharacter::Server_FireWeapon_Validate(const FVector& TargetLocation)
{
    // 检查参数是否合法
    if (!TargetLocation.IsFinite())
    {
        return false; // 无效参数，拒绝执行
    }

    // 检查射击距离是否合理
    float Distance = FVector::Dist(GetActorLocation(), TargetLocation);
    if (Distance > MaxWeaponRange)
    {
        return false; // 超出武器范围，可能是作弊
    }

    return true; // 验证通过
}

void AMyCharacter::Server_UseItem_Implementation(int32 ItemId)
{
    // 验证物品是否在背包中
    if (!Inventory.Contains(ItemId))
    {
        return; // 物品不存在
    }

    // 使用物品
    UInventoryItem* Item = Inventory[ItemId];
    if (Item && Item->CanUse(this))
    {
        Item->Use(this);

        // 从背包移除（如果是消耗品）
        if (Item->IsConsumable())
        {
            Inventory.Remove(ItemId);
        }
    }
}

bool AMyCharacter::Server_UseItem_Validate(int32 ItemId)
{
    // 简单验证
    return ItemId >= 0 && ItemId < 10000; // 合理的物品ID范围
}
```

### 2.2 Server RPC执行条件

```cpp
// Server RPC的执行条件：
// 1. 调用者必须是客户端
// 2. Actor必须被网络复制 (bReplicates = true)
// 3. Actor的LocalRole必须是AutonomousProxy或拥有Authority

void AMyCharacter::TryServerRPC()
{
    // 检查是否可以调用Server RPC
    if (GetLocalRole() < ROLE_AutonomousProxy)
    {
        UE_LOG(LogTemp, Warning, TEXT("Cannot call Server RPC: Role is %d"), GetLocalRole());
        return;
    }

    // 调用RPC
    Server_DoSomething();
}

// RPC调用者的角色要求：
// AutonomousProxy - 可以调用Server RPC（PlayerController拥有的Actor）
// SimulatedProxy - 不能调用Server RPC
// Authority - 可以调用但会在本地执行（已经是服务器）
```

### 2.3 Server RPC与所有权

```cpp
// Server RPC的所有权验证是UE5反作弊的重要机制

// 示例：只有拥有者才能调用的Server RPC
UFUNCTION(Server, Reliable, WithValidation)
void Server_ChangeInventory(int32 SlotIndex, AInventoryItem* NewItem);

bool AMyCharacter::Server_ChangeInventory_Validate(int32 SlotIndex, AInventoryItem* NewItem)
{
    // UE自动验证调用者是否拥有此Actor
    // 如果客户端不拥有此Actor，RPC会被丢弃

    // 额外验证
    if (SlotIndex < 0 || SlotIndex >= Inventory.Num())
    {
        return false;
    }

    return true;
}

void AMyCharacter::Server_ChangeInventory_Implementation(int32 SlotIndex, AInventoryItem* NewItem)
{
    // 到这里，已经确认：
    // 1. 调用者拥有此Actor
    // 2. 参数验证通过
    Inventory[SlotIndex] = NewItem;
}
```

---

## 三、Client RPC详解

### 3.1 基本用法

```cpp
// MyPlayerController.h
UCLASS()
class AMyPlayerController : public APlayerController
{
    GENERATED_BODY()

public:
    // Client RPC - 只发给特定客户端
    UFUNCTION(Client, Reliable)
    void Client_ShowNotification(const FString& Message);

    UFUNCTION(Client, Reliable)
    void Client_UpdateScore(int32 NewScore);

    UFUNCTION(Client, Unreliable)
    void Client_PlaySoundAtLocation(USoundBase* Sound, FVector Location);

private:
    UFUNCTION(Client, Reliable)
    void Client_KickPlayer(const FString& Reason);
};

// MyPlayerController.cpp
void AMyPlayerController::Client_ShowNotification_Implementation(const FString& Message)
{
    // 这个代码只在目标客户端执行
    if (IsLocalController())
    {
        // 显示通知UI
        if (UMyHUD* HUD = Cast<UMyHUD>(GetHUD()))
        {
            HUD->ShowNotification(Message);
        }
    }
}

void AMyPlayerController::Client_UpdateScore_Implementation(int32 NewScore)
{
    if (IsLocalController())
    {
        // 更新本地分数显示
        CurrentScore = NewScore;
        UpdateScoreUI();
    }
}

void AMyPlayerController::Client_PlaySoundAtLocation_Implementation(USoundBase* Sound, FVector Location)
{
    if (IsLocalController() && Sound)
    {
        UGameplayStatics::PlaySoundAtLocation(this, Sound, Location);
    }
}

void AMyPlayerController::Client_KickPlayer_Implementation(const FString& Reason)
{
    // 显示踢出原因
    UE_LOG(LogTemp, Warning, TEXT("Kicked from server: %s"), *Reason);

    // 返回主菜单
    ClientReturnToMainMenuWithTextReason(FText::FromString(Reason));
}
```

### 3.2 Client RPC执行条件

```cpp
// Client RPC执行条件：
// 1. 必须在服务器端调用
// 2. Actor必须被网络复制
// 3. 必须有有效的目标客户端（通过NetConnection确定）

// 正确的Client RPC调用方式
void AMyGameMode::NotifyPlayer(APlayerController* PC, const FString& Message)
{
    if (PC && PC->IsA<APlayerController>())
    {
        // 确保是服务器端
        if (GetNetMode() != NM_Client)
        {
            PC->Client_ShowNotification(Message);
        }
    }
}

// 常见错误示例
void AMyActor::WrongClientRPCUsage()
{
    // 错误：在客户端调用Client RPC
    // Client RPC只能在服务器端调用
    Client_ShowNotification(TEXT("Test")); // 可能不会执行
}
```

### 3.3 Client RPC与PlayerController

```cpp
// PlayerController是Client RPC最常用的Actor
// 因为每个PlayerController对应一个特定的客户端连接

void AMyGameMode::OnPlayerJoined(APlayerController* NewPlayer)
{
    // 发送欢迎消息给新玩家
    NewPlayer->Client_ShowNotification(TEXT("Welcome to the server!"));

    // 发送服务器规则
    NewPlayer->Client_ShowNotification(GetServerRules());
}

void AMyGameState::BroadcastToAllPlayers(const FString& Message)
{
    // 向所有玩家广播消息
    for (FConstPlayerControllerIterator It = GetWorld()->GetPlayerControllerIterator(); It; ++It)
    {
        APlayerController* PC = It->Get();
        if (PC)
        {
            PC->Client_ShowNotification(Message);
        }
    }
}

void AMyGameMode::KickPlayer(APlayerController* PlayerToKick, const FString& Reason)
{
    if (PlayerToKick)
    {
        PlayerToKick->Client_KickPlayer(Reason);
        // 延迟踢出，让消息先发送
        FTimerHandle KickTimer;
        GetWorld()->GetTimerManager().SetTimer(KickTimer, [this, PlayerToKick]()
        {
            PlayerToKick->Destroy();
        }, 1.0f, false);
    }
}
```

---

## 四、Multicast RPC详解

### 4.1 基本用法

```cpp
// MyCharacter.h
UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    // Multicast RPC - 发给所有客户端
    UFUNCTION(NetMulticast, Reliable)
    void Multicast_PlayDeathAnimation();

    UFUNCTION(NetMulticast, Unreliable)
    void Multicast_SpawnParticleEffect(UParticleSystem* Effect, FVector Location);

    UFUNCTION(NetMulticast, Reliable)
    void Multicast_SetTeamColor(FLinearColor TeamColor);

    UFUNCTION(NetMulticast, Reliable)
    void Multicast_ChatMessage(const FString& SenderName, const FString& Message);
};

// MyCharacter.cpp
void AMyCharacter::Multicast_PlayDeathAnimation_Implementation()
{
    // 在所有客户端执行死亡动画
    if (UAnimInstance* AnimInst = GetMesh()->GetAnimInstance())
    {
        AnimInst->Montage_Play(DeathMontage);
    }

    // 禁用碰撞
    SetActorEnableCollision(false);

    // 延迟销毁
    FTimerHandle DestroyTimer;
    GetWorld()->GetTimerManager().SetTimer(DestroyTimer, [this]()
    {
        Destroy();
    }, 5.0f, false);
}

void AMyCharacter::Multicast_SpawnParticleEffect_Implementation(UParticleSystem* Effect, FVector Location)
{
    if (Effect)
    {
        UGameplayStatics::SpawnEmitterAtLocation(GetWorld(), Effect, Location);
    }
}

void AMyCharacter::Multicast_SetTeamColor_Implementation(FLinearColor TeamColor)
{
    // 更新材质颜色
    if (UMaterialInstanceDynamic* DynMaterial = GetMesh()->CreateDynamicMaterialInstance(0))
    {
        DynMaterial->SetVectorParameterValue(FName("TeamColor"), TeamColor);
    }
}

void AMyCharacter::Multicast_ChatMessage_Implementation(const FString& SenderName, const FString& Message)
{
    // 显示聊天消息（在所有客户端）
    if (UMyHUD* HUD = Cast<UMyHUD>(GetWorld()->GetFirstPlayerController()->GetHUD()))
    {
        HUD->AddChatMessage(SenderName, Message);
    }
}
```

### 4.2 Multicast执行条件

```cpp
// Multicast RPC执行条件：
// 1. 必须在服务器端调用
// 2. Actor必须被网络复制
// 3. 会发送给所有相关连接

// Multicast的特殊行为：
// - 在服务器端调用时，也会在服务器本地执行
// - 如果Actor不在某些客户端的视野内，可能不会发送给那些客户端

void AMyCharacter::TestMulticastBehavior()
{
    if (HasAuthority())
    {
        // 服务器调用Multicast
        Multicast_TestFunction();

        // 注意：上面的调用会在服务器本地也执行
        // 这是Multicast的特性
    }
}

void AMyCharacter::Multicast_TestFunction_Implementation()
{
    // 这会在所有相关端执行
    // 包括服务器自己（如果从服务器调用）
    UE_LOG(LogTemp, Log, TEXT("Multicast executed on: %s"),
        HasAuthority() ? TEXT("Server") : TEXT("Client"));
}
```

### 4.3 Multicast vs 属性复制

```cpp
// 什么时候用Multicast，什么时候用属性复制？

// 使用属性复制：
// 1. 需要保持同步的状态
// 2. 新玩家加入时需要获取当前值
// 3. 需要OnRep回调

// 使用Multicast：
// 1. 一次性事件
// 2. 不需要保持状态
// 3. 瞬时效果（声音、粒子）

// 示例对比

// 方案A：属性复制（适合状态）
UPROPERTY(ReplicatedUsing = OnRep_Health)
float Health;

UFUNCTION()
void OnRep_Health(float OldHealth)
{
    // 新玩家加入时会收到当前值
    UpdateHealthBar();
}

// 方案B：Multicast（适合事件）
UFUNCTION(NetMulticast, Reliable)
void Multicast_OnDamageTaken(float Damage);

void AMyCharacter::TakeDamage(float Damage)
{
    Health -= Damage;

    // 广播受伤事件
    Multicast_OnDamageTaken(Damage);
}

void AMyCharacter::Multicast_OnDamageTaken_Implementation(float Damage)
{
    // 播放受伤特效（一次性事件）
    PlayDamageEffect();
    PlayDamageSound();
}
```

---

## 五、RPC可靠性

### 5.1 Reliable vs Unreliable

```cpp
// Reliable RPC - 保证送达
// - 如果丢包会重发
// - 保证执行顺序
// - 适合重要操作

// Unreliable RPC - 不保证送达
// - 可能丢失
// - 不保证顺序
// - 适合频繁更新的非关键数据

// 使用Reliable的场景
UFUNCTION(Server, Reliable, WithValidation)
void Server_PurchaseItem(int32 ItemId);  // 购买物品，必须可靠

UFUNCTION(Server, Reliable, WithValidation)
void Server_EquipWeapon(AWeapon* Weapon);  // 装备武器，必须可靠

UFUNCTION(Client, Reliable)
void Client_UpdateInventory(const TArray<FItemData>& Items);  // 库存更新，必须可靠

// 使用Unreliable的场景
UFUNCTION(NetMulticast, Unreliable)
void Multicast_UpdateVelocity(FVector Velocity);  // 速度更新，下一帧会更新

UFUNCTION(Client, Unreliable)
void Client_UpdateMovementPrediction(FVector Position);  // 移动预测，频繁更新

UFUNCTION(NetMulticast, Unreliable)
void Multicast_PlayFootstepSound();  // 脚步声，丢失也没关系
```

### 5.2 可靠性实现原理

```
Reliable RPC流程:

客户端                                    服务器
   │                                        │
   │  ─────── RPC包 (Seq=1) ─────────────►  │
   │                                        │
   │  ◄─────── ACK (Seq=1) ───────────────  │
   │                                        │
   │  ─────── RPC包 (Seq=2) ─────────────►  │
   │         (丢失)                          │
   │                                        │
   │  ◄─────── ACK (Seq=1) ───────────────  │
   │    (超时重传)                           │
   │                                        │
   │  ─────── RPC包 (Seq=2) ─────────────►  │
   │         (重发)                          │
   │                                        │
   │  ◄─────── ACK (Seq=2) ───────────────  │
   │                                        │
```

### 5.3 RPC队列管理

```cpp
// 避免RPC队列堵塞

// 错误：每帧发送大量Reliable RPC
void AMyCharacter::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    // 错误！每帧发送Reliable RPC
    Server_UpdatePosition(GetActorLocation());  // 会导致队列堵塞
}

// 正确：使用属性复制或Unreliable RPC
UPROPERTY(Replicated)
FVector ReplicatedPosition;

// 或
UFUNCTION(Server, Unreliable)
void Server_UpdatePosition(FVector Position);

// 限制RPC频率
void AMyCharacter::TrySendRPC()
{
    float CurrentTime = GetWorld()->GetTimeSeconds();
    if (CurrentTime - LastRPCTime >= MinRPCInterval)
    {
        Server_DoSomething();
        LastRPCTime = CurrentTime;
    }
}
```

---

## 六、RPC参数限制

### 6.1 参数类型支持

```cpp
// 支持的参数类型
UFUNCTION(Server, Reliable)
void Server_BasicTypes(bool bFlag, int32 IntValue, float FloatValue, FName NameValue, FString StringValue);

UFUNCTION(Server, Reliable)
void Server_VectorTypes(FVector VectorValue, FRotator RotatorValue, FTransform TransformValue);

UFUNCTION(Server, Reliable)
void Server_ObjectPointer(AActor* ActorPtr, UObject* ObjectPtr);

UFUNCTION(Server, Reliable)
void Server_ClassPointer(TSubclassOf<AActor> ActorClass);

// 支持容器类型
UFUNCTION(Server, Reliable)
void Server_Array(const TArray<int32>& IntArray);

UFUNCTION(Server, Reliable)
void Server_Struct(const FMyStruct& MyStruct);

// 不支持的参数类型
// - 指针的指针
// - 引用（const引用可以）
// - 某些复杂模板类型
```

### 6.2 参数序列化

```cpp
// 自定义结构的序列化
USTRUCT(BlueprintType)
struct FCustomData
{
    GENERATED_BODY()

    UPROPERTY()
    int32 Id;

    UPROPERTY()
    FString Name;

    UPROPERTY()
    FVector Location;

    // 自定义序列化（可选）
    bool NetSerialize(FArchive& Ar, class UPackageMap* Map, bool& bOutSuccess)
    {
        Ar << Id;
        Ar << Name;

        // 优化Vector序列化
        FVector_NetQuantize QuantizedLocation = Location;
        Ar << QuantizedLocation;
        Location = QuantizedLocation;

        bOutSuccess = true;
        return true;
    }
};

// 使用自定义NetSerialize
template<>
struct TStructOpsTypeTraits<FCustomData> : public TStructOpsTypeTraitsBase2<FCustomData>
{
    enum
    {
        WithNetSerializer = true,
    };
};
```

### 6.3 参数大小优化

```cpp
// 优化RPC参数大小

// 不好：传输完整对象
UFUNCTION(Server, Reliable)
void Server_EquipWeapon(AWeapon* Weapon);  // 只传输对象引用

// 更好：传输ID
UFUNCTION(Server, Reliable)
void Server_EquipWeaponById(int32 WeaponId);

// 优化：使用量化类型
UFUNCTION(Server, Reliable)
void Server_SetTargetLocation(const FVector_NetQuantize& Location);  // 压缩位置

UFUNCTION(Server, Reliable)
void Server_SetTargetRotation(const FRotator_NetQuantize& Rotation);  // 压缩旋转

// 布尔参数优化
UFUNCTION(Server, Reliable)
void Server_SetFlags(bool bFlag1, bool bFlag2, bool bFlag3);  // 每个bool占1字节

// 更好：使用位掩码
UFUNCTION(Server, Reliable)
void Server_SetFlags(uint8 Flags);  // 所有bool压缩到1字节
```

---

## 七、RPC与复制系统交互

### 7.1 RPC触发属性更新

```cpp
// 常见模式：RPC触发状态改变，属性复制同步状态

UCLASS()
class AMyWeapon : public AActor
{
    GENERATED_BODY()

public:
    // 属性
    UPROPERTY(ReplicatedUsing = OnRep_Ammo)
    int32 CurrentAmmo;

    UPROPERTY(ReplicatedUsing = OnRep_ReloadState)
    bool bIsReloading;

    // RPC
    UFUNCTION(Server, Reliable)
    void Server_Reload();

    UFUNCTION(Server, Reliable, WithValidation)
    void Server_Fire();

    // Multicast
    UFUNCTION(NetMulticast, Unreliable)
    void Multicast_PlayFireEffects();

private:
    UFUNCTION()
    void OnRep_Ammo(int32 OldAmmo);

    UFUNCTION()
    void OnRep_ReloadState();
};

void AMyWeapon::Server_Reload_Implementation()
{
    if (bIsReloading || CurrentAmmo >= MaxAmmo)
    {
        return;
    }

    bIsReloading = true;  // 属性复制会同步给客户端

    // 开始装弹计时器
    FTimerHandle ReloadTimer;
    GetWorld()->GetTimerManager().SetTimer(ReloadTimer, [this]()
    {
        CurrentAmmo = MaxAmmo;  // 属性复制会同步
        bIsReloading = false;   // 属性复制会同步
    }, ReloadTime, false);
}

bool AMyWeapon::Server_Fire_Validate()
{
    return CurrentAmmo > 0;
}

void AMyWeapon::Server_Fire_Implementation()
{
    CurrentAmmo--;  // 属性复制会同步

    // 执行射击逻辑
    PerformFire();

    // 广播射击效果（一次性事件，用Multicast）
    Multicast_PlayFireEffects();
}

void AMyWeapon::OnRep_Ammo(int32 OldAmmo)
{
    // 客户端响应弹药变化
    UpdateAmmoUI();
}
```

### 7.2 组合使用RPC和属性复制

```cpp
// 设计模式：状态+事件

UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    // === 状态属性（通过属性复制同步）===
    UPROPERTY(ReplicatedUsing = OnRep_Health)
    float Health;

    UPROPERTY(ReplicatedUsing = OnRep_IsDead)
    bool bIsDead;

    UPROPERTY(Replicated)
    int32 TeamId;

    // === 事件RPC（通过RPC广播）===

    // 受伤事件
    UFUNCTION(NetMulticast, Reliable)
    void Multicast_OnDamaged(float Damage, AActor* DamageCauser);

    // 死亡事件
    UFUNCTION(NetMulticast, Reliable)
    void Multicast_OnDeath(AActor* Killer);

    // 复活事件
    UFUNCTION(NetMulticast, Reliable)
    void Multicast_OnRespawn();

    // === 请求RPC（客户端请求服务器）===

    UFUNCTION(Server, Reliable, WithValidation)
    void Server_RequestRespawn();

private:
    UFUNCTION()
    void OnRep_Health(float OldHealth);

    UFUNCTION()
    void OnRep_IsDead();
};

// 实现
void AMyCharacter::TakeDamage(float Damage, AActor* DamageCauser)
{
    if (bIsDead)
    {
        return;
    }

    Health = FMath::Clamp(Health - Damage, 0.0f, MaxHealth);

    // 广播受伤事件
    Multicast_OnDamaged(Damage, DamageCauser);

    if (Health <= 0.0f)
    {
        bIsDead = true;
        Multicast_OnDeath(DamageCauser);
    }
}

void AMyCharacter::Multicast_OnDamaged_Implementation(float Damage, AActor* DamageCauser)
{
    // 在所有客户端播放受伤效果
    PlayDamageVFX();
    PlayDamageSFX();
}

void AMyCharacter::Server_RequestRespawn_Implementation()
{
    if (bIsDead)
    {
        RespawnPlayer();
    }
}

bool AMyCharacter::Server_RequestRespawn_Validate()
{
    return true; // 简单验证
}

void AMyCharacter::RespawnPlayer()
{
    Health = MaxHealth;
    bIsDead = false;

    // 广播复活事件
    Multicast_OnRespawn();

    // 移动到出生点
    SetActorLocation(GetSpawnPoint());
}

void AMyCharacter::OnRep_Health(float OldHealth)
{
    // 客户端更新血条
    UpdateHealthBar();
}
```

---

## 八、RPC性能优化

### 8.1 减少RPC调用频率

```cpp
// 使用节流（Throttling）限制RPC频率

UCLASS()
class AMyCharacter : public ACharacter
{
    // RPC节流变量
    float LastRPCSendTime = 0.0f;
    float MinRPCInterval = 0.1f; // 最小间隔100ms

    // 累积待发送的数据
    FVector PendingPosition;
    bool bHasPendingUpdate = false;

public:
    void UpdateMovement(FVector NewPosition)
    {
        PendingPosition = NewPosition;
        bHasPendingUpdate = true;

        float CurrentTime = GetWorld()->GetTimeSeconds();
        if (CurrentTime - LastRPCSendTime >= MinRPCInterval)
        {
            FlushPendingRPC();
        }
    }

    void FlushPendingRPC()
    {
        if (bHasPendingUpdate)
        {
            Server_UpdatePosition(PendingPosition);
            bHasPendingUpdate = false;
            LastRPCSendTime = GetWorld()->GetTimeSeconds();
        }
    }

    UFUNCTION(Server, Unreliable)
    void Server_UpdatePosition(FVector Position);
};
```

### 8.2 合并RPC调用

```cpp
// 将多个小RPC合并成一个

// 不好：多次RPC
void AMyCharacter::UpdateStats()
{
    Server_SetHealth(Health);
    Server_SetStamina(Stamina);
    Server_SetMana(Mana);
}

// 更好：一次RPC
USTRUCT()
struct FPlayerStats
{
    GENERATED_BODY()

    UPROPERTY()
    float Health;

    UPROPERTY()
    float Stamina;

    UPROPERTY()
    float Mana;
};

UFUNCTION(Server, Reliable)
void Server_UpdateStats(const FPlayerStats& Stats);

void AMyCharacter::UpdateStats()
{
    FPlayerStats Stats;
    Stats.Health = Health;
    Stats.Stamina = Stamina;
    Stats.Mana = Mana;

    Server_UpdateStats(Stats);
}
```

### 8.3 RPC带宽分析

```cpp
// 使用网络分析工具

// 控制台命令
// stat net - 显示网络统计
// net.Profile - 网络性能分析

// 代码中获取统计信息
void AMyGameInstance::LogRPCStats()
{
    UNetDriver* NetDriver = GetWorld()->GetNetDriver();
    if (NetDriver)
    {
        // 获取RPC统计
        int32 TotalRPCs = 0;
        int32 TotalBytes = 0;

        // 统计逻辑...

        UE_LOG(LogTemp, Log, TEXT("RPC Stats: %d calls, %d bytes"), TotalRPCs, TotalBytes);
    }
}

// 优化建议
// 1. 将Reliable RPC转为Unreliable（如果可以接受丢失）
// 2. 将频繁RPC转为属性复制
// 3. 减少RPC参数大小
// 4. 使用事件聚合
```

### 8.4 最佳实践总结

```cpp
// RPC最佳实践清单

// 1. 使用正确的RPC类型
// - 客户端->服务器：Server RPC
// - 服务器->特定客户端：Client RPC
// - 服务器->所有客户端：Multicast RPC

// 2. 合理选择可靠性
// - 重要操作：Reliable
// - 频繁更新：Unreliable

// 3. 使用WithValidation保护Server RPC
UFUNCTION(Server, Reliable, WithValidation)
void Server_ImportantAction(int32 Value);

// 4. 避免在Tick中调用RPC
// 使用定时器或事件驱动

// 5. 最小化参数
// 使用ID而非完整对象，使用量化类型

// 6. 合理使用属性复制替代RPC
// 状态用属性，事件用RPC

// 7. 注意RPC队列
// 避免发送大量Reliable RPC导致堵塞
```

---

## 九、实践任务

### 任务：实现完整的武器系统RPC

```cpp
// MyWeapon.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyWeapon.generated.h"

// 武器状态
UENUM(BlueprintType)
enum class EWeaponState : uint8
{
    Idle,
    Firing,
    Reloading,
    Disabled
};

// 武器数据
USTRUCT(BlueprintType)
struct FWeaponData
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 MaxAmmo = 30;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 CurrentAmmo = 30;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float Damage = 10.0f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float FireRate = 0.1f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float ReloadTime = 2.0f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float Range = 10000.0f;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnAmmoChangedDelegate, int32, OldAmmo, int32, NewAmmo);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnWeaponStateChangedDelegate, EWeaponState, NewState);

UCLASS()
class AMyWeapon : public AActor
{
    GENERATED_BODY()

public:
    AMyWeapon();

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    // 状态属性
    UPROPERTY(ReplicatedUsing = OnRep_WeaponData, BlueprintReadOnly, Category = "Weapon")
    FWeaponData WeaponData;

    UPROPERTY(ReplicatedUsing = OnRep_WeaponState, BlueprintReadOnly, Category = "Weapon")
    EWeaponState CurrentState;

    // 委托
    UPROPERTY(BlueprintAssignable, Category = "Weapon")
    FOnAmmoChangedDelegate OnAmmoChanged;

    UPROPERTY(BlueprintAssignable, Category = "Weapon")
    FOnWeaponStateChangedDelegate OnWeaponStateChanged;

    // RPC函数
    UFUNCTION(Server, Reliable, WithValidation)
    void Server_RequestFire(const FVector_NetQuantize& TargetLocation);

    UFUNCTION(Server, Reliable, WithValidation)
    void Server_RequestReload();

    UFUNCTION(Server, Reliable, WithValidation)
    void Server_RequestStopFire();

    // Multicast函数
    UFUNCTION(NetMulticast, Unreliable)
    void Multicast_PlayFireEffects(const FVector_NetQuantize& HitLocation);

    UFUNCTION(NetMulticast, Reliable)
    void Multicast_PlayReloadEffects();

    UFUNCTION(NetMulticast, Reliable)
    void Multicast_OnWeaponEmpty();

    // Client函数
    UFUNCTION(Client, Reliable)
    void Client_NotifyHit(AActor* HitActor, float Damage);

    // 公共函数
    UFUNCTION(BlueprintCallable, Category = "Weapon")
    bool CanFire() const;

    UFUNCTION(BlueprintCallable, Category = "Weapon")
    bool CanReload() const;

    UFUNCTION(BlueprintCallable, Category = "Weapon")
    float GetAmmoPercentage() const;

protected:
    virtual void BeginPlay() override;

    UFUNCTION()
    void OnRep_WeaponData(const FWeaponData& OldData);

    UFUNCTION()
    void OnRep_WeaponState(EWeaponState OldState);

private:
    FTimerHandle FireTimerHandle;
    FTimerHandle ReloadTimerHandle;

    void SetWeaponState(EWeaponState NewState);
    void PerformFire(const FVector& TargetLocation);
    void CompleteReload();
    void HandleFire(const FVector_NetQuantize& TargetLocation);
};

// MyWeapon.cpp
#include "MyWeapon.h"
#include "Net/UnrealNetwork.h"
#include "Kismet/GameplayStatics.h"

AMyWeapon::AMyWeapon()
{
    PrimaryActorTick.bCanEverTick = false;
    bReplicates = true;

    CurrentState = EWeaponState::Idle;
}

void AMyWeapon::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(AMyWeapon, WeaponData);
    DOREPLIFETIME(AMyWeapon, CurrentState);
}

void AMyWeapon::BeginPlay()
{
    Super::BeginPlay();
}

bool AMyWeapon::CanFire() const
{
    return CurrentState == EWeaponState::Idle &&
           WeaponData.CurrentAmmo > 0;
}

bool AMyWeapon::CanReload() const
{
    return CurrentState == EWeaponState::Idle &&
           WeaponData.CurrentAmmo < WeaponData.MaxAmmo;
}

float AMyWeapon::GetAmmoPercentage() const
{
    return static_cast<float>(WeaponData.CurrentAmmo) / static_cast<float>(WeaponData.MaxAmmo);
}

// ========== Server RPC实现 ==========

bool AMyWeapon::Server_RequestFire_Validate(const FVector_NetQuantize& TargetLocation)
{
    // 验证目标位置在合理范围内
    float Distance = FVector::Dist(GetActorLocation(), TargetLocation);
    return Distance <= WeaponData.Range * 1.5f; // 允许一定误差
}

void AMyWeapon::Server_RequestFire_Implementation(const FVector_NetQuantize& TargetLocation)
{
    if (!CanFire())
    {
        return;
    }

    HandleFire(TargetLocation);
}

bool AMyWeapon::Server_RequestReload_Validate()
{
    return true;
}

void AMyWeapon::Server_RequestReload_Implementation()
{
    if (!CanReload())
    {
        return;
    }

    SetWeaponState(EWeaponState::Reloading);

    // 设置装弹计时器
    GetWorld()->GetTimerManager().SetTimer(
        ReloadTimerHandle,
        this,
        &AMyWeapon::CompleteReload,
        WeaponData.ReloadTime,
        false
    );

    // 广播装弹效果
    Multicast_PlayReloadEffects();
}

bool AMyWeapon::Server_RequestStopFire_Validate()
{
    return true;
}

void AMyWeapon::Server_RequestStopFire_Implementation()
{
    if (CurrentState == EWeaponState::Firing)
    {
        GetWorld()->GetTimerManager().ClearTimer(FireTimerHandle);
        SetWeaponState(EWeaponState::Idle);
    }
}

// ========== 内部函数 ==========

void AMyWeapon::HandleFire(const FVector_NetQuantize& TargetLocation)
{
    if (!HasAuthority())
    {
        return;
    }

    SetWeaponState(EWeaponState::Firing);
    PerformFire(TargetLocation);

    // 设置射击间隔计时器
    GetWorld()->GetTimerManager().SetTimer(
        FireTimerHandle,
        [this, TargetLocation]()
        {
            if (WeaponData.CurrentAmmo > 0)
            {
                PerformFire(TargetLocation);
            }
            else
            {
                SetWeaponState(EWeaponState::Idle);
                Multicast_OnWeaponEmpty();
            }
        },
        WeaponData.FireRate,
        true
    );
}

void AMyWeapon::PerformFire(const FVector& TargetLocation)
{
    // 减少弹药
    int32 OldAmmo = WeaponData.CurrentAmmo;
    WeaponData.CurrentAmmo--;

    // 执行射线检测
    FVector StartLocation = GetActorLocation();
    FVector Direction = (TargetLocation - StartLocation).GetSafeNormal();
    FVector EndLocation = StartLocation + Direction * WeaponData.Range;

    FHitResult HitResult;
    FCollisionQueryParams Params;
    Params.AddIgnoredActor(GetOwner());

    bool bHit = GetWorld()->LineTraceSingleByChannel(
        HitResult,
        StartLocation,
        EndLocation,
        ECC_Pawn,
        Params
    );

    FVector HitLocation = bHit ? HitResult.Location : EndLocation;

    // 广播射击效果
    Multicast_PlayFireEffects(HitLocation);

    // 如果命中，通知客户端
    if (bHit && HitResult.GetActor())
    {
        Client_NotifyHit(HitResult.GetActor(), WeaponData.Damage);
    }

    // 触发弹药变化委托
    OnAmmoChanged.Broadcast(OldAmmo, WeaponData.CurrentAmmo);
}

void AMyWeapon::CompleteReload()
{
    WeaponData.CurrentAmmo = WeaponData.MaxAmmo;
    SetWeaponState(EWeaponState::Idle);
}

void AMyWeapon::SetWeaponState(EWeaponState NewState)
{
    EWeaponState OldState = CurrentState;
    CurrentState = NewState;
    OnWeaponStateChanged.Broadcast(NewState);
}

// ========== Multicast实现 ==========

void AMyWeapon::Multicast_PlayFireEffects_Implementation(const FVector_NetQuantize& HitLocation)
{
    // 在所有客户端播放射击特效
    if (FireEffect)
    {
        UGameplayStatics::SpawnEmitterAtLocation(
            GetWorld(),
            FireEffect,
            GetActorLocation()
        );
    }

    if (FireSound)
    {
        UGameplayStatics::PlaySoundAtLocation(
            this,
            FireSound,
            GetActorLocation()
        );
    }
}

void AMyWeapon::Multicast_PlayReloadEffects_Implementation()
{
    // 播放装弹动画和音效
    if (ReloadSound)
    {
        UGameplayStatics::PlaySoundAtLocation(
            this,
            ReloadSound,
            GetActorLocation()
        );
    }
}

void AMyWeapon::Multicast_OnWeaponEmpty_Implementation()
{
    // 播放弹药耗尽提示
    UE_LOG(LogTemp, Log, TEXT("Weapon is empty!"));
}

// ========== Client实现 ==========

void AMyWeapon::Client_NotifyHit_Implementation(AActor* HitActor, float Damage)
{
    // 在客户端显示命中反馈
    if (HitActor)
    {
        UE_LOG(LogTemp, Log, TEXT("Hit %s for %.1f damage"), *HitActor->GetName(), Damage);
    }
}

// ========== OnRep实现 ==========

void AMyWeapon::OnRep_WeaponData(const FWeaponData& OldData)
{
    if (WeaponData.CurrentAmmo != OldData.CurrentAmmo)
    {
        OnAmmoChanged.Broadcast(OldData.CurrentAmmo, WeaponData.CurrentAmmo);
    }
}

void AMyWeapon::OnRep_WeaponState(EWeaponState OldState)
{
    OnWeaponStateChanged.Broadcast(CurrentState);
}
```

---

## 十、总结

本课我们学习了：

1. **RPC类型**：Server RPC、Client RPC、Multicast RPC的区别与用法
2. **执行条件**：理解不同RPC的调用限制
3. **可靠性**：Reliable与Unreliable的选择策略
4. **参数限制**：支持的类型与序列化机制
5. **性能优化**：减少调用频率、合并RPC、带宽优化

---

## 十一、下节预告

**第4课：变量复制与回调**

将深入学习：
- OnRep回调函数机制
- 条件复制详解
- 复杂数据类型复制
- 最佳实践与优化

---

*课程版本：1.0*
*最后更新：2026-04-09*