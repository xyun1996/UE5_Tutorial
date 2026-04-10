# 第14课：用户数据管理

> 本课深入讲解UE5的用户数据管理系统，包括认证系统设计、账号数据持久化和跨设备数据同步。

---

## 课程目标

- 掌握用户认证系统设计
- 学会账号数据持久化实现
- 理解跨设备数据同步机制
- 掌握数据迁移策略
- 学会数据备份与恢复

---

## 一、用户认证系统设计

### 1.1 认证系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                    认证系统架构                              │
│                                                              │
│  客户端                                                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  1. 用户输入凭据                                     │    │
│  │  2. 发送认证请求                                     │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│                          ▼                                   │
│  认证服务器                                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  3. 验证凭据                                         │    │
│  │  4. 检查权限                                         │    │
│  │  5. 生成令牌                                         │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│                          ▼                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  6. 返回访问令牌 + 刷新令牌                          │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│                          ▼                                   │
│  游戏服务器                                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  7. 验证令牌有效性                                   │    │
│  │  8. 加载用户数据                                     │    │
│  │  9. 建立游戏会话                                     │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 认证管理器实现

```cpp
// MyAuthenticationManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "MyAuthenticationManager.generated.h"

// 认证结果
UENUM(BlueprintType)
enum class EAuthResult : uint8
{
    Success,
    InvalidCredentials,
    AccountBanned,
    ServerError,
    NetworkError,
    TokenExpired
};

// 用户信息
USTRUCT(BlueprintType)
struct FUserInfo
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString UserId;

    UPROPERTY(BlueprintReadOnly)
    FString Username;

    UPROPERTY(BlueprintReadOnly)
    FString Email;

    UPROPERTY(BlueprintReadOnly)
    FString DisplayName;

    UPROPERTY(BlueprintReadOnly)
    int32 AccountLevel = 0;

    UPROPERTY(BlueprintReadOnly)
    TArray<FString> Permissions;
};

// 认证令牌
USTRUCT()
struct FAuthToken
{
    GENERATED_BODY()

    FString AccessToken;
    FString RefreshToken;
    float ExpiresIn;
    float RefreshExpiresIn;
    FDateTime IssueTime;

    bool IsValid() const
    {
        FDateTime Now = FDateTime::UtcNow();
        FTimespan Elapsed = Now - IssueTime;
        return Elapsed.GetTotalSeconds() < ExpiresIn;
    }

    bool NeedsRefresh() const
    {
        FDateTime Now = FDateTime::UtcNow();
        FTimespan Elapsed = Now - IssueTime;
        return Elapsed.GetTotalSeconds() > ExpiresIn * 0.8f; // 80%时刷新
    }
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnAuthComplete, EAuthResult, Result, const FUserInfo&, UserInfo);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnLogoutComplete, bool, bSuccess);

UCLASS()
class UMyAuthenticationManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 登录方式
    UFUNCTION(BlueprintCallable, Category = "Auth")
    void LoginWithCredentials(const FString& Username, const FString& Password);

    UFUNCTION(BlueprintCallable, Category = "Auth")
    void LoginWithSteam();

    UFUNCTION(BlueprintCallable, Category = "Auth")
    void LoginWithToken(const FString& Token);

    UFUNCTION(BlueprintCallable, Category = "Auth")
    void LoginAsGuest();

    // 登出
    UFUNCTION(BlueprintCallable, Category = "Auth")
    void Logout();

    // 令牌管理
    UFUNCTION(BlueprintCallable, Category = "Auth")
    void RefreshToken();

    UFUNCTION(BlueprintPure, Category = "Auth")
    bool IsLoggedIn() const;

    UFUNCTION(BlueprintPure, Category = "Auth")
    FUserInfo GetUserInfo() const;

    UFUNCTION(BlueprintPure, Category = "Auth")
    FString GetAccessToken() const;

    // 委托
    UPROPERTY(BlueprintAssignable)
    FOnAuthComplete OnLoginComplete;

    UPROPERTY(BlueprintAssignable)
    FOnLogoutComplete OnLogoutComplete;

protected:
    FUserInfo CurrentUser;
    FAuthToken CurrentToken;
    bool bIsLoggedIn = false;

    // 配置
    UPROPERTY(Config)
    FString AuthServerURL = TEXT("https://auth.example.com");

    UPROPERTY(Config)
    FString ApiKey;

    // 持久化
    void SaveAuthToken();
    bool LoadSavedToken();
    void ClearSavedToken();

    // 内部方法
    void OnLoginResponse(bool bSuccess, const FString& Response);
    void ValidateAndRefreshToken();
};

// MyAuthenticationManager.cpp
#include "MyAuthenticationManager.h"
#include "Kismet/GameplayStatics.h"
#include "JsonObjectConverter.h"

void UMyAuthenticationManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    // 尝试加载保存的令牌
    if (LoadSavedToken() && CurrentToken.IsValid())
    {
        // 自动刷新令牌
        RefreshToken();
    }
}

void UMyAuthenticationManager::Deinitialize()
{
    SaveAuthToken();
    Super::Deinitialize();
}

void UMyAuthenticationManager::LoginWithCredentials(const FString& Username, const FString& Password)
{
    // 构建请求
    TSharedPtr<FJsonObject> RequestJson = MakeShareable(new FJsonObject);
    RequestJson->SetStringField(TEXT("username"), Username);
    RequestJson->SetStringField(TEXT("password"), Password);
    RequestJson->SetStringField(TEXT("grant_type"), TEXT("password"));

    FString RequestBody;
    FJsonObjectConverter::UStructToJsonObjectString(RequestJson.ToSharedRef(), RequestBody);

    // 发送HTTP请求（这里需要HTTP模块）
    // TSharedRef<IHttpRequest> Request = FHttpModule::Get().CreateRequest();
    // Request->SetURL(AuthServerURL + TEXT("/login"));
    // Request->SetVerb(TEXT("POST"));
    // Request->SetHeader(TEXT("Content-Type"), TEXT("application/json"));
    // Request->SetContentAsString(RequestBody);
    // Request->ProcessRequest();

    UE_LOG(LogTemp, Log, TEXT("Login request sent for user: %s"), *Username);
}

void UMyAuthenticationManager::LoginWithSteam()
{
    // 获取Steam认证票据
    // FString SteamTicket = GetSteamAuthTicket();

    // 构建请求并发送
    // ...
}

void UMyAuthenticationManager::Logout()
{
    // 发送登出请求到服务器
    // ...

    // 清理本地数据
    CurrentUser = FUserInfo();
    CurrentToken = FAuthToken();
    bIsLoggedIn = false;
    ClearSavedToken();

    OnLogoutComplete.Broadcast(true);
}

void UMyAuthenticationManager::RefreshToken()
{
    if (CurrentToken.RefreshToken.IsEmpty())
    {
        return;
    }

    // 发送刷新请求
    // ...

    UE_LOG(LogTemp, Log, TEXT("Refreshing token"));
}

bool UMyAuthenticationManager::IsLoggedIn() const
{
    return bIsLoggedIn && CurrentToken.IsValid();
}

FUserInfo UMyAuthenticationManager::GetUserInfo() const
{
    return CurrentUser;
}

FString UMyAuthenticationManager::GetAccessToken() const
{
    return CurrentToken.AccessToken;
}

void UMyAuthenticationManager::SaveAuthToken()
{
    if (!CurrentToken.AccessToken.IsEmpty())
    {
        // 保存到本地存储
        UGameplayStatics::SaveGameToSlot(/* ... */);
    }
}

bool UMyAuthenticationManager::LoadSavedToken()
{
    // 从本地存储加载
    // UGameplayStatics::LoadGameFromSlot(/* ... */);
    return false;
}

void UMyAuthenticationManager::ClearSavedToken()
{
    // 清除本地存储
}
```

