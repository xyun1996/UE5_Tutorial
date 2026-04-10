# 第5课：网络同步策略

> 本课深入讲解UE5的网络同步策略，包括状态同步、客户端预测、服务器权威和延迟补偿等核心技术。

---

## 课程目标

- 理解状态同步与输入同步的区别
- 掌握客户端预测的实现原理
- 学会设计服务器权威架构
- 了解网络延迟补偿技术
- 掌握插值与平滑处理技巧

---

## 一、网络同步模式概述

### 1.1 同步模式对比

```
┌─────────────────────────────────────────────────────────────┐
│                      状态同步模式                            │
│                                                              │
│  服务器                                                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  真实状态                                            │    │
│  │  Position: (100, 50, 200)                           │    │
│  │  Velocity: (5, 0, 3)                                │    │
│  │  Health: 80                                         │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│                          │ 同步状态                          │
│                          ▼                                   │
│  客户端                                                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  复制状态                                            │    │
│  │  Position: (100, 50, 200)                           │    │
│  │  Velocity: (5, 0, 3)                                │    │
│  │  Health: 80                                         │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                      输入同步模式                            │
│                                                              │
│  客户端                                                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  输入                                                │    │
│  │  MoveForward: 1.0                                    │    │
│  │  MoveRight: 0.5                                      │    │
│  │  Jump: true                                          │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│                          │ 发送输入                          │
│                          ▼                                   │
│  服务器                                                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  执行输入 → 计算结果                                 │    │
│  │  结果状态: Position, Velocity, etc.                  │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│                          │ 返回结果                          │
│                          ▼                                   │
│  客户端                                                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  应用服务器结果                                      │    │
│  │  可能需要回滚本地预测                                │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 选择同步策略

| 场景 | 推荐同步策略 | 原因 |
|------|------------|------|
| 玩家移动 | 输入同步+预测 | 需要即时响应 |
| AI行为 | 状态同步 | 服务器控制 |
| 物理对象 | 状态同步+物理预测 | 复杂计算在服务器 |
| 门/开关 | 状态同步 | 简单可靠 |
| 射击命中 | 输入同步+服务器验证 | 需要防作弊 |
| 动画状态 | 状态同步 | 视觉一致性 |

---

## 二、状态同步详解

### 2.1 基本状态同步

```cpp
// 基本状态同步示例
UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    // 同步的状态属性
    UPROPERTY(Replicated)
    FVector ReplicatedLocation;

    UPROPERTY(Replicated)
    FRotator ReplicatedRotation;

    UPROPERTY(ReplicatedUsing = OnRep_Health)
    float Health;

    virtual void Tick(float DeltaTime) override;

protected:
    UFUNCTION()
    void OnRep_Health(float OldHealth);
};

void AMyCharacter::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    if (HasAuthority())
    {
        // 服务器更新状态
        ReplicatedLocation = GetActorLocation();
        ReplicatedRotation = GetActorRotation();
    }
    else
    {
        // 客户端应用服务器状态
        SetActorLocation(ReplicatedLocation);
        SetActorRotation(ReplicatedRotation);
    }
}
```

### 2.2 平滑插值

```cpp
// 平滑插值处理网络抖动
UCLASS()
class AMyCharacter : public ACharacter
{
    // 插值参数
    float InterpolationSpeed = 10.0f;
    FVector TargetLocation;
    FRotator TargetRotation;

public:
    virtual void Tick(float DeltaTime) override;

private:
    void InterpolateToTarget(float DeltaTime);
};

void AMyCharacter::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    if (HasAuthority())
    {
        // 服务器：更新目标位置
        ReplicatedLocation = GetActorLocation();
    }
    else
    {
        // 客户端：平滑插值到目标位置
        TargetLocation = ReplicatedLocation;
        InterpolateToTarget(DeltaTime);
    }
}

void AMyCharacter::InterpolateToTarget(float DeltaTime)
{
    // 位置插值
    FVector CurrentLocation = GetActorLocation();
    FVector NewLocation = FMath::VInterpTo(
        CurrentLocation,
        TargetLocation,
        DeltaTime,
        InterpolationSpeed
    );
    SetActorLocation(NewLocation);

    // 旋转插值
    FRotator CurrentRotation = GetActorRotation();
    FRotator NewRotation = FMath::RInterpTo(
        CurrentRotation,
        TargetRotation,
        DeltaTime,
        InterpolationSpeed
    );
    SetActorRotation(NewRotation);
}
```

### 2.3 外推预测

```cpp
// 当网络更新延迟时，使用外推预测

UCLASS()
class AMyActor : public AActor
{
    GENERATED_BODY()

    // 速度信息（用于外推）
    UPROPERTY(Replicated)
    FVector ReplicatedVelocity;

    // 上次更新时间
    float LastReplicationTime;

    // 是否使用外推
    bool bUseExtrapolation = true;
    float MaxExtrapolationTime = 0.5f; // 最大外推时间

public:
    virtual void Tick(float DeltaTime) override;

private:
    void ApplyExtrapolation(float DeltaTime);
};

