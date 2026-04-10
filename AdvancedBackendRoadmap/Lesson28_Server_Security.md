# 第28课：服务器安全

> 学习目标：掌握输入验证与过滤、反作弊系统设计、加密通信和API安全最佳实践。

---

## 28.1 输入验证与过滤

### 28.1.1 输入验证系统

```csharp
// InputValidationSubsystem.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "InputValidationSubsystem.generated.h"

UENUM(BlueprintType)
enum class EValidationResult : uint8
{
    Valid,
    Invalid,
    TooLong,
    InvalidCharacters,
    BlockedContent,
    Suspicious
};

USTRUCT(BlueprintType)
struct FValidationConfig
{
    GENERATED_BODY()

    // 字符串长度限制
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 MaxStringLength = 256;

    // 用户名长度
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 MinUsernameLength = 3;
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 MaxUsernameLength = 32;

    // 禁止的字符模式
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TArray<FString> BlockedPatterns;

    // 敏感词列表
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TArray<FString> SensitiveWords;

    // 允许的特殊字符
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString AllowedSpecialChars = TEXT("_-");

    // 是否启用XSS防护
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bEnableXSSProtection = true;

    // 是否启用SQL注入防护
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bEnableSQLInjectionProtection = true;
};

USTRUCT(BlueprintType)
struct FValidationReport
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    EValidationResult Result = EValidationResult::Valid;

    UPROPERTY(BlueprintReadOnly)
    FString OriginalInput;

    UPROPERTY(BlueprintReadOnly)
    FString SanitizedInput;

    UPROPERTY(BlueprintReadOnly)
    FString ErrorMessage;

    UPROPERTY(BlueprintReadOnly)
    TArray<FString> Warnings;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnSuspiciousInput, const FString&, PlayerId, const FString&, InputType);

UCLASS()
class INPUTVALIDATION_API UInputValidationSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 验证字符串输入
    UFUNCTION(BlueprintCallable, Category = "Validation")
    FValidationReport ValidateString(const FString& Input, const FString& Context = TEXT(""));

    // 验证用户名
    UFUNCTION(BlueprintCallable, Category = "Validation")
    FValidationReport ValidateUsername(const FString& Username);

    // 验证聊天消息
    UFUNCTION(BlueprintCallable, Category = "Validation")
    FValidationReport ValidateChatMessage(const FString& Message);

    // 验证数值范围
    UFUNCTION(BlueprintCallable, Category = "Validation")
    bool ValidateNumericRange(float Value, float Min, float Max, const FString& Context = TEXT(""));

    // 验证ID格式
    UFUNCTION(BlueprintCallable, Category = "Validation")
    bool ValidateId(const FString& Id, const FString& ExpectedPrefix = TEXT(""));

    // 清理输入
    UFUNCTION(BlueprintCallable, Category = "Validation")
    FString SanitizeInput(const FString& Input);

    // 检查敏感词
    UFUNCTION(BlueprintCallable, Category = "Validation")
    bool ContainsSensitiveWord(const FString& Input, FString& FoundWord);

    // 配置
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Validation")
    FValidationConfig Config;

    UPROPERTY(BlueprintAssignable, Category = "Validation")
    FOnSuspiciousInput OnSuspiciousInput;

private:
    // SQL注入模式
    TArray<FString> SQLInjectionPatterns;

    // XSS模式
    TArray<FString> XSSPatterns;

    // 检测到的可疑行为计数
    TMap<FString, int32> SuspiciousInputCount;

    void InitializePatterns();
    bool CheckSQLInjection(const FString& Input);
    bool CheckXSS(const FString& Input);
    FString RemoveDangerousChars(const FString& Input);
    void LogSuspiciousInput(const FString& PlayerId, const FString& InputType, const FString& Input);
};
```

