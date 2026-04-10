# 第29课：防盗版与防盗版设计

> 学习目标：掌握DRM集成方案、许可验证机制、Steam/Epic平台集成和资源加密保护技术。

---

## 29.1 DRM集成方案

### 29.1.1 DRM系统架构

```csharp
// DRMManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "DRMManager.generated.h"

UENUM(BlueprintType)
enum class EDRMStatus : uint8
{
    NotInitialized,
    Checking,
    Valid,
    Invalid,
    Expired,
    Revoked,
    OfflineMode,
    Error
};

UENUM(BlueprintType)
enum class EDRMProvider : uint8
{
    None,
    Steam,
    Epic,
    GOG,
    Custom
};

USTRUCT(BlueprintType)
struct FLicenseInfo
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString LicenseId;

    UPROPERTY(BlueprintReadOnly)
    FString ProductId;

    UPROPERTY(BlueprintReadOnly)
    FString UserId;

    UPROPERTY(BlueprintReadOnly)
    FDateTime ExpirationDate;

    UPROPERTY(BlueprintReadOnly)
    TArray<FString> Entitlements;

    UPROPERTY(BlueprintReadOnly)
    EDRMProvider Provider;

    UPROPERTY(BlueprintReadOnly)
    bool bIsValid = false;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnDRMStatusChanged, EDRMStatus, NewStatus);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnLicenseVerified, const FLicenseInfo&, License);

UCLASS()
class DRMSYSTEM_API UDRMManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 初始化DRM
    UFUNCTION(BlueprintCallable, Category = "DRM")
    void InitializeDRM(EDRMProvider Provider, const FString& AppId);

    // 验证许可
    UFUNCTION(BlueprintCallable, Category = "DRM")
    void VerifyLicense();

    // 离线验证
    UFUNCTION(BlueprintCallable, Category = "DRM")
    bool VerifyOfflineLicense();

    // 获取当前状态
    UFUNCTION(BlueprintPure, Category = "DRM")
    EDRMStatus GetStatus() const { return CurrentStatus; }

    // 获取许可信息
    UFUNCTION(BlueprintPure, Category = "DRM")
    FLicenseInfo GetLicenseInfo() const { return CurrentLicense; }

    // 检查权限
    UFUNCTION(BlueprintCallable, Category = "DRM")
    bool HasEntitlement(const FString& EntitlementId) const;

    // 刷新许可
    UFUNCTION(BlueprintCallable, Category = "DRM")
    void RefreshLicense();

    // 设置离线模式
    UFUNCTION(BlueprintCallable, Category = "DRM")
    void SetOfflineMode(bool bEnabled);

    UPROPERTY(BlueprintAssignable, Category = "DRM")
    FOnDRMStatusChanged OnStatusChanged;

    UPROPERTY(BlueprintAssignable, Category = "DRM")
    FOnLicenseVerified OnLicenseVerified;

    // 配置
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "DRM")
    bool bRequireOnlineVerification = true;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "DRM")
    int32 OfflineGracePeriodDays = 7;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "DRM")
    int32 VerificationIntervalMinutes = 60;

private:
    EDRMStatus CurrentStatus = EDRMStatus::NotInitialized;
    EDRMProvider CurrentProvider = EDRMProvider::None;
    FLicenseInfo CurrentLicense;
    FString ApplicationId;
    bool bOfflineMode = false;

    FTimerHandle VerificationTimerHandle;

    // 离线令牌
    FString OfflineToken;
    FDateTime LastOnlineVerification;

    void SetStatus(EDRMStatus NewStatus);
    bool PerformOnlineVerification();
    void SaveOfflineToken();
    bool LoadOfflineToken();
    FString GenerateMachineId();
};
```

