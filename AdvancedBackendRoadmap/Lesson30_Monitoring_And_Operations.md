# 第30课：监控与运维

> 学习目标：掌握服务器健康监控、日志聚合与分析、告警系统设计和自动化运维脚本。

---

## 30.1 服务器健康监控

### 30.1.1 健康检查系统

```csharp
// HealthMonitorSubsystem.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "HealthMonitorSubsystem.generated.h"

UENUM(BlueprintType)
enum class EHealthStatus : uint8
{
    Healthy,
    Degraded,
    Unhealthy,
    Unknown
};

USTRUCT(BlueprintType)
struct FHealthCheckResult
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString CheckName;

    UPROPERTY(BlueprintReadOnly)
    EHealthStatus Status = EHealthStatus::Unknown;

    UPROPERTY(BlueprintReadOnly)
    FString Message;

    UPROPERTY(BlueprintReadOnly)
    double LastCheckTime = 0.0;

    UPROPERTY(BlueprintReadOnly)
    float ResponseTimeMs = 0.0f;
};

USTRUCT(BlueprintType)
struct FServerHealthReport
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    EHealthStatus OverallStatus = EHealthStatus::Unknown;

    UPROPERTY(BlueprintReadOnly)
    TArray<FHealthCheckResult> Checks;

    UPROPERTY(BlueprintReadOnly)
    double ReportTime = 0.0;

    UPROPERTY(BlueprintReadOnly)
    FString ServerId;

    UPROPERTY(BlueprintReadOnly)
    FString ServerVersion;

    UPROPERTY(BlueprintReadOnly)
    int32 UptimeSeconds = 0;

    // 资源指标
    UPROPERTY(BlueprintReadOnly)
    float CpuUsagePercent = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    float MemoryUsagePercent = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    int64 MemoryUsedMB = 0;

    UPROPERTY(BlueprintReadOnly)
    int64 MemoryTotalMB = 0;

    // 网络指标
    UPROPERTY(BlueprintReadOnly)
    int32 ConnectedPlayers = 0;

    UPROPERTY(BlueprintReadOnly)
    float NetworkInKBps = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    float NetworkOutKBps = 0.0f;

    // 游戏指标
    UPROPERTY(BlueprintReadOnly)
    int32 ActiveGameSessions = 0;

    UPROPERTY(BlueprintReadOnly)
    float AverageFrameTimeMs = 0.0f;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnHealthStatusChanged, const FServerHealthReport&, Report);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnHealthCheckFailed, const FString&, CheckName);

UCLASS()
class HEALTHMONITOR_API UHealthMonitorSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 注册健康检查
    UFUNCTION(BlueprintCallable, Category = "Health")
    void RegisterHealthCheck(const FString& CheckName, float CheckIntervalSeconds = 30.0f);

    // 移除健康检查
    UFUNCTION(BlueprintCallable, Category = "Health")
    void UnregisterHealthCheck(const FString& CheckName);

    // 执行所有检查
    UFUNCTION(BlueprintCallable, Category = "Health")
    FServerHealthReport RunAllChecks();

    // 执行单个检查
    UFUNCTION(BlueprintCallable, Category = "Health")
    FHealthCheckResult RunCheck(const FString& CheckName);

    // 获取最新报告
    UFUNCTION(BlueprintPure, Category = "Health")
    FServerHealthReport GetLatestReport() const { return LatestReport; }

    // 获取HTTP健康端点响应
    UFUNCTION(BlueprintCallable, Category = "Health")
    FString GetHealthEndpointResponse();

    // 配置
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Health")
    float DefaultCheckInterval = 30.0f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Health")
    float CpuWarningThreshold = 80.0f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Health")
    float CpuCriticalThreshold = 95.0f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Health")
    float MemoryWarningThreshold = 80.0f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Health")
    float MemoryCriticalThreshold = 95.0f;

    UPROPERTY(BlueprintAssignable, Category = "Health")
    FOnHealthStatusChanged OnStatusChanged;

    UPROPERTY(BlueprintAssignable, Category = "Health")
    FOnHealthCheckFailed OnCheckFailed;

private:
    struct FHealthCheckEntry
    {
        FString Name;
        float IntervalSeconds;
        double LastRunTime;
        FHealthCheckResult LastResult;
    };

    TMap<FString, FHealthCheckEntry> RegisteredChecks;
    FServerHealthReport LatestReport;
    FString ServerId;
    FDateTime StartTime;
    FTimerHandle MonitoringTimerHandle;

    // 内置检查
    FHealthCheckResult CheckCpuHealth();
    FHealthCheckResult CheckMemoryHealth();
    FHealthCheckResult CheckNetworkHealth();
    FHealthCheckResult CheckDatabaseHealth();
    FHealthCheckResult CheckGameSessionHealth();
    FHealthCheckResult CheckDiskSpaceHealth();

    void RunMonitoringTick();
    EHealthStatus DetermineOverallStatus();
    void UpdateResourceMetrics();
    void UpdateNetworkMetrics();
};
```

