# 第16课：映射系统（Map Transfer）

> 本课深入讲解UE5的地图切换和服务器旅行系统，包括无缝旅行、加载界面和多关卡服务器架构。

---

## 课程目标

- 掌握关卡切换与服务器旅行机制
- 学会实现无缝旅行（Seamless Travel）
- 理解加载进度与界面设计
- 掌握多关卡服务器架构
- 学会动态关卡流式加载

---

## 一、地图切换机制

### 1.1 地图切换类型

```
┌─────────────────────────────────────────────────────────────┐
│                    地图切换类型对比                          │
│                                                              │
│  硬切换（Hard Travel）                                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  特点：完全卸载当前地图，加载新地图                   │    │
│  │  流程：                                              │    │
│  │  1. 断开所有客户端连接                              │    │
│  │  2. 卸载当前World                                   │    │
│  │  3. 加载新World                                     │    │
│  │  4. 客户端重新连接                                  │    │
│  │  缺点：中断体验、黑屏时间长                          │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  无缝旅行（Seamless Travel）                                 │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  特点：保持连接，平滑过渡                             │    │
│  │  流程：                                              │    │
│  │  1. 通知客户端准备切换                              │    │
│  │  2. 客户端加载过渡地图                              │    │
│  │  3. 服务器切换到目标地图                            │    │
│  │  4. 客户端连接到新地图                              │    │
│  │  优点：体验流畅、保持连接                            │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  流式加载（Level Streaming）                                 │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  特点：动态加载/卸载关卡子集                          │    │
│  │  流程：                                              │    │
│  │  1. 主关卡始终加载                                  │    │
│  │  2. 根据玩家位置动态加载子关卡                      │    │
│  │  3. 远离子关卡自动卸载                              │    │
│  │  优点：无缝世界、内存优化                            │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、服务器旅行

### 2.1 基本旅行命令

```cpp
// MyTravelManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "MyTravelManager.generated.h"

// 旅行类型
UENUM(BlueprintType)
enum class ETravelType : uint8
{
    Absolute,       // 完全切换到新URL
    Partial,        // 部分切换（保留某些状态）
    Seamless        // 无缝旅行
};

// 旅行状态
UENUM(BlueprintType)
enum class ETravelState : uint8
{
    Idle,
    Preparing,
    InTransition,
    Loading,
    Complete,
    Failed
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnTravelStart, const FString&, Destination);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnTravelComplete, bool, bSuccess, const FString&, MapName);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnTravelProgress, float, Progress);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnTravelError, const FString&, ErrorMessage);

UCLASS()
class UMyTravelManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 服务器旅行（服务器调用）
    UFUNCTION(BlueprintCallable, Category = "Travel", meta = (WorldContext = "WorldContextObject"))
    void ServerTravel(UObject* WorldContextObject, const FString& MapName, bool bSeamless = true);

    // 客户端旅行
    UFUNCTION(BlueprintCallable, Category = "Travel")
    void ClientTravel(const FString& URL, ETravelType Type = ETravelType::Absolute);

    // 旅行到特定服务器
    UFUNCTION(BlueprintCallable, Category = "Travel")
    void TravelToServer(const FString& Address, int32 Port);

    // 取消旅行
    UFUNCTION(BlueprintCallable, Category = "Travel")
    void CancelTravel();

    // 状态查询
    UFUNCTION(BlueprintPure, Category = "Travel")
    ETravelState GetTravelState() const { return CurrentState; }

    UFUNCTION(BlueprintPure, Category = "Travel")
    FString GetCurrentDestination() const { return Destination; }

    // 委托
    UPROPERTY(BlueprintAssignable)
    FOnTravelStart OnTravelStart;

    UPROPERTY(BlueprintAssignable)
    FOnTravelComplete OnTravelComplete;

    UPROPERTY(BlueprintAssignable)
    FOnTravelProgress OnTravelProgress;

    UPROPERTY(BlueprintAssignable)
    FOnTravelError OnTravelError;

protected:
    ETravelState CurrentState = ETravelState::Idle;
    FString Destination;
    FTimerHandle TravelTimeoutHandle;

    void SetTravelState(ETravelState NewState);
    void OnTravelTimeout();
    FString BuildTravelURL(const FString& MapName, const FString& Options);
};