void AMyActor::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    if (!HasAuthority())
    {
        float TimeSinceReplication = GetWorld()->GetTimeSeconds() - LastReplicationTime;

        if (bUseExtrapolation && TimeSinceReplication < MaxExtrapolationTime)
        {
            // 使用外推
            ApplyExtrapolation(DeltaTime);
        }
    }
}

void AMyActor::ApplyExtrapolation(float DeltaTime)
{
    // 基于速度预测下一帧位置
    FVector PredictedLocation = GetActorLocation() + ReplicatedVelocity * DeltaTime;

    // 可以添加更多预测逻辑（如考虑重力）
    // PredictedLocation += FVector(0, 0, -980) * DeltaTime * DeltaTime * 0.5f;

    SetActorLocation(PredictedLocation);
}

// 在收到状态更新时
void AMyActor::OnRep_ReplicatedLocation()
{
    LastReplicationTime = GetWorld()->GetTimeSeconds();

    // 可以选择平滑过渡而不是直接跳转
    TargetLocation = ReplicatedLocation;
}
```

---

## 三、客户端预测

### 3.1 预测原理

```
客户端预测流程:

客户端                                    服务器
   │                                        │
   │  1. 玩家输入                           │
   │     ▼                                  │
   │  2. 本地预测执行                        │
   │     │                                  │
   │     ├── 保存输入到缓冲区                │
   │     ├── 应用移动                        │
   │     └── 显示预测结果                    │
   │                                        │
   │  3. 发送输入到服务器                    │
   │     ─────────────────────────────────►  │
   │                                        │
   │                               4. 服务器执行
   │                                  并计算真实状态
   │                                        │
   │  ◄─────────────────────────────────    │
   │  5. 接收服务器状态                      │
   │     │                                  │
   │     ▼                                  │
   │  6. 与预测对比                          │
   │     ├── 匹配？继续                      │
   │     └── 不匹配？回滚重放                │
   │                                        │
```

### 3.2 实现客户端预测

```cpp
// MyPredictedCharacter.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Character.h"
#include "MyPredictedCharacter.generated.h"

// 保存的输入状态
USTRUCT()
struct FSavedMove
{
    GENERATED_BODY()

    float Timestamp;
    FVector InputVector;
    FRotator Rotation;
    bool bJumped;

    // 预测的结果
    FVector PredictedLocation;
    FRotator PredictedRotation;
};

UCLASS()
class AMyPredictedCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    AMyPredictedCharacter();

    virtual void SetupPlayerInputComponent(class UInputComponent* PlayerInputComponent) override;
    virtual void Tick(float DeltaTime) override;
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

protected:
    // 输入处理
    void MoveForward(float Value);
    void MoveRight(float Value);
    void Jump();

    // 服务器RPC
    UFUNCTION(Server, Reliable)
    void Server_SendMove(const FSavedMove& Move);

    UFUNCTION(Client, Reliable)
    void Client_UpdatePosition(const FVector& ServerPosition);

    // 复制属性
    UPROPERTY(ReplicatedUsing = OnRep_ServerPosition)
    FVector ServerPosition;

    UPROPERTY(Replicated)
    FRotator ServerRotation;

    UFUNCTION()
    void OnRep_ServerPosition();

private:
    // 输入缓冲
    TArray<FSavedMove> MoveBuffer;
    int32 MoveIndex = 0;

    // 预测设置
    float PredictionErrorThreshold = 10.0f; // 位置误差阈值
    float ReconciliationSpeed = 10.0f;      // 和解速度

    // 当前输入
    FVector CurrentInput;
    bool bWantsToJump;

    // 预测函数
    void ApplyPrediction(float DeltaTime);
    void ReconcileWithServer();
    void ReplayMoves(const FVector& FromPosition);
};

// MyPredictedCharacter.cpp
#include "MyPredictedCharacter.h"
#include "Net/UnrealNetwork.h"

AMyPredictedCharacter::AMyPredictedCharacter()
{
    PrimaryActorTick.bCanEverTick = true;
    bReplicates = true;
}

void AMyPredictedCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);

    PlayerInputComponent->BindAxis("MoveForward", this, &AMyPredictedCharacter::MoveForward);
    PlayerInputComponent->BindAxis("MoveRight", this, &AMyPredictedCharacter::MoveRight);
    PlayerInputComponent->BindAction("Jump", IE_Pressed, this, &AMyPredictedCharacter::Jump);
}

void AMyPredictedCharacter::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(AMyPredictedCharacter, ServerPosition);
    DOREPLIFETIME(AMyPredictedCharacter, ServerRotation);
}

void AMyPredictedCharacter::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    if (IsLocallyControlled())
    {
        // 客户端：执行预测
        ApplyPrediction(DeltaTime);
    }
    else
    {
        // 非本地控制：使用插值
        // 由标准复制处理
    }
}

// ========== 输入处理 ==========

void AMyPredictedCharacter::MoveForward(float Value)
{
    CurrentInput.X = Value;
}

void AMyPredictedCharacter::MoveRight(float Value)
{
    CurrentInput.Y = Value;
}