```csharp
// HealthMonitorSubsystem.cpp
#include "HealthMonitorSubsystem.h"
#include "HAL/PlatformMemory.h"
#include "HAL/PlatformProcess.h"
#include "Engine/World.h"
#include "TimerManager.h"

void UHealthMonitorSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    StartTime = FDateTime::UtcNow();
    ServerId = FPlatformProcess::ComputerName() + "_" + FGuid::NewGuid().ToString().Left(8);

    // 注册默认检查
    RegisterHealthCheck(TEXT("cpu"), 30.0f);
    RegisterHealthCheck(TEXT("memory"), 30.0f);
    RegisterHealthCheck(TEXT("network"), 60.0f);
    RegisterHealthCheck(TEXT("database"), 60.0f);
    RegisterHealthCheck(TEXT("game_sessions"), 30.0f);
    RegisterHealthCheck(TEXT("disk_space"), 300.0f);

    // 启动监控
    if (UWorld* World = GetWorld())
    {
        World->GetTimerManager().SetTimer(
            MonitoringTimerHandle,
            this,
            &UHealthMonitorSubsystem::RunMonitoringTick,
            10.0f,
            true
        );
    }
}

void UHealthMonitorSubsystem::Deinitialize()
{
    if (UWorld* World = GetWorld())
    {
        World->GetTimerManager().ClearTimer(MonitoringTimerHandle);
    }

    Super::Deinitialize();
}

void UHealthMonitorSubsystem::RegisterHealthCheck(const FString& CheckName, float CheckIntervalSeconds)
{
    FHealthCheckEntry Entry;
    Entry.Name = CheckName;
    Entry.IntervalSeconds = CheckIntervalSeconds;
    Entry.LastRunTime = 0.0;

    RegisteredChecks.Add(CheckName, Entry);
}

void UHealthMonitorSubsystem::UnregisterHealthCheck(const FString& CheckName)
{
    RegisteredChecks.Remove(CheckName);
}

FServerHealthReport UHealthMonitorSubsystem::RunAllChecks()
{
    FServerHealthReport Report;
    Report.ReportTime = FPlatformTime::Seconds();
    Report.ServerId = ServerId;
    Report.ServerVersion = "1.0.0";  // 从配置读取

    // 计算运行时间
    FTimespan Uptime = FDateTime::UtcNow() - StartTime;
    Report.UptimeSeconds = Uptime.GetTotalSeconds();

    // 更新资源指标
    UpdateResourceMetrics();
    Report.CpuUsagePercent = LatestReport.CpuUsagePercent;
    Report.MemoryUsagePercent = LatestReport.MemoryUsagePercent;
    Report.MemoryUsedMB = LatestReport.MemoryUsedMB;
    Report.MemoryTotalMB = LatestReport.MemoryTotalMB;

    // 更新网络指标
    UpdateNetworkMetrics();
    Report.ConnectedPlayers = LatestReport.ConnectedPlayers;
    Report.NetworkInKBps = LatestReport.NetworkInKBps;
    Report.NetworkOutKBps = LatestReport.NetworkOutKBps;

    // 执行各检查
    double CurrentTime = FPlatformTime::Seconds();

    for (auto& Pair : RegisteredChecks)
    {
        FHealthCheckEntry& Entry = Pair.Value;

        // 检查是否需要运行
        if (CurrentTime - Entry.LastRunTime >= Entry.IntervalSeconds)
        {
            Entry.LastResult = RunCheck(Entry.Name);
            Entry.LastRunTime = CurrentTime;
        }

        Report.Checks.Add(Entry.LastResult);
    }

    // 确定总体状态
    Report.OverallStatus = DetermineOverallStatus();

    // 检查状态变化
    if (Report.OverallStatus != LatestReport.OverallStatus)
    {
        OnStatusChanged.Broadcast(Report);
    }

    LatestReport = Report;
    return Report;
}

FHealthCheckResult UHealthMonitorSubsystem::RunCheck(const FString& CheckName)
{
    FHealthCheckResult Result;
    Result.CheckName = CheckName;
    Result.LastCheckTime = FPlatformTime::Seconds();

    double StartTime = FPlatformTime::Seconds();

    if (CheckName == TEXT("cpu"))
    {
        Result = CheckCpuHealth();
    }
    else if (CheckName == TEXT("memory"))
    {
        Result = CheckMemoryHealth();
    }
    else if (CheckName == TEXT("network"))
    {
        Result = CheckNetworkHealth();
    }
    else if (CheckName == TEXT("database"))
    {
        Result = CheckDatabaseHealth();
    }
    else if (CheckName == TEXT("game_sessions"))
    {
        Result = CheckGameSessionHealth();
    }
    else if (CheckName == TEXT("disk_space"))
    {
        Result = CheckDiskSpaceHealth();
    }
    else
    {
        Result.Status = EHealthStatus::Unknown;
        Result.Message = TEXT("Unknown health check");
    }

    Result.ResponseTimeMs = (FPlatformTime::Seconds() - StartTime) * 1000.0f;

    // 检查失败通知
    if (Result.Status == EHealthStatus::Unhealthy)
    {
        OnCheckFailed.Broadcast(CheckName);
    }

    return Result;
}

FHealthCheckResult UHealthMonitorSubsystem::CheckCpuHealth()
{
    FHealthCheckResult Result;
    Result.CheckName = TEXT("cpu");

    // 获取CPU使用率
    float CpuUsage = 0.0f;
    // FPlatformProcess::GetCPUUsage(CpuUsage);  // 平台特定

    // 简化实现
    CpuUsage = LatestReport.CpuUsagePercent;

    if (CpuUsage >= CpuCriticalThreshold)
    {
        Result.Status = EHealthStatus::Unhealthy;
        Result.Message = FString::Printf(TEXT("CPU usage critical: %.1f%%"), CpuUsage);
    }
    else if (CpuUsage >= CpuWarningThreshold)
    {
        Result.Status = EHealthStatus::Degraded;
        Result.Message = FString::Printf(TEXT("CPU usage high: %.1f%%"), CpuUsage);
    }
    else
    {
        Result.Status = EHealthStatus::Healthy;
        Result.Message = FString::Printf(TEXT("CPU usage normal: %.1f%%"), CpuUsage);
    }

    return Result;
}

FHealthCheckResult UHealthMonitorSubsystem::CheckMemoryHealth()
{
    FHealthCheckResult Result;
    Result.CheckName = TEXT("memory");

    FPlatformMemoryStats MemStats = FPlatformMemory::GetStats();
    float MemUsagePercent = (float)MemStats.UsedPhysical / MemStats.TotalPhysical * 100.0f;

    if (MemUsagePercent >= MemoryCriticalThreshold)
    {
        Result.Status = EHealthStatus::Unhealthy;
        Result.Message = FString::Printf(TEXT("Memory usage critical: %.1f%%"), MemUsagePercent);
    }
    else if (MemUsagePercent >= MemoryWarningThreshold)
    {
        Result.Status = EHealthStatus::Degraded;
        Result.Message = FString::Printf(TEXT("Memory usage high: %.1f%%"), MemUsagePercent);
    }
    else
    {
        Result.Status = EHealthStatus::Healthy;
        Result.Message = FString::Printf(TEXT("Memory usage normal: %.1f%%"), MemUsagePercent);
    }

    return Result;
}

FHealthCheckResult UHealthMonitorSubsystem::CheckNetworkHealth()
{
    FHealthCheckResult Result;
    Result.CheckName = TEXT("network");

    // 检查网络连接状态
    // 这里可以检查到关键服务的连接

    Result.Status = EHealthStatus::Healthy;
    Result.Message = FString::Printf(
        TEXT("Network: %.1f KB/s in, %.1f KB/s out"),
        LatestReport.NetworkInKBps, LatestReport.NetworkOutKBps
    );

    return Result;
}

FHealthCheckResult UHealthMonitorSubsystem::CheckDatabaseHealth()
{
    FHealthCheckResult Result;
    Result.CheckName = TEXT("database");

    // 检查数据库连接
    // 执行简单查询测试延迟

    Result.Status = EHealthStatus::Healthy;
    Result.Message = TEXT("Database connection OK");

    return Result;
}

FHealthCheckResult UHealthMonitorSubsystem::CheckGameSessionHealth()
{
    FHealthCheckResult Result;
    Result.CheckName = TEXT("game_sessions");

    int32 ActiveSessions = 0;
    int32 ActivePlayers = 0;

    if (UWorld* World = GetWorld())
    {
        // 计算活跃会话
        for (auto It = World->GetPlayerControllerIterator(); It; ++It)
        {
            ActivePlayers++;
        }
    }

    Result.Status = EHealthStatus::Healthy;
    Result.Message = FString::Printf(
        TEXT("Active sessions: %d, Players: %d"),
        ActiveSessions, ActivePlayers
    );

    return Result;
}

FHealthCheckResult UHealthMonitorSubsystem::CheckDiskSpaceHealth()
{
    FHealthCheckResult Result;
    Result.CheckName = TEXT("disk_space");

    // 检查磁盘空间
    FString SaveDir = FPaths::ProjectSavedDir();
    int64 TotalSpace = 0;
    int64 FreeSpace = 0;

    // FPlatformMisc::GetDiskSpace(SaveDir, TotalSpace, FreeSpace);

    float FreePercent = (float)FreeSpace / TotalSpace * 100.0f;

    if (FreePercent < 5.0f)
    {
        Result.Status = EHealthStatus::Unhealthy;
        Result.Message = FString::Printf(TEXT("Disk space critical: %.1f%% free"), FreePercent);
    }
    else if (FreePercent < 15.0f)
    {
        Result.Status = EHealthStatus::Degraded;
        Result.Message = FString::Printf(TEXT("Disk space low: %.1f%% free"), FreePercent);
    }
    else
    {
        Result.Status = EHealthStatus::Healthy;
        Result.Message = FString::Printf(TEXT("Disk space OK: %.1f%% free"), FreePercent);
    }

    return Result;
}

EHealthStatus UHealthMonitorSubsystem::DetermineOverallStatus()
{
    EHealthStatus OverallStatus = EHealthStatus::Healthy;

    for (const auto& Pair : RegisteredChecks)
    {
        const FHealthCheckResult& CheckResult = Pair.Value.LastResult;

        if (CheckResult.Status == EHealthStatus::Unhealthy)
        {
            return EHealthStatus::Unhealthy;
        }
        else if (CheckResult.Status == EHealthStatus::Degraded)
        {
            OverallStatus = EHealthStatus::Degraded;
        }
    }

    return OverallStatus;
}

void UHealthMonitorSubsystem::UpdateResourceMetrics()
{
    FPlatformMemoryStats MemStats = FPlatformMemory::GetStats();

    LatestReport.MemoryTotalMB = MemStats.TotalPhysical / (1024 * 1024);
    LatestReport.MemoryUsedMB = MemStats.UsedPhysical / (1024 * 1024);
    LatestReport.MemoryUsagePercent = (float)MemStats.UsedPhysical / MemStats.TotalPhysical * 100.0f;

    // CPU使用率需要平台特定实现
    LatestReport.CpuUsagePercent = 25.0f;  // 简化
}

void UHealthMonitorSubsystem::UpdateNetworkMetrics()
{
    if (UWorld* World = GetWorld())
    {
        LatestReport.ConnectedPlayers = 0;

        for (auto It = World->GetPlayerControllerIterator(); It; ++It)
        {
            LatestReport.ConnectedPlayers++;
        }
    }
}

void UHealthMonitorSubsystem::RunMonitoringTick()
{
    RunAllChecks();
}

FString UHealthMonitorSubsystem::GetHealthEndpointResponse()
{
    FServerHealthReport Report = GetLatestReport();

    FString Json = FString::Printf(TEXT(
        "{"
        "\"status\":\"%s\","
        "\"serverId\":\"%s\","
        "\"uptime\":%d,"
        "\"checks\":["
    ),
        Report.OverallStatus == EHealthStatus::Healthy ? TEXT("healthy") :
        Report.OverallStatus == EHealthStatus::Degraded ? TEXT("degraded") : TEXT("unhealthy"),
        *Report.ServerId,
        Report.UptimeSeconds
    );

    for (int32 i = 0; i < Report.Checks.Num(); i++)
    {
        const FHealthCheckResult& Check = Report.Checks[i];
        Json += FString::Printf(TEXT(
            "%s{\"name\":\"%s\",\"status\":\"%s\",\"message\":\"%s\",\"responseTime\":%.2f}"
        ),
            i > 0 ? TEXT(",") : TEXT(""),
            *Check.CheckName,
            Check.Status == EHealthStatus::Healthy ? TEXT("healthy") :
            Check.Status == EHealthStatus::Degraded ? TEXT("degraded") : TEXT("unhealthy"),
            *Check.Message,
            Check.ResponseTimeMs
        );
    }

    Json += TEXT("]}");

    return Json;
}
```