```csharp
// DRMManager.cpp
#include "DRMManager.h"
#include "Misc/FileHelper.h"
#include "Misc/Paths.h"
#include "HAL/PlatformProcess.h"
#include "HttpModule.h"
#include "Interfaces/IHttpRequest.h"
#include "Interfaces/IHttpResponse.h"

void UDRMManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    // 尝试加载离线令牌
    LoadOfflineToken();
}

void UDRMManager::Deinitialize()
{
    if (GetWorld())
    {
        GetWorld()->GetTimerManager().ClearTimer(VerificationTimerHandle);
    }

    Super::Deinitialize();
}

void UDRMManager::InitializeDRM(EDRMProvider Provider, const FString& AppId)
{
    CurrentProvider = Provider;
    ApplicationId = AppId;

    SetStatus(EDRMStatus::Checking);

    // 根据提供商初始化
    switch (Provider)
    {
        case EDRMProvider::Steam:
            // Steam初始化
            break;

        case EDRMProvider::Epic:
            // Epic初始化
            break;

        case EDRMProvider::Custom:
            // 自定义DRM初始化
            break;

        default:
            break;
    }

    // 执行初始验证
    VerifyLicense();

    // 设置定时验证
    if (GetWorld())
    {
        GetWorld()->GetTimerManager().SetTimer(
            VerificationTimerHandle,
            this,
            &UDRMManager::RefreshLicense,
            VerificationIntervalMinutes * 60.0f,
            true
        );
    }
}

void UDRMManager::VerifyLicense()
{
    if (bOfflineMode)
    {
        if (VerifyOfflineLicense())
        {
            SetStatus(EDRMStatus::OfflineMode);
        }
        else
        {
            SetStatus(EDRMStatus::Invalid);
        }
        return;
    }

    SetStatus(EDRMStatus::Checking);

    if (PerformOnlineVerification())
    {
        LastOnlineVerification = FDateTime::UtcNow();
        SaveOfflineToken();
    }
}

bool UDRMManager::PerformOnlineVerification()
{
    // 根据提供商执行在线验证
    FString MachineId = GenerateMachineId();

    switch (CurrentProvider)
    {
        case EDRMProvider::Steam:
            // Steam在线验证
            return true;  // 简化

        case EDRMProvider::Epic:
            // Epic在线验证
            return true;

        case EDRMProvider::Custom:
        {
            // 自定义服务器验证
            TSharedRef<IHttpRequest> Request = FHttpModule::Get().CreateRequest();
            Request->SetURL("https://license.mygame.com/verify");
            Request->SetVerb("POST");
            Request->SetHeader("Content-Type", "application/json");

            FString JsonPayload = FString::Printf(
                TEXT("{\"app_id\":\"%s\",\"machine_id\":\"%s\"}"),
                *ApplicationId, *MachineId
            );
            Request->SetContentAsString(JsonPayload);

            Request->OnProcessRequestComplete().BindLambda([this](FHttpRequestPtr Req, FHttpResponsePtr Resp, bool bSuccess)
            {
                if (bSuccess && Resp.IsValid())
                {
                    // 解析响应
                    // 设置许可信息
                    SetStatus(EDRMStatus::Valid);
                }
                else
                {
                    SetStatus(EDRMStatus::Invalid);
                }
            });

            Request->ProcessRequest();
            return false;  // 异步，等待回调
        }

        default:
            return false;
    }
}

bool UDRMManager::VerifyOfflineLicense()
{
    if (OfflineToken.IsEmpty())
    {
        return false;
    }

    // 检查离线宽限期
    FTimespan TimeSinceOnline = FDateTime::UtcNow() - LastOnlineVerification;
    if (TimeSinceOnline.GetTotalDays() > OfflineGracePeriodDays)
    {
        UE_LOG(LogTemp, Warning, TEXT("Offline grace period expired"));
        return false;
    }

    // 验证离线令牌
    // 这里可以验证令牌签名、机器ID等

    return true;
}

void UDRMManager::SaveOfflineToken()
{
    FString TokenPath = FPaths::Combine(FPaths::ProjectSavedDir(), TEXT("license.dat"));

    FString TokenData = FString::Printf(
        TEXT("%s|%s|%s"),
        *OfflineToken,
        *LastOnlineVerification.ToString(),
        *GenerateMachineId()
    );

    // 加密令牌数据
    TArray<uint8> EncryptedData;
    // ... 加密逻辑

    FFileHelper::SaveArrayToFile(EncryptedData, *TokenPath);
}

bool UDRMManager::LoadOfflineToken()
{
    FString TokenPath = FPaths::Combine(FPaths::ProjectSavedDir(), TEXT("license.dat"));

    if (!FPaths::FileExists(TokenPath))
    {
        return false;
    }

    TArray<uint8> EncryptedData;
    FFileHelper::LoadFileToArray(EncryptedData, *TokenPath);

    // 解密并解析
    // ...

    return true;
}

FString UDRMManager::GenerateMachineId()
{
    // 生成基于硬件的唯一标识
    FString CpuId = FPlatformMisc::GetProcessorBrand();
    FString MachineName = FPlatformProcess::ComputerName();
    FString OsId = FPlatformMisc::GetOSVersion();

    // 组合并哈希
    FString Combined = FString::Printf(TEXT("%s|%s|%s"), *CpuId, *MachineName, *OsId);

    return FMD5::HashAnsiString(*Combined);
}

void UDRMManager::SetStatus(EDRMStatus NewStatus)
{
    if (CurrentStatus != NewStatus)
    {
        CurrentStatus = NewStatus;
        OnStatusChanged.Broadcast(NewStatus);

        UE_LOG(LogTemp, Log, TEXT("DRM Status changed to: %d"), (int32)NewStatus);
    }
}

bool UDRMManager::HasEntitlement(const FString& EntitlementId) const
{
    return CurrentLicense.Entitlements.Contains(EntitlementId);
}

void UDRMManager::RefreshLicense()
{
    VerifyLicense();
}

void UDRMManager::SetOfflineMode(bool bEnabled)
{
    bOfflineMode = bEnabled;

    if (bOfflineMode)
    {
        VerifyOfflineLicense();
    }
    else
    {
        VerifyLicense();
    }
}
```