---

## 二、账号数据持久化

### 2.1 用户数据模型

```cpp
// MyUserDataManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "MyUserDataManager.generated.h"

// 玩家档案数据
USTRUCT(BlueprintType)
struct FPlayerProfile
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString PlayerId;

    UPROPERTY(BlueprintReadWrite)
    FString DisplayName;

    UPROPERTY(BlueprintReadOnly)
    int32 Level = 1;

    UPROPERTY(BlueprintReadOnly)
    int64 Experience = 0;

    UPROPERTY(BlueprintReadOnly)
    FString AvatarUrl;

    UPROPERTY(BlueprintReadOnly)
    FString Title;

    UPROPERTY(BlueprintReadWrite)
    FLinearColor CustomColor = FLinearColor::White;
};

// 玩家统计数据
USTRUCT(BlueprintType)
struct FPlayerStats
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    int32 TotalKills = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 TotalDeaths = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 Wins = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 Losses = 0;

    UPROPERTY(BlueprintReadOnly)
    int64 TotalScore = 0;

    UPROPERTY(BlueprintReadOnly)
    float TotalPlayTime = 0.0f;

    float GetKDRatio() const
    {
        return TotalDeaths > 0 ? static_cast<float>(TotalKills) / TotalDeaths : TotalKills;
    }

    float GetWinRate() const
    {
        int32 TotalGames = Wins + Losses;
        return TotalGames > 0 ? static_cast<float>(Wins) / TotalGames : 0.0f;
    }
};

// 玩家物品数据
USTRUCT(BlueprintType)
struct FPlayerItem
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString ItemId;

    UPROPERTY(BlueprintReadOnly)
    FString ItemType;

    UPROPERTY(BlueprintReadOnly)
    int32 Quantity = 1;

    UPROPERTY(BlueprintReadOnly)
    FDateTime AcquiredTime;

    UPROPERTY(BlueprintReadOnly)
    bool bIsEquipped = false;
};

// 玩家设置
USTRUCT(BlueprintType)
struct FPlayerSettings
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadWrite)
    float MouseSensitivity = 1.0f;

    UPROPERTY(BlueprintReadWrite)
    float MusicVolume = 1.0f;

    UPROPERTY(BlueprintReadWrite)
    float SFXVolume = 1.0f;

    UPROPERTY(BlueprintReadWrite)
    float VoiceVolume = 1.0f;

    UPROPERTY(BlueprintReadWrite)
    bool bVoiceChatEnabled = true;

    UPROPERTY(BlueprintReadWrite)
    bool bShowFPS = false;

    UPROPERTY(BlueprintReadWrite)
    int32 GraphicsQuality = 2;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnUserDataLoaded, bool, bSuccess);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnUserDataSaved, bool, bSuccess);

UCLASS()
class UMyUserDataManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 数据加载
    UFUNCTION(BlueprintCallable, Category = "UserData")
    void LoadUserData(const FString& UserId);

    UFUNCTION(BlueprintCallable, Category = "UserData")
    void SaveUserData();

    UFUNCTION(BlueprintCallable, Category = "UserData")
    void ForceSaveUserData();

    // 数据访问
    UFUNCTION(BlueprintPure, Category = "UserData")
    FPlayerProfile GetProfile() const { return Profile; }

    UFUNCTION(BlueprintPure, Category = "UserData")
    FPlayerStats GetStats() const { return Stats; }

    UFUNCTION(BlueprintPure, Category = "UserData")
    TArray<FPlayerItem> GetInventory() const { return Inventory; }

    UFUNCTION(BlueprintPure, Category = "UserData")
    FPlayerSettings GetSettings() const { return Settings; }

    // 数据修改
    UFUNCTION(BlueprintCallable, Category = "UserData")
    void UpdateProfile(const FPlayerProfile& NewProfile);

    UFUNCTION(BlueprintCallable, Category = "UserData")
    void AddExperience(int64 Amount);

    UFUNCTION(BlueprintCallable, Category = "UserData")
    void UpdateStats(const FPlayerStats& NewStats);

    UFUNCTION(BlueprintCallable, Category = "UserData")
    bool AddItem(const FString& ItemId, int32 Quantity = 1);

    UFUNCTION(BlueprintCallable, Category = "UserData")
    bool RemoveItem(const FString& ItemId, int32 Quantity = 1);

    UFUNCTION(BlueprintCallable, Category = "UserData")
    void UpdateSettings(const FPlayerSettings& NewSettings);

    // 委托
    UPROPERTY(BlueprintAssignable)
    FOnUserDataLoaded OnDataLoaded;

    UPROPERTY(BlueprintAssignable)
    FOnUserDataSaved OnDataSaved;

protected:
    FPlayerProfile Profile;
    FPlayerStats Stats;
    TArray<FPlayerItem> Inventory;
    FPlayerSettings Settings;

    FString CurrentUserId;
    bool bIsDataLoaded = false;
    bool bHasUnsavedChanges = false;

    // 自动保存
    FTimerHandle AutoSaveTimer;
    float AutoSaveInterval = 60.0f;

    void StartAutoSave();
    void StopAutoSave();
    void OnAutoSave();

    // 服务器通信
    void SendLoadRequest(const FString& UserId);
    void SendSaveRequest();
    void OnLoadResponse(bool bSuccess, const FString& Response);
    void OnSaveResponse(bool bSuccess);
};
```

