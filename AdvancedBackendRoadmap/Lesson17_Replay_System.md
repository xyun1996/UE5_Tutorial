# 第17课：回放系统

> 本课深入讲解UE5的网络回放系统，包括录制、回放实现、数据存储和性能优化。

---

## 课程目标

- 理解网络回放架构原理
- 掌握录制与回放实现方法
- 学会回放数据存储管理
- 掌握回放编辑与剪辑技术
- 学会回放系统性能优化

---

## 一、网络回放架构

### 1.1 回放系统概述

```
┌─────────────────────────────────────────────────────────────┐
│                    回放系统架构                              │
│                                                              │
│  录制阶段                                                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  游戏运行                                            │    │
│  │     │                                                │    │
│  │     ▼                                                │    │
│  │  ┌─────────────┐                                    │    │
│  │  │ 帧数据收集  │                                    │    │
│  │  │ - 网络包    │                                    │    │
│  │  │ - Actor状态│                                    │    │
│  │  │ - RPC调用  │                                    │    │
│  │  └──────┬──────┘                                    │    │
│  │         │                                            │    │
│  │         ▼                                            │    │
│  │  ┌─────────────┐                                    │    │
│  │  │ 数据序列化  │                                    │    │
│  │  │ - 压缩      │                                    │    │
│  │  │ - 索引构建  │                                    │    │
│  │  └──────┬──────┘                                    │    │
│  │         │                                            │    │
│  │         ▼                                            │    │
│  │  ┌─────────────┐                                    │    │
│  │  │ 文件存储    │                                    │    │
│  │  └─────────────┘                                    │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  回放阶段                                                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  加载回放文件                                        │    │
│  │     │                                                │    │
│  │     ▼                                                │    │
│  │  ┌─────────────┐                                    │    │
│  │  │ 数据反序列化│                                    │    │
│  │  └──────┬──────┘                                    │    │
│  │         │                                            │    │
│  │         ▼                                            │    │
│  │  ┌─────────────┐                                    │    │
│  │  │ 时间轴控制  │                                    │    │
│  │  │ - 播放      │                                    │    │
│  │  │ - 暂停      │                                    │    │
│  │  │ - 跳转      │                                    │    │
│  │  │ - 倍速      │                                    │    │
│  │  └──────┬──────┘                                    │    │
│  │         │                                            │    │
│  │         ▼                                            │    │
│  │  ┌─────────────┐                                    │    │
│  │  │ 状态重现    │                                    │    │
│  │  └─────────────┘                                    │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、录制实现

### 2.1 回放录制管理器

```cpp
// MyReplayRecorder.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "MyReplayRecorder.generated.h"

// 录制配置
USTRUCT(BlueprintType)
struct FReplayConfig
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString ReplayName;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString FriendlyName;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bRecordAudio = false;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 MaxDurationMinutes = 30;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bCompressData = true;
};

// 录制状态
UENUM(BlueprintType)
enum class ERecordingState : uint8
{
    Idle,
    Recording,
    Paused,
    Stopped
};

// 回放信息
USTRUCT(BlueprintType)
struct FReplayInfo
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString ReplayId;

    UPROPERTY(BlueprintReadOnly)
    FString Name;

    UPROPERTY(BlueprintReadOnly)
    FString MapName;

    UPROPERTY(BlueprintReadOnly)
    FDateTime RecordTime;

    UPROPERTY(BlueprintReadOnly)
    float Duration;

    UPROPERTY(BlueprintReadOnly)
    int64 FileSize;

    UPROPERTY(BlueprintReadOnly)
    TArray<FString> Players;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnRecordingStateChanged, ERecordingState, NewState);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnRecordingComplete, const FReplayInfo&, ReplayInfo);