---

## 29.2 Steam集成

### 29.2.1 Steam SDK集成

```csharp
// SteamDRMIntegration.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "SteamDRMIntegration.generated.h"

USTRUCT(BlueprintType)
struct FSteamUserInfo
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString SteamId;

    UPROPERTY(BlueprintReadOnly)
    FString PersonaName;

    UPROPERTY(BlueprintReadOnly)
    int32 PersonaState = 0;  // 0=offline, 1=online, etc.

    UPROPERTY(BlueprintReadOnly)
    FString AvatarUrl;

    UPROPERTY(BlueprintReadOnly)
    bool bIsOwned = false;
};

USTRUCT(BlueprintType)
struct FSteamDLCInfo
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    int32 AppId;

    UPROPERTY(BlueprintReadOnly)
    FString Name;

    UPROPERTY(BlueprintReadOnly)
    bool bInstalled = false;

    UPROPERTY(BlueprintReadOnly)
    bool bAvailable = false;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnSteamInitialized, bool, bSuccess);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnSteamOverlayToggled, bool, bIsOpen);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnDLCInstalled, int32, AppId);

UCLASS()
class STEAMINTEGRATION_API USteamDRMIntegration : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 初始化Steam
    UFUNCTION(BlueprintCallable, Category = "Steam")
    bool InitializeSteam(uint32 AppId);

    // 获取用户信息
    UFUNCTION(BlueprintCallable, Category = "Steam")
    FSteamUserInfo GetUserInfo();

    // 检查拥有权
    UFUNCTION(BlueprintCallable, Category = "Steam")
    bool CheckOwnership();

    // DLC管理
    UFUNCTION(BlueprintCallable, Category = "Steam")
    bool IsDLCOwned(int32 DLCAppId) const;

    UFUNCTION(BlueprintCallable, Category = "Steam")
    bool IsDLCInstalled(int32 DLCAppId) const;

    UFUNCTION(BlueprintCallable, Category = "Steam")
    TArray<FSteamDLCInfo> GetOwnedDLCs() const;

    // 成就
    UFUNCTION(BlueprintCallable, Category = "Steam")
    bool UnlockAchievement(const FString& AchievementId);

    UFUNCTION(BlueprintCallable, Category = "Steam")
    float GetAchievementProgress(const FString& AchievementId);

    // 排行榜
    UFUNCTION(BlueprintCallable, Category = "Steam")
    bool UploadScore(const FString& LeaderboardName, int32 Score);

    // 云存储
    UFUNCTION(BlueprintCallable, Category = "Steam")
    bool SaveToCloud(const FString& FileName, const TArray<uint8>& Data);

    UFUNCTION(BlueprintCallable, Category = "Steam")
    bool LoadFromCloud(const FString& FileName, TArray<uint8>& OutData);

    // 微交易
    UFUNCTION(BlueprintCallable, Category = "Steam")
    bool InitiatePurchase(int32 ItemId, const FString& Currency, int32 Amount);

    UFUNCTION(BlueprintPure, Category = "Steam")
    bool IsOverlayActive() const;

    UPROPERTY(BlueprintAssignable, Category = "Steam")
    FOnSteamInitialized OnInitialized;

    UPROPERTY(BlueprintAssignable, Category = "Steam")
    FOnSteamOverlayToggled OnOverlayToggled;

    UPROPERTY(BlueprintAssignable, Category = "Steam")
    FOnDLCInstalled OnDLCInstalled;

private:
    bool bSteamInitialized = false;
    uint32 CurrentAppId = 0;
    FSteamUserInfo CurrentUser;

    void OnSteamOverlayActivated(bool bIsOpen);
    void OnDLCInstalledCallback(int32 DLCAppId);

    void PollSteamCallbacks();
    FTimerHandle CallbackTimerHandle;
};
```