---

## 三、跨设备数据同步

### 3.1 数据同步管理器

```cpp
// MyDataSyncManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "MyDataSyncManager.generated.h"

// 同步状态
UENUM(BlueprintType)
enum class ESyncStatus : uint8
{
    NotSynced,
    Syncing,
    Synced,
    Conflict,
    Error
};

// 同步冲突
USTRUCT(BlueprintType)
struct FSyncConflict
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString FieldName;

    UPROPERTY(BlueprintReadOnly)
    FString LocalValue;

    UPROPERTY(BlueprintReadOnly)
    FString ServerValue;

    UPROPERTY(BlueprintReadOnly)
    FDateTime LocalTimestamp;

    UPROPERTY(BlueprintReadOnly)
    FDateTime ServerTimestamp;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnSyncComplete, bool, bSuccess);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnSyncStatusChanged, ESyncStatus, NewStatus);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnSyncConflictDetected, const TArray<FSyncConflict>&, Conflicts);

UCLASS()
class UMyDataSyncManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 同步操作
    UFUNCTION(BlueprintCallable, Category = "Sync")
    void SyncAll();

    UFUNCTION(BlueprintCallable, Category = "Sync")
    void SyncProfile();

    UFUNCTION(BlueprintCallable, Category = "Sync")
    void SyncInventory();

    UFUNCTION(BlueprintCallable, Category = "Sync")
    void SyncSettings();

    // 冲突解决
    UFUNCTION(BlueprintCallable, Category = "Sync")
    void ResolveConflict(const FString& FieldName, bool bUseServerValue);

    UFUNCTION(BlueprintCallable, Category = "Sync")
    void ResolveAllConflicts(bool bUseServerValues);

    // 状态
    UFUNCTION(BlueprintPure, Category = "Sync")
    ESyncStatus GetSyncStatus() const { return CurrentStatus; }

    UFUNCTION(BlueprintPure, Category = "Sync")
    FDateTime GetLastSyncTime() const { return LastSyncTime; }

    UFUNCTION(BlueprintPure, Category = "Sync")
    bool HasPendingChanges() const { return bHasPendingChanges; }

    // 委托
    UPROPERTY(BlueprintAssignable)
    FOnSyncComplete OnSyncComplete;

    UPROPERTY(BlueprintAssignable)
    FOnSyncStatusChanged OnSyncStatusChanged;

    UPROPERTY(BlueprintAssignable)
    FOnSyncConflictDetected OnSyncConflictDetected;

protected:
    ESyncStatus CurrentStatus = ESyncStatus::NotSynced;
    FDateTime LastSyncTime;
    bool bHasPendingChanges = false;

    TArray<FSyncConflict> PendingConflicts;

    void SetStatus(ESyncStatus NewStatus);
    void DetectConflicts(const TMap<FString, FString>& LocalData, const TMap<FString, FString>& ServerData);
    void ApplyServerData(const TMap<FString, FString>& ServerData);
    void MarkDataChanged(const FString& DataType);
};

// MyDataSyncManager.cpp
#include "MyDataSyncManager.h"

void UMyDataSyncManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    // 监听数据变化
    // MyUserDataManager->OnDataChanged.AddUObject(this, &UMyDataSyncManager::MarkDataChanged);
}

void UMyDataSyncManager::SyncAll()
{
    if (CurrentStatus == ESyncStatus::Syncing)
    {
        UE_LOG(LogTemp, Warning, TEXT("Sync already in progress"));
        return;
    }

    SetStatus(ESyncStatus::Syncing);

    // 执行全量同步
    // 1. 发送本地数据到服务器
    // 2. 接收服务器数据
    // 3. 检测冲突
    // 4. 解决或报告冲突
    // 5. 应用最终数据

    // 模拟同步完成
    FTimerHandle SyncTimer;
    GetWorld()->GetTimerManager().SetTimer(SyncTimer, [this]()
    {
        LastSyncTime = FDateTime::UtcNow();
        bHasPendingChanges = false;
        SetStatus(ESyncStatus::Synced);
        OnSyncComplete.Broadcast(true);
    }, 2.0f, false);
}

void UMyDataSyncManager::DetectConflicts(const TMap<FString, FString>& LocalData, const TMap<FString, FString>& ServerData)
{
    PendingConflicts.Empty();

    for (const auto& Pair : LocalData)
    {
        const FString& Key = Pair.Key;
        const FString& LocalValue = Pair.Value;

        if (const FString* ServerValue = ServerData.Find(Key))
        {
            if (LocalValue != *ServerValue)
            {
                // 检测到冲突
                FSyncConflict Conflict;
                Conflict.FieldName = Key;
                Conflict.LocalValue = LocalValue;
                Conflict.ServerValue = *ServerValue;
                Conflict.LocalTimestamp = FDateTime::UtcNow(); // 从本地数据获取
                Conflict.ServerTimestamp = FDateTime::UtcNow(); // 从服务器数据获取

                PendingConflicts.Add(Conflict);
            }
        }
    }

    if (PendingConflicts.Num() > 0)
    {
        SetStatus(ESyncStatus::Conflict);
        OnSyncConflictDetected.Broadcast(PendingConflicts);
    }
}

void UMyDataSyncManager::ResolveConflict(const FString& FieldName, bool bUseServerValue)
{
    for (int32 i = PendingConflicts.Num() - 1; i >= 0; i--)
    {
        if (PendingConflicts[i].FieldName == FieldName)
        {
            if (bUseServerValue)
            {
                // 应用服务器值
                // ApplyServerValue(FieldName, PendingConflicts[i].ServerValue);
            }

            PendingConflicts.RemoveAt(i);
            break;
        }
    }

    if (PendingConflicts.Num() == 0)
    {
        SetStatus(ESyncStatus::Synced);
    }
}

void UMyDataSyncManager::ResolveAllConflicts(bool bUseServerValues)
{
    for (const FSyncConflict& Conflict : PendingConflicts)
    {
        if (bUseServerValues)
        {
            // ApplyServerValue(Conflict.FieldName, Conflict.ServerValue);
        }
    }

    PendingConflicts.Empty();
    SetStatus(ESyncStatus::Synced);
}

void UMyDataSyncManager::SetStatus(ESyncStatus NewStatus)
{
    if (CurrentStatus != NewStatus)
    {
        CurrentStatus = NewStatus;
        OnSyncStatusChanged.Broadcast(NewStatus);
    }
}

void UMyDataSyncManager::MarkDataChanged(const FString& DataType)
{
    bHasPendingChanges = true;
}
```