```csharp
// InputValidationSubsystem.cpp
#include "InputValidationSubsystem.h"
#include "Regex.h"

void UInputValidationSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
    InitializePatterns();
}

void UInputValidationSubsystem::InitializePatterns()
{
    // SQL注入模式
    SQLInjectionPatterns = {
        TEXT("('|\")(\\s|\\+)+(SELECT|INSERT|DELETE|UPDATE|DROP|EXEC|UNION)"),
        TEXT("(SELECT|INSERT|DELETE|UPDATE|DROP|EXEC|UNION).+(FROM|INTO|TABLE|DATABASE)"),
        TEXT("(';|--|\\)\\s*;|\\)\\s+--|\\)\\s+})"),
        TEXT("(@@version|@@servername|@@LANGUAGE)"),
        TEXT("(WAITFOR\\s+DELAY|BENCHMARK\\s*\\()"),
        TEXT("(xp_|sp_)"),
        TEXT("(CONCAT|CHAR\\(|ASCII\\()")
    };

    // XSS模式
    XSSPatterns = {
        TEXT("(<script|</script)"),
        TEXT("(javascript:|vbscript:|onload\\s*=)"),
        TEXT("(onerror\\s*=|onclick\\s*=|onmouseover\\s*=)"),
        TEXT("(eval\\s*\\(|expression\\s*\\()"),
        TEXT("(<iframe|<object|<embed)"),
        TEXT("(document\\.(cookie|location|write))"),
        TEXT("(alert\\s*\\(|prompt\\s*\\(|confirm\\s*\\()")
    };

    // 加载敏感词列表（可以从文件加载）
    Config.SensitiveWords = {
        TEXT("admin"), TEXT("password"), TEXT("secret"),
        TEXT("hack"), TEXT("cheat"), TEXT("exploit")
    };
}

FValidationReport UInputValidationSubsystem::ValidateString(const FString& Input, const FString& Context)
{
    FValidationReport Report;
    Report.OriginalInput = Input;

    // 检查长度
    if (Input.Len() > Config.MaxStringLength)
    {
        Report.Result = EValidationResult::TooLong;
        Report.ErrorMessage = FString::Printf(TEXT("Input exceeds maximum length of %d"), Config.MaxStringLength);
        return Report;
    }

    // 检查空输入
    if (Input.IsEmpty())
    {
        Report.Result = EValidationResult::Invalid;
        Report.ErrorMessage = TEXT("Input cannot be empty");
        return Report;
    }

    // 检查非法字符
    FString AllowedChars = TEXT("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789") + Config.AllowedSpecialChars + TEXT(" ");
    for (TCHAR Char : Input)
    {
        if (!AllowedChars.Contains(FString(1, &Char)))
        {
            Report.Result = EValidationResult::InvalidCharacters;
            Report.ErrorMessage = FString::Printf(TEXT("Invalid character: %c"), Char);
            Report.Warnings.Add(Report.ErrorMessage);
        }
    }

    // 检查SQL注入
    if (Config.bEnableSQLInjectionProtection && CheckSQLInjection(Input))
    {
        Report.Result = EValidationResult::Suspicious;
        Report.ErrorMessage = TEXT("Potential SQL injection detected");
        LogSuspiciousInput(TEXT("Unknown"), Context, Input);
        return Report;
    }

    // 检查XSS
    if (Config.bEnableXSSProtection && CheckXSS(Input))
    {
        Report.Result = EValidationResult::Suspicious;
        Report.ErrorMessage = TEXT("Potential XSS attack detected");
        LogSuspiciousInput(TEXT("Unknown"), Context, Input);
        return Report;
    }

    // 检查敏感词
    FString FoundWord;
    if (ContainsSensitiveWord(Input, FoundWord))
    {
        Report.Result = EValidationResult::BlockedContent;
        Report.ErrorMessage = FString::Printf(TEXT("Blocked word detected: %s"), *FoundWord);
        Report.Warnings.Add(Report.ErrorMessage);
    }

    // 清理并返回
    Report.SanitizedInput = SanitizeInput(Input);
    Report.Result = EValidationResult::Valid;

    return Report;
}

FValidationReport UInputValidationSubsystem::ValidateUsername(const FString& Username)
{
    FValidationReport Report = ValidateString(Username, TEXT("username"));

    if (Report.Result != EValidationResult::Valid)
    {
        return Report;
    }

    // 检查用户名长度
    if (Username.Len() < Config.MinUsernameLength)
    {
        Report.Result = EValidationResult::Invalid;
        Report.ErrorMessage = FString::Printf(TEXT("Username must be at least %d characters"), Config.MinUsernameLength);
        return Report;
    }

    if (Username.Len() > Config.MaxUsernameLength)
    {
        Report.Result = EValidationResult::TooLong;
        Report.ErrorMessage = FString::Printf(TEXT("Username must be at most %d characters"), Config.MaxUsernameLength);
        return Report;
    }

    // 用户名不能以数字开头
    if (FChar::IsDigit(Username[0]))
    {
        Report.Result = EValidationResult::Invalid;
        Report.ErrorMessage = TEXT("Username cannot start with a number");
        return Report;
    }

    return Report;
}

FValidationReport UInputValidationSubsystem::ValidateChatMessage(const FString& Message)
{
    FValidationReport Report = ValidateString(Message, TEXT("chat"));

    // 额外的聊天验证
    if (Report.Result == EValidationResult::Valid)
    {
        // 检查重复字符（刷屏）
        int32 ConsecutiveCount = 1;
        int32 MaxConsecutive = 10;
        for (int32 i = 1; i < Message.Len(); i++)
        {
            if (Message[i] == Message[i - 1])
            {
                ConsecutiveCount++;
                if (ConsecutiveCount > MaxConsecutive)
                {
                    Report.Warnings.Add(TEXT("Message may contain spam"));
                    break;
                }
            }
            else
            {
                ConsecutiveCount = 1;
            }
        }
    }

    return Report;
}

bool UInputValidationSubsystem::ValidateNumericRange(float Value, float Min, float Max, const FString& Context)
{
    if (Value < Min || Value > Max)
    {
        UE_LOG(LogTemp, Warning, TEXT("Numeric validation failed: %f not in range [%f, %f], Context: %s"),
            Value, Min, Max, *Context);
        return false;
    }
    return true;
}

bool UInputValidationSubsystem::ValidateId(const FString& Id, const FString& ExpectedPrefix)
{
    if (Id.IsEmpty())
    {
        return false;
    }

    // 检查前缀
    if (!ExpectedPrefix.IsEmpty() && !Id.StartsWith(ExpectedPrefix))
    {
        return false;
    }

    // ID格式：字母开头，只包含字母数字和下划线
    if (!FChar::IsAlpha(Id[0]))
    {
        return false;
    }

    for (int32 i = 1; i < Id.Len(); i++)
    {
        if (!FChar::IsAlnum(Id[i]) && Id[i] != '_')
        {
            return false;
        }
    }

    return true;
}

FString UInputValidationSubsystem::SanitizeInput(const FString& Input)
{
    FString Sanitized = Input;

    // 移除危险字符
    Sanitized = RemoveDangerousChars(Sanitized);

    // HTML实体编码（用于显示）
    Sanitized.ReplaceInline(TEXT("&"), TEXT("&amp;"));
    Sanitized.ReplaceInline(TEXT("<"), TEXT("&lt;"));
    Sanitized.ReplaceInline(TEXT(">"), TEXT("&gt;"));
    Sanitized.ReplaceInline(TEXT("\""), TEXT("&quot;"));
    Sanitized.ReplaceInline(TEXT("'"), TEXT("&#39;"));

    return Sanitized;
}

bool UInputValidationSubsystem::ContainsSensitiveWord(const FString& Input, FString& FoundWord)
{
    FString LowerInput = Input.ToLower();

    for (const FString& Word : Config.SensitiveWords)
    {
        if (LowerInput.Contains(Word.ToLower()))
        {
            FoundWord = Word;
            return true;
        }
    }

    return false;
}

bool UInputValidationSubsystem::CheckSQLInjection(const FString& Input)
{
    FString LowerInput = Input.ToLower();

    for (const FString& Pattern : SQLInjectionPatterns)
    {
        FRegex Regex(FString::Printf(TEXT("(%s)"), *Pattern));
        FRegexMatcher Matcher(Regex, LowerInput);

        if (Matcher.FindNext())
        {
            UE_LOG(LogTemp, Warning, TEXT("SQL injection pattern detected: %s in input"), *Pattern);
            return true;
        }
    }

    return false;
}

bool UInputValidationSubsystem::CheckXSS(const FString& Input)
{
    FString LowerInput = Input.ToLower();

    for (const FString& Pattern : XSSPatterns)
    {
        FRegex Regex(FString::Printf(TEXT("(%s)"), *Pattern));
        FRegexMatcher Matcher(Regex, LowerInput);

        if (Matcher.FindNext())
        {
            UE_LOG(LogTemp, Warning, TEXT("XSS pattern detected: %s in input"), *Pattern);
            return true;
        }
    }

    return false;
}

FString UInputValidationSubsystem::RemoveDangerousChars(const FString& Input)
{
    FString Result;
    Result.Reserve(Input.Len());

    for (TCHAR Char : Input)
    {
        // 移除控制字符
        if (Char >= 32 || Char == '\n' || Char == '\r' || Char == '\t')
        {
            Result.AppendChar(Char);
        }
    }

    return Result;
}

void UInputValidationSubsystem::LogSuspiciousInput(const FString& PlayerId, const FString& InputType, const FString& Input)
{
    FString Key = PlayerId + TEXT("_") + InputType;
    SuspiciousInputCount.Add(Key, SuspiciousInputCount.FindRef(Key) + 1);

    UE_LOG(LogTemp, Warning, TEXT("Suspicious input from %s, type: %s, count: %d"),
        *PlayerId, *InputType, SuspiciousInputCount[Key]);

    OnSuspiciousInput.Broadcast(PlayerId, InputType);
}
```