// MyTravelManager.cpp
#include "MyTravelManager.h"
#include "Kismet/GameplayStatics.h"
#include "GameFramework/PlayerController.h"

void UMyTravelManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
    CurrentState = ETravelState::Idle;
}

void UMyTravelManager::ServerTravel(UObject* WorldContextObject, const FString& MapName, bool bSeamless)
{
    UWorld* World = GEngine->GetWorldFromContextObject(WorldContextObject, EGetWorldErrorMode::LogAndReturnNull);
    if (!World)
    {
        OnTravelError.Broadcast(TEXT("Invalid world context"));
        return;
    }

    SetTravelState(ETravelState::Preparing);
    Destination = MapName;

    // 构建旅行URL
    FString TravelURL = BuildTravelURL(MapName, TEXT("Listen"));

    if (bSeamless)
    {
        // 无缝旅行
        World->ServerTravel(TravelURL + TEXT("?Seamless"));
    }
    else
    {
        // 硬切换
        World->ServerTravel(TravelURL);
    }

    OnTravelStart.Broadcast(MapName);

    // 设置超时
    GetWorld()->GetTimerManager().SetTimer(
        TravelTimeoutHandle,
        this,
        &UMyTravelManager::OnTravelTimeout,
        60.0f,
        false
    );
}

void UMyTravelManager::ClientTravel(const FString& URL, ETravelType Type)
{
    APlayerController* PC = GetWorld()->GetFirstPlayerController();
    if (!PC)
    {
        OnTravelError.Broadcast(TEXT("No player controller"));
        return;
    }

    SetTravelState(ETravelState::Preparing);
    Destination = URL;

    ETravelType::Type UnrealTravelType;
    switch (Type)
    {
        case ETravelType::Absolute:
            UnrealTravelType = TRAVEL_Absolute;
            break;
        case ETravelType::Partial:
            UnrealTravelType = TRAVEL_Partial;
            break;
        case ETravelType::Seamless:
            UnrealTravelType = TRAVEL_Relative;
            break;
        default:
            UnrealTravelType = TRAVEL_Absolute;
    }

    PC->ClientTravel(URL, UnrealTravelType);
    OnTravelStart.Broadcast(URL);
}

void UMyTravelManager::TravelToServer(const FString& Address, int32 Port)
{
    FString URL = FString::Printf(TEXT("%s:%d"), *Address, Port);
    ClientTravel(URL, ETravelType::Absolute);
}

void UMyTravelManager::CancelTravel()
{
    if (CurrentState == ETravelState::Idle)
    {
        return;
    }

    GetWorld()->GetTimerManager().ClearTimer(TravelTimeoutHandle);
    SetTravelState(ETravelState::Idle);
    Destination.Empty();
}

void UMyTravelManager::SetTravelState(ETravelState NewState)
{
    if (CurrentState != NewState)
    {
        CurrentState = NewState;
        UE_LOG(LogTemp, Log, TEXT("Travel state changed to: %d"), static_cast<int32>(NewState));
    }
}

void UMyTravelManager::OnTravelTimeout()
{
    SetTravelState(ETravelState::Failed);
    OnTravelError.Broadcast(TEXT("Travel timeout"));
}

FString UMyTravelManager::BuildTravelURL(const FString& MapName, const FString& Options)
{
    FString URL = MapName;

    if (!Options.IsEmpty())
    {
        URL += TEXT("?") + Options;
    }

    return URL;
}
```

---

## 三、无缝旅行实现

### 3.1 无缝旅行配置

```cpp
// DefaultEngine.ini 配置
[/Script/Engine.Engine]
; 启用无缝旅行
bUseSeamlessTravel=true

[/Script/Engine.GameEngine]
; 过渡地图
TransitionMap=/Game/Maps/TransitionMap

[/Script/Engine.GameMode]
; 旅行设置
bUseSeamlessTravel=true
```

### 3.2 GameMode无缝旅行处理

```cpp
// MyGameMode.h
UCLASS()
class AMyGameMode : public AGameModeBase
{
    GENERATED_BODY()

public:
    // 无缝旅行配置
    UPROPERTY(EditDefaultsOnly, Category = "Travel")
    bool bUseSeamlessTravel = true;

    UPROPERTY(EditDefaultsOnly, Category = "Travel")
    TSoftObjectPtr<UWorld> TransitionMap;