void AMyPredictedCharacter::Jump()
{
    bWantsToJump = true;
}

// ========== 预测执行 ==========

void AMyPredictedCharacter::ApplyPrediction(float DeltaTime)
{
    // 保存当前状态
    FSavedMove NewMove;
    NewMove.Timestamp = GetWorld()->GetTimeSeconds();
    NewMove.InputVector = CurrentInput;
    NewMove.Rotation = GetActorRotation();
    NewMove.bJumped = bWantsToJump;
    NewMove.PredictedLocation = GetActorLocation();
    NewMove.PredictedRotation = GetActorRotation();

    // 执行本地预测移动
    FVector MovementDirection = GetActorRotation().RotateVector(CurrentInput);
    AddMovementInput(MovementDirection, 1.0f);

    if (bWantsToJump && CanJump())
    {
        bPressedJump = true;
    }

    // 保存预测结果
    NewMove.PredictedLocation = GetActorLocation();
    NewMove.PredictedRotation = GetActorRotation();

    // 添加到缓冲区
    MoveBuffer.Add(NewMove);
    MoveIndex++;

    // 发送到服务器
    Server_SendMove(NewMove);

    // 重置输入
    bWantsToJump = false;
}

// ========== 服务器处理 ==========

void AMyPredictedCharacter::Server_SendMove_Implementation(const FSavedMove& Move)
{
    // 验证移动合法性
    if (Move.InputVector.Size() > 1.0f)
    {
        // 可能是作弊
        return;
    }

    // 保存旧位置用于验证
    FVector OldLocation = GetActorLocation();

    // 在服务器执行移动
    FVector MovementDirection = GetActorRotation().RotateVector(Move.InputVector);
    AddMovementInput(MovementDirection, 1.0f);

    if (Move.bJumped && CanJump())
    {
        bPressedJump = true;
    }

    // 更新服务器状态
    ServerPosition = GetActorLocation();
    ServerRotation = GetActorRotation();

    // 发送确认给客户端
    // 客户端将通过OnRep收到更新
}

// ========== 客户端和解 ==========

void AMyPredictedCharacter::OnRep_ServerPosition()
{
    if (!IsLocallyControlled())
    {
        // 非本地控制的角色直接更新
        SetActorLocation(ServerPosition);
        return;
    }

    // 本地控制：进行和解
    ReconcileWithServer();
}

void AMyPredictedCharacter::ReconcileWithServer()
{
    // 计算预测误差
    FVector PredictedLocation = GetActorLocation();
    float Error = FVector::Dist(PredictedLocation, ServerPosition);

    if (Error > PredictionErrorThreshold)
    {
        UE_LOG(LogTemp, Warning, TEXT("Prediction error: %.2f"), Error);

        // 服务器为准，回滚到服务器位置
        SetActorLocation(ServerPosition);
        SetActorRotation(ServerRotation);

        // 重放所有未确认的移动
        ReplayMoves(ServerPosition);
    }
    else
    {
        // 小误差，平滑过渡
        FVector NewLocation = FMath::VInterpTo(
            PredictedLocation,
            ServerPosition,
            GetWorld()->GetDeltaSeconds(),
            ReconciliationSpeed
        );
        SetActorLocation(NewLocation);
    }
}

void AMyPredictedCharacter::ReplayMoves(const FVector& FromPosition)
{
    // 移除已确认的旧移动
    while (MoveBuffer.Num() > 0)
    {
        const FSavedMove& Move = MoveBuffer[0];

        // 这里应该有服务器确认机制来确定哪些移动已处理
        // 简化版本：假设所有移动都需要重放

        // 执行重放
        SetActorLocation(Move.PredictedLocation);
        SetActorRotation(Move.PredictedRotation);

        MoveBuffer.RemoveAt(0);
    }
}

void AMyPredictedCharacter::Client_UpdatePosition_Implementation(const FVector& InServerPosition)
{
    ServerPosition = InServerPosition;
    ReconcileWithServer();
}
```

### 3.3 UE5内置预测系统

```cpp
// UE5的角色移动组件已经实现了完整的预测系统
// 推荐直接使用或继承UCharacterMovementComponent

// 标准设置
AMyCharacter::AMyCharacter()
{
    // 启用网络移动预测
    GetCharacterMovement()->NetworkSmoothingMode = ENetworkSmoothingMode::Exponential;

    // 预测设置
    GetCharacterMovement()->NetworkMaxSmoothUpdateDistance = 256.0f;
    GetCharacterMovement()->NetworkNoSmoothUpdateDistance = 384.0f;
}

// 自定义移动模式
UCLASS()
class UMyMovementComponent : public UCharacterMovementComponent
{
    GENERATED_BODY()

public:
    // 重写预测相关函数
    virtual void TickComponent(float DeltaTime,
                               enum ELevelTick TickType,
                               FActorComponentTickFunction* ThisTickFunction) override;

    virtual void PerformMovement(float DeltaTime) override;

protected:
    // 保存移动状态
    virtual void FNetworkPredictionData_Client_Character* AllocatePredictionData(
        const FNetworkPredictionData_Client_Character& Source) const override;