```csharp
// SteamDRMIntegration.cpp
#include "SteamDRMIntegration.h"
#include "TimerManager.h"

// Steam API头文件（需要Steam SDK）
// #include "steam/steam_api.h"

void USteamDRMIntegration::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
}

void USteamDRMIntegration::Deinitialize()
{
    if (GetWorld())
    {
        GetWorld()->GetTimerManager().ClearTimer(CallbackTimerHandle);
    }

    // Steam API关闭
    // SteamAPI_Shutdown();

    Super::Deinitialize();
}

bool USteamDRMIntegration::InitializeSteam(uint32 AppId)
{
    if (bSteamInitialized)
    {
        return true;
    }

    CurrentAppId = AppId;

    // 设置App ID
    FString AppIdString = FString::FromInt(AppId);
    FPlatformMisc::SetEnvironmentVar(TEXT("SteamAppId"), *AppIdString);

    // 初始化Steam API
    // bool bSuccess = SteamAPI_Init();

    // 简化实现
    bool bSuccess = true;

    if (bSuccess)
    {
        bSteamInitialized = true;

        // 获取用户信息
        CurrentUser = GetUserInfo();

        // 启动回调轮询
        if (GetWorld())
        {
            GetWorld()->GetTimerManager().SetTimer(
                CallbackTimerHandle,
                this,
                &USteamDRMIntegration::PollSteamCallbacks,
                0.1f,
                true
            );
        }

        UE_LOG(LogTemp, Log, TEXT("Steam initialized for AppId: %d"), AppId);
        OnInitialized.Broadcast(true);
        return true;
    }
    else
    {
        UE_LOG(LogTemp, Error, TEXT("Failed to initialize Steam"));
        OnInitialized.Broadcast(false);
        return false;
    }
}

FSteamUserInfo USteamDRMIntegration::GetUserInfo()
{
    FSteamUserInfo Info;

    if (!bSteamInitialized)
    {
        return Info;
    }

    // 获取Steam用户信息
    // ISteamUser* SteamUser = SteamUser();
    // if (SteamUser)
    // {
    //     Info.SteamId = FString::Printf(TEXT("%llu"), SteamUser->GetSteamID().ConvertToUint64());
    //     Info.bIsOwned = SteamUser->BIsLoggedOn();
    // }

    // ISteamFriends* SteamFriends = SteamFriends();
    // if (SteamFriends)
    // {
    //     Info.PersonaName = FString(SteamFriends->GetPersonaName());
    //     Info.PersonaState = SteamFriends->GetPersonaState();
    // }

    // 简化实现
    Info.SteamId = TEXT("76561198000000000");
    Info.PersonaName = TEXT("Player");
    Info.PersonaState = 1;
    Info.bIsOwned = true;

    return Info;
}

bool USteamDRMIntegration::CheckOwnership()
{
    if (!bSteamInitialized)
    {
        return false;
    }

    // 检查用户是否拥有游戏
    // ISteamUser* SteamUser = SteamUser();
    // return SteamUser && SteamUser->BIsLoggedOn();

    return CurrentUser.bIsOwned;
}

bool USteamDRMIntegration::IsDLCOwned(int32 DLCAppId) const
{
    if (!bSteamInitialized)
    {
        return false;
    }

    // ISteamApps* SteamApps = SteamApps();
    // return SteamApps && SteamApps->BIsSubscribedApp(DLCAppId);

    return true;
}

bool USteamDRMIntegration::IsDLCInstalled(int32 DLCAppId) const
{
    if (!bSteamInitialized)
    {
        return false;
    }

    // ISteamApps* SteamApps = SteamApps();
    // return SteamApps && SteamApps->BIsDlcInstalled(DLCAppId);

    return true;
}

TArray<FSteamDLCInfo> USteamDRMIntegration::GetOwnedDLCs() const
{
    TArray<FSteamDLCInfo> DLCs;

    if (!bSteamInitialized)
    {
        return DLCs;
    }

    // ISteamApps* SteamApps = SteamApps();
    // if (SteamApps)
    // {
    //     int32 DLCCount = SteamApps->GetDLCCount();
    //     for (int32 i = 0; i < DLCCount; i++)
    //     {
    //         int32 AppId;
    //         bool bAvailable;
    //         char Name[128];
    //         if (SteamApps->BGetDLCDataByIndex(i, &AppId, &bAvailable, Name, sizeof(Name)))
    //         {
    //             FSteamDLCInfo Info;
    //             Info.AppId = AppId;
    //             Info.Name = FString(Name);
    //             Info.bAvailable = bAvailable;
    //             Info.bInstalled = SteamApps->BIsDlcInstalled(AppId);
    //             DLCs.Add(Info);
    //         }
    //     }
    // }

    return DLCs;
}

bool USteamDRMIntegration::UnlockAchievement(const FString& AchievementId)
{
    if (!bSteamInitialized)
    {
        return false;
    }

    // ISteamUserStats* SteamStats = SteamUserStats();
    // if (SteamStats)
    // {
    //     bool bSuccess = SteamStats->SetAchievement(TCHAR_TO_UTF8(*AchievementId));
    //     SteamStats->StoreStats();
    //     return bSuccess;
    // }

    return true;
}

bool USteamDRMIntegration::SaveToCloud(const FString& FileName, const TArray<uint8>& Data)
{
    if (!bSteamInitialized)
    {
        return false;
    }

    // ISteamRemoteStorage* RemoteStorage = SteamRemoteStorage();
    // if (RemoteStorage)
    // {
    //     int32 Written = RemoteStorage->FileWrite(TCHAR_TO_UTF8(*FileName), Data.GetData(), Data.Num());
    //     return Written == Data.Num();
    // }

    return true;
}

bool USteamDRMIntegration::LoadFromCloud(const FString& FileName, TArray<uint8>& OutData)
{
    if (!bSteamInitialized)
    {
        return false;
    }

    // ISteamRemoteStorage* RemoteStorage = SteamRemoteStorage();
    // if (RemoteStorage && RemoteStorage->FileExists(TCHAR_TO_UTF8(*FileName)))
    // {
    //     int32 Size = RemoteStorage->GetFileSize(TCHAR_TO_UTF8(*FileName));
    //     OutData.SetNumUninitialized(Size);
    //     int32 Read = RemoteStorage->FileRead(TCHAR_TO_UTF8(*FileName), OutData.GetData(), Size);
    //     return Read == Size;
    // }

    return false;
}

void USteamDRMIntegration::PollSteamCallbacks()
{
    // 处理Steam回调
    // SteamAPI_RunCallbacks();
}

void USteamDRMIntegration::OnSteamOverlayActivated(bool bIsOpen)
{
    OnOverlayToggled.Broadcast(bIsOpen);
}

void USteamDRMIntegration::OnDLCInstalledCallback(int32 DLCAppId)
{
    OnDLCInstalled.Broadcast(DLCAppId);
}
```