    // 旅行处理
    virtual void ProcessClientTravel(APlayerController* Player, const FString& TravelURL, bool bSeamless) override;

    virtual bool HasMatchStarted() const override;
    virtual bool HasMatchEnded() const override;

    // 保存/恢复玩家状态
    virtual void PostSeamlessTravel() override;
    virtual void HandleSeamlessTravelPlayer(AController*& C) override;

    // 开始旅行
    UFUNCTION(BlueprintCallable, Category = "Game")
    void StartTravel(const FString& DestinationMap);

    UFUNCTION(BlueprintCallable, Category = "Game")
    void StartTravelWithOptions(const FString& DestinationMap, const FString& Options);

protected:
    // 保存旅行数据
    USTRUCT()
    struct FTravelPlayerData
    {
        FString PlayerId;
        FString PlayerName;
        int32 TeamId;
        FVector LastLocation;
        FRotator LastRotation;
        float Health;
    };

    TMap<FString, FTravelPlayerData> SavedPlayerData;

    void SavePlayerDataForTravel();
    void RestorePlayerDataAfterTravel();
};

// MyGameMode.cpp
#include "MyGameMode.h"
#include "Kismet/GameplayStatics.h"

void AMyGameMode::StartTravel(const FString& DestinationMap)
{
    StartTravelWithOptions(DestinationMap, TEXT(""));
}

void AMyGameMode::StartTravelWithOptions(const FString& DestinationMap, const FString& Options)
{
    if (!HasAuthority())
    {
        UE_LOG(LogTemp, Warning, TEXT("StartTravel called on client"));
        return;
    }

    // 保存玩家数据
    SavePlayerDataForTravel();

    // 构建旅行URL
    FString TravelURL = DestinationMap + TEXT("?Listen");

    if (!Options.IsEmpty())
    {
        TravelURL += TEXT("?") + Options;
    }

    if (bUseSeamlessTravel)
    {
        TravelURL += TEXT("?Seamless");
    }

    UE_LOG(LogTemp, Log, TEXT("Starting travel to: %s"), *TravelURL);

    // 执行旅行
    GetWorld()->ServerTravel(TravelURL);
}

void AMyGameMode::SavePlayerDataForTravel()
{
    SavedPlayerData.Empty();

    for (FConstPlayerControllerIterator It = GetWorld()->GetPlayerControllerIterator(); It; ++It)
    {
        APlayerController* PC = It->Get();
        if (!PC) continue;

        FTravelPlayerData Data;

        // 获取玩家标识
        if (APlayerState* PS = PC->GetPlayerState<APlayerState>())
        {
            Data.PlayerId = PS->GetUniqueId().ToString();
            Data.PlayerName = PS->GetPlayerName();
        }

        // 保存位置
        if (APawn* Pawn = PC->GetPawn())
        {
            Data.LastLocation = Pawn->GetActorLocation();
            Data.LastRotation = Pawn->GetActorRotation();
        }

        // 保存其他数据（血量等）
        // ...

        SavedPlayerData.Add(Data.PlayerId, Data);
    }

    UE_LOG(LogTemp, Log, TEXT("Saved %d players for travel"), SavedPlayerData.Num());
}

void AMyGameMode::RestorePlayerDataAfterTravel()
{
    for (FConstPlayerControllerIterator It = GetWorld()->GetPlayerControllerIterator(); It; ++It)
    {
        APlayerController* PC = It->Get();
        if (!PC) continue;

        if (APlayerState* PS = PC->GetPlayerState<APlayerState>())
        {
            FString PlayerId = PS->GetUniqueId().ToString();

            if (FTravelPlayerData* Data = SavedPlayerData.Find(PlayerId))
            {
                // 恢复玩家数据
                if (APawn* Pawn = PC->GetPawn())
                {
                    Pawn->SetActorLocation(Data->LastLocation);
                    Pawn->SetActorRotation(Data->LastRotation);
                }

                UE_LOG(LogTemp, Log, TEXT("Restored player %s"), *Data->PlayerName);
            }
        }
    }

    // 清理保存的数据
    SavedPlayerData.Empty();
}