    // 应用保存的移动
    virtual void PerformSavedMove(const FSavedMove& Move) override;

    // 验证移动
    virtual bool VerifyMove(const FSavedMove& Move) override;
};
```

---

## 四、服务器权威设计

### 4.1 权威模式架构

```
┌─────────────────────────────────────────────────────────────┐
│                      服务器权威架构                          │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                     服务器                           │    │
│  │                                                      │    │
│  │  ┌─────────────┐    ┌─────────────┐                │    │
│  │  │  GameMode   │    │  GameState  │                │    │
│  │  │ (游戏规则)   │    │ (游戏状态)   │                │    │
│  │  └──────┬──────┘    └──────┬──────┘                │    │
│  │         │                  │                        │    │
│  │         └────────┬─────────┘                        │    │
│  │                  ▼                                  │    │
│  │  ┌───────────────────────────────────────────────┐ │    │
│  │  │              游戏逻辑层                        │ │    │
│  │  │  - 伤害计算                                   │ │    │
│  │  │  - 物品验证                                   │ │    │
│  │  │  - 规则判定                                   │ │    │
│  │  │  - AI决策                                    │ │    │
│  │  └───────────────────────────────────────────────┘ │    │
│  │                  │                                  │    │
│  │                  ▼                                  │    │
│  │  ┌───────────────────────────────────────────────┐ │    │
│  │  │              状态存储                          │ │    │
│  │  │  - Actor状态                                  │ │    │
│  │  │  - 世界状态                                   │ │    │
│  │  └───────────────────────────────────────────────┘ │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│                          │ 只同步结果                        │
│                          ▼                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                     客户端                          │    │
│  │                                                      │    │
│  │  ┌─────────────┐    ┌─────────────┐                │    │
│  │  │PlayerController│  │  UI/HUD    │                │    │
│  │  │ (输入/预测)   │    │ (显示)     │                │    │
│  │  └─────────────┘    └─────────────┘                │    │
│  │                                                      │    │
│  │  - 接收输入                                          │    │
│  │  - 本地预测（乐观）                                   │    │
│  │  - 渲染和音效                                        │    │
│  │  - UI更新                                           │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 服务器权威实现

```cpp
// 伤害系统的服务器权威设计

UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    // 属性（服务器权威）
    UPROPERTY(ReplicatedUsing = OnRep_Health)
    float Health;

    UPROPERTY(Replicated)
    float MaxHealth;

    UPROPERTY(Replicated)
    float Shield;

    // 客户端请求攻击
    UFUNCTION(Server, Reliable, WithValidation)
    void Server_RequestAttack(AActor* Target);

    // 客户端请求使用物品
    UFUNCTION(Server, Reliable, WithValidation)
    void Server_RequestUseItem(int32 ItemIndex);

    // 客户端请求治疗
    UFUNCTION(Server, Reliable, WithValidation)
    void Server_RequestHeal(float Amount);

    // 服务器广播伤害结果
    UFUNCTION(NetMulticast, Reliable)
    void Multicast_OnDamageReceived(float Damage, AActor* DamageCauser);

    UFUNCTION(NetMulticast, Reliable)
    void Multicast_OnDeath(AActor* Killer);

protected:
    UFUNCTION()
    void OnRep_Health(float OldHealth);

private:
    // 服务器端伤害计算
    float CalculateDamage(AActor* Target, float BaseDamage);
    void ApplyDamage(AActor* Target, float Damage);
    bool ValidateAttack(AActor* Target);
};

// 实现
void AMyCharacter::Server_RequestAttack_Implementation(AActor* Target)
{
    if (!ValidateAttack(Target))
    {
        UE_LOG(LogTemp, Warning, TEXT("Invalid attack request"));
        return;
    }

    // 服务器计算伤害
    float BaseDamage = GetWeaponDamage();
    float FinalDamage = CalculateDamage(Target, BaseDamage);

    // 应用伤害
    ApplyDamage(Target, FinalDamage);
}

bool AMyCharacter::Server_RequestAttack_Validate(AActor* Target)
{
    // 验证攻击合法性
    if (!Target)
    {
        return false;
    }

    // 检查目标是否可攻击
    if (!Target->CanBeDamaged())
    {
        return false;
    }

    // 检查距离
    float Distance = FVector::Dist(GetActorLocation(), Target->GetActorLocation());
    if (Distance > GetAttackRange())
    {
        return false;
    }

    return true;
}

float AMyCharacter::CalculateDamage(AActor* Target, float BaseDamage)
{
    float FinalDamage = BaseDamage;

    // 应用护甲减伤
    if (AMyCharacter* TargetCharacter = Cast<AMyCharacter>(Target))
    {
        float ArmorReduction = TargetCharacter->Shield * 0.5f;
        FinalDamage *= (1.0f - ArmorReduction);
    }

    // 应用暴击
    if (FMath::Rand() < CritChance)
    {
        FinalDamage *= CritMultiplier;
    }

    return FinalDamage;
}

void AMyCharacter::ApplyDamage(AActor* Target, float Damage)
{
    if (AMyCharacter* TargetCharacter = Cast<AMyCharacter>(Target))
    {
        float OldHealth = TargetCharacter->Health;
        TargetCharacter->Health = FMath::Clamp(
            TargetCharacter->Health - Damage,
            0.0f,
            TargetCharacter->MaxHealth
        );

        // 广播伤害事件
        TargetCharacter->Multicast_OnDamageReceived(Damage, this);

        // 检查死亡
        if (TargetCharacter->Health <= 0.0f)
        {
            TargetCharacter->Multicast_OnDeath(this);
        }
    }
}
```