---

## 28.2 反作弊系统设计

### 28.2.1 服务器端验证

```csharp
// AntiCheatSubsystem.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "AntiCheatSubsystem.generated.h"

UENUM(BlueprintType)
enum class ECheatType : uint8
{
    SpeedHack,
    Teleport,
    ImpossibleAction,
    MemoryModification,
    PacketManipulation,
    Aimbot,
    WallHack,
    ResourceManipulation,
    TimestampManipulation
};

USTRUCT(BlueprintType)
struct FCheatDetection
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString PlayerId;

    UPROPERTY(BlueprintReadOnly)
    ECheatType CheatType;

    UPROPERTY(BlueprintReadOnly)
    float Confidence = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    FString Details;

    UPROPERTY(BlueprintReadOnly)
    double DetectionTime;
};

USTRUCT(BlueprintType)
struct FPlayerMovementSnapshot
{
    GENERATED_BODY()

    UPROPERTY()
    FVector Location;

    UPROPERTY()
    FRotator Rotation;

    UPROPERTY()
    FVector Velocity;

    UPROPERTY()
    double Timestamp;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnCheatDetected, const FCheatDetection&, Detection);

UCLASS()
class ANTICHEAT_API UAntiCheatSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 验证移动
    UFUNCTION(BlueprintCallable, Category = "AntiCheat")
    bool ValidateMovement(const FString& PlayerId, const FVector& NewLocation, const FVector& Velocity, float DeltaTime);

    // 验证射击
    UFUNCTION(BlueprintCallable, Category = "AntiCheat")
    bool ValidateShot(const FString& PlayerId, const FVector& StartLocation, const FVector& EndLocation, float Timestamp);

    // 验证资源变化
    UFUNCTION(BlueprintCallable, Category = "AntiCheat")
    bool ValidateResourceChange(const FString& PlayerId, const FString& ResourceType, int32 OldValue, int32 NewValue);

    // 验证时间戳
    UFUNCTION(BlueprintCallable, Category = "AntiCheat")
    bool ValidateTimestamp(const FString& PlayerId, float ClientTimestamp, float ServerTimestamp);

    // 验证动作频率
    UFUNCTION(BlueprintCallable, Category = "AntiCheat")
    bool ValidateActionRate(const FString& PlayerId, const FString& ActionType);

    // 获取玩家作弊分数
    UFUNCTION(BlueprintPure, Category = "AntiCheat")
    float GetPlayerCheatScore(const FString& PlayerId) const;

    // 获取检测历史
    UFUNCTION(BlueprintCallable, Category = "AntiCheat")
    TArray<FCheatDetection> GetPlayerDetectionHistory(const FString& PlayerId) const;

    // 配置
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AntiCheat")
    float MaxMoveSpeed = 1000.0f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AntiCheat")
    float MaxAcceleration = 2000.0f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AntiCheat")
    float MaxTurnRate = 720.0f;  // 度/秒

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AntiCheat")
    float TimestampTolerance = 0.5f;  // 秒

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AntiCheat")
    float CheatScoreThreshold = 100.0f;

    UPROPERTY(BlueprintAssignable, Category = "AntiCheat")
    FOnCheatDetected OnCheatDetected;

private:
    // 玩家移动历史
    TMap<FString, TArray<FPlayerMovementSnapshot>> MovementHistory;

    // 玩家作弊分数
    TMap<FString, float> CheatScores;

    // 检测历史
    TMap<FString, TArray<FCheatDetection>> DetectionHistory;

    // 动作频率追踪
    TMap<FString, TMap<FString, TArray<double>>> ActionRateTracking;

    // 服务器时间偏移追踪
    TMap<FString, float> ServerTimeOffset;

    void RecordCheatDetection(const FString& PlayerId, ECheatType Type, float Confidence, const FString& Details);
    void UpdateCheatScore(const FString& PlayerId, float Points);

    bool CheckSpeedHack(const FString& PlayerId, const FVector& NewLocation, const FVector& Velocity, float DeltaTime);
    bool CheckTeleport(const FString& PlayerId, const FVector& NewLocation, const FVector& LastLocation, float DeltaTime);
    bool CheckAimbot(const FString& PlayerId, const FVector& StartLocation, const FVector& EndLocation);
};
```