void AMyGameMode::PostSeamlessTravel()
{
    Super::PostSeamlessTravel();

    UE_LOG(LogTemp, Log, TEXT("Post seamless travel"));

    // 恢复玩家数据
    RestorePlayerDataAfterTravel();
}

void AMyGameMode::HandleSeamlessTravelPlayer(AController*& C)
{
    Super::HandleSeamlessTravelPlayer(C);

    // 处理旅行中的玩家
    if (APlayerController* PC = Cast<APlayerController>(C))
    {
        // 确保玩家正确初始化
        // ...
    }
}
```

---

## 四、加载进度与界面

### 4.1 加载进度追踪

```cpp
// MyLoadingScreenManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "MyLoadingScreenManager.generated.h"

// 加载阶段
UENUM(BlueprintType)
enum class ELoadingPhase : uint8
{
    None,
    PreLoading,
    LoadingMap,
    PostLoading,
    Complete
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnLoadingPhaseChanged, ELoadingPhase, Phase);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnLoadingProgressUpdated, float, Progress);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnLoadingMapChanged, const FString&, MapName);

UCLASS()
class UMyLoadingScreenManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 开始加载
    void BeginLoading(const FString& MapName);

    // 更新进度
    void UpdateProgress(float Progress);
    void SetPhase(ELoadingPhase Phase);

    // 完成加载
    void FinishLoading();

    // 状态查询
    UFUNCTION(BlueprintPure, Category = "Loading")
    ELoadingPhase GetCurrentPhase() const { return CurrentPhase; }

    UFUNCTION(BlueprintPure, Category = "Loading")
    float GetCurrentProgress() const { return CurrentProgress; }

    UFUNCTION(BlueprintPure, Category = "Loading")
    FString GetLoadingMapName() const { return LoadingMapName; }

    UFUNCTION(BlueprintPure, Category = "Loading")
    bool IsLoading() const { return bIsLoading; }

    // 加载提示
    UFUNCTION(BlueprintCallable, Category = "Loading")
    FString GetRandomLoadingTip() const;

    // 委托
    UPROPERTY(BlueprintAssignable)
    FOnLoadingPhaseChanged OnPhaseChanged;

    UPROPERTY(BlueprintAssignable)
    FOnLoadingProgressUpdated OnProgressUpdated;

    UPROPERTY(BlueprintAssignable)
    FOnLoadingMapChanged OnMapChanged;

protected:
    ELoadingPhase CurrentPhase = ELoadingPhase::None;
    float CurrentProgress = 0.0f;
    FString LoadingMapName;
    bool bIsLoading = false;

    // 加载提示列表
    UPROPERTY(EditDefaultsOnly, Category = "Loading")
    TArray<FString> LoadingTips;

    // 预估加载时间（用于模拟进度）
    TMap<FString, float> EstimatedLoadTimes;

    void SimulateProgress();
};

// MyLoadingScreenManager.cpp
#include "MyLoadingScreenManager.h"

void UMyLoadingScreenManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    // 初始化加载提示
    LoadingTips = {
        TEXT("Tip: Use cover to avoid enemy fire!"),
        TEXT("Tip: Teamwork is key to victory!"),
        TEXT("Tip: Check your minimap for objectives!"),
        TEXT("Tip: Upgrade your weapons for better performance!"),
        TEXT("Tip: Complete daily challenges for bonus rewards!")
    };
}

void UMyLoadingScreenManager::BeginLoading(const FString& MapName)
{
    LoadingMapName = MapName;
    CurrentProgress = 0.0f;
    bIsLoading = true;

    SetPhase(ELoadingPhase::PreLoading);
    OnMapChanged.Broadcast(MapName);

    UE_LOG(LogTemp, Log, TEXT("Begin loading: %s"), *MapName);
}

void UMyLoadingScreenManager::UpdateProgress(float Progress)
{
    CurrentProgress = FMath::Clamp(Progress, 0.0f, 1.0f);
    OnProgressUpdated.Broadcast(CurrentProgress);
}

void UMyLoadingScreenManager::SetPhase(ELoadingPhase Phase)
{
    if (CurrentPhase != Phase)
    {
        CurrentPhase = Phase;
        OnPhaseChanged.Broadcast(Phase);
    }
}