---

## 30.2 日志聚合与分析

### 30.2.1 结构化日志系统

```csharp
// StructuredLogManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "StructuredLogManager.generated.h"

UENUM(BlueprintType)
enum class ELogLevel : uint8
{
    Debug,
    Info,
    Warning,
    Error,
    Critical
};

USTRUCT(BlueprintType)
struct FStructuredLogEntry
{
    GENERATED_BODY()

    UPROPERTY()
    FString LogId;

    UPROPERTY()
    FDateTime Timestamp;

    UPROPERTY()
    ELogLevel Level;

    UPROPERTY()
    FString Message;

    UPROPERTY()
    FString Source;

    UPROPERTY()
    FString Category;

    UPROPERTY()
    FString ServerId;

    UPROPERTY()
    FString TraceId;

    UPROPERTY()
    FString UserId;

    UPROPERTY()
    FString SessionId;

    UPROPERTY()
    TMap<FString, FString> Metadata;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnLogEntry, const FStructuredLogEntry&, Entry);

UCLASS()
class STRUCTUREDLOG_API UStructuredLogManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 记录日志
    UFUNCTION(BlueprintCallable, Category = "Logging")
    void Log(ELogLevel Level, const FString& Message, const FString& Category = TEXT(""), const TMap<FString, FString>& Metadata = TMap<FString, FString>());

    // 便捷方法
    void Debug(const FString& Message, const FString& Category = TEXT(""));
    void Info(const FString& Message, const FString& Category = TEXT(""));
    void Warning(const FString& Message, const FString& Category = TEXT(""));
    void Error(const FString& Message, const FString& Category = TEXT(""));
    void Critical(const FString& Message, const FString& Category = TEXT(""));

    // 带上下文的日志
    void LogWithContext(ELogLevel Level, const FString& Message, const FString& Category, const FString& TraceId, const FString& UserId);

    // 查询日志
    UFUNCTION(BlueprintCallable, Category = "Logging")
    TArray<FStructuredLogEntry> QueryLogs(const FString& Filter, int32 MaxResults = 100);

    // 导出日志
    UFUNCTION(BlueprintCallable, Category = "Logging")
    void ExportLogs(const FString& FilePath, bool bJsonFormat = true);

    // 发送到远程
    UFUNCTION(BlueprintCallable, Category = "Logging")
    void FlushToRemote();

    // 配置
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Logging")
    ELogLevel MinLogLevel = ELogLevel::Info;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Logging")
    int32 MaxLocalLogs = 10000;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Logging")
    FString RemoteLogEndpoint;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Logging")
    bool bEnableRemoteLogging = false;

    UPROPERTY(BlueprintAssignable, Category = "Logging")
    FOnLogEntry OnLogEntry;

private:
    TArray<FStructuredLogEntry> LogBuffer;
    FString ServerId;
    FString CurrentTraceId;
    FString CurrentSessionId;
    FTimerHandle FlushTimerHandle;
    FCriticalSection BufferMutex;

    FString GenerateLogId();
    FString FormatAsJson(const FStructuredLogEntry& Entry);
    FString FormatAsText(const FStructuredLogEntry& Entry);
    void WriteToFile(const FString& FilePath, const FString& Content);
    void SendToRemoteEndpoint(const TArray<FStructuredLogEntry>& Entries);
    FString LogLevelToString(ELogLevel Level);
};
```

