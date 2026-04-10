# 第18课：网络预测系统

> 本课深入讲解UE5的网络预测系统，包括客户端预测、服务器校正和移动预测实现。

---

## 课程目标

- 深入理解客户端预测原理
- 掌握服务器校正机制
- 学会移动预测实现
- 理解物理预测挑战
- 掌握预测系统调试方法

---

## 一、客户端预测原理

### 1.1 为什么需要客户端预测

```
没有预测的情况:

客户端                         服务器
   │                             │
   │  输入: 按W移动              │
   │  ──────────────────────►    │
   │         (延迟50ms)          │
   │                             │ 处理输入
   │                             │ 更新位置
   │  ◄──────────────────────    │
   │         (延迟50ms)          │ 返回新位置
   │                             │
   │  显示新位置                 │
   │                             │
总延迟: 100ms (往返延迟)
玩家感觉: 卡顿、不流畅


有预测的情况:

客户端                         服务器
   │                             │
   │  输入: 按W移动              │
   │  ──────────────────────►    │
   │                             │
   │  立即预测移动               │
   │  显示预测位置               │
   │                             │ 处理输入
   │                             │ 更新位置
   │  ◄──────────────────────    │
   │                             │ 返回真实位置
   │                             │
   │  对比预测与真实位置         │
   │  误差小: 继续               │
   │  误差大: 校正到真实位置     │
   │                             │
玩家感觉: 流畅、即时响应
```

### 1.2 预测核心组件