void UMyLoadingScreenManager::FinishLoading()
{
    CurrentProgress = 1.0f;
    SetPhase(ELoadingPhase::Complete);
    bIsLoading = false;

    OnProgressUpdated.Broadcast(1.0f);

    UE_LOG(LogTemp, Log, TEXT("Loading complete"));
}

FString UMyLoadingScreenManager::GetRandomLoadingTip() const
{
    if (LoadingTips.Num() == 0)
    {
        return TEXT("");
    }

    int32 Index = FMath::RandRange(0, LoadingTips.Num() - 1);
    return LoadingTips[Index];
}
```

### 4.2 加载界面Widget

```cpp
// WBP_LoadingScreen.h
UCLASS()
class UWBP_LoadingScreen : public UUserWidget
{
    GENERATED_BODY()

public:
    virtual void NativeConstruct() override;
    virtual void NativeTick(const FGeometry& MyGeometry, float InDeltaTime) override;

    // 绑定到加载管理器
    void BindToLoadingManager();

protected:
    // UI组件
    UPROPERTY(meta = (BindWidget))
    class UTextBlock* MapNameText;

    UPROPERTY(meta = (BindWidget))
    class UProgressBar* LoadingProgressBar;

    UPROPERTY(meta = (BindWidget))
    class UTextBlock* LoadingTipText;

    UPROPERTY(meta = (BindWidget))
    class UTextBlock* PercentageText;

    // 动画
    UPROPERTY(Transient, meta = (BindWidgetAnim))
    class UWidgetAnimation* FadeInAnimation;

    UPROPERTY(Transient, meta = (BindWidgetAnim))
    class UWidgetAnimation* FadeOutAnimation;

private:
    UFUNCTION()
    void OnPhaseChanged(ELoadingPhase Phase);

    UFUNCTION()
    void OnProgressUpdated(float Progress);

    UFUNCTION()
    void OnMapChanged(const FString& MapName);

    void UpdateUI();
};

// WBP_LoadingScreen.cpp
#include "WBP_LoadingScreen.h"
#include "MyLoadingScreenManager.h"
#include "Components/TextBlock.h"
#include "Components/ProgressBar.h"

void UWBP_LoadingScreen::NativeConstruct()
{
    Super::NativeConstruct();

    BindToLoadingManager();
}

void UWBP_LoadingScreen::BindToLoadingManager()
{
    UGameInstance* GI = GetGameInstance();
    if (UMyLoadingScreenManager* Manager = GI->GetSubsystem<UMyLoadingScreenManager>())
    {
        Manager->OnPhaseChanged.AddDynamic(this, &UWBP_LoadingScreen::OnPhaseChanged);
        Manager->OnProgressUpdated.AddDynamic(this, &UWBP_LoadingScreen::OnProgressUpdated);
        Manager->OnMapChanged.AddDynamic(this, &UWBP_LoadingScreen::OnMapChanged);

        // 初始化UI
        LoadingTipText->SetText(FText::FromString(Manager->GetRandomLoadingTip()));
    }
}

void UWBP_LoadingScreen::OnPhaseChanged(ELoadingPhase Phase)
{
    switch (Phase)
    {
        case ELoadingPhase::PreLoading:
            if (FadeInAnimation)
            {
                PlayAnimation(FadeInAnimation);
            }
            break;

        case ELoadingPhase::Complete:
            if (FadeOutAnimation)
            {
                PlayAnimation(FadeOutAnimation);
            }
            break;

        default:
            break;
    }
}

void UWBP_LoadingScreen::OnProgressUpdated(float Progress)
{
    if (LoadingProgressBar)
    {
        LoadingProgressBar->SetPercent(Progress);
    }

    if (PercentageText)
    {
        FString Percentage = FString::Printf(TEXT("%d%%"), FMath::RoundToInt(Progress * 100));
        PercentageText->SetText(FText::FromString(Percentage));
    }
}

void UWBP_LoadingScreen::OnMapChanged(const FString& MapName)
{
    if (MapNameText)
    {
        MapNameText->SetText(FText::FromString(MapName));
    }
}
```

---

## 五、多关卡服务器架构

### 5.1 多关卡管理器

```cpp
// MyMultiLevelManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "MyMultiLevelManager.generated.h"