```csharp
// AntiCheatSubsystem.cpp
#include "AntiCheatSubsystem.h"
#include "Kismet/KismetMathLibrary.h"

void UAntiCheatSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
}

bool UAntiCheatSubsystem::ValidateMovement(const FString& PlayerId, const FVector& NewLocation, const FVector& Velocity, float DeltaTime)
{
    TArray<FPlayerMovementSnapshot>& History = MovementHistory.FindOrAdd(PlayerId);

    // 获取上一个快照
    if (History.Num() > 0)
    {
        const FPlayerMovementSnapshot& LastSnapshot = History.Last();
        double TimeDiff = FPlatformTime::Seconds() - LastSnapshot.Timestamp;

        if (TimeDiff > 0.001 && TimeDiff < 5.0)
        {
            FVector LastLocation = LastSnapshot.Location;
            float Distance = FVector::Dist(NewLocation, LastLocation);
            float ExpectedMaxDistance = MaxMoveSpeed * TimeDiff;

            // 检查速度作弊
            if (CheckSpeedHack(PlayerId, NewLocation, Velocity, TimeDiff))
            {
                return false;
            }

            // 检查瞬移
            if (CheckTeleport(PlayerId, NewLocation, LastLocation, TimeDiff))
            {
                return false;
            }
        }
    }

    // 记录当前快照
    FPlayerMovementSnapshot NewSnapshot;
    NewSnapshot.Location = NewLocation;
    NewSnapshot.Velocity = Velocity;
    NewSnapshot.Timestamp = FPlatformTime::Seconds();

    History.Add(NewSnapshot);

    // 限制历史长度
    if (History.Num() > 60)
    {
        History.RemoveAt(0);
    }

    return true;
}

bool UAntiCheatSubsystem::CheckSpeedHack(const FString& PlayerId, const FVector& NewLocation, const FVector& Velocity, float DeltaTime)
{
    TArray<FPlayerMovementSnapshot>& History = MovementHistory.FindOrAdd(PlayerId);

    if (History.Num() < 2)
    {
        return false;
    }

    const FPlayerMovementSnapshot& LastSnapshot = History.Last();
    FVector Distance = NewLocation - LastSnapshot.Location;
    float ActualSpeed = Distance.Size() / DeltaTime;

    // 检查是否超过最大速度
    float SpeedRatio = ActualSpeed / MaxMoveSpeed;

    if (SpeedRatio > 1.5f)  // 允许50%的误差
    {
        FString Details = FString::Printf(
            TEXT("Speed hack detected: %.1f units/s (max: %.1f), ratio: %.2f"),
            ActualSpeed, MaxMoveSpeed, SpeedRatio
        );

        RecordCheatDetection(PlayerId, ECheatType::SpeedHack, SpeedRatio - 1.0f, Details);
        return true;
    }

    // 检查加速度
    FVector LastVelocity = LastSnapshot.Velocity;
    FVector Acceleration = (Velocity - LastVelocity) / DeltaTime;
    float AccelerationMag = Acceleration.Size();

    if (AccelerationMag > MaxAcceleration * 2.0f)
    {
        FString Details = FString::Printf(
            TEXT("Impossible acceleration: %.1f (max: %.1f)"),
            AccelerationMag, MaxAcceleration
        );

        RecordCheatDetection(PlayerId, ECheatType::ImpossibleAction, 0.8f, Details);
        return true;
    }

    return false;
}

bool UAntiCheatSubsystem::CheckTeleport(const FString& PlayerId, const FVector& NewLocation, const FVector& LastLocation, float DeltaTime)
{
    float Distance = FVector::Dist(NewLocation, LastLocation);
    float MaxPossibleDistance = MaxMoveSpeed * DeltaTime * 2.0f;  // 双倍容差

    if (Distance > MaxPossibleDistance)
    {
        FString Details = FString::Printf(
            TEXT("Teleport detected: %.1f units in %.3f seconds"),
            Distance, DeltaTime
        );

        RecordCheatDetection(PlayerId, ECheatType::Teleport, 0.9f, Details);
        return true;
    }

    return false;
}

bool UAntiCheatSubsystem::ValidateShot(const FString& PlayerId, const FVector& StartLocation, const FVector& EndLocation)
{
    // 获取玩家最后位置
    TArray<FPlayerMovementSnapshot>* History = MovementHistory.Find(PlayerId);
    if (!History || History->Num() == 0)
    {
        return true;  // 没有历史数据，暂时通过
    }

    const FPlayerMovementSnapshot& LastSnapshot = History->Last();

    // 检查射击起点是否合理
    float DistanceToPlayer = FVector::Dist(StartLocation, LastSnapshot.Location);
    if (DistanceToPlayer > 200.0f)  // 超过2米
    {
        FString Details = FString::Printf(
            TEXT("Shot from invalid location: %.1f units from player"),
            DistanceToPlayer
        );

        RecordCheatDetection(PlayerId, ECheatType::WallHack, 0.7f, Details);
        return false;
    }

    // 检查瞄准辅助（Aimbot）
    if (CheckAimbot(PlayerId, StartLocation, EndLocation))
    {
        return false;
    }

    return true;
}

bool UAntiCheatSubsystem::CheckAimbot(const FString& PlayerId, const FVector& StartLocation, const FVector& EndLocation)
{
    // 分析瞄准模式
    // 这里可以检查：
    // 1. 瞄准的平滑度
    // 2. 命中率异常
    // 3. 反应时间异常

    // 简化实现：检查瞄准角度变化是否过于完美
    static TArray<float> AngleHistory;
    FVector Direction = (EndLocation - StartLocation).GetSafeNormal();

    // 计算角度变化
    if (AngleHistory.Num() > 0)
    {
        // 分析历史数据...
    }

    return false;
}

bool UAntiCheatSubsystem::ValidateResourceChange(const FString& PlayerId, const FString& ResourceType, int32 OldValue, int32 NewValue)
{
    // 资源只能增加或正常消耗
    if (NewValue < 0)
    {
        FString Details = FString::Printf(
            TEXT("Negative resource: %s = %d"),
            *ResourceType, NewValue
        );

        RecordCheatDetection(PlayerId, ECheatType::ResourceManipulation, 1.0f, Details);
        return false;
    }

    // 检查异常增加
    int32 Change = NewValue - OldValue;
    if (Change > 10000)  // 阈值可根据游戏调整
    {
        FString Details = FString::Printf(
            TEXT("Suspicious resource gain: %s +%d"),
            *ResourceType, Change
        );

        RecordCheatDetection(PlayerId, ECheatType::ResourceManipulation, 0.6f, Details);
        return false;
    }

    return true;
}

bool UAntiCheatSubsystem::ValidateTimestamp(const FString& PlayerId, float ClientTimestamp, float ServerTimestamp)
{
    float& Offset = ServerTimeOffset.FindOrAdd(PlayerId);

    // 计算时间差
    float TimeDiff = ServerTimestamp - ClientTimestamp;

    // 更新偏移（使用滑动平均）
    Offset = Offset * 0.9f + TimeDiff * 0.1f;

    // 检查是否超出容忍范围
    if (FMath::Abs(TimeDiff - Offset) > TimestampTolerance)
    {
        FString Details = FString::Printf(
            TEXT("Timestamp manipulation: client=%.3f, server=%.3f, diff=%.3f"),
            ClientTimestamp, ServerTimestamp, TimeDiff
        );

        RecordCheatDetection(PlayerId, ECheatType::TimestampManipulation, 0.8f, Details);
        return false;
    }

    return true;
}

bool UAntiCheatSubsystem::ValidateActionRate(const FString& PlayerId, const FString& ActionType)
{
    TMap<FString, TArray<double>>& PlayerActions = ActionRateTracking.FindOrAdd(PlayerId);
    TArray<double>& ActionTimes = PlayerActions.FindOrAdd(ActionType);

    double CurrentTime = FPlatformTime::Seconds();
    ActionTimes.Add(CurrentTime);

    // 清理旧记录（只保留最近10秒）
    ActionTimes.RemoveAll([CurrentTime](double Time)
    {
        return CurrentTime - Time > 10.0;
    });

    // 检查动作频率
    int32 RecentCount = ActionTimes.Num();

    // 定义各动作的最大频率
    static TMap<FString, int32> MaxActionRates = {
        {TEXT("Fire"), 30},       // 每秒最多30次
        {TEXT("Jump"), 5},        // 每秒最多5次
        {TEXT("Interact"), 10},   // 每秒最多10次
        {TEXT("UseItem"), 5}      // 每秒最多5次
    };

    int32 MaxRate = MaxActionRates.FindRef(ActionType);
    if (MaxRate > 0 && RecentCount > MaxRate)
    {
        FString Details = FString::Printf(
            TEXT("Action rate exceeded: %s - %d in 10s (max: %d)"),
            *ActionType, RecentCount, MaxRate * 10
        );

        RecordCheatDetection(PlayerId, ECheatType::ImpossibleAction, 0.7f, Details);
        return false;
    }

    return true;
}

float UAntiCheatSubsystem::GetPlayerCheatScore(const FString& PlayerId) const
{
    const float* Score = CheatScores.Find(PlayerId);
    return Score ? *Score : 0.0f;
}

TArray<FCheatDetection> UAntiCheatSubsystem::GetPlayerDetectionHistory(const FString& PlayerId) const
{
    const TArray<FCheatDetection>* History = DetectionHistory.Find(PlayerId);
    return History ? *History : TArray<FCheatDetection>();
}

void UAntiCheatSubsystem::RecordCheatDetection(const FString& PlayerId, ECheatType Type, float Confidence, const FString& Details)
{
    FCheatDetection Detection;
    Detection.PlayerId = PlayerId;
    Detection.CheatType = Type;
    Detection.Confidence = Confidence;
    Detection.Details = Details;
    Detection.DetectionTime = FPlatformTime::Seconds();

    // 添加到历史
    TArray<FCheatDetection>& History = DetectionHistory.FindOrAdd(PlayerId);
    History.Add(Detection);

    // 限制历史长度
    if (History.Num() > 100)
    {
        History.RemoveAt(0);
    }

    // 更新作弊分数
    float ScoreIncrease = Confidence * 10.0f;
    UpdateCheatScore(PlayerId, ScoreIncrease);

    // 广播事件
    OnCheatDetected.Broadcast(Detection);

    UE_LOG(LogTemp, Warning, TEXT("Cheat detected: Player=%s, Type=%d, Confidence=%.2f, Details=%s"),
        *PlayerId, (int32)Type, Confidence, *Details);
}

void UAntiCheatSubsystem::UpdateCheatScore(const FString& PlayerId, float Points)
{
    float CurrentScore = CheatScores.FindRef(PlayerId);
    float NewScore = CurrentScore + Points;

    CheatScores.Add(PlayerId, NewScore);

    // 检查是否超过阈值
    if (NewScore >= CheatScoreThreshold)
    {
        UE_LOG(LogTemp, Error, TEXT("Player %s exceeded cheat score threshold: %.1f"),
            *PlayerId, NewScore);

        // 可以触发封禁、踢出等操作
    }
}
```