UCLASS()
class UMyReplayRecorder : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 录制控制
    UFUNCTION(BlueprintCallable, Category = "Replay")
    bool StartRecording(const FReplayConfig& Config);

    UFUNCTION(BlueprintCallable, Category = "Replay")
    void StopRecording();

    UFUNCTION(BlueprintCallable, Category = "Replay")
    void PauseRecording();

    UFUNCTION(BlueprintCallable, Category = "Replay")
    void ResumeRecording();

    // 状态查询
    UFUNCTION(BlueprintPure, Category = "Replay")
    ERecordingState GetRecordingState() const { return CurrentState; }

    UFUNCTION(BlueprintPure, Category = "Replay")
    float GetCurrentRecordingTime() const;

    UFUNCTION(BlueprintPure, Category = "Replay")
    FString GetCurrentReplayName() const { return CurrentReplayName; }

    // 回放管理
    UFUNCTION(BlueprintCallable, Category = "Replay")
    TArray<FReplayInfo> GetAvailableReplays();

    UFUNCTION(BlueprintCallable, Category = "Replay")
    bool DeleteReplay(const FString& ReplayId);

    UFUNCTION(BlueprintCallable, Category = "Replay")
    void RenameReplay(const FString& ReplayId, const FString& NewName);

    // 委托
    UPROPERTY(BlueprintAssignable)
    FOnRecordingStateChanged OnStateChanged;

    UPROPERTY(BlueprintAssignable)
    FOnRecordingComplete OnRecordingComplete;

protected:
    ERecordingState CurrentState = ERecordingState::Idle;
    FString CurrentReplayName;
    float RecordingStartTime;
    FString ReplayDirectory;

    // 内部方法
    void SetupReplayDirectory();
    FString GenerateReplayId() const;
    void SaveReplayMetadata(const FString& ReplayId, const FReplayConfig& Config, float Duration);
    FString GetReplayFilePath(const FString& ReplayId) const;
};

// MyReplayRecorder.cpp
#include "MyReplayRecorder.h"
#include "GameFramework/PlayerController.h"
#include "HAL/PlatformFileManager.h"

void UMyReplayRecorder::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    SetupReplayDirectory();
}

void UMyReplayRecorder::Deinitialize()
{
    if (CurrentState == ERecordingState::Recording)
    {
        StopRecording();
    }

    Super::Deinitialize();
}

void UMyReplayRecorder::SetupReplayDirectory()
{
    ReplayDirectory = FPaths::ProjectSavedDir() / TEXT("Replays");

    IPlatformFile& PlatformFile = FPlatformFileManager::Get().GetPlatformFile();
    if (!PlatformFile.DirectoryExists(*ReplayDirectory))
    {
        PlatformFile.CreateDirectory(*ReplayDirectory);
    }
}

bool UMyReplayRecorder::StartRecording(const FReplayConfig& Config)
{
    if (CurrentState == ERecordingState::Recording)
    {
        UE_LOG(LogTemp, Warning, TEXT("Already recording"));
        return false;
    }

    CurrentReplayName = Config.ReplayName.IsEmpty() ? GenerateReplayId() : Config.ReplayName;

    // 使用UE内置回放系统
    UGameInstance* GI = GetGameInstance();
    if (GI && GI->GetWorld())
    {
        // 启动录制
        FString ReplayName = CurrentReplayName;

        // 使用DemoRecording
        // UGameplayStatics::StartRecordingReplay(GI, ReplayName, Config.FriendlyName);
    }

    RecordingStartTime = GetWorld()->GetTimeSeconds();
    CurrentState = ERecordingState::Recording;
    OnStateChanged.Broadcast(CurrentState);

    UE_LOG(LogTemp, Log, TEXT("Started recording: %s"), *CurrentReplayName);
    return true;
}

void UMyReplayRecorder::StopRecording()
{
    if (CurrentState != ERecordingState::Recording && CurrentState != ERecordingState::Paused)
    {
        return;
    }

    float Duration = GetWorld()->GetTimeSeconds() - RecordingStartTime;

    // 停止录制
    UGameInstance* GI = GetGameInstance();
    if (GI)
    {
        // UGameplayStatics::StopRecordingReplay(GI);
    }

    // 保存元数据
    FReplayConfig Config;
    Config.ReplayName = CurrentReplayName;
    SaveReplayMetadata(CurrentReplayName, Config, Duration);

    CurrentState = ERecordingState::Stopped;
    OnStateChanged.Broadcast(CurrentState);

    FReplayInfo Info;
    Info.ReplayId = CurrentReplayName;
    Info.Name = CurrentReplayName;
    Info.Duration = Duration;
    OnRecordingComplete.Broadcast(Info);

    UE_LOG(LogTemp, Log, TEXT("Stopped recording: %s, Duration: %.2f"), *CurrentReplayName, Duration);

    CurrentState = ERecordingState::Idle;
    OnStateChanged.Broadcast(CurrentState);
}