```csharp
// StructuredLogManager.cpp
#include "StructuredLogManager.h"
#include "Misc/FileHelper.h"
#include "Misc/Paths.h"
#include "HttpModule.h"
#include "JsonUtilities.h"

void UStructuredLogManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    ServerId = FPlatformProcess::ComputerName() + "_" + FGuid::NewGuid().ToString().Left(8);

    // 定时刷新到远程
    if (bEnableRemoteLogging && !RemoteLogEndpoint.IsEmpty())
    {
        if (UWorld* World = GetWorld())
        {
            World->GetTimerManager().SetTimer(
                FlushTimerHandle,
                this,
                &UStructuredLogManager::FlushToRemote,
                10.0f,
                true
            );
        }
    }
}

void UStructuredLogManager::Deinitialize()
{
    if (UWorld* World = GetWorld())
    {
        World->GetTimerManager().ClearTimer(FlushTimerHandle);
    }

    // 最终刷新
    if (bEnableRemoteLogging)
    {
        FlushToRemote();
    }

    Super::Deinitialize();
}

void UStructuredLogManager::Log(ELogLevel Level, const FString& Message, const FString& Category, const TMap<FString, FString>& Metadata)
{
    if (Level < MinLogLevel)
    {
        return;
    }

    FStructuredLogEntry Entry;
    Entry.LogId = GenerateLogId();
    Entry.Timestamp = FDateTime::UtcNow();
    Entry.Level = Level;
    Entry.Message = Message;
    Entry.Category = Category;
    Entry.ServerId = ServerId;
    Entry.TraceId = CurrentTraceId;
    Entry.SessionId = CurrentSessionId;
    Entry.Metadata = Metadata;

    {
        FScopeLock Lock(&BufferMutex);
        LogBuffer.Add(Entry);

        // 限制缓冲区大小
        if (LogBuffer.Num() > MaxLocalLogs)
        {
            LogBuffer.RemoveAt(0, LogBuffer.Num() - MaxLocalLogs);
        }
    }

    // 广播事件
    OnLogEntry.Broadcast(Entry);

    // 同时输出到UE日志
    FString Formatted = FString::Printf(TEXT("[%s] %s: %s"),
        *Entry.LogId, *LogLevelToString(Level), *Message);

    switch (Level)
    {
        case ELogLevel::Debug:
            UE_LOG(LogTemp, Verbose, TEXT("%s"), *Formatted);
            break;
        case ELogLevel::Info:
            UE_LOG(LogTemp, Log, TEXT("%s"), *Formatted);
            break;
        case ELogLevel::Warning:
            UE_LOG(LogTemp, Warning, TEXT("%s"), *Formatted);
            break;
        case ELogLevel::Error:
            UE_LOG(LogTemp, Error, TEXT("%s"), *Formatted);
            break;
        case ELogLevel::Critical:
            UE_LOG(LogTemp, Error, TEXT("[CRITICAL] %s"), *Formatted);
            break;
    }
}

void UStructuredLogManager::Debug(const FString& Message, const FString& Category)
{
    Log(ELogLevel::Debug, Message, Category);
}

void UStructuredLogManager::Info(const FString& Message, const FString& Category)
{
    Log(ELogLevel::Info, Message, Category);
}

void UStructuredLogManager::Warning(const FString& Message, const FString& Category)
{
    Log(ELogLevel::Warning, Message, Category);
}

void UStructuredLogManager::Error(const FString& Message, const FString& Category)
{
    Log(ELogLevel::Error, Message, Category);
}

void UStructuredLogManager::Critical(const FString& Message, const FString& Category)
{
    Log(ELogLevel::Critical, Message, Category);
}

void UStructuredLogManager::LogWithContext(ELogLevel Level, const FString& Message, const FString& Category, const FString& TraceId, const FString& UserId)
{
    TMap<FString, FString> Metadata;
    Metadata.Add("userId", UserId);

    FString OldTraceId = CurrentTraceId;
    CurrentTraceId = TraceId;

    Log(Level, Message, Category, Metadata);

    CurrentTraceId = OldTraceId;
}

TArray<FStructuredLogEntry> UStructuredLogManager::QueryLogs(const FString& Filter, int32 MaxResults)
{
    TArray<FStructuredLogEntry> Results;

    FScopeLock Lock(&BufferMutex);

    for (int32 i = LogBuffer.Num() - 1; i >= 0 && Results.Num() < MaxResults; i--)
    {
        const FStructuredLogEntry& Entry = LogBuffer[i];

        if (Filter.IsEmpty() ||
            Entry.Message.Contains(Filter) ||
            Entry.Category.Contains(Filter))
        {
            Results.Add(Entry);
        }
    }

    return Results;
}

void UStructuredLogManager::ExportLogs(const FString& FilePath, bool bJsonFormat)
{
    FString Content;

    FScopeLock Lock(&BufferMutex);

    if (bJsonFormat)
    {
        Content = TEXT("[\n");
        for (int32 i = 0; i < LogBuffer.Num(); i++)
        {
            Content += FormatAsJson(LogBuffer[i]);
            if (i < LogBuffer.Num() - 1)
            {
                Content += TEXT(",");
            }
            Content += TEXT("\n");
        }
        Content += TEXT("]");
    }
    else
    {
        for (const FStructuredLogEntry& Entry : LogBuffer)
        {
            Content += FormatAsText(Entry) + TEXT("\n");
        }
    }

    WriteToFile(FilePath, Content);
}

void UStructuredLogManager::FlushToRemote()
{
    TArray<FStructuredLogEntry> EntriesToSend;

    {
        FScopeLock Lock(&BufferMutex);
        EntriesToSend = LogBuffer;
        LogBuffer.Empty();
    }

    if (EntriesToSend.Num() > 0)
    {
        SendToRemoteEndpoint(EntriesToSend);
    }
}

FString UStructuredLogManager::GenerateLogId()
{
    return FGuid::NewGuid().ToString();
}

FString UStructuredLogManager::FormatAsJson(const FStructuredLogEntry& Entry)
{
    TSharedPtr<FJsonObject> JsonObject = MakeShareable(new FJsonObject);

    JsonObject->SetStringField("logId", Entry.LogId);
    JsonObject->SetStringField("timestamp", Entry.Timestamp.ToIso8601());
    JsonObject->SetStringField("level", LogLevelToString(Entry.Level));
    JsonObject->SetStringField("message", Entry.Message);
    JsonObject->SetStringField("category", Entry.Category);
    JsonObject->SetStringField("serverId", Entry.ServerId);

    if (!Entry.TraceId.IsEmpty())
    {
        JsonObject->SetStringField("traceId", Entry.TraceId);
    }
    if (!Entry.UserId.IsEmpty())
    {
        JsonObject->SetStringField("userId", Entry.UserId);
    }
    if (!Entry.SessionId.IsEmpty())
    {
        JsonObject->SetStringField("sessionId", Entry.SessionId);
    }

    // 添加元数据
    TSharedPtr<FJsonObject> MetaObject = MakeShareable(new FJsonObject);
    for (const auto& Pair : Entry.Metadata)
    {
        MetaObject->SetStringField(Pair.Key, Pair.Value);
    }
    JsonObject->SetObjectField("metadata", MetaObject);

    FString Output;
    TJsonWriterFactory<>::Create(&Output)->WriteObject(JsonObject.ToSharedRef());

    return Output;
}

FString UStructuredLogManager::FormatAsText(const FStructuredLogEntry& Entry)
{
    return FString::Printf(TEXT("[%s] [%s] [%s] %s"),
        *Entry.Timestamp.ToString(),
        *LogLevelToString(Entry.Level),
        *Entry.Category,
        *Entry.Message
    );
}

void UStructuredLogManager::WriteToFile(const FString& FilePath, const FString& Content)
{
    FFileHelper::SaveStringToFile(Content, *FilePath);
}

void UStructuredLogManager::SendToRemoteEndpoint(const TArray<FStructuredLogEntry>& Entries)
{
    if (RemoteLogEndpoint.IsEmpty())
    {
        return;
    }

    // 构建批量JSON
    FString JsonBody = TEXT("{\"logs\":[");
    for (int32 i = 0; i < Entries.Num(); i++)
    {
        JsonBody += FormatAsJson(Entries[i]);
        if (i < Entries.Num() - 1)
        {
            JsonBody += TEXT(",");
        }
    }
    JsonBody += TEXT("]}");

    TSharedRef<IHttpRequest> Request = FHttpModule::Get().CreateRequest();
    Request->SetURL(RemoteLogEndpoint);
    Request->SetVerb("POST");
    Request->SetHeader("Content-Type", "application/json");
    Request->SetHeader("X-Server-Id", ServerId);
    Request->SetContentAsString(JsonBody);

    Request->OnProcessRequestComplete().BindLambda([](FHttpRequestPtr Req, FHttpResponsePtr Resp, bool bSuccess)
    {
        if (!bSuccess)
        {
            UE_LOG(LogTemp, Warning, TEXT("Failed to send logs to remote endpoint"));
        }
    });

    Request->ProcessRequest();
}

FString UStructuredLogManager::LogLevelToString(ELogLevel Level)
{
    switch (Level)
    {
        case ELogLevel::Debug: return TEXT("DEBUG");
        case ELogLevel::Info: return TEXT("INFO");
        case ELogLevel::Warning: return TEXT("WARNING");
        case ELogLevel::Error: return TEXT("ERROR");
        case ELogLevel::Critical: return TEXT("CRITICAL");
        default: return TEXT("UNKNOWN");
    }
}
```