### 4.3 验证与防作弊

```cpp
// 服务器端验证系统

UCLASS()
class AMyGameMode : public AGameModeBase
{
    GENERATED_BODY()

public:
    // 验证玩家位置
    bool ValidatePlayerPosition(APlayerController* Player, const FVector& ReportedPosition);

    // 验证射击
    bool ValidateShot(APlayerController* Player, const FVector& Start, const FVector& End);

    // 验证移动速度
    bool ValidateMovementSpeed(APlayerController* Player, float ReportedSpeed);

    // 验证物品获取
    bool ValidateItemPickup(APlayerController* Player, AActor* Item);

private:
    // 验证参数
    float MaxPositionError = 100.0f;
    float MaxSpeedMultiplier = 1.2f;
    float MaxPickupDistance = 500.0f;
};

bool AMyGameMode::ValidatePlayerPosition(APlayerController* Player, const FVector& ReportedPosition)
{
    if (!Player || !Player->GetPawn())
    {
        return false;
    }

    FVector ServerPosition = Player->GetPawn()->GetActorLocation();
    float Error = FVector::Dist(ServerPosition, ReportedPosition);

    if (Error > MaxPositionError)
    {
        // 位置误差过大，可能是加速作弊
        UE_LOG(LogTemp, Warning, TEXT("Player %s position error: %.2f"),
            *Player->GetName(), Error);

        // 可以选择踢出玩家或修正位置
        return false;
    }

    return true;
}

bool AMyGameMode::ValidateShot(APlayerController* Player, const FVector& Start, const FVector& End)
{
    if (!Player || !Player->GetPawn())
    {
        return false;
    }

    FVector ServerPosition = Player->GetPawn()->GetActorLocation();

    // 验证射击起点是否合理
    float DistanceToStart = FVector::Dist(ServerPosition, Start);
    if (DistanceToStart > 100.0f)
    {
        UE_LOG(LogTemp, Warning, TEXT("Shot validation failed: distance to start %.2f"), DistanceToStart);
        return false;
    }

    // 服务器执行射线检测验证命中
    FHitResult HitResult;
    FCollisionQueryParams Params;
    Params.AddIgnoredActor(Player->GetPawn());

    bool bHit = GetWorld()->LineTraceSingleByChannel(
        HitResult,
        Start,
        End,
        ECC_Pawn,
        Params
    );

    return bHit;
}

bool AMyGameMode::ValidateItemPickup(APlayerController* Player, AActor* Item)
{
    if (!Player || !Player->GetPawn() || !Item)
    {
        return false;
    }

    FVector PlayerPosition = Player->GetPawn()->GetActorLocation();
    FVector ItemPosition = Item->GetActorLocation();

    float Distance = FVector::Dist(PlayerPosition, ItemPosition);

    if (Distance > MaxPickupDistance)
    {
        UE_LOG(LogTemp, Warning, TEXT("Invalid item pickup: distance %.2f"), Distance);
        return false;
    }

    return true;
}
```

---

## 五、延迟补偿

### 5.1 延迟补偿原理

```
延迟补偿射击示例:

客户端 (延迟100ms)                    服务器
     │                                 │
     │ T=0: 玩家看到敌人位置            │
     │     敌人在位置A                  │
     │                                 │
     │ T=0: 发射子弹                    │
     │     ─────────────────────────►   │
     │                                 │ T=100ms: 服务器收到
     │                                 │   敌人已在位置B
     │                                 │
     │                                 │ 延迟补偿：
     │                                 │   1. 保存敌人历史位置
     │                                 │   2. 回滚到T=0时的位置A
     │                                 │   3. 执行射线检测
     │                                 │   4. 确认命中
     │                                 │   5. 恢复到位置B
     │                                 │
     │ ◄─────────────────────────────  │
     │ T=200ms: 收到命中确认            │
     │                                 │
```

### 5.2 实现延迟补偿系统