// 关卡信息
USTRUCT(BlueprintType)
struct FLevelInfo
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString LevelName;

    UPROPERTY(BlueprintReadOnly)
    FString DisplayName;

    UPROPERTY(BlueprintReadOnly)
    FString AssetPath;

    UPROPERTY(BlueprintReadOnly)
    FVector DefaultSpawnLocation;

    UPROPERTY(BlueprintReadOnly)
    bool bRequiresLoading = true;

    UPROPERTY(BlueprintReadOnly)
    int32 MaxPlayers = 32;
};

// 关卡加载状态
UENUM(BlueprintType)
enum class ELevelLoadState : uint8
{
    Unloaded,
    Loading,
    Loaded,
    Failed
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnLevelLoadComplete, const FString&, LevelName, bool, bSuccess);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnLevelUnloadComplete, const FString&, LevelName);

UCLASS()
class UMyMultiLevelManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 关卡注册
    UFUNCTION(BlueprintCallable, Category = "Levels")
    void RegisterLevel(const FLevelInfo& LevelInfo);

    UFUNCTION(BlueprintPure, Category = "Levels")
    TArray<FString> GetRegisteredLevels() const;

    UFUNCTION(BlueprintPure, Category = "Levels")
    FLevelInfo GetLevelInfo(const FString& LevelName) const;

    // 流式加载
    UFUNCTION(BlueprintCallable, Category = "Levels")
    void LoadLevel(const FString& LevelName);

    UFUNCTION(BlueprintCallable, Category = "Levels")
    void UnloadLevel(const FString& LevelName);

    UFUNCTION(BlueprintCallable, Category = "Levels")
    void LoadLevelsInRadius(const FVector& Location, float Radius);

    UFUNCTION(BlueprintCallable, Category = "Levels")
    void UnloadLevelsOutsideRadius(const FVector& Location, float Radius);

    // 状态查询
    UFUNCTION(BlueprintPure, Category = "Levels")
    ELevelLoadState GetLevelState(const FString& LevelName) const;

    UFUNCTION(BlueprintPure, Category = "Levels")
    bool IsLevelLoaded(const FString& LevelName) const;

    UFUNCTION(BlueprintPure, Category = "Levels")
    TArray<FString> GetLoadedLevels() const;

    // 传送玩家
    UFUNCTION(BlueprintCallable, Category = "Levels")
    void TeleportPlayerToLevel(APlayerController* Player, const FString& LevelName, const FVector& Location);

    // 委托
    UPROPERTY(BlueprintAssignable)
    FOnLevelLoadComplete OnLevelLoadComplete;

    UPROPERTY(BlueprintAssignable)
    FOnLevelUnloadComplete OnLevelUnloadComplete;

private:
    TMap<FString, FLevelInfo> RegisteredLevels;
    TMap<FString, ELevelLoadState> LevelStates;

    UPROPERTY()
    TMap<FString, ULevelStreaming*> StreamingLevels;

    FString CurrentMainLevel;

    void OnLevelLoaded(const FString& LevelName);
    void OnLevelUnloaded(const FString& LevelName);
    FName GenerateStreamingLevelName(const FString& LevelName);
};

// MyMultiLevelManager.cpp
#include "MyMultiLevelManager.h"
#include "Engine/LevelStreaming.h"
#include "Kismet/GameplayStatics.h"

void UMyMultiLevelManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    // 注册默认关卡
    FLevelInfo LobbyLevel;
    LobbyLevel.LevelName = TEXT("Lobby");
    LobbyLevel.DisplayName = TEXT("Main Lobby");
    LobbyLevel.AssetPath = TEXT("/Game/Maps/Lobby");
    LobbyLevel.DefaultSpawnLocation = FVector(0, 0, 100);
    RegisterLevel(LobbyLevel);

    FLevelInfo GameLevel;
    GameLevel.LevelName = TEXT("GameArena");
    GameLevel.DisplayName = TEXT("Game Arena");
    GameLevel.AssetPath = TEXT("/Game/Maps/GameArena");
    GameLevel.DefaultSpawnLocation = FVector(1000, 1000, 100);
    RegisterLevel(GameLevel);
}

void UMyMultiLevelManager::RegisterLevel(const FLevelInfo& LevelInfo)
{
    RegisteredLevels.Add(LevelInfo.LevelName, LevelInfo);
    LevelStates.Add(LevelInfo.LevelName, ELevelLoadState::Unloaded);

    UE_LOG(LogTemp, Log, TEXT("Registered level: %s"), *LevelInfo.LevelName);
}