---

## 29.3 Epic在线服务集成

### 29.3.1 EOS集成

```csharp
// EpicOnlineServicesIntegration.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "EpicOnlineServicesIntegration.generated.h"

USTRUCT(BlueprintType)
struct FEOSUserInfo
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString EpicAccountId;

    UPROPERTY(BlueprintReadOnly)
    FString DisplayName;

    UPROPERTY(BlueprintReadOnly)
    FString ProductUserId;

    UPROPERTY(BlueprintReadOnly)
    bool bIsLoggedIn = false;
};

USTRUCT(BlueprintType)
struct FEOSEntitlement
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString EntitlementId;

    UPROPERTY(BlueprintReadOnly)
    FString CatalogItemId;

    UPROPERTY(BlueprintReadOnly)
    FString Name;

    UPROPERTY(BlueprintReadOnly)
    bool bIsRedeemed = false;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnEOSLoginComplete, bool, bSuccess);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnEntitlementVerified, const FEOSEntitlement&, Entitlement);

UCLASS()
class EOSINTEGRATION_API UEpicOnlineServicesIntegration : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 初始化EOS
    UFUNCTION(BlueprintCallable, Category = "EOS")
    bool InitializeEOS(const FString& ProductId, const FString& SandboxId, const FString& ClientId, const FString& ClientSecret);

    // 登录
    UFUNCTION(BlueprintCallable, Category = "EOS")
    void LoginWithAccountPortal();

    UFUNCTION(BlueprintCallable, Category = "EOS")
    void LoginWithPersistedAuth();

    UFUNCTION(BlueprintCallable, Category = "EOS")
    void LoginWithDevAuth(const FString& DevAuthHost, int32 Port);

    // 登出
    UFUNCTION(BlueprintCallable, Category = "EOS")
    void Logout();

    // 用户信息
    UFUNCTION(BlueprintPure, Category = "EOS")
    FEOSUserInfo GetUserInfo() const { return CurrentUser; }

    UFUNCTION(BlueprintPure, Category = "EOS")
    bool IsLoggedIn() const { return CurrentUser.bIsLoggedIn; }

    // 权限验证
    UFUNCTION(BlueprintCallable, Category = "EOS")
    void VerifyEntitlement(const FString& EntitlementId);

    UFUNCTION(BlueprintCallable, Category = "EOS")
    void QueryEntitlements();

    UFUNCTION(BlueprintPure, Category = "EOS")
    TArray<FEOSEntitlement> GetEntitlements() const;

    // 云存储
    UFUNCTION(BlueprintCallable, Category = "EOS")
    void SaveToCloud(const FString& SlotName, const TArray<uint8>& Data);

    UFUNCTION(BlueprintCallable, Category = "EOS")
    void LoadFromCloud(const FString& SlotName);

    // 成就
    UFUNCTION(BlueprintCallable, Category = "EOS")
    void UnlockAchievement(const FString& AchievementId);

    UPROPERTY(BlueprintAssignable, Category = "EOS")
    FOnEOSLoginComplete OnLoginComplete;

    UPROPERTY(BlueprintAssignable, Category = "EOS")
    FOnEntitlementVerified OnEntitlementVerified;

private:
    bool bEOSInitialized = false;
    FEOSUserInfo CurrentUser;
    TArray<FEOSEntitlement> CachedEntitlements;

    FString ProductId;
    FString SandboxId;

    void HandleLoginResult(bool bSuccess, const FString& EpicAccountId, const FString& ProductUserId);
    void HandleEntitlementQueryResult(const TArray<FEOSEntitlement>& Entitlements);
};
```