```cpp
// LagCompensationSystem.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "LagCompensationSystem.generated.h"

// 历史位置记录
USTRUCT()
struct FPositionRecord
{
    GENERATED_BODY()

    float Timestamp;
    FVector Position;
    FRotator Rotation;
    FVector Velocity;
};

// 可回滚的Actor
UCLASS()
class ALagCompensatableActor : public AActor
{
    GENERATED_BODY()

public:
    ALagCompensatableActor();

    virtual void Tick(float DeltaTime) override;

    // 获取历史位置
    bool GetPositionAtTime(float Time, FVector& OutPosition, FRotator& OutRotation);

    // 回滚到指定时间
    void RollbackToTime(float Time);

    // 恢复当前位置
    void RestoreCurrentPosition();

private:
    // 历史记录
    TArray<FPositionRecord> PositionHistory;
    float HistoryDuration = 1.0f; // 保存1秒的历史

    // 当前位置备份（用于恢复）
    FVector BackupPosition;
    FRotator BackupRotation;
};

// 延迟补偿管理器
UCLASS()
class ULagCompensationManager : public UObject
{
    GENERATED_BODY()

public:
    static ULagCompensationManager* Get(UWorld* World);

    // 注册Actor
    void RegisterActor(ALagCompensatableActor* Actor);
    void UnregisterActor(ALagCompensatableActor* Actor);

    // 执行延迟补偿射线检测
    bool LagCompensatedLineTrace(
        const FVector& Start,
        const FVector& End,
        float ClientTimestamp,
        FHitResult& OutHit,
        AActor* IgnoreActor = nullptr
    );

    // 延迟补偿检测（带恢复）
    bool PerformLagCompensatedCheck(
        float ClientTimestamp,
        TFunctionRef<bool()> CheckFunction
    );

private:
    UPROPERTY()
    TArray<ALagCompensatableActor*> RegisteredActors;
};

// LagCompensationSystem.cpp
#include "LagCompensationSystem.h"

ALagCompensatableActor::ALagCompensatableActor()
{
    PrimaryActorTick.bCanEverTick = true;
    bReplicates = true;
}

void ALagCompensatableActor::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    if (HasAuthority())
    {
        // 记录当前位置
        FPositionRecord Record;
        Record.Timestamp = GetWorld()->GetTimeSeconds();
        Record.Position = GetActorLocation();
        Record.Rotation = GetActorRotation();
        Record.Velocity = GetVelocity();

        PositionHistory.Add(Record);

        // 清理过旧的记录
        float CurrentTime = GetWorld()->GetTimeSeconds();
        while (PositionHistory.Num() > 0 &&
               CurrentTime - PositionHistory[0].Timestamp > HistoryDuration)
        {
            PositionHistory.RemoveAt(0);
        }
    }
}

bool ALagCompensatableActor::GetPositionAtTime(float Time, FVector& OutPosition, FRotator& OutRotation)
{
    // 二分查找最近的记录
    for (int32 i = PositionHistory.Num() - 1; i >= 0; i--)
    {
        if (PositionHistory[i].Timestamp <= Time)
        {
            if (i < PositionHistory.Num() - 1)
            {
                // 插值计算精确位置
                float Alpha = (Time - PositionHistory[i].Timestamp) /
                             (PositionHistory[i + 1].Timestamp - PositionHistory[i].Timestamp);

                OutPosition = FMath::Lerp(
                    PositionHistory[i].Position,
                    PositionHistory[i + 1].Position,
                    Alpha
                );
                OutRotation = FMath::Lerp(
                    PositionHistory[i].Rotation,
                    PositionHistory[i + 1].Rotation,
                    Alpha
                );
            }
            else
            {
                OutPosition = PositionHistory[i].Position;
                OutRotation = PositionHistory[i].Rotation;
            }
            return true;
        }
    }

    return false;
}

void ALagCompensatableActor::RollbackToTime(float Time)
{
    // 备份当前位置
    BackupPosition = GetActorLocation();
    BackupRotation = GetActorRotation();

    // 回滚到历史位置
    FVector HistoricalPosition;
    FRotator HistoricalRotation;

    if (GetPositionAtTime(Time, HistoricalPosition, HistoricalRotation))
    {
        SetActorLocation(HistoricalPosition);
        SetActorRotation(HistoricalRotation);
    }
}

void ALagCompensatableActor::RestoreCurrentPosition()
{
    SetActorLocation(BackupPosition);
    SetActorRotation(BackupRotation);
}

// 延迟补偿管理器实现
ULagCompensationManager* ULagCompensationManager::Get(UWorld* World)
{
    // 通常作为子系统或存储在GameMode中
    // 简化实现
    static TMap<UWorld*, ULagCompensationManager*> Managers;
    if (!Managers.Contains(World))
    {
        Managers.Add(World, NewObject<ULagCompensationManager>());
    }
    return Managers[World];
}

void ULagCompensationManager::RegisterActor(ALagCompensatableActor* Actor)
{
    if (!RegisteredActors.Contains(Actor))
    {
        RegisteredActors.Add(Actor);
    }
}

void ULagCompensationManager::UnregisterActor(ALagCompensatableActor* Actor)
{
    RegisteredActors.Remove(Actor);
}

bool ULagCompensationManager::LagCompensatedLineTrace(
    const FVector& Start,
    const FVector& End,
    float ClientTimestamp,
    FHitResult& OutHit,
    AActor* IgnoreActor)
{
    // 1. 回滚所有Actor到客户端时间
    for (ALagCompensatableActor* Actor : RegisteredActors)
    {
        if (Actor != IgnoreActor)
        {
            Actor->RollbackToTime(ClientTimestamp);
        }
    }

    // 2. 执行射线检测
    FCollisionQueryParams Params;
    Params.AddIgnoredActor(IgnoreActor);

    bool bHit = GetWorld()->LineTraceSingleByChannel(
        OutHit,
        Start,
        End,
        ECC_Pawn,
        Params
    );

    // 3. 恢复所有Actor位置
    for (ALagCompensatableActor* Actor : RegisteredActors)
    {
        if (Actor != IgnoreActor)
        {
            Actor->RestoreCurrentPosition();
        }
    }

    return bHit;
}

bool ULagCompensationManager::PerformLagCompensatedCheck(
    float ClientTimestamp,
    TFunctionRef<bool()> CheckFunction)
{
    // 回滚
    for (ALagCompensatableActor* Actor : RegisteredActors)
    {
        Actor->RollbackToTime(ClientTimestamp);
    }

    // 执行检查
    bool bResult = CheckFunction();

    // 恢复
    for (ALagCompensatableActor* Actor : RegisteredActors)
    {
        Actor->RestoreCurrentPosition();
    }

    return bResult;
}
```