---

## 28.3 加密通信

### 28.3.1 TLS/SSL配置

```csharp
// TLSCryptoManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "TLSCryptoManager.generated.h"

USTRUCT(BlueprintType)
struct FCryptoConfig
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString CertificatePath;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString PrivateKeyPath;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bRequireClientCertificate = false;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString CipherList = TEXT("ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384");
};

UCLASS()
class TLSCRYPTO_API UTLSCryptoManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 生成密钥对
    UFUNCTION(BlueprintCallable, Category = "Crypto")
    bool GenerateKeyPair(const FString& PublicKeyPath, const FString& PrivateKeyPath, int32 KeySize = 2048);

    // 加密数据
    UFUNCTION(BlueprintCallable, Category = "Crypto")
    bool EncryptData(const TArray<uint8>& Plaintext, const FString& PublicKeyPath, TArray<uint8>& OutCiphertext);

    // 解密数据
    UFUNCTION(BlueprintCallable, Category = "Crypto")
    bool DecryptData(const TArray<uint8>& Ciphertext, const FString& PrivateKeyPath, TArray<uint8>& OutPlaintext);

    // 签名数据
    UFUNCTION(BlueprintCallable, Category = "Crypto")
    bool SignData(const TArray<uint8>& Data, const FString& PrivateKeyPath, TArray<uint8>& OutSignature);

    // 验证签名
    UFUNCTION(BlueprintCallable, Category = "Crypto")
    bool VerifySignature(const TArray<uint8>& Data, const TArray<uint8>& Signature, const FString& PublicKeyPath);

    // 哈希数据
    UFUNCTION(BlueprintCallable, Category = "Crypto")
    FString HashData(const TArray<uint8>& Data);

    // HMAC
    UFUNCTION(BlueprintCallable, Category = "Crypto")
    FString ComputeHMAC(const TArray<uint8>& Data, const FString& Key);

    // 生成随机Token
    UFUNCTION(BlueprintCallable, Category = "Crypto")
    FString GenerateToken(int32 Length = 32);

private:
    FCryptoConfig Config;
};
```