---

## 四、数据迁移与备份

### 4.1 数据迁移系统

```cpp
// MyDataMigration.h
#pragma once

#include "CoreMinimal.h"
#include "MyDataMigration.generated.h"

// 迁移记录
USTRUCT()
struct FMigrationRecord
{
    GENERATED_BODY()

    FString SourcePlatform;
    FString TargetPlatform;
    FDateTime MigrationTime;
    bool bSuccess;
    FString ErrorMessage;
};

UCLASS()
class UMyDataMigration : public UObject
{
    GENERATED_BODY()

public:
    // 迁移数据到新平台
    UFUNCTION(BlueprintCallable, Category = "Migration")
    void MigrateToPlatform(const FString& TargetPlatform);

    // 检查迁移状态
    UFUNCTION(BlueprintPure, Category = "Migration")
    bool IsMigrationComplete() const;

    // 获取迁移历史
    UFUNCTION(BlueprintPure, Category = "Migration")
    TArray<FMigrationRecord> GetMigrationHistory() const;

private:
    TArray<FMigrationRecord> MigrationHistory;

    bool ExportData(TArray<uint8>& OutData);
    bool ImportData(const TArray<uint8>& Data);
    bool ValidateMigration(const FString& TargetPlatform);
    void LogMigration(const FString& Source, const FString& Target, bool bSuccess, const FString& Error);
};
```

---

## 五、总结

本课我们学习了：

1. **认证系统**：多种登录方式和令牌管理
2. **数据持久化**：用户数据模型设计
3. **跨设备同步**：冲突检测和解决
4. **数据迁移**：平台间数据转移

---

## 六、下节预告

**第15课：云存储集成**

将深入学习：
- AWS S3集成
- Azure Blob Storage
- 游戏资源云存储

---

*课程版本：1.0*
*最后更新：2026-04-10*