void UMyMultiLevelManager::LoadLevel(const FString& LevelName)
{
    if (!RegisteredLevels.Contains(LevelName))
    {
        UE_LOG(LogTemp, Warning, TEXT("Unknown level: %s"), *LevelName);
        OnLevelLoadComplete.Broadcast(LevelName, false);
        return;
    }

    if (LevelStates[LevelName] == ELevelLoadState::Loaded)
    {
        UE_LOG(LogTemp, Log, TEXT("Level already loaded: %s"), *LevelName);
        OnLevelLoadComplete.Broadcast(LevelName, true);
        return;
    }

    LevelStates[LevelName] = ELevelLoadState::Loading;

    FLevelInfo Info = RegisteredLevels[LevelName];

    // 创建流式关卡
    ULevelStreaming* StreamingLevel = UGameplayStatics::GetStreamingLevel(GetWorld(), FName(*LevelName));

    if (!StreamingLevel)
    {
        // 创建新的流式关卡
        FString LevelPath = Info.AssetPath + TEXT(".") + LevelName;
        StreamingLevel = UGameplayStatics::LoadStreamLevel(GetWorld(), FName(*LevelName), true, false, ELatentActionManager::None);
    }
    else
    {
        // 激活已存在的流式关卡
        StreamingLevel->SetShouldBeLoaded(true);
        StreamingLevel->SetShouldBeVisible(true);
    }

    if (StreamingLevel)
    {
        StreamingLevels.Add(LevelName, StreamingLevel);
        LevelStates[LevelName] = ELevelLoadState::Loaded;
        OnLevelLoadComplete.Broadcast(LevelName, true);
    }
    else
    {
        LevelStates[LevelName] = ELevelLoadState::Failed;
        OnLevelLoadComplete.Broadcast(LevelName, false);
    }
}

void UMyMultiLevelManager::UnloadLevel(const FString& LevelName)
{
    if (LevelStates[LevelName] != ELevelLoadState::Loaded)
    {
        return;
    }

    if (ULevelStreaming* StreamingLevel = StreamingLevels.FindRef(LevelName))
    {
        StreamingLevel->SetShouldBeLoaded(false);
        StreamingLevel->SetShouldBeVisible(false);
        StreamingLevels.Remove(LevelName);
    }

    LevelStates[LevelName] = ELevelLoadState::Unloaded;
    OnLevelUnloadComplete.Broadcast(LevelName);
}

void UMyMultiLevelManager::TeleportPlayerToLevel(APlayerController* Player, const FString& LevelName, const FVector& Location)
{
    if (!Player || !RegisteredLevels.Contains(LevelName))
    {
        return;
    }

    // 确保目标关卡已加载
    if (!IsLevelLoaded(LevelName))
    {
        LoadLevel(LevelName);
    }

    // 传送玩家
    if (APawn* Pawn = Player->GetPawn())
    {
        Pawn->SetActorLocation(Location);
    }
    else
    {
        // 如果玩家没有Pawn，在目标位置生成
        Player->SetInitialLocationAndRotation(Location, FRotator::ZeroRotator);
    }
}

bool UMyMultiLevelManager::IsLevelLoaded(const FString& LevelName) const
{
    return LevelStates.Contains(LevelName) && LevelStates[LevelName] == ELevelLoadState::Loaded;
}

TArray<FString> UMyMultiLevelManager::GetLoadedLevels() const
{
    TArray<FString> Result;

    for (const auto& Pair : LevelStates)
    {
        if (Pair.Value == ELevelLoadState::Loaded)
        {
            Result.Add(Pair.Key);
        }
    }

    return Result;
}
```

---

## 六、总结

本课我们学习了：

1. **地图切换类型**：硬切换、无缝旅行、流式加载
2. **服务器旅行**：ServerTravel和ClientTravel
3. **无缝旅行实现**：保存恢复玩家状态
4. **加载界面**：进度追踪和UI设计
5. **多关卡架构**：动态加载和卸载

---

## 七、下节预告

**第17课：回放系统**

将深入学习：
- 网络回放架构
- 录制与回放实现
- 回放数据存储

---

*课程版本：1.0*
*最后更新：2026-04-10*