```cpp
// MyPredictionSystem.h
#pragma once

#include "CoreMinimal.h"
#include "MyPredictionSystem.generated.h"

// 预测移动状态
USTRUCT()
struct FPredictedMove
{
    GENERATED_BODY()

    float Timestamp;
    FVector InputVector;
    FRotator Rotation;
    float DeltaTime;

    // 预测结果
    FVector PredictedLocation;
    FRotator PredictedRotation;
    FVector PredictedVelocity;

    // 服务器确认
    bool bConfirmed = false;
    FVector ServerLocation;
};

// 预测配置
USTRUCT()
struct FPredictionConfig
{
    GENERATED_BODY()

    // 最大预测帧数
    int32 MaxPredictedMoves = 64;

    // 校正阈值
    float CorrectionThreshold = 10.0f; // cm

    // 平滑校正速度
    float CorrectionSpeed = 10.0f;

    // 是否启用预测
    bool bEnablePrediction = true;
};

UCLASS()
class UMyPredictionSystem : public UObject
{
    GENERATED_BODY()

public:
    UMyPredictionSystem();

    // 预测流程
    FPredictedMove CreatePredictedMove(const FVector& Input, float DeltaTime);
    void ApplyPredictedMove(const FPredictedMove& Move);
    void ConfirmMove(float Timestamp, const FVector& ServerLocation);
    void Reconcile(const FVector& ServerLocation, const FVector& ServerVelocity);

    // 状态
    FVector GetPredictedLocation() const { return CurrentLocation; }
    FVector GetPredictedVelocity() const { return CurrentVelocity; }
    int32 GetUnconfirmedMoveCount() const { return UnconfirmedMoves.Num(); }

    // 配置
    void SetConfig(const FPredictionConfig& Config) { CurrentConfig = Config; }

private:
    TArray<FPredictedMove> UnconfirmedMoves;
    FPredictionConfig CurrentConfig;

    FVector CurrentLocation;
    FVector CurrentVelocity;
    FRotator CurrentRotation;

    float CurrentTime;

    // 预测逻辑
    FVector PredictLocation(const FVector& StartLocation, const FVector& Input, float DeltaTime);
    void RemoveConfirmedMoves(float UpToTimestamp);
    void ReplayUnconfirmedMoves(const FVector& FromLocation);
};

// MyPredictionSystem.cpp
#include "MyPredictionSystem.h"

UMyPredictionSystem::UMyPredictionSystem()
{
    CurrentLocation = FVector::ZeroVector;
    CurrentVelocity = FVector::ZeroVector;
    CurrentTime = 0.0f;
}

FPredictedMove UMyPredictionSystem::CreatePredictedMove(const FVector& Input, float DeltaTime)
{
    FPredictedMove Move;
    Move.Timestamp = CurrentTime;
    Move.InputVector = Input;
    Move.DeltaTime = DeltaTime;

    // 执行预测
    Move.PredictedLocation = PredictLocation(CurrentLocation, Input, DeltaTime);
    Move.PredictedRotation = CurrentRotation;

    return Move;
}

void UMyPredictionSystem::ApplyPredictedMove(const FPredictedMove& Move)
{
    if (!CurrentConfig.bEnablePrediction)
    {
        return;
    }

    // 添加到未确认列表
    UnconfirmedMoves.Add(Move);

    // 应用预测
    CurrentLocation = Move.PredictedLocation;
    CurrentTime += Move.DeltaTime;

    // 更新Actor位置
    // Actor->SetActorLocation(CurrentLocation);
}

void UMyPredictionSystem::ConfirmMove(float Timestamp, const FVector& ServerLocation)
{
    // 找到对应的预测移动
    for (FPredictedMove& Move : UnconfirmedMoves)
    {
        if (FMath::IsNearlyEqual(Move.Timestamp, Timestamp, 0.001f))
        {
            Move.bConfirmed = true;
            Move.ServerLocation = ServerLocation;
            break;
        }
    }

    // 清理已确认的旧移动
    RemoveConfirmedMoves(Timestamp);
}

void UMyPredictionSystem::Reconcile(const FVector& ServerLocation, const FVector& ServerVelocity)
{
    float Error = FVector::Dist(CurrentLocation, ServerLocation);

    if (Error > CurrentConfig.CorrectionThreshold)
    {
        UE_LOG(LogTemp, Warning, TEXT("Prediction error: %.2f cm, reconciling"), Error);

        // 校正到服务器位置
        CurrentLocation = ServerLocation;
        CurrentVelocity = ServerVelocity;

        // 重放所有未确认的移动
        ReplayUnconfirmedMoves(ServerLocation);
    }
    else if (Error > 1.0f)
    {
        // 小误差，平滑校正
        CurrentLocation = FMath::VInterpTo(
            CurrentLocation,
            ServerLocation,
            1.0f / 60.0f,
            CurrentConfig.CorrectionSpeed
        );
    }
}

FVector UMyPredictionSystem::PredictLocation(const FVector& StartLocation, const FVector& Input, float DeltaTime)
{
    // 简单的移动预测
    FVector Acceleration = Input * 1000.0f; // 加速度
    FVector NewVelocity = CurrentVelocity + Acceleration * DeltaTime;
    NewVelocity = NewVelocity.GetClampedToMaxSize(600.0f); // 最大速度

    FVector NewLocation = StartLocation + NewVelocity * DeltaTime;

    return NewLocation;
}

void UMyPredictionSystem::RemoveConfirmedMoves(float UpToTimestamp)
{
    UnconfirmedMoves.RemoveAll([UpToTimestamp](const FPredictedMove& Move)
    {
        return Move.Timestamp <= UpToTimestamp;
    });
}

void UMyPredictionSystem::ReplayUnconfirmedMoves(const FVector& FromLocation)
{
    FVector ReplayLocation = FromLocation;

    for (FPredictedMove& Move : UnconfirmedMoves)
    {
        Move.PredictedLocation = PredictLocation(ReplayLocation, Move.InputVector, Move.DeltaTime);
        ReplayLocation = Move.PredictedLocation;
    }

    // 应用重放后的位置
    if (UnconfirmedMoves.Num() > 0)
    {
        CurrentLocation = UnconfirmedMoves.Last().PredictedLocation;
    }
}
```

---

## 二、服务器校正机制

### 2.1 服务器验证