---

## 29.4 资源加密保护

### 29.4.1 资源加密系统

```csharp
// AssetEncryptionSystem.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "AssetEncryptionSystem.generated.h"

UENUM(BlueprintType)
enum class EEncryptionMethod : uint8
{
    AES_256,
    ChaCha20,
    Custom
};

USTRUCT(BlueprintType)
struct FEncryptionConfig
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    EEncryptionMethod Method = EEncryptionMethod::AES_256;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bEncryptOnCook = true;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bDecryptOnLoad = true;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TArray<FString> EncryptedExtensions;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bUseDerivationKey = true;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 KeyDerivationIterations = 10000;
};

UCLASS()
class ASSETENCRYPTION_API UAssetEncryptionSystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 加密资产
    UFUNCTION(BlueprintCallable, Category = "Encryption")
    bool EncryptAsset(const FString& AssetPath, const FString& OutputPath);

    // 解密资产
    UFUNCTION(BlueprintCallable, Category = "Encryption")
    bool DecryptAsset(const FString& EncryptedPath, TArray<uint8>& OutData);

    // 批量加密
    UFUNCTION(BlueprintCallable, Category = "Encryption")
    void EncryptDirectory(const FString& SourceDir, const FString& OutputDir, const TArray<FString>& Extensions);

    // 运行时解密注册
    void RegisterDecryptor();

    // 生成加密密钥
    UFUNCTION(BlueprintCallable, Category = "Encryption")
    FString GenerateEncryptionKey();

    // 设置密钥
    void SetEncryptionKey(const TArray<uint8>& Key);

    // 配置
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Encryption")
    FEncryptionConfig Config;

private:
    TArray<uint8> MasterKey;
    TMap<FString, TArray<uint8>> AssetKeys;

    bool EncryptData_AES256(const TArray<uint8>& Plaintext, TArray<uint8>& OutCiphertext);
    bool DecryptData_AES256(const TArray<uint8>& Ciphertext, TArray<uint8>& OutPlaintext);

    bool EncryptData_ChaCha20(const TArray<uint8>& Plaintext, TArray<uint8>& OutCiphertext);
    bool DecryptData_ChaCha20(const TArray<uint8>& Ciphertext, TArray<uint8>& OutPlaintext);

    TArray<uint8> DeriveKey(const FString& AssetPath);
    FString GetObfuscatedKeyPath(const FString& AssetPath) const;
};
```