### 28.3.2 消息加密层

```csharp
// EncryptedMessageHandler.h
#pragma once

#include "CoreMinimal.h"

#include "EncryptedMessageHandler.generated.h"

USTRUCT()
struct FEncryptedMessage
{
    GENERATED_BODY()

    UPROPERTY()
    TArray<uint8> IV;  // 初始化向量

    UPROPERTY()
    TArray<uint8> EncryptedData;

    UPROPERTY()
    TArray<uint8> HMAC;  // 消息认证码

    UPROPERTY()
    int64 Timestamp;

    UPROPERTY()
    int32 SequenceNumber;
};

UCLASS()
class ENCRYPTEDMSG_API UEncryptedMessageHandler : public UObject
{
    GENERATED_BODY()

public:
    // 初始化加密处理器
    void Initialize(const TArray<uint8>& InSharedSecret);

    // 加密消息
    bool EncryptMessage(const TArray<uint8>& Plaintext, FEncryptedMessage& OutMessage);

    // 解密消息
    bool DecryptMessage(const FEncryptedMessage& Message, TArray<uint8>& OutPlaintext);

    // 验证消息
    bool ValidateMessage(const FEncryptedMessage& Message);

    // 设置序列号
    void SetSequenceNumber(int32 InSequence) { SequenceNumber = InSequence; }

private:
    TArray<uint8> SharedSecret;
    TArray<uint8> SessionKey;
    int32 SequenceNumber = 0;
    double LastMessageTime = 0.0;

    // 重放攻击防护
    TSet<int64> SeenSequences;
    int32 MaxSequenceHistory = 1000;

    bool DeriveSessionKey();
    void RotateSessionKey();
    bool CheckReplayAttack(int32 Sequence);
};
```