void UMyReplayRecorder::PauseRecording()
{
    if (CurrentState != ERecordingState::Recording)
    {
        return;
    }

    CurrentState = ERecordingState::Paused;
    OnStateChanged.Broadcast(CurrentState);
}

void UMyReplayRecorder::ResumeRecording()
{
    if (CurrentState != ERecordingState::Paused)
    {
        return;
    }

    CurrentState = ERecordingState::Recording;
    OnStateChanged.Broadcast(CurrentState);
}

float UMyReplayRecorder::GetCurrentRecordingTime() const
{
    if (CurrentState == ERecordingState::Recording || CurrentState == ERecordingState::Paused)
    {
        return GetWorld()->GetTimeSeconds() - RecordingStartTime;
    }
    return 0.0f;
}

TArray<FReplayInfo> UMyReplayRecorder::GetAvailableReplays()
{
    TArray<FReplayInfo> Replays;

    IPlatformFile& PlatformFile = FPlatformFileManager::Get().GetPlatformFile();
    TArray<FString> Files;
    PlatformFile.FindFilesRecursively(Files, *ReplayDirectory, TEXT(".replay"));

    for (const FString& File : Files)
    {
        FReplayInfo Info;
        Info.ReplayId = FPaths::GetBaseFilename(File);
        Info.Name = Info.ReplayId;
        Info.FileSize = PlatformFile.FileSize(*File);

        // 从元数据加载更多信息
        // ...

        Replays.Add(Info);
    }

    return Replays;
}

bool UMyReplayRecorder::DeleteReplay(const FString& ReplayId)
{
    FString FilePath = GetReplayFilePath(ReplayId);

    IPlatformFile& PlatformFile = FPlatformFileManager::Get().GetPlatformFile();
    if (PlatformFile.FileExists(*FilePath))
    {
        return PlatformFile.DeleteFile(*FilePath);
    }

    return false;
}

FString UMyReplayRecorder::GenerateReplayId() const
{
    return FString::Printf(TEXT("Replay_%s"), *FDateTime::Now().ToString());
}

void UMyReplayRecorder::SaveReplayMetadata(const FString& ReplayId, const FReplayConfig& Config, float Duration)
{
    FString MetadataPath = GetReplayFilePath(ReplayId) + TEXT(".meta");

    TSharedPtr<FJsonObject> JsonObject = MakeShareable(new FJsonObject);
    JsonObject->SetStringField(TEXT("id"), ReplayId);
    JsonObject->SetStringField(TEXT("name"), Config.FriendlyName);
    JsonObject->SetNumberField(TEXT("duration"), Duration);
    JsonObject->SetStringField(TEXT("timestamp"), FDateTime::Now().ToString());

    FString OutputString;
    TJsonWriter<>::Create(&OutputString)->WriteObject(JsonObject.ToSharedRef());

    FFileHelper::SaveStringToFile(OutputString, *MetadataPath);
}

FString UMyReplayRecorder::GetReplayFilePath(const FString& ReplayId) const
{
    return ReplayDirectory / ReplayId + TEXT(".replay");
}
```

---

## 三、回放播放实现

### 3.1 回放播放器

```cpp
// MyReplayPlayer.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "MyReplayPlayer.generated.h"

// 播放状态
UENUM(BlueprintType)
enum class EPlaybackState : uint8
{
    Stopped,
    Playing,
    Paused,
    Ended
};

// 播放配置
USTRUCT(BlueprintType)
struct FPlaybackConfig
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float PlaySpeed = 1.0f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bLoop = false;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bShowHUD = true;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FVector SpectatorLocation;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FRotator SpectatorRotation;
};

// 关键帧标记
USTRUCT(BlueprintType)
struct FKeyframeMarker
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString Name;

    UPROPERTY(BlueprintReadOnly)
    float Timestamp;

    UPROPERTY(BlueprintReadOnly)
    FString Description;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnPlaybackStateChanged, EPlaybackState, NewState);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnPlaybackProgress, float, CurrentTime, float, TotalTime);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnReplayLoaded, bool, bSuccess);