```csharp
// AssetEncryptionSystem.cpp
#include "AssetEncryptionSystem.h"
#include "Misc/FileHelper.h"
#include "Misc/Paths.h"
#include "HAL/PlatformFileManager.h"

void UAssetEncryptionSystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    // 加载主密钥（从安全存储）
    FString KeyPath = FPaths::Combine(FPaths::ProjectDir(), TEXT("Content/Security/master.key"));
    if (FPaths::FileExists(KeyPath))
    {
        TArray<uint8> KeyData;
        FFileHelper::LoadFileToArray(KeyData, *KeyPath);
        SetEncryptionKey(KeyData);
    }

    // 注册解密器
    RegisterDecryptor();
}

bool UAssetEncryptionSystem::EncryptAsset(const FString& AssetPath, const FString& OutputPath)
{
    // 读取原始资产
    TArray<uint8> AssetData;
    if (!FFileHelper::LoadFileToArray(AssetData, *AssetPath))
    {
        UE_LOG(LogTemp, Error, TEXT("Failed to load asset: %s"), *AssetPath);
        return false;
    }

    // 加密
    TArray<uint8> EncryptedData;
    bool bSuccess = false;

    switch (Config.Method)
    {
        case EEncryptionMethod::AES_256:
            bSuccess = EncryptData_AES256(AssetData, EncryptedData);
            break;

        case EEncryptionMethod::ChaCha20:
            bSuccess = EncryptData_ChaCha20(AssetData, EncryptedData);
            break;

        default:
            break;
    }

    if (!bSuccess)
    {
        return false;
    }

    // 写入加密文件
    return FFileHelper::SaveArrayToFile(EncryptedData, *OutputPath);
}

bool UAssetEncryptionSystem::DecryptAsset(const FString& EncryptedPath, TArray<uint8>& OutData)
{
    // 读取加密数据
    TArray<uint8> EncryptedData;
    if (!FFileHelper::LoadFileToArray(EncryptedData, *EncryptedPath))
    {
        return false;
    }

    // 解密
    switch (Config.Method)
    {
        case EEncryptionMethod::AES_256:
            return DecryptData_AES256(EncryptedData, OutData);

        case EEncryptionMethod::ChaCha20:
            return DecryptData_ChaCha20(EncryptedData, OutData);

        default:
            return false;
    }
}

bool UAssetEncryptionSystem::EncryptData_AES256(const TArray<uint8>& Plaintext, TArray<uint8>& OutCiphertext)
{
    if (MasterKey.Num() != 32)
    {
        UE_LOG(LogTemp, Error, TEXT("Invalid master key size for AES-256"));
        return false;
    }

    // 生成随机IV
    TArray<uint8> IV;
    IV.SetNumUninitialized(16);
    FMath::GetRandSeed();  // 使用更好的随机源

    // AES-256-CBC加密
    // 这里使用简化实现，实际应使用加密库

    // 构建输出：IV + 密文
    OutCiphertext = IV;
    OutCiphertext.Append(Plaintext);  // 简化：实际应加密

    return true;
}

bool UAssetEncryptionSystem::DecryptData_AES256(const TArray<uint8>& Ciphertext, TArray<uint8>& OutPlaintext)
{
    if (Ciphertext.Num() < 16)
    {
        return false;
    }

    // 提取IV
    TArray<uint8> IV;
    IV.Append(Ciphertext.GetData(), 16);

    // 解密
    OutPlaintext.Append(Ciphertext.GetData() + 16, Ciphertext.Num() - 16);

    return true;
}

void UAssetEncryptionSystem::EncryptDirectory(const FString& SourceDir, const FString& OutputDir, const TArray<FString>& Extensions)
{
    TArray<FString> Files;
    IFileManager::Get().FindFilesRecursive(Files, *SourceDir, TEXT("*"), true, false);

    int32 EncryptedCount = 0;

    for (const FString& File : Files)
    {
        FString Extension = FPaths::GetExtension(File).ToLower();

        if (Extensions.Contains(Extension))
        {
            FString RelativePath = File.RightChop(SourceDir.Len());
            FString OutputPath = FPaths::Combine(OutputDir, RelativePath + TEXT(".enc"));

            if (EncryptAsset(File, OutputPath))
            {
                EncryptedCount++;
            }
        }
    }

    UE_LOG(LogTemp, Log, TEXT("Encrypted %d files"), EncryptedCount);
}

TArray<uint8> UAssetEncryptionSystem::DeriveKey(const FString& AssetPath)
{
    // 使用PBKDF2派生资产特定密钥
    TArray<uint8> DerivedKey;
    DerivedKey.SetNumUninitialized(32);

    // 简化实现
    FString Salt = AssetPath + FString::FromInt(FDateTime::UtcNow().ToUnixTimestamp());

    // 实际应使用HKDF或PBKDF2
    for (int32 i = 0; i < 32; i++)
    {
        DerivedKey[i] = MasterKey[i % MasterKey.Num()] ^ Salt[i % Salt.Len()];
    }

    return DerivedKey;
}

void UAssetEncryptionSystem::SetEncryptionKey(const TArray<uint8>& Key)
{
    MasterKey = Key;
}

void UAssetEncryptionSystem::RegisterDecryptor()
{
    // 注册资产加载回调
    // 在资产加载时自动解密

    // FCoreDelegates::OnAssetLoaded.AddLambda([this](UObject* Asset)
    // {
    //     if (Config.bDecryptOnLoad)
    //     {
    //         // 检查并解密
    //     }
    // });
}

FString UAssetEncryptionSystem::GenerateEncryptionKey()
{
    // 生成256位密钥
    TArray<uint8> Key;
    Key.SetNumUninitialized(32);

    for (int32 i = 0; i < 32; i++)
    {
        Key[i] = FMath::RandRange(0, 255);
    }

    // 转换为Base64字符串
    return FBase64::Encode(Key);
}
```