---

## 28.4 API安全最佳实践

### 28.4.1 API认证中间件

```csharp
// APIAuthMiddleware.h
#pragma once

#include "CoreMinimal.h"
#include "HttpServerResponse.h"

#include "APIAuthMiddleware.generated.h"

UENUM(BlueprintType)
enum class EAuthResult : uint8
{
    Authorized,
    Unauthorized,
    Forbidden,
    RateLimited,
    InvalidToken
};

USTRUCT(BlueprintType)
struct FAuthToken
{
    GENERATED_BODY()

    UPROPERTY()
    FString Token;

    UPROPERTY()
    FString UserId;

    UPROPERTY()
    TArray<FString> Scopes;

    UPROPERTY()
    double ExpirationTime;

    UPROPERTY()
    double IssuedTime;
};

UCLASS()
class APISECURITY_API UAPIAuthMiddleware : public UObject
{
    GENERATED_BODY()

public:
    // 验证请求
    EAuthResult ValidateRequest(const FHttpServerRequest& Request, FAuthToken& OutToken);

    // 生成访问令牌
    FString GenerateAccessToken(const FString& UserId, const TArray<FString>& Scopes, int32 ExpiresInMinutes = 60);

    // 刷新令牌
    FString RefreshAccessToken(const FString& RefreshToken);

    // 撤销令牌
    bool RevokeToken(const FString& Token);

    // 检查权限
    bool CheckScope(const FAuthToken& Token, const FString& RequiredScope);

    // 配置
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 AccessTokenExpirationMinutes = 60;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 RefreshTokenExpirationDays = 30;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 MaxRequestsPerMinute = 60;

private:
    // 活跃令牌缓存
    TMap<FString, FAuthToken> TokenCache;

    // 刷新令牌
    TMap<FString, FString> RefreshTokens;

    // 速率限制
    TMap<FString, TArray<double>> RateLimitCache;

    FString ExtractBearerToken(const FHttpServerRequest& Request);
    bool ValidateTokenSignature(const FString& Token);
    bool CheckRateLimit(const FString& ClientId);
    TArray<uint8> SignToken(const FString& Payload);
};
```