```cpp
// MyServerCorrection.h
#pragma once

#include "CoreMinimal.h"
#include "MyServerCorrection.generated.h"

// 移动验证结果
UENUM()
enum class EMoveValidationResult : uint8
{
    Valid,
    InvalidPosition,
    InvalidVelocity,
    SpeedHack,
    TeleportHack,
    TimestampManipulation
};

// 服务器移动状态
USTRUCT()
struct FServerMoveState
{
    GENERATED_BODY()

    FVector LastValidLocation;
    FVector LastValidVelocity;
    float LastMoveTimestamp;
    int32 LastMoveSequence;

    // 验证参数
    float MaxSpeed = 600.0f;
    float MaxAcceleration = 2000.0f;
    float MaxStepHeight = 45.0f;
    float Tolerance = 5.0f; // 容差
};

UCLASS()
class UMyServerCorrection : public UObject
{
    GENERATED_BODY()

public:
    // 验证移动
    EMoveValidationResult ValidateMove(const FPredictedMove& Move, FServerMoveState& State);

    // 计算正确位置
    FVector ComputeCorrectPosition(const FPredictedMove& Move, const FServerMoveState& State);

    // 检测作弊
    bool DetectSpeedHack(const FVector& Start, const FVector& End, float DeltaTime, float MaxSpeed);
    bool DetectTeleportHack(const FVector& Start, const FVector& End, float MaxDistance);

    // 校正响应
    struct FCorrectionResponse
    {
        bool bNeedsCorrection;
        FVector CorrectedLocation;
        FVector CorrectedVelocity;
        FString Reason;
    };

    FCorrectionResponse GenerateCorrection(const FPredictedMove& Move, const FServerMoveState& State);

private:
    // 历史记录用于检测
    TArray<FPredictedMove> MoveHistory;
    int32 MaxHistorySize = 64;
};

// MyServerCorrection.cpp
#include "MyServerCorrection.h"

EMoveValidationResult UMyServerCorrection::ValidateMove(const FPredictedMove& Move, FServerMoveState& State)
{
    // 1. 验证时间戳
    if (Move.Timestamp <= State.LastMoveTimestamp)
    {
        return EMoveValidationResult::TimestampManipulation;
    }

    // 2. 计算期望位移
    float TimeDelta = Move.Timestamp - State.LastMoveTimestamp;
    FVector Movement = Move.PredictedLocation - State.LastValidLocation;
    float Distance = Movement.Size();

    // 3. 检测速度作弊
    float ActualSpeed = Distance / TimeDelta;
    if (ActualSpeed > State.MaxSpeed * 1.5f) // 50%容差
    {
        return EMoveValidationResult::SpeedHack;
    }

    // 4. 检测传送作弊
    if (Distance > State.MaxSpeed * TimeDelta * 2.0f)
    {
        return EMoveValidationResult::TeleportHack;
    }

    // 5. 验证位置合理性
    // 检查是否在有效区域内
    // ...

    return EMoveValidationResult::Valid;
}

FVector UMyServerCorrection::ComputeCorrectPosition(const FPredictedMove& Move, const FServerMoveState& State)
{
    // 基于服务器物理计算正确位置
    FVector CorrectedLocation = State.LastValidLocation;

    // 应用输入
    FVector Acceleration = Move.InputVector * State.MaxAcceleration;
    FVector Velocity = State.LastValidVelocity + Acceleration * Move.DeltaTime;
    Velocity = Velocity.GetClampedToMaxSize(State.MaxSpeed);

    CorrectedLocation += Velocity * Move.DeltaTime;

    return CorrectedLocation;
}

bool UMyServerCorrection::DetectSpeedHack(const FVector& Start, const FVector& End, float DeltaTime, float MaxSpeed)
{
    float Distance = FVector::Dist(Start, End);
    float ActualSpeed = Distance / DeltaTime;

    // 考虑网络延迟和预测误差
    float AllowedSpeed = MaxSpeed * 1.3f;

    return ActualSpeed > AllowedSpeed;
}

UMyServerCorrection::FCorrectionResponse UMyServerCorrection::GenerateCorrection(
    const FPredictedMove& Move, const FServerMoveState& State)
{
    FCorrectionResponse Response;

    EMoveValidationResult ValidationResult = ValidateMove(Move, State);

    if (ValidationResult != EMoveValidationResult::Valid)
    {
        Response.bNeedsCorrection = true;
        Response.CorrectedLocation = ComputeCorrectPosition(Move, State);
        Response.Reason = FString::Printf(TEXT("Validation failed: %d"), static_cast<int32>(ValidationResult));
    }
    else
    {
        Response.bNeedsCorrection = false;
    }

    return Response;
}
```