### 5.3 在射击系统中应用延迟补偿

```cpp
// MyWeaponWithLagCompensation.h
UCLASS()
class AMyWeaponWithLagCompensation : public AActor
{
    GENERATED_BODY()

public:
    UFUNCTION(Server, Reliable, WithValidation)
    void Server_FireWithTimestamp(
        const FVector_NetQuantize& Start,
        const FVector_NetQuantize& End,
        float ClientTimestamp
    );

private:
    bool PerformLagCompensatedShot(
        const FVector& Start,
        const FVector& End,
        float ClientTimestamp,
        FHitResult& OutHit
    );
};

// 实现
void AMyWeaponWithLagCompensation::Server_FireWithTimestamp_Implementation(
    const FVector_NetQuantize& Start,
    const FVector_NetQuantize& End,
    float ClientTimestamp)
{
    // 计算服务器对应的时间
    float ServerTime = GetWorld()->GetTimeSeconds();
    UNetConnection* Connection = GetOwner()->GetNetConnection();
    if (Connection)
    {
        // 使用客户端和服务器的时钟差
        float ClockDelta = Connection->GetAvgLag();
        ClientTimestamp += ClockDelta; // 调整到服务器时间线
    }

    FHitResult HitResult;
    if (PerformLagCompensatedShot(Start, End, ClientTimestamp, HitResult))
    {
        // 处理命中
        if (HitResult.GetActor())
        {
            ApplyDamageToActor(HitResult.GetActor(), BaseDamage);
        }
    }
}

bool AMyWeaponWithLagCompensation::PerformLagCompensatedShot(
    const FVector& Start,
    const FVector& End,
    float ClientTimestamp,
    FHitResult& OutHit)
{
    ULagCompensationManager* Manager = ULagCompensationManager::Get(GetWorld());
    if (Manager)
    {
        return Manager->LagCompensatedLineTrace(
            Start,
            End,
            ClientTimestamp,
            OutHit,
            GetOwner()
        );
    }

    // 回退到普通射线检测
    FCollisionQueryParams Params;
    Params.AddIgnoredActor(GetOwner());
    return GetWorld()->LineTraceSingleByChannel(OutHit, Start, End, ECC_Pawn, Params);
}
```

---

## 六、插值与平滑处理

### 6.1 位置插值