---

## 28.5 DDoS防护策略

### 28.5.1 流量控制

```csharp
// DDoSProtectionSubsystem.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "DDoSProtectionSubsystem.generated.h"

USTRUCT(BlueprintType)
struct FConnectionStats
{
    GENERATED_BODY()

    UPROPERTY()
    FString ClientIP;

    UPROPERTY()
    int32 ConnectionCount = 0;

    UPROPERTY()
    int32 PacketCount = 0;

    UPROPERTY()
    int64 BytesReceived = 0;

    UPROPERTY()
    double FirstConnectionTime = 0.0;

    UPROPERTY()
    double LastActivityTime = 0.0;

    UPROPERTY()
    bool bIsBlocked = false;
};

USTRUCT(BlueprintType)
struct FDDoSConfig
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 MaxConnectionsPerIP = 10;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 MaxPacketsPerSecond = 1000;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int64 MaxBytesPerSecond = 1048576;  // 1MB

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 BlockDurationMinutes = 30;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 ConnectionTimeoutSeconds = 60;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnIPBlocked, const FString&, IP, const FString&, Reason);

UCLASS()
class DDOSPROTECTION_API UDDoSProtectionSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 检查连接
    bool CheckConnection(const FString& ClientIP);

    // 记录活动
    void RecordActivity(const FString& ClientIP, int32 BytesReceived);

    // 检查是否被阻止
    UFUNCTION(BlueprintPure, Category = "DDoS")
    bool IsIPBlocked(const FString& ClientIP) const;

    // 手动阻止/解除
    UFUNCTION(BlueprintCallable, Category = "DDoS")
    void BlockIP(const FString& ClientIP, int32 DurationMinutes = 30, const FString& Reason = TEXT("Manual"));

    UFUNCTION(BlueprintCallable, Category = "DDoS")
    void UnblockIP(const FString& ClientIP);

    // 获取统计
    UFUNCTION(BlueprintPure, Category = "DDoS")
    TArray<FConnectionStats> GetTopTrafficSources(int32 Count = 10) const;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "DDoS")
    FDDoSConfig Config;

    UPROPERTY(BlueprintAssignable, Category = "DDoS")
    FOnIPBlocked OnIPBlocked;

private:
    TMap<FString, FConnectionStats> IPStats;
    TMap<FString, double> BlockedIPs;
    FTimerHandle CleanupTimerHandle;

    void CleanupStaleData();
    bool CheckRateLimits(const FString& ClientIP, const FConnectionStats& Stats);
};
```

---

## 28.6 实践任务

### 任务1：实现输入验证系统

为游戏聊天系统实现：
- 敏感词过滤
- XSS防护
- 长度限制

### 任务2：设计反作弊系统

实现移动验证：
- 速度检测
- 瞬移检测
- 异常行为记录

### 任务3：实现API认证

为游戏服务器API实现：
- JWT令牌认证
- 速率限制
- 权限检查

---

## 28.7 总结

本课学习了：
- 输入验证与过滤技术
- 反作弊系统设计与实现
- TLS/SSL加密通信
- API安全最佳实践
- DDoS防护策略

**下一课预告**：防盗版与防盗版设计 - DRM集成、许可验证和资源加密保护。

---

*课程版本：1.0*
*最后更新：2026-04-10*