---

## 三、移动预测实现

### 3.1 完整移动预测组件

```cpp
// MyMovementPredictionComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "MyMovementPredictionComponent.generated.h"

// 移动输入
USTRUCT()
struct FMovementInput
{
    GENERATED_BODY()

    FVector MoveInput;
    FVector LookInput;
    bool bJumpPressed;
    bool bCrouchPressed;
    bool bSprintPressed;
};

// 移动状态
USTRUCT()
struct FMovementState
{
    GENERATED_BODY()

    FVector Location;
    FRotator Rotation;
    FVector Velocity;
    bool bIsOnGround;
    bool bIsCrouching;
    bool bIsSprinting;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnMovementCorrected, const FVector&, CorrectedLocation);

UCLASS(ClassGroup = "Movement", meta = (BlueprintSpawnableComponent))
class UMyMovementPredictionComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    UMyMovementPredictionComponent();

    virtual void TickComponent(float DeltaTime, ELevelTick TickType,
                               FActorComponentTickFunction* ThisTickFunction) override;

    // 输入
    void AddMovementInput(const FVector& Input);
    void AddLookInput(const FVector& Input);
    void Jump();
    void Crouch(bool bCrouch);
    void Sprint(bool bSprint);

    // 状态
    UFUNCTION(BlueprintPure, Category = "Movement")
    FMovementState GetCurrentState() const { return CurrentState; }

    UFUNCTION(BlueprintPure, Category = "Movement")
    bool IsPredictionEnabled() const { return bPredictionEnabled; }

    // 配置
    UPROPERTY(EditAnywhere, Category = "Movement")
    float WalkSpeed = 400.0f;

    UPROPERTY(EditAnywhere, Category = "Movement")
    float SprintSpeed = 600.0f;

    UPROPERTY(EditAnywhere, Category = "Movement")
    float CrouchSpeed = 200.0f;

    UPROPERTY(EditAnywhere, Category = "Movement")
    float JumpVelocity = 420.0f;

    UPROPERTY(EditAnywhere, Category = "Movement")
    float Gravity = -980.0f;

    UPROPERTY(EditAnywhere, Category = "Prediction")
    bool bPredictionEnabled = true;

    UPROPERTY(EditAnywhere, Category = "Prediction")
    float CorrectionThreshold = 10.0f;

    // 委托
    UPROPERTY(BlueprintAssignable)
    FOnMovementCorrected OnMovementCorrected;

protected:
    virtual void BeginPlay() override;

private:
    FMovementState CurrentState;
    FMovementInput CurrentInput;
    TArray<FPredictedMove> UnconfirmedMoves;

    float CurrentTime;
    int32 SequenceNumber;

    // 预测
    void ProcessPrediction(float DeltaTime);
    FMovementState PredictMovement(const FMovementState& StartState, const FMovementInput& Input, float DeltaTime);

    // 服务器通信
    UFUNCTION(Server, Reliable, WithValidation)
    void Server_SendMove(const FPredictedMove& Move);

    UFUNCTION(Client, Reliable)
    void Client_CorrectMove(int32 Sequence, const FMovementState& CorrectedState);

    // 校正
    void ApplyCorrection(const FMovementState& CorrectedState);
    void ReplayUnconfirmedMoves(const FMovementState& FromState);
};

// MyMovementPredictionComponent.cpp
#include "MyMovementPredictionComponent.h"
#include "GameFramework/Character.h"
#include "GameFramework/CharacterMovementComponent.h"

UMyMovementPredictionComponent::UMyMovementPredictionComponent()
{
    PrimaryComponentTick.bCanEverTick = true;
    SequenceNumber = 0;
    CurrentTime = 0.0f;
}

void UMyMovementPredictionComponent::BeginPlay()
{
    Super::BeginPlay();

    // 初始化状态
    AActor* Owner = GetOwner();
    if (Owner)
    {
        CurrentState.Location = Owner->GetActorLocation();
        CurrentState.Rotation = Owner->GetActorRotation();
    }
}

void UMyMovementPredictionComponent::TickComponent(float DeltaTime, ELevelTick TickType,
                                                    FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

    if (IsPredictionEnabled() && GetOwner()->GetLocalRole() == ROLE_AutonomousProxy)
    {
        ProcessPrediction(DeltaTime);
    }
}

void UMyMovementPredictionComponent::ProcessPrediction(float DeltaTime)
{
    // 创建预测移动
    FPredictedMove Move;
    Move.Timestamp = CurrentTime;
    Move.DeltaTime = DeltaTime;
    Move.Sequence = SequenceNumber++;

    // 预测下一状态
    FMovementState PredictedState = PredictMovement(CurrentState, CurrentInput, DeltaTime);

    Move.PredictedLocation = PredictedState.Location;
    Move.PredictedVelocity = PredictedState.Velocity;

    // 添加到未确认列表
    UnconfirmedMoves.Add(Move);

    // 应用预测
    CurrentState = PredictedState;
    CurrentTime += DeltaTime;

    // 更新Actor
    AActor* Owner = GetOwner();
    if (Owner)
    {
        Owner->SetActorLocation(CurrentState.Location);
        Owner->SetActorRotation(CurrentState.Rotation);
    }

    // 发送到服务器
    Server_SendMove(Move);
}

FMovementState UMyMovementPredictionComponent::PredictMovement(
    const FMovementState& StartState, const FMovementInput& Input, float DeltaTime)
{
    FMovementState NewState = StartState;

    // 计算速度
    float MaxSpeed = WalkSpeed;
    if (Input.bSprintPressed)
    {
        MaxSpeed = SprintSpeed;
    }
    else if (NewState.bIsCrouching)
    {
        MaxSpeed = CrouchSpeed;
    }

    // 应用移动输入
    FVector Acceleration = Input.MoveInput * MaxSpeed * 10.0f;
    NewState.Velocity += Acceleration * DeltaTime;
    NewState.Velocity = NewState.Velocity.GetClampedToMaxSize(MaxSpeed);

    // 应用重力
    if (!NewState.bIsOnGround)
    {
        NewState.Velocity.Z += Gravity * DeltaTime;
    }

    // 处理跳跃
    if (Input.bJumpPressed && NewState.bIsOnGround)
    {
        NewState.Velocity.Z = JumpVelocity;
        NewState.bIsOnGround = false;
    }

    // 更新位置
    NewState.Location += NewState.Velocity * DeltaTime;

    // 简单地面检测
    if (NewState.Location.Z <= 0.0f)
    {
        NewState.Location.Z = 0.0f;
        NewState.Velocity.Z = 0.0f;
        NewState.bIsOnGround = true;
    }

    return NewState;
}

void UMyMovementPredictionComponent::AddMovementInput(const FVector& Input)
{
    CurrentInput.MoveInput = Input;
}

void UMyMovementPredictionComponent::AddLookInput(const FVector& Input)
{
    CurrentInput.LookInput = Input;
}

void UMyMovementPredictionComponent::Jump()
{
    CurrentInput.bJumpPressed = true;
}

void UMyMovementPredictionComponent::Crouch(bool bCrouch)
{
    CurrentInput.bCrouchPressed = bCrouch;
    CurrentState.bIsCrouching = bCrouch;
}

void UMyMovementPredictionComponent::Sprint(bool bSprint)
{
    CurrentInput.bSprintPressed = bSprint;
    CurrentState.bIsSprinting = bSprint;
}

bool UMyMovementPredictionComponent::Server_SendMove_Validate(const FPredictedMove& Move)
{
    return true;
}

void UMyMovementPredictionComponent::Server_SendMove_Implementation(const FPredictedMove& Move)
{
    // 服务器处理移动
    // 验证并计算真实状态
    // 返回校正（如果需要）

    AActor* Owner = GetOwner();
    if (!Owner)
    {
        return;
    }

    // 应用移动到服务器状态
    // ...

    // 返回确认
    Client_CorrectMove(Move.Sequence, CurrentState);
}

void UMyMovementPredictionComponent::Client_CorrectMove_Implementation(int32 Sequence, const FMovementState& CorrectedState)
{
    // 找到对应的移动
    FPredictedMove* CorrespondingMove = nullptr;
    for (auto& Move : UnconfirmedMoves)
    {
        if (Move.Sequence == Sequence)
        {
            CorrespondingMove = &Move;
            break;
        }
    }

    if (!CorrespondingMove)
    {
        return;
    }

    // 计算误差
    float Error = FVector::Dist(CorrespondingMove->PredictedLocation, CorrectedState.Location);

    if (Error > CorrectionThreshold)
    {
        // 需要校正
        ApplyCorrection(CorrectedState);
    }

    // 移除已确认的移动
    UnconfirmedMoves.RemoveAll([Sequence](const FPredictedMove& Move)
    {
        return Move.Sequence <= Sequence;
    });
}

void UMyMovementPredictionComponent::ApplyCorrection(const FMovementState& CorrectedState)
{
    float Error = FVector::Dist(CurrentState.Location, CorrectedState.Location);

    UE_LOG(LogTemp, Warning, TEXT("Movement correction: %.2f cm"), Error);

    CurrentState = CorrectedState;

    // 更新Actor
    AActor* Owner = GetOwner();
    if (Owner)
    {
        Owner->SetActorLocation(CurrentState.Location);
    }

    // 重放未确认的移动
    ReplayUnconfirmedMoves(CorrectedState);

    OnMovementCorrected.Broadcast(CurrentState.Location);
}

void UMyMovementPredictionComponent::ReplayUnconfirmedMoves(const FMovementState& FromState)
{
    if (UnconfirmedMoves.Num() == 0)
    {
        return;
    }

    FMovementState ReplayState = FromState;

    for (FPredictedMove& Move : UnconfirmedMoves)
    {
        // 重放输入（需要存储）
        // ...

        // 更新预测
        Move.PredictedLocation = ReplayState.Location;
    }

    // 应用最终状态
    if (UnconfirmedMoves.Num() > 0)
    {
        CurrentState.Location = UnconfirmedMoves.Last().PredictedLocation;
    }
}
```