---

## 30.3 告警系统设计

### 30.3.1 多通道告警系统

```csharp
// AlertSystem.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "AlertSystem.generated.h"

UENUM(BlueprintType)
enum class EAlertSeverity : uint8
{
    Info,
    Warning,
    Error,
    Critical
};

UENUM(BlueprintType)
enum class EAlertChannel : uint8
{
    Email,
    Slack,
    Discord,
    Webhook,
    SMS,
    Push
};

USTRUCT(BlueprintType)
struct FAlert
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString AlertId;

    UPROPERTY(BlueprintReadOnly)
    FString Title;

    UPROPERTY(BlueprintReadOnly)
    FString Message;

    UPROPERTY(BlueprintReadOnly)
    EAlertSeverity Severity = EAlertSeverity::Info;

    UPROPERTY(BlueprintReadOnly)
    FString Source;

    UPROPERTY(BlueprintReadOnly)
    TMap<FString, FString> Metadata;

    UPROPERTY(BlueprintReadOnly)
    FDateTime CreatedAt;

    UPROPERTY(BlueprintReadOnly)
    bool bAcknowledged = false;

    UPROPERTY(BlueprintReadOnly)
    int32 OccurrenceCount = 1;
};

USTRUCT(BlueprintType)
struct FAlertRule
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString RuleName;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString Condition;  // 表达式如 "cpu_usage > 90"

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    EAlertSeverity Severity = EAlertSeverity::Warning;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float CooldownMinutes = 5.0f;  // 相同告警冷却时间

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TArray<EAlertChannel> Channels;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString MessageTemplate;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bEnabled = true;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnAlertTriggered, const FAlert&, Alert);

UCLASS()
class ALERTSYSTEM_API UAlertSystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 触发告警
    UFUNCTION(BlueprintCallable, Category = "Alerts")
    void TriggerAlert(const FString& Title, const FString& Message, EAlertSeverity Severity, const TMap<FString, FString>& Metadata = TMap<FString, FString>());

    // 添加规则
    UFUNCTION(BlueprintCallable, Category = "Alerts")
    void AddAlertRule(const FAlertRule& Rule);

    // 评估规则
    UFUNCTION(BlueprintCallable, Category = "Alerts")
    void EvaluateRules(const TMap<FString, float>& Metrics);

    // 确认告警
    UFUNCTION(BlueprintCallable, Category = "Alerts")
    void AcknowledgeAlert(const FString& AlertId);

    // 获取活跃告警
    UFUNCTION(BlueprintCallable, Category = "Alerts")
    TArray<FAlert> GetActiveAlerts() const;

    // 发送测试告警
    UFUNCTION(BlueprintCallable, Category = "Alerts")
    void SendTestAlert();

    UPROPERTY(BlueprintAssignable, Category = "Alerts")
    FOnAlertTriggered OnAlertTriggered;

    // 通道配置
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Alerts")
    FString SlackWebhookUrl;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Alerts")
    FString DiscordWebhookUrl;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Alerts")
    FString EmailRecipients;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Alerts")
    FString GenericWebhookUrl;

private:
    TArray<FAlertRule> AlertRules;
    TArray<FAlert> ActiveAlerts;
    TMap<FString, double> LastAlertTime;  // 冷却追踪

    FString GenerateAlertId();
    bool CheckCooldown(const FString& RuleName);
    void SendToChannel(EAlertChannel Channel, const FAlert& Alert);
    void SendToSlack(const FAlert& Alert);
    void SendToDiscord(const FAlert& Alert);
    void SendToEmail(const FAlert& Alert);
    void SendToWebhook(const FAlert& Alert);
    FString SeverityToColor(EAlertSeverity Severity) const;
};
```