UCLASS()
class UMyReplayPlayer : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 播放控制
    UFUNCTION(BlueprintCallable, Category = "Replay")
    bool LoadReplay(const FString& ReplayId);

    UFUNCTION(BlueprintCallable, Category = "Replay")
    void Play();

    UFUNCTION(BlueprintCallable, Category = "Replay")
    void Pause();

    UFUNCTION(BlueprintCallable, Category = "Replay")
    void Stop();

    UFUNCTION(BlueprintCallable, Category = "Replay")
    void Seek(float Time);

    UFUNCTION(BlueprintCallable, Category = "Replay")
    void SetPlaySpeed(float Speed);

    UFUNCTION(BlueprintCallable, Category = "Replay")
    void SkipToNextKeyframe();

    UFUNCTION(BlueprintCallable, Category = "Replay")
    void SkipToPreviousKeyframe();

    // 观察模式
    UFUNCTION(BlueprintCallable, Category = "Replay")
    void SpectatePlayer(const FString& PlayerId);

    UFUNCTION(BlueprintCallable, Category = "Replay")
    void SetFreeCamera(const FVector& Location, const FRotator& Rotation);

    // 状态查询
    UFUNCTION(BlueprintPure, Category = "Replay")
    EPlaybackState GetPlaybackState() const { return CurrentState; }

    UFUNCTION(BlueprintPure, Category = "Replay")
    float GetCurrentTime() const { return CurrentTime; }

    UFUNCTION(BlueprintPure, Category = "Replay")
    float GetTotalDuration() const { return TotalDuration; }

    UFUNCTION(BlueprintPure, Category = "Replay")
    float GetPlaySpeed() const { return CurrentConfig.PlaySpeed; }

    UFUNCTION(BlueprintPure, Category = "Replay")
    TArray<FKeyframeMarker> GetKeyframes() const { return Keyframes; }

    UFUNCTION(BlueprintPure, Category = "Replay")
    bool IsReplayLoaded() const { return bIsLoaded; }

    // 委托
    UPROPERTY(BlueprintAssignable)
    FOnPlaybackStateChanged OnStateChanged;

    UPROPERTY(BlueprintAssignable)
    FOnPlaybackProgress OnProgress;

    UPROPERTY(BlueprintAssignable)
    FOnReplayLoaded OnReplayLoaded;

protected:
    EPlaybackState CurrentState = EPlaybackState::Stopped;
    FPlaybackConfig CurrentConfig;
    float CurrentTime = 0.0f;
    float TotalDuration = 0.0f;
    bool bIsLoaded = false;

    TArray<FKeyframeMarker> Keyframes;
    FString CurrentReplayId;

    // 播放控制器
    UPROPERTY()
    APlayerController* SpectatorController;

    void UpdatePlayback(float DeltaTime);
    void LoadKeyframes(const FString& ReplayId);
    void ApplyPlaySpeed();
};
```

---

## 四、回放数据存储

### 4.1 回放数据管理

```cpp
// MyReplayDataManager.h
#pragma once

#include "CoreMinimal.h"
#include "MyReplayDataManager.generated.h"

// 回放数据块
USTRUCT()
struct FReplayDataChunk
{
    GENERATED_BODY()

    float Timestamp;
    TArray<uint8> Data;
    TArray<uint8> Checksum;
};

// 回放索引
USTRUCT()
struct FReplayIndex
{
    GENERATED_BODY()

    TArray<float> KeyframeTimes;
    TMap<float, int64> ChunkOffsets;
    TMap<float, FString> EventMarkers;
};

// 回放头部
USTRUCT()
struct FReplayHeader
{
    GENERATED_BODY()

    FString MagicNumber = TEXT("UE5REPLAY");
    int32 Version = 1;
    FString ReplayId;
    FString MapName;
    FString GameMode;
    FDateTime RecordTime;
    float Duration;
    int32 ChunkCount;
    TArray<FString> Players;
};

UCLASS()
class UMyReplayDataManager : public UObject
{
    GENERATED_BODY()

public:
    // 文件操作
    bool SaveReplay(const FString& Path, const TArray<FReplayDataChunk>& Chunks, const FReplayHeader& Header);
    bool LoadReplay(const FString& Path, TArray<FReplayDataChunk>& OutChunks, FReplayHeader& OutHeader);

    // 流式加载
    FReplayDataChunk LoadChunkAtTime(const FString& Path, float Time, const FReplayIndex& Index);
    TArray<FReplayDataChunk> LoadChunkRange(const FString& Path, float StartTime, float EndTime, const FReplayIndex& Index);