```cpp
// 高级位置插值系统
UCLASS()
class USmoothInterpolationComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    USmoothInterpolationComponent();

    virtual void TickComponent(float DeltaTime,
                               ELevelTick TickType,
                               FActorComponentTickFunction* ThisTickFunction) override;

    // 设置目标位置
    void SetTargetLocation(const FVector& NewTarget);

    // 设置插值速度
    void SetInterpolationSpeed(float Speed) { InterpolationSpeed = Speed; }

    // 网络平滑模式
    enum class ESmoothMode
    {
        Linear,      // 线性插值
        Exponential, // 指数插值（默认）
        DequeBased   // 基于双端队列的平滑
    };

    void SetSmoothMode(ESmoothMode Mode) { SmoothMode = Mode; }

private:
    FVector TargetLocation;
    FRotator TargetRotation;

    float InterpolationSpeed;
    ESmoothMode SmoothMode;

    // 用于DequeBased平滑
    struct FSample
    {
        float Time;
        FVector Location;
    };
    TArray<FSample> SampleBuffer;
    float BufferDuration = 0.2f;
};

void USmoothInterpolationComponent::TickComponent(
    float DeltaTime,
    ELevelTick TickType,
    FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

    if (!GetOwner()->HasAuthority())
    {
        switch (SmoothMode)
        {
            case ESmoothMode::Linear:
            {
                FVector NewLocation = FMath::Lerp(
                    GetOwner()->GetActorLocation(),
                    TargetLocation,
                    DeltaTime * InterpolationSpeed
                );
                GetOwner()->SetActorLocation(NewLocation);
                break;
            }

            case ESmoothMode::Exponential:
            {
                FVector NewLocation = FMath::VInterpTo(
                    GetOwner()->GetActorLocation(),
                    TargetLocation,
                    DeltaTime,
                    InterpolationSpeed
                );
                GetOwner()->SetActorLocation(NewLocation);
                break;
            }

            case ESmoothMode::DequeBased:
            {
                // 基于时间缓冲的平滑
                float CurrentTime = GetWorld()->GetTimeSeconds();
                float TargetTime = CurrentTime - BufferDuration;

                // 从缓冲中获取目标时间的位置
                FVector InterpolatedLocation = TargetLocation;

                if (SampleBuffer.Num() >= 2)
                {
                    for (int32 i = 0; i < SampleBuffer.Num() - 1; i++)
                    {
                        if (SampleBuffer[i].Time <= TargetTime &&
                            SampleBuffer[i + 1].Time >= TargetTime)
                        {
                            float Alpha = (TargetTime - SampleBuffer[i].Time) /
                                        (SampleBuffer[i + 1].Time - SampleBuffer[i].Time);

                            InterpolatedLocation = FMath::Lerp(
                                SampleBuffer[i].Location,
                                SampleBuffer[i + 1].Location,
                                Alpha
                            );
                            break;
                        }
                    }
                }

                GetOwner()->SetActorLocation(InterpolatedLocation);
                break;
            }
        }
    }
}

void USmoothInterpolationComponent::SetTargetLocation(const FVector& NewTarget)
{
    TargetLocation = NewTarget;

    // 记录样本用于DequeBased平滑
    if (SmoothMode == ESmoothMode::DequeBased)
    {
        FSample NewSample;
        NewSample.Time = GetWorld()->GetTimeSeconds();
        NewSample.Location = NewTarget;
        SampleBuffer.Add(NewSample);

        // 清理旧样本
        float CurrentTime = GetWorld()->GetTimeSeconds();
        SampleBuffer.RemoveAll([CurrentTime, this](const FSample& Sample)
        {
            return CurrentTime - Sample.Time > BufferDuration * 2.0f;
        });
    }
}
```

### 6.2 动画平滑

```cpp
// 网络动画平滑
UCLASS()
class UNetworkAnimationComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    // 存储动画状态的更新时间
    UPROPERTY(ReplicatedUsing = OnRep_AnimationState)
    FNetworkAnimationState AnimationState;

    UFUNCTION()
    void OnRep_AnimationState(const FNetworkAnimationState& OldState);

private:
    // 动画状态
    USTRUCT()
    struct FNetworkAnimationState
    {
        GENERATED_BODY()

        UPROPERTY()
        float Speed;

        UPROPERTY()
        bool bIsJumping;

        UPROPERTY()
        bool bIsCrouching;

        UPROPERTY()
        FRotator AimRotation;

        float Timestamp; // 非复制，本地计算
    };

    // 平滑后的值
    float SmoothedSpeed;
    FRotator SmoothedAimRotation;
    float SmoothTime = 0.1f;
};

void UNetworkAnimationComponent::OnRep_AnimationState(const FNetworkAnimationState& OldState)
{
    // 计算动画状态的平滑过渡
    float DeltaTime = GetWorld()->GetDeltaSeconds();

    // 速度平滑
    SmoothedSpeed = FMath::FInterpTo(
        SmoothedSpeed,
        AnimationState.Speed,
        DeltaTime,
        1.0f / SmoothTime
    );

    // 瞄准旋转平滑
    SmoothedAimRotation = FMath::RInterpTo(
        SmoothedAimRotation,
        AnimationState.AimRotation,
        DeltaTime,
        1.0f / SmoothTime
    );

    // 应用到动画蓝图
    if (USkeletalMeshComponent* Mesh = GetOwner()->FindComponentByClass<USkeletalMeshComponent>())
    {
        if (UAnimInstance* AnimInstance = Mesh->GetAnimInstance())
        {
            AnimInstance->SetFloatParameterValue(FName("Speed"), SmoothedSpeed);
            AnimInstance->SetBoolParameterValue(FName("IsJumping"), AnimationState.bIsJumping);
        }
    }
}
```

---

## 七、实践任务

### 任务：实现完整的网络移动系统

```cpp
// 实现一个包含预测、和解、延迟补偿的完整移动系统
// 参考 UE5 的 CharacterMovementComponent 实现

// 提示：
// 1. 使用 SaveMove 保存玩家输入
// 2. 客户端执行预测
// 3. 服务器验证并返回结果
// 4. 客户端进行和解
```

---

## 八、总结

本课我们学习了：

1. **同步模式**：状态同步与输入同步的区别与选择
2. **客户端预测**：实现即时响应的预测系统
3. **服务器权威**：设计安全权威的服务器架构
4. **延迟补偿**：为高延迟玩家提供公平体验
5. **插值平滑**：消除网络抖动带来的视觉问题

---

## 九、下节预告

**第6课：Dedicated Server开发**

将深入学习：
- Dedicated Server编译与部署
- 服务器启动参数配置
- GameMode与GameState设计
- PlayerController的服务器逻辑

---

*课程版本：1.0*
*最后更新：2026-04-10*