---

## 四、物理预测挑战

### 4.1 物理预测难点

```cpp
// 物理预测的挑战

/*
挑战1：物理确定性
- 物理引擎在不同机器上可能产生不同结果
- 浮点精度差异
- 解决方案：使用确定性物理或服务器权威

挑战2：碰撞响应
- 客户端和服务器碰撞结果可能不同
- 解决方案：简化碰撞或服务器校正

挑战3：复杂交互
- 多物体交互难以预测
- 解决方案：延迟显示或降低预测复杂度
*/

// 简化的物理预测
UCLASS()
class UMyPhysicsPrediction : public UObject
{
public:
    // 确定性物理模拟
    FVector SimulatePhysicsDeterministic(
        const FVector& StartLocation,
        const FVector& StartVelocity,
        float DeltaTime,
        uint32 RandomSeed)
    {
        // 使用固定种子确保确定性
        FRandomStream RandomStream(RandomSeed);

        FVector NewLocation = StartLocation;
        FVector Velocity = StartVelocity;

        // 简化的物理步骤
        // 避免使用非确定性操作

        return NewLocation;
    }

    // 物理状态同步
    void SyncPhysicsState(const FVector& ServerLocation, const FVector& ServerVelocity)
    {
        // 直接采用服务器状态
        // 放弃客户端物理预测
    }
};
```

---

## 五、总结

本课我们学习了：

1. **预测原理**：为什么需要预测以及预测流程
2. **服务器校正**：验证和校正机制
3. **移动预测**：完整的移动预测组件实现
4. **物理预测**：挑战和解决方案

---

## 六、下节预告

**第19课：ReplicationGraph进阶**

将深入学习：
- ReplicationGraph架构深入
- 自定义SpatialGrid策略
- 性能分析工具

---

*课程版本：1.0*
*最后更新：2026-04-10*