    // 索引操作
    bool BuildIndex(const FString& Path, FReplayIndex& OutIndex);
    bool SaveIndex(const FString& Path, const FReplayIndex& Index);
    bool LoadIndex(const FString& Path, FReplayIndex& OutIndex);

    // 压缩
    TArray<uint8> CompressData(const TArray<uint8>& Data);
    TArray<uint8> DecompressData(const TArray<uint8>& CompressedData);

    // 校验
    bool ValidateChunk(const FReplayDataChunk& Chunk);
    bool ValidateReplayFile(const FString& Path);

    // 云存储集成
    bool UploadToCloud(const FString& ReplayId, const FString& CloudPath);
    bool DownloadFromCloud(const FString& CloudPath, const FString& LocalPath);
    TArray<FString> ListCloudReplays();

private:
    FString ReplayDirectory;

    TArray<uint8> CalculateChecksum(const TArray<uint8>& Data);
    void WriteHeader(FArchive& Ar, const FReplayHeader& Header);
    void ReadHeader(FArchive& Ar, FReplayHeader& OutHeader);
};
```

---

## 五、回放编辑与剪辑

### 5.1 回放编辑器

```cpp
// MyReplayEditor.h
#pragma once

#include "CoreMinimal.h"
#include "MyReplayEditor.generated.h"

// 剪辑片段
USTRUCT(BlueprintType)
struct FReplayClip
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadWrite)
    FString ClipName;

    UPROPERTY(BlueprintReadWrite)
    float StartTime;

    UPROPERTY(BlueprintReadWrite)
    float EndTime;

    UPROPERTY(BlueprintReadWrite)
    FString Description;

    UPROPERTY(BlueprintReadWrite)
    TArray<FString> Tags;
};

// 编辑操作
UENUM(BlueprintType)
enum class EEditOperation : uint8
{
    Cut,
    Copy,
    Paste,
    Delete,
    Merge
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnEditComplete, bool, bSuccess);

UCLASS()
class UMyReplayEditor : public UObject
{
    GENERATED_BODY()

public:
    // 剪辑操作
    UFUNCTION(BlueprintCallable, Category = "Replay Editor")
    void CreateClip(const FString& ReplayId, float StartTime, float EndTime, const FString& ClipName);

    UFUNCTION(BlueprintCallable, Category = "Replay Editor")
    void DeleteClip(const FString& ReplayId, const FString& ClipName);

    UFUNCTION(BlueprintCallable, Category = "Replay Editor")
    TArray<FReplayClip> GetClips(const FString& ReplayId);

    // 编辑操作
    UFUNCTION(BlueprintCallable, Category = "Replay Editor")
    void CutSegment(const FString& SourceReplay, float StartTime, float EndTime);

    UFUNCTION(BlueprintCallable, Category = "Replay Editor")
    void MergeReplays(const TArray<FString>& ReplayIds, const FString& OutputName);

    UFUNCTION(BlueprintCallable, Category = "Replay Editor")
    void ExportClip(const FString& ReplayId, const FString& ClipName, const FString& OutputPath);

    // 标记操作
    UFUNCTION(BlueprintCallable, Category = "Replay Editor")
    void AddMarker(const FString& ReplayId, float Time, const FString& MarkerName);

    UFUNCTION(BlueprintCallable, Category = "Replay Editor")
    void RemoveMarker(const FString& ReplayId, const FString& MarkerName);

    // 导出
    UFUNCTION(BlueprintCallable, Category = "Replay Editor")
    void ExportToVideo(const FString& ReplayId, float StartTime, float EndTime, const FString& OutputPath);

    // 委托
    UPROPERTY(BlueprintAssignable)
    FOnEditComplete OnEditComplete;

protected:
    TMap<FString, TArray<FReplayClip>> ReplayClips;

    bool ValidateTimeRange(float StartTime, float EndTime);
    void CopyReplayData(const FString& Source, const FString& Destination, float StartTime, float EndTime);
};
```

---

## 六、总结

本课我们学习了：

1. **回放架构**：录制和播放流程
2. **录制实现**：控制和管理
3. **播放控制**：时间轴和速度
4. **数据存储**：压缩和索引
5. **编辑功能**：剪辑和导出

---

## 七、下节预告

**第18课：网络预测系统**

将深入学习：
- 客户端预测原理
- 服务器校正机制
- 移动预测实现

---

*课程版本：1.0*
*最后更新：2026-04-10*