---

## 30.4 自动化运维脚本

### 30.4.1 运维自动化管理器

```csharp
// OperationsAutomationManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "OperationsAutomationManager.generated.h"

UENUM(BlueprintType)
enum class EMaintenanceTaskType : uint8
{
    Cleanup,
    Backup,
    Restart,
    Update,
    Scale,
    Custom
};

USTRUCT(BlueprintType)
struct FMaintenanceTask
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString TaskName;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    EMaintenanceTaskType Type;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString Schedule;  // Cron表达式

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString ScriptPath;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TMap<FString, FString> Parameters;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bEnabled = true;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 MaxRetries = 3;
};

USTRUCT(BlueprintType)
struct FMaintenanceResult
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString TaskName;

    UPROPERTY(BlueprintReadOnly)
    bool bSuccess = false;

    UPROPERTY(BlueprintReadOnly)
    FString Output;

    UPROPERTY(BlueprintReadOnly)
    FString Error;

    UPROPERTY(BlueprintReadOnly)
    float DurationSeconds = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    FDateTime ExecutionTime;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnMaintenanceComplete, const FMaintenanceResult&, Result);

UCLASS()
class OPSAUTOMATION_API UOperationsAutomationManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 注册维护任务
    UFUNCTION(BlueprintCallable, Category = "Operations")
    void RegisterTask(const FMaintenanceTask& Task);

    // 执行任务
    UFUNCTION(BlueprintCallable, Category = "Operations")
    void ExecuteTask(const FString& TaskName);

    // 执行脚本
    UFUNCTION(BlueprintCallable, Category = "Operations")
    FMaintenanceResult ExecuteScript(const FString& ScriptPath, const TMap<FString, FString>& Parameters);

    // 获取任务状态
    UFUNCTION(BlueprintPure, Category = "Operations")
    bool IsTaskRunning(const FString& TaskName) const;

    // 获取任务历史
    UFUNCTION(BlueprintCallable, Category = "Operations")
    TArray<FMaintenanceResult> GetTaskHistory(const FString& TaskName) const;

    // 内置维护操作
    UFUNCTION(BlueprintCallable, Category = "Operations")
    void PerformCleanup();

    UFUNCTION(BlueprintCallable, Category = "Operations")
    void PerformBackup(const FString& BackupPath);

    UFUNCTION(BlueprintCallable, Category = "Operations")
    void PerformLogRotation();

    UPROPERTY(BlueprintAssignable, Category = "Operations")
    FOnMaintenanceComplete OnMaintenanceComplete;

private:
    TMap<FString, FMaintenanceTask> RegisteredTasks;
    TMap<FString, TArray<FMaintenanceResult>> TaskHistory;
    TSet<FString> RunningTasks;
    FTimerHandle SchedulerTimerHandle;

    void CheckScheduledTasks();
    bool ShouldRunTask(const FMaintenanceTask& Task);
    FString ParseCronExpression(const FString& CronExpr);
};
```

---

## 30.5 实践任务

### 任务1：实现健康监控仪表盘

创建服务器健康监控系统：
- CPU/内存监控
- 网络状态监控
- 健康检查端点

### 任务2：配置告警系统

设置多通道告警：
- Slack集成
- 邮件通知
- 自定义Webhook

### 任务3：编写自动化脚本

创建维护自动化：
- 日志清理脚本
- 自动备份任务
- 定时重启脚本

---

## 30.6 总结

本课学习了：
- 服务器健康监控系统
- 结构化日志管理
- 多通道告警系统
- 自动化运维脚本
- 故障排查流程

**下一阶段预告**：实战项目 - 多人在线游戏完整开发流程。

---

*课程版本：1.0*
*最后更新：2026-04-10*