---

## 29.5 离线模式设计

### 29.5.1 离线授权系统

```csharp
// OfflineAuthorizationManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "OfflineAuthorizationManager.generated.h"

USTRUCT(BlueprintType)
struct FOfflineToken
{
    GENERATED_BODY()

    UPROPERTY()
    FString TokenId;

    UPROPERTY()
    FString MachineId;

    UPROPERTY()
    FString UserId;

    UPROPERTY()
    FString ProductId;

    UPROPERTY()
    int64 IssueTime;

    UPROPERTY()
    int64 ExpirationTime;

    UPROPERTY()
    FString Signature;
};

UCLASS()
class OFFLINEAUTH_API UOfflineAuthorizationManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 生成离线令牌
    UFUNCTION(BlueprintCallable, Category = "OfflineAuth")
    FString GenerateOfflineToken(const FString& UserId, int32 ValidityDays = 7);

    // 验证离线令牌
    UFUNCTION(BlueprintCallable, Category = "OfflineAuth")
    bool ValidateOfflineToken(const FString& Token);

    // 刷新离线令牌
    UFUNCTION(BlueprintCallable, Category = "OfflineAuth")
    bool RefreshOfflineToken();

    // 检查离线状态
    UFUNCTION(BlueprintPure, Category = "OfflineAuth")
    bool IsOfflineModeAvailable() const;

    UFUNCTION(BlueprintPure, Category = "OfflineAuth")
    int32 GetRemainingOfflineDays() const;

    // 管理离线令牌
    UFUNCTION(BlueprintCallable, Category = "OfflineAuth")
    void ClearOfflineTokens();

    // 配置
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "OfflineAuth")
    int32 MaxOfflineDays = 30;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "OfflineAuth")
    int32 MaxOfflineLogins = 5;

private:
    TArray<FOfflineToken> StoredTokens;
    FString CurrentMachineId;
    FString TokenStoragePath;

    bool SaveToken(const FOfflineToken& Token);
    bool LoadTokens();
    FString SignToken(const FOfflineToken& Token);
    bool VerifySignature(const FOfflineToken& Token);
    FString GetCurrentMachineId();
};
```

---

## 29.6 实践任务

### 任务1：实现Steam集成

为游戏添加Steam DRM：
- 初始化Steam SDK
- 检查拥有权
- DLC验证

### 任务2：实现资源加密

创建资源加密系统：
- AES加密实现
- 自动加密流程
- 运行时解密

### 任务3：设计离线模式

实现离线授权：
- 令牌生成
- 本地验证
- 宽限期管理

---

## 29.7 总结

本课学习了：
- DRM系统架构设计
- Steam平台集成
- Epic在线服务集成
- 资源加密保护技术
- 离线模式设计

**下一课预告**：监控与运维 - 服务器健康监控、日志聚合与告警系统设计。

---

*课程版本：1.0*
*最后更新：2026-04-10*
