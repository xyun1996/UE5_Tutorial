# 第22课：PlayFab集成

> 学习目标：掌握PlayFab SDK在UE5中的集成，实现用户认证、云脚本、匹配系统和数据分析功能。

---

## 22.1 PlayFab SDK 集成

### 22.1.1 安装PlayFab SDK

**下载并安装SDK：**

1. 从PlayFab官网或GitHub下载UE4/UE5 SDK
2. 将SDK插件复制到项目的Plugins目录
3. 在项目的Build.cs中添加依赖

```csharp
// ProjectName.Build.cs
public class ProjectName : ModuleRules
{
    public ProjectName(ReadOnlyTargetRules Target) : base(Target)
    {
        PublicDependencyModuleNames.AddRange(new string[] {
            "Core", "CoreUObject", "Engine", "InputCore",
            "PlayFab", "PlayFabCpp", "PlayFabCommon"
        });
    }
}
```

### 22.1.2 PlayFab配置

```csharp
// PlayFabConfigSubsystem.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "PlayFab.h"
#include "PlayFabSettings.h"

#include "PlayFabConfigSubsystem.generated.h"

USTRUCT(BlueprintType)
struct FPlayFabConfig
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "PlayFab")
    FString TitleId;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "PlayFab")
    FString SecretKey; // 仅服务器端使用

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "PlayFab")
    FString ProductionEnvironmentUrl = "https://titleId.playfabapi.com";

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "PlayFab")
    bool bEnableCompression = true;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnPlayFabConfigured, bool, bSuccess);

UCLASS()
class PLAYFABTUTORIAL_API UPlayFabConfigSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    UFUNCTION(BlueprintCallable, Category = "PlayFab")
    void ConfigurePlayFab(const FPlayFabConfig& Config);

    UFUNCTION(BlueprintPure, Category = "PlayFab")
    bool IsConfigured() const { return bIsConfigured; }

    UFUNCTION(BlueprintPure, Category = "PlayFab")
    FString GetTitleId() const { return TitleId; }

    UPROPERTY(BlueprintAssignable, Category = "PlayFab")
    FOnPlayFabConfigured OnPlayFabConfigured;

private:
    bool bIsConfigured = false;
    FString TitleId;
};
```

```csharp
// PlayFabConfigSubsystem.cpp
#include "PlayFabConfigSubsystem.h"
#include "PlayFabSettings.h"

void UPlayFabConfigSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    // 从配置文件加载默认设置
    // 可从GameConfig.ini读取
}

void UPlayFabConfigSubsystem::ConfigurePlayFab(const FPlayFabConfig& Config)
{
    if (Config.TitleId.IsEmpty())
    {
        UE_LOG(LogTemp, Error, TEXT("PlayFab TitleId is empty"));
        OnPlayFabConfigured.Broadcast(false);
        return;
    }

    TitleId = Config.TitleId;

    // 配置PlayFab设置
    PlayFab::PlayFabSettings::staticSettings->titleId = TCHAR_TO_UTF8(*Config.TitleId);

    // 设置生产环境URL
    FString EnvironmentUrl = Config.ProductionEnvironmentUrl;
    EnvironmentUrl = EnvironmentUrl.Replace(TEXT("titleId"), *Config.TitleId);
    PlayFab::PlayFabSettings::staticSettings->productionEnvironmentUrl = TCHAR_TO_UTF8(*EnvironmentUrl);

    bIsConfigured = true;
    UE_LOG(LogTemp, Log, TEXT("PlayFab configured for TitleId: %s"), *Config.TitleId);

    OnPlayFabConfigured.Broadcast(true);
}
```

---

## 22.2 用户认证系统

### 22.2.1 多种认证方式实现

```csharp
// PlayFabAuthManager.h
#pragma once

#include "CoreMinimal.h"
#include "PlayFabAuthenticationModels.h"
#include "PlayFabClientAPI.h"

#include "PlayFabAuthManager.generated.h"

USTRUCT(BlueprintType)
struct FPlayFabUserInfo
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString PlayFabId;

    UPROPERTY(BlueprintReadOnly)
    FString SessionTicket;

    UPROPERTY(BlueprintReadOnly)
    FString EntityToken;

    UPROPERTY(BlueprintReadOnly)
    FString DisplayName;

    UPROPERTY(BlueprintReadOnly)
    FString AvatarUrl;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnLoginComplete, bool, bSuccess, FPlayFabUserInfo, UserInfo);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnLogoutComplete, bool, bSuccess);

UCLASS()
class PLAYFABTUTORIAL_API UPlayFabAuthManager : public UObject
{
    GENERATED_BODY()

public:
    // 匿名登录（设备ID）
    UFUNCTION(BlueprintCallable, Category = "PlayFab|Auth")
    void LoginWithDevice(const FString& DeviceId, const FString& DisplayName = TEXT(""));

    // 自定义ID登录
    UFUNCTION(BlueprintCallable, Category = "PlayFab|Auth")
    void LoginWithCustomId(const FString& CustomId, const FString& DisplayName = TEXT(""));

    // 邮箱密码登录
    UFUNCTION(BlueprintCallable, Category = "PlayFab|Auth")
    void LoginWithEmail(const FString& Email, const FString& Password);

    // 注册新账号
    UFUNCTION(BlueprintCallable, Category = "PlayFab|Auth")
    void RegisterWithEmail(const FString& Email, const FString& Password, const FString& DisplayName);

    // Steam登录
    UFUNCTION(BlueprintCallable, Category = "PlayFab|Auth")
    void LoginWithSteam(const FString& SteamTicket);

    // Google登录
    UFUNCTION(BlueprintCallable, Category = "PlayFab|Auth")
    void LoginWithGoogle(const FString& ServerAuthCode);

    // Apple登录
    UFUNCTION(BlueprintCallable, Category = "PlayFab|Auth")
    void LoginWithApple(const FString& IdentityToken);

    // 登出
    UFUNCTION(BlueprintCallable, Category = "PlayFab|Auth")
    void Logout();

    // 检查登录状态
    UFUNCTION(BlueprintPure, Category = "PlayFab|Auth")
    bool IsLoggedIn() const;

    // 获取当前用户信息
    UFUNCTION(BlueprintPure, Category = "PlayFab|Auth")
    FPlayFabUserInfo GetUserInfo() const { return CurrentUser; }

    UPROPERTY(BlueprintAssignable, Category = "PlayFab|Auth")
    FOnLoginComplete OnLoginComplete;

    UPROPERTY(BlueprintAssignable, Category = "PlayFab|Auth")
    FOnLogoutComplete OnLogoutComplete;

private:
    FPlayFabUserInfo CurrentUser;
    bool bIsLoggedIn = false;

    void HandleLoginResult(const PlayFab::ClientModels::FLoginResult& Result);
    void HandleLoginError(const PlayFab::FPlayFabError& Error);
};
```

```csharp
// PlayFabAuthManager.cpp
#include "PlayFabAuthManager.h"
#include "PlayFabSettings.h"

void UPlayFabAuthManager::LoginWithDevice(const FString& DeviceId, const FString& DisplayName)
{
    using namespace PlayFab::ClientModels;

    FLoginWithCustomIDRequest Request;
    Request.CustomId = DeviceId.IsEmpty() ? FPlatformMisc::GetDeviceId() : DeviceId;
    Request.CreateAccount = true;

    // 设置玩家数据
    if (!DisplayName.IsEmpty())
    {
        FGetPlayerCombinedInfoRequestParams InfoRequestParams;
        InfoRequestParams.GetPlayerProfile = true;
        InfoRequestParams.GetTitleData = true;
        Request.InfoRequestParameters = InfoRequestParams;
    }

    PlayFab::PlayFabClientAPI clientAPI;
    clientAPI.LoginWithCustomID(Request,
        PlayFab::FLoginWithCustomIDDelegate::CreateLambda([this](const FLoginResult& Result)
        {
            HandleLoginResult(Result);
        }),
        PlayFab::FPlayFabErrorDelegate::CreateLambda([this](const PlayFab::FPlayFabError& Error)
        {
            HandleLoginError(Error);
        })
    );
}

void UPlayFabAuthManager::LoginWithCustomId(const FString& CustomId, const FString& DisplayName)
{
    using namespace PlayFab::ClientModels;

    FLoginWithCustomIDRequest Request;
    Request.CustomId = CustomId;
    Request.CreateAccount = true;

    PlayFab::PlayFabClientAPI clientAPI;
    clientAPI.LoginWithCustomID(Request,
        PlayFab::FLoginWithCustomIDDelegate::CreateLambda([this](const FLoginResult& Result)
        {
            HandleLoginResult(Result);
        }),
        PlayFab::FPlayFabErrorDelegate::CreateLambda([this](const PlayFab::FPlayFabError& Error)
        {
            HandleLoginError(Error);
        })
    );
}

void UPlayFabAuthManager::LoginWithEmail(const FString& Email, const FString& Password)
{
    using namespace PlayFab::ClientModels;

    FLoginWithEmailAddressRequest Request;
    Request.Email = Email;
    Request.Password = Password;

    PlayFab::PlayFabClientAPI clientAPI;
    clientAPI.LoginWithEmailAddress(Request,
        PlayFab::FLoginWithEmailAddressDelegate::CreateLambda([this](const FLoginResult& Result)
        {
            HandleLoginResult(Result);
        }),
        PlayFab::FPlayFabErrorDelegate::CreateLambda([this](const PlayFab::FPlayFabError& Error)
        {
            HandleLoginError(Error);
        })
    );
}

void UPlayFabAuthManager::RegisterWithEmail(const FString& Email, const FString& Password, const FString& DisplayName)
{
    using namespace PlayFab::ClientModels;

    FRegisterPlayFabUserRequest Request;
    Request.Email = Email;
    Request.Password = Password;
    Request.DisplayName = DisplayName;
    Request.RequireBothUsernameAndEmail = false;

    PlayFab::PlayFabClientAPI clientAPI;
    clientAPI.RegisterPlayFabUser(Request,
        PlayFab::FRegisterPlayFabUserDelegate::CreateLambda([this](const FRegisterPlayFabUserResult& Result)
        {
            // 注册成功后自动登录
            FPlayFabUserInfo Info;
            Info.PlayFabId = Result.PlayFabId;
            Info.DisplayName = DisplayName;
            CurrentUser = Info;
            bIsLoggedIn = true;

            OnLoginComplete.Broadcast(true, CurrentUser);
        }),
        PlayFab::FPlayFabErrorDelegate::CreateLambda([this](const PlayFab::FPlayFabError& Error)
        {
            HandleLoginError(Error);
        })
    );
}

void UPlayFabAuthManager::LoginWithSteam(const FString& SteamTicket)
{
    using namespace PlayFab::ClientModels;

    FLoginWithSteamRequest Request;
    Request.SteamTicket = SteamTicket;
    Request.CreateAccount = true;

    PlayFab::PlayFabClientAPI clientAPI;
    clientAPI.LoginWithSteam(Request,
        PlayFab::FLoginWithSteamDelegate::CreateLambda([this](const FLoginResult& Result)
        {
            HandleLoginResult(Result);
        }),
        PlayFab::FPlayFabErrorDelegate::CreateLambda([this](const PlayFab::FPlayFabError& Error)
        {
            HandleLoginError(Error);
        })
    );
}

void UPlayFabAuthManager::HandleLoginResult(const PlayFab::ClientModels::FLoginResult& Result)
{
    CurrentUser.PlayFabId = Result.PlayFabId;
    CurrentUser.SessionTicket = Result.SessionTicket;

    // 获取实体令牌
    if (Result.EntityToken.IsValid())
    {
        CurrentUser.EntityToken = Result.EntityToken->EntityToken;
    }

    // 获取玩家信息
    if (Result.InfoResultPayload.IsValid())
    {
        if (Result.InfoResultPayload->PlayerProfile.IsValid())
        {
            CurrentUser.DisplayName = Result.InfoResultPayload->PlayerProfile->DisplayName;
            CurrentUser.AvatarUrl = Result.InfoResultPayload->PlayerProfile->AvatarUrl;
        }
    }

    bIsLoggedIn = true;
    UE_LOG(LogTemp, Log, TEXT("PlayFab login successful: %s"), *CurrentUser.PlayFabId);

    OnLoginComplete.Broadcast(true, CurrentUser);
}

void UPlayFabAuthManager::HandleLoginError(const PlayFab::FPlayFabError& Error)
{
    UE_LOG(LogTemp, Error, TEXT("PlayFab login failed: %s"), *Error.ErrorMessage);

    FPlayFabUserInfo EmptyInfo;
    OnLoginComplete.Broadcast(false, EmptyInfo);
}

void UPlayFabAuthManager::Logout()
{
    CurrentUser = FPlayFabUserInfo();
    bIsLoggedIn = false;
    OnLogoutComplete.Broadcast(true);
}

bool UPlayFabAuthManager::IsLoggedIn() const
{
    return bIsLoggedIn;
}
```

---

## 22.3 玩家数据管理

### 22.3.1 用户数据存储

```csharp
// PlayFabDataManager.h
#pragma once

#include "CoreMinimal.h"
#include "PlayFabDataModels.h"
#include "PlayFabClientAPI.h"
#include "PlayFabServerAPI.h"

#include "PlayFabDataManager.generated.h"

USTRUCT(BlueprintType)
struct FPlayerData
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadWrite)
    TMap<FString, FString> TitleData;

    UPROPERTY(BlueprintReadWrite)
    TMap<FString, FString> UserData;

    UPROPERTY(BlueprintReadWrite)
    TArray<FString> Inventory;

    UPROPERTY(BlueprintReadWrite)
    int32 VirtualCurrency = 0;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnDataLoaded, bool, bSuccess, FPlayerData, Data);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnDataSaved, bool, bSuccess);

UCLASS()
class PLAYFABTUTORIAL_API UPlayFabDataManager : public UObject
{
    GENERATED_BODY()

public:
    // 获取玩家数据
    UFUNCTION(BlueprintCallable, Category = "PlayFab|Data")
    void LoadPlayerData();

    // 保存玩家数据
    UFUNCTION(BlueprintCallable, Category = "PlayFab|Data")
    void SavePlayerData(const TMap<FString, FString>& DataToSave);

    // 获取特定数据键
    UFUNCTION(BlueprintCallable, Category = "PlayFab|Data")
    void GetUserData(const TArray<FString>& Keys);

    // 设置单个数据键
    UFUNCTION(BlueprintCallable, Category = "PlayFab|Data")
    void SetUserData(const FString& Key, const FString& Value);

    // 获取标题数据（全局配置）
    UFUNCTION(BlueprintCallable, Category = "PlayFab|Data")
    void GetTitleData(const TArray<FString>& Keys);

    // 获取玩家统计
    UFUNCTION(BlueprintCallable, Category = "PlayFab|Data")
    void GetPlayerStatistics(const TArray<FString>& StatisticNames);

    // 更新玩家统计
    UFUNCTION(BlueprintCallable, Category = "PlayFab|Data")
    void UpdatePlayerStatistics(const TMap<FString, int32>& Statistics);

    UPROPERTY(BlueprintAssignable, Category = "PlayFab|Data")
    FOnDataLoaded OnDataLoaded;

    UPROPERTY(BlueprintAssignable, Category = "PlayFab|Data")
    FOnDataSaved OnDataSaved;

private:
    FPlayerData CachedData;
};
```

```csharp
// PlayFabDataManager.cpp
#include "PlayFabDataManager.h"

void UPlayFabDataManager::LoadPlayerData()
{
    using namespace PlayFab::ClientModels;

    FGetPlayerCombinedInfoRequestParams Params;
    Params.GetUserAccountInfo = true;
    Params.GetPlayerStatistics = true;
    Params.GetUserData = true;
    Params.GetTitleData = true;
    Params.GetUserInventory = true;
    Params.GetUserVirtualCurrency = true;

    FGetPlayerCombinedInfoRequest Request;
    Request.InfoRequestParameters = Params;

    PlayFab::PlayFabClientAPI clientAPI;
    clientAPI.GetPlayerCombinedInfo(Request,
        PlayFab::FGetPlayerCombinedInfoDelegate::CreateLambda([this](const FGetPlayerCombinedInfoResult& Result)
        {
            FPlayerData Data;

            // 解析用户数据
            if (Result.InfoResultPayload.IsValid())
            {
                auto& Payload = Result.InfoResultPayload;

                // 用户数据
                for (const auto& Pair : Payload->UserData)
                {
                    Data.UserData.Add(Pair.Key, Pair.Value.Value);
                }

                // 标题数据
                for (const auto& Pair : Payload->TitleData)
                {
                    Data.TitleData.Add(Pair.Key, Pair.Value);
                }

                // 库存
                for (const auto& Item : Payload->UserInventory)
                {
                    Data.Inventory.Add(Item.ItemId);
                }

                // 虚拟货币
                for (const auto& Pair : Payload->UserVirtualCurrency)
                {
                    Data.VirtualCurrency = Pair.Value;
                    break; // 假设只有一种货币
                }
            }

            CachedData = Data;
            OnDataLoaded.Broadcast(true, Data);
        }),
        PlayFab::FPlayFabErrorDelegate::CreateLambda([this](const PlayFab::FPlayFabError& Error)
        {
            UE_LOG(LogTemp, Error, TEXT("Failed to load player data: %s"), *Error.ErrorMessage);
            OnDataLoaded.Broadcast(false, FPlayerData());
        })
    );
}

void UPlayFabDataManager::SavePlayerData(const TMap<FString, FString>& DataToSave)
{
    using namespace PlayFab::ClientModels;

    FUpdateUserDataRequest Request;
    for (const auto& Pair : DataToSave)
    {
        FUserDataRecord Record;
        Record.Value = Pair.Value;
        Request.Data.Add(Pair.Key, Record);
    }

    PlayFab::PlayFabClientAPI clientAPI;
    clientAPI.UpdateUserData(Request,
        PlayFab::FUpdateUserDataDelegate::CreateLambda([this](const FUpdateUserDataResult& Result)
        {
            OnDataSaved.Broadcast(true);
        }),
        PlayFab::FPlayFabErrorDelegate::CreateLambda([this](const PlayFab::FPlayFabError& Error)
        {
            UE_LOG(LogTemp, Error, TEXT("Failed to save player data: %s"), *Error.ErrorMessage);
            OnDataSaved.Broadcast(false);
        })
    );
}

void UPlayFabDataManager::SetUserData(const FString& Key, const FString& Value)
{
    TMap<FString, FString> Data;
    Data.Add(Key, Value);
    SavePlayerData(Data);
}

void UPlayFabDataManager::UpdatePlayerStatistics(const TMap<FString, int32>& Statistics)
{
    using namespace PlayFab::ClientModels;

    FUpdatePlayerStatisticsRequest Request;
    for (const auto& Pair : Statistics)
    {
        FStatisticUpdate Update;
        Update.StatisticName = Pair.Key;
        Update.Value = Pair.Value;
        Request.Statistics.Add(Update);
    }

    PlayFab::PlayFabClientAPI clientAPI;
    clientAPI.UpdatePlayerStatistics(Request,
        PlayFab::FUpdatePlayerStatisticsDelegate::CreateLambda([this](const FUpdatePlayerStatisticsResult& Result)
        {
            OnDataSaved.Broadcast(true);
        }),
        PlayFab::FPlayFabErrorDelegate::CreateLambda([this](const PlayFab::FPlayFabError& Error)
        {
            UE_LOG(LogTemp, Error, TEXT("Failed to update statistics: %s"), *Error.ErrorMessage);
            OnDataSaved.Broadcast(false);
        })
    );
}
```

---

## 22.4 CloudScript 云脚本

### 22.4.1 客户端调用CloudScript

```csharp
// PlayFabCloudScriptManager.h
#pragma once

#include "CoreMinimal.h"
#include "PlayFabClientAPI.h"
#include "PlayFabCloudScriptModels.h"

#include "PlayFabCloudScriptManager.generated.h"

USTRUCT(BlueprintType)
struct FCloudScriptResult
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    bool bSuccess = false;

    UPROPERTY(BlueprintReadOnly)
    FString ReturnValue;

    UPROPERTY(BlueprintReadOnly)
    float ExecutionTime = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    int32 RequestsCount = 0;

    UPROPERTY(BlueprintReadOnly)
    FString Error;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnCloudScriptExecuted, FCloudScriptResult, Result);

UCLASS()
class PLAYFABTUTORIAL_API UPlayFabCloudScriptManager : public UObject
{
    GENERATED_BODY()

public:
    // 执行云脚本函数
    UFUNCTION(BlueprintCallable, Category = "PlayFab|CloudScript")
    void ExecuteFunction(const FString& FunctionName, const FString& FunctionParameter);

    // 执行云脚本（带JSON参数）
    UFUNCTION(BlueprintCallable, Category = "PlayFab|CloudScript")
    void ExecuteFunctionWithJson(const FString& FunctionName, const TSharedPtr<FJsonObject>& Params);

    // 执行实体云脚本（需要Entity Token）
    UFUNCTION(BlueprintCallable, Category = "PlayFab|CloudScript")
    void ExecuteEntityFunction(const FString& FunctionName, const FString& FunctionParameter);

    UPROPERTY(BlueprintAssignable, Category = "PlayFab|CloudScript")
    FOnCloudScriptExecuted OnCloudScriptExecuted;

private:
    void HandleCloudScriptResult(const FString& ResponseJson);
};
```

```csharp
// PlayFabCloudScriptManager.cpp
#include "PlayFabCloudScriptManager.h"
#include "JsonUtilities.h"

void UPlayFabCloudScriptManager::ExecuteFunction(const FString& FunctionName, const FString& FunctionParameter)
{
    using namespace PlayFab::CloudScriptModels;

    FExecuteFunctionRequest Request;
    Request.FunctionName = FunctionName;
    Request.FunctionParameter = FunctionParameter;

    PlayFab::PlayFabCloudScriptAPI cloudScriptAPI;
    cloudScriptAPI.ExecuteFunction(Request,
        PlayFab::CloudScriptModels::FExecuteFunctionDelegate::CreateLambda([this](const FExecuteFunctionResult& Result)
        {
            FCloudScriptResult ScriptResult;
            ScriptResult.bSuccess = true;
            ScriptResult.ExecutionTime = Result.ExecutionTimeSeconds;
            ScriptResult.RequestsCount = Result.Requests;

            if (Result.ReturnValue.IsValid())
            {
                ScriptResult.ReturnValue = Result.ReturnValue->AsString();
            }

            OnCloudScriptExecuted.Broadcast(ScriptResult);
        }),
        PlayFab::FPlayFabErrorDelegate::CreateLambda([this](const PlayFab::FPlayFabError& Error)
        {
            FCloudScriptResult ScriptResult;
            ScriptResult.bSuccess = false;
            ScriptResult.Error = Error.ErrorMessage;
            OnCloudScriptExecuted.Broadcast(ScriptResult);
        })
    );
}

void UPlayFabCloudScriptManager::ExecuteEntityFunction(const FString& FunctionName, const FString& FunctionParameter)
{
    using namespace PlayFab::CloudScriptModels;

    FExecuteEntityCloudScriptRequest Request;
    Request.FunctionName = FunctionName;
    Request.FunctionParameter = FunctionParameter;

    PlayFab::PlayFabCloudScriptAPI cloudScriptAPI;
    cloudScriptAPI.ExecuteEntityCloudScript(Request,
        PlayFab::CloudScriptModels::FExecuteEntityCloudScriptDelegate::CreateLambda([this](const FExecuteEntityCloudScriptResult& Result)
        {
            FCloudScriptResult ScriptResult;
            ScriptResult.bSuccess = true;
            ScriptResult.ExecutionTime = Result.ExecutionTimeSeconds;

            if (Result.FunctionResult.IsValid())
            {
                ScriptResult.ReturnValue = Result.FunctionResult->AsString();
            }

            OnCloudScriptExecuted.Broadcast(ScriptResult);
        }),
        PlayFab::FPlayFabErrorDelegate::CreateLambda([this](const PlayFab::FPlayFabError& Error)
        {
            FCloudScriptResult ScriptResult;
            ScriptResult.bSuccess = false;
            ScriptResult.Error = Error.ErrorMessage;
            OnCloudScriptExecuted.Broadcast(ScriptResult);
        })
    );
}
```

### 22.4.2 CloudScript示例

**购买物品的云脚本：**

```javascript
// CloudScript - PurchaseItem
handlers.PurchaseItem = function (args, context) {
    var itemId = args.itemId;
    var price = args.price;
    var currency = args.currency || "GC"; // 默认金币

    // 获取玩家当前货币
    var userInventory = server.GetUserInventory({
        PlayFabId: currentPlayerId
    });

    var currentBalance = userInventory.VirtualCurrency[currency] || 0;

    // 验证余额
    if (currentBalance < price) {
        return {
            success: false,
            error: "Insufficient funds",
            balance: currentBalance
        };
    }

    // 执行购买
    var purchaseResult = server.PurchaseItem({
        CatalogVersion: "MainCatalog",
        ItemId: itemId,
        VirtualCurrency: currency,
        Price: price
    });

    // 记录交易
    server.WriteTitleEvent({
        EventName: "item_purchased",
        Body: {
            itemId: itemId,
            price: price,
            currency: currency,
            playerId: currentPlayerId
        }
    });

    return {
        success: true,
        itemId: itemId,
        newBalance: currentBalance - price
    };
};
```

**每日奖励的云脚本：**

```javascript
// CloudScript - ClaimDailyReward
handlers.ClaimDailyReward = function (args, context) {
    var now = new Date();
    var today = now.toDateString();

    // 获取上次领取时间
    var userData = server.GetUserData({
        PlayFabId: currentPlayerId,
        Keys: ["lastDailyReward"]
    });

    var lastClaim = userData.Data.lastDailyReward;

    if (lastClaim && lastClaim.Value === today) {
        return {
            success: false,
            error: "Already claimed today",
            nextClaimTime: new Date(now.setDate(now.getDate() + 1)).toISOString()
        };
    }

    // 计算连续登录天数
    var loginData = server.GetUserData({
        PlayFabId: currentPlayerId,
        Keys: ["consecutiveLoginDays"]
    });

    var consecutiveDays = 1;
    var lastLoginDate = loginData.Data.consecutiveLoginDays ?
        new Date(loginData.Data.consecutiveLoginDays.Value) : null;

    if (lastLoginDate) {
        var diffDays = Math.floor((now - lastLoginDate) / (1000 * 60 * 60 * 24));
        if (diffDays === 1) {
            consecutiveDays = parseInt(loginData.Data.consecutiveLoginDays.Value) + 1;
        }
    }

    // 计算奖励
    var baseReward = 100;
    var bonusMultiplier = Math.min(consecutiveDays, 7);
    var totalReward = baseReward * bonusMultiplier;

    // 发放奖励
    server.AddUserVirtualCurrency({
        PlayFabId: currentPlayerId,
        VirtualCurrency: "GC",
        Amount: totalReward
    });

    // 更新数据
    server.UpdateUserData({
        Data: {
            lastDailyReward: today,
            consecutiveLoginDays: consecutiveDays.toString()
        }
    });

    return {
        success: true,
        reward: totalReward,
        consecutiveDays: consecutiveDays,
        bonusMultiplier: bonusMultiplier
    };
};
```

---

## 22.5 PlayFab Matchmaking 匹配系统

### 22.5.1 匹配系统实现

```csharp
// PlayFabMatchmakingManager.h
#pragma once

#include "CoreMinimal.h"
#include "PlayFabMatchmakingModels.h"
#include "PlayFabMatchmakingAPI.h"

#include "PlayFabMatchmakingManager.generated.h"

USTRUCT(BlueprintType)
struct FMatchmakingTicket
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString TicketId;

    UPROPERTY(BlueprintReadOnly)
    FString Status;

    UPROPERTY(BlueprintReadOnly)
    FString MatchId;

    UPROPERTY(BlueprintReadOnly)
    FString ConnectionString;
};

USTRUCT(BlueprintType)
struct FMatchPlayer
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadWrite)
    FString PlayerId;

    UPROPERTY(BlueprintReadWrite)
    FString TeamId;

    UPROPERTY(BlueprintReadWrite)
    TMap<FString, float> Attributes;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnTicketCreated, bool, bSuccess, FString, TicketId);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnMatchFound, bool, bSuccess, FMatchmakingTicket, Ticket);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnMatchCancelled, bool, bSuccess);

UCLASS()
class PLAYFABTUTORIAL_API UPlayFabMatchmakingManager : public UObject
{
    GENERATED_BODY()

public:
    // 创建匹配票据
    UFUNCTION(BlueprintCallable, Category = "PlayFab|Matchmaking")
    void CreateTicket(const FString& QueueName, const TArray<FMatchPlayer>& Players, int32 GiveUpAfterSeconds = 60);

    // 获取票据状态
    UFUNCTION(BlueprintCallable, Category = "PlayFab|Matchmaking")
    void GetTicket(const FString& TicketId);

    // 取消匹配
    UFUNCTION(BlueprintCallable, Category = "PlayFab|Matchmaking")
    void CancelTicket(const FString& TicketId);

    // 开始轮询票据状态
    UFUNCTION(BlueprintCallable, Category = "PlayFab|Matchmaking")
    void StartTicketPolling(const FString& TicketId, float PollInterval = 5.0f);

    // 停止轮询
    UFUNCTION(BlueprintCallable, Category = "PlayFab|Matchmaking")
    void StopTicketPolling();

    // 加入匹配
    UFUNCTION(BlueprintCallable, Category = "PlayFab|Matchmaking")
    void JoinMatch(const FString& MatchId, const FString& ConnectionString);

    UPROPERTY(BlueprintAssignable, Category = "PlayFab|Matchmaking")
    FOnTicketCreated OnTicketCreated;

    UPROPERTY(BlueprintAssignable, Category = "PlayFab|Matchmaking")
    FOnMatchFound OnMatchFound;

    UPROPERTY(BlueprintAssignable, Category = "PlayFab|Matchmaking")
    FOnMatchCancelled OnMatchCancelled;

private:
    FTimerHandle PollingTimerHandle;
    FString CurrentTicketId;
    FString CurrentQueueName;

    void PollTicketStatus();
};
```

```csharp
// PlayFabMatchmakingManager.cpp
#include "PlayFabMatchmakingManager.h"
#include "TimerManager.h"

void UPlayFabMatchmakingManager::CreateTicket(const FString& QueueName, const TArray<FMatchPlayer>& Players, int32 GiveUpAfterSeconds)
{
    using namespace PlayFab::MatchmakingModels;

    FCreateMatchmakingTicketRequest Request;
    Request.QueueName = QueueName;
    Request.GiveUpAfterSeconds = GiveUpAfterSeconds;

    // 设置玩家属性
    for (const auto& Player : Players)
    {
        FMatchmakingPlayer PlayerInfo;
        PlayerInfo.Entity = PlayFab::PlayFabSettings::staticPlayerEntity;
        PlayerInfo.Entity.Id = Player.PlayerId;

        // 设置属性
        for (const auto& Attr : Player.Attributes)
        {
            FMatchmakingPlayerAttributes Attributes;
            Attributes.DataObject = Attr.Value;
            PlayerInfo.Attributes = Attributes;
        }

        Request.Members.Add(PlayerInfo);
    }

    CurrentQueueName = QueueName;

    PlayFab::PlayFabMatchmakingAPI matchmakingAPI;
    matchmakingAPI.CreateMatchmakingTicket(Request,
        FCreateMatchmakingTicketDelegate::CreateLambda([this](const FCreateMatchmakingTicketResult& Result)
        {
            CurrentTicketId = Result.TicketId;
            OnTicketCreated.Broadcast(true, Result.TicketId);
        }),
        PlayFab::FPlayFabErrorDelegate::CreateLambda([this](const PlayFab::FPlayFabError& Error)
        {
            UE_LOG(LogTemp, Error, TEXT("Failed to create matchmaking ticket: %s"), *Error.ErrorMessage);
            OnTicketCreated.Broadcast(false, TEXT(""));
        })
    );
}

void UPlayFabMatchmakingManager::GetTicket(const FString& TicketId)
{
    using namespace PlayFab::MatchmakingModels;

    FGetMatchmakingTicketRequest Request;
    Request.TicketId = TicketId;
    Request.QueueName = CurrentQueueName;

    PlayFab::PlayFabMatchmakingAPI matchmakingAPI;
    matchmakingAPI.GetMatchmakingTicket(Request,
        FGetMatchmakingTicketDelegate::CreateLambda([this](const FGetMatchmakingTicketResult& Result)
        {
            FMatchmakingTicket Ticket;
            Ticket.TicketId = Result.TicketId;
            Ticket.Status = Result.Status;

            if (Result.Status == TEXT("Matched"))
            {
                Ticket.MatchId = Result.MatchId;

                // 获取匹配详情
                GetMatch(Result.MatchId);
            }

            OnMatchFound.Broadcast(Result.Status == TEXT("Matched"), Ticket);
        }),
        PlayFab::FPlayFabErrorDelegate::CreateLambda([this](const PlayFab::FPlayFabError& Error)
        {
            UE_LOG(LogTemp, Error, TEXT("Failed to get ticket: %s"), *Error.ErrorMessage);
            OnMatchFound.Broadcast(false, FMatchmakingTicket());
        })
    );
}

void UPlayFabMatchmakingManager::CancelTicket(const FString& TicketId)
{
    using namespace PlayFab::MatchmakingModels;

    FCancelMatchmakingTicketRequest Request;
    Request.TicketId = TicketId;
    Request.QueueName = CurrentQueueName;

    PlayFab::PlayFabMatchmakingAPI matchmakingAPI;
    matchmakingAPI.CancelMatchmakingTicket(Request,
        FCancelMatchmakingTicketDelegate::CreateLambda([this](const FCancelMatchmakingTicketResult& Result)
        {
            StopTicketPolling();
            OnMatchCancelled.Broadcast(true);
        }),
        PlayFab::FPlayFabErrorDelegate::CreateLambda([this](const PlayFab::FPlayFabError& Error)
        {
            OnMatchCancelled.Broadcast(false);
        })
    );
}

void UPlayFabMatchmakingManager::StartTicketPolling(const FString& TicketId, float PollInterval)
{
    CurrentTicketId = TicketId;

    // 设置定时轮询
    GetWorld()->GetTimerManager().SetTimer(
        PollingTimerHandle,
        this,
        &UPlayFabMatchmakingManager::PollTicketStatus,
        PollInterval,
        true
    );
}

void UPlayFabMatchmakingManager::StopTicketPolling()
{
    GetWorld()->GetTimerManager().ClearTimer(PollingTimerHandle);
}

void UPlayFabMatchmakingManager::PollTicketStatus()
{
    if (!CurrentTicketId.IsEmpty())
    {
        GetTicket(CurrentTicketId);
    }
}

void UPlayFabMatchmakingManager::GetMatch(const FString& MatchId)
{
    using namespace PlayFab::MatchmakingModels;

    FGetMatchRequest Request;
    Request.MatchId = MatchId;
    Request.QueueName = CurrentQueueName;
    Request.ReturnMemberAttributes = true;

    PlayFab::PlayFabMatchmakingAPI matchmakingAPI;
    matchmakingAPI.GetMatch(Request,
        FGetMatchDelegate::CreateLambda([this](const FGetMatchResult& Result)
        {
            // 获取连接信息
            FString ConnectionString = Result.ServerDetails.ConnectionString;

            FMatchmakingTicket Ticket;
            Ticket.MatchId = Result.MatchId;
            Ticket.ConnectionString = ConnectionString;
            Ticket.Status = TEXT("Matched");

            OnMatchFound.Broadcast(true, Ticket);
        }),
        PlayFab::FPlayFabErrorDelegate::CreateLambda([this](const PlayFab::FPlayFabError& Error)
        {
            UE_LOG(LogTemp, Error, TEXT("Failed to get match: %s"), *Error.ErrorMessage);
        })
    );
}
```

---

## 22.6 PlayFab数据分析

### 22.6.1 事件追踪系统

```csharp
// PlayFabAnalyticsManager.h
#pragma once

#include "CoreMinimal.h"
#include "PlayFabEventsModels.h"
#include "PlayFabEventsAPI.h"

#include "PlayFabAnalyticsManager.generated.h"

USTRUCT(BlueprintType)
struct FGameEvent
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadWrite)
    FString EventName;

    UPROPERTY(BlueprintReadWrite)
    TMap<FString, FString> Properties;

    UPROPERTY(BlueprintReadWrite)
    double Timestamp = 0.0;
};

UCLASS()
class PLAYFABTUTORIAL_API UPlayFabAnalyticsManager : public UObject
{
    GENERATED_BODY()

public:
    // 记录事件
    UFUNCTION(BlueprintCallable, Category = "PlayFab|Analytics")
    void WriteEvent(const FString& EventName, const TMap<FString, FString>& Properties);

    // 批量记录事件
    UFUNCTION(BlueprintCallable, Category = "PlayFab|Analytics")
    void WriteEvents(const TArray<FGameEvent>& Events);

    // 常用事件便捷方法
    UFUNCTION(BlueprintCallable, Category = "PlayFab|Analytics")
    void TrackLevelStart(const FString& LevelName);

    UFUNCTION(BlueprintCallable, Category = "PlayFab|Analytics")
    void TrackLevelComplete(const FString& LevelName, float Duration, int32 Score);

    UFUNCTION(BlueprintCallable, Category = "PlayFab|Analytics")
    void TrackLevelFail(const FString& LevelName, float Duration, const FString& Reason);

    UFUNCTION(BlueprintCallable, Category = "PlayFab|Analytics")
    void TrackPurchase(const FString& ItemId, int32 Price, const FString& Currency);

    UFUNCTION(BlueprintCallable, Category = "PlayFab|Analytics")
    void TrackTutorialComplete(const FString& TutorialId);

private:
    FString GetTimestamp();
};
```

```csharp
// PlayFabAnalyticsManager.cpp
#include "PlayFabAnalyticsManager.h"
#include "DateTime.h"

void UPlayFabAnalyticsManager::WriteEvent(const FString& EventName, const TMap<FString, FString>& Properties)
{
    using namespace PlayFab::EventsModels;

    FWriteEventsRequest Request;

    FEventContents Event;
    Event.Name = EventName;
    Event.EventNamespace = "com.mycompany.mygame";

    // 设置事件属性
    TSharedPtr<FJsonObject> BodyJson = MakeShareable(new FJsonObject);
    for (const auto& Pair : Properties)
    {
        BodyJson->SetStringField(Pair.Key, Pair.Value);
    }

    Event.Payload = BodyJson;
    Request.Events.Add(Event);

    PlayFab::PlayFabEventsAPI eventsAPI;
    eventsAPI.WriteEvents(Request,
        FWriteEventsDelegate::CreateLambda([](const FWriteEventsResponse& Result)
        {
            UE_LOG(LogTemp, Verbose, TEXT("Event written successfully"));
        }),
        PlayFab::FPlayFabErrorDelegate::CreateLambda([](const PlayFab::FPlayFabError& Error)
        {
            UE_LOG(LogTemp, Error, TEXT("Failed to write event: %s"), *Error.ErrorMessage);
        })
    );
}

void UPlayFabAnalyticsManager::TrackLevelStart(const FString& LevelName)
{
    TMap<FString, FString> Properties;
    Properties.Add("level_name", LevelName);
    Properties.Add("timestamp", GetTimestamp());
    WriteEvent("level_start", Properties);
}

void UPlayFabAnalyticsManager::TrackLevelComplete(const FString& LevelName, float Duration, int32 Score)
{
    TMap<FString, FString> Properties;
    Properties.Add("level_name", LevelName);
    Properties.Add("duration_seconds", FString::SanitizeFloat(Duration));
    Properties.Add("score", FString::FromInt(Score));
    Properties.Add("timestamp", GetTimestamp());
    WriteEvent("level_complete", Properties);
}

void UPlayFabAnalyticsManager::TrackLevelFail(const FString& LevelName, float Duration, const FString& Reason)
{
    TMap<FString, FString> Properties;
    Properties.Add("level_name", LevelName);
    Properties.Add("duration_seconds", FString::SanitizeFloat(Duration));
    Properties.Add("failure_reason", Reason);
    Properties.Add("timestamp", GetTimestamp());
    WriteEvent("level_fail", Properties);
}

void UPlayFabAnalyticsManager::TrackPurchase(const FString& ItemId, int32 Price, const FString& Currency)
{
    TMap<FString, FString> Properties;
    Properties.Add("item_id", ItemId);
    Properties.Add("price", FString::FromInt(Price));
    Properties.Add("currency", Currency);
    Properties.Add("timestamp", GetTimestamp());
    WriteEvent("item_purchase", Properties);
}

FString UPlayFabAnalyticsManager::GetTimestamp()
{
    return FDateTime::UtcNow().ToIso8601();
}
```

---

## 22.7 实践任务

### 任务1：实现完整的登录流程

创建一个包含以下功能的登录系统：
- 匿名登录（首次启动自动创建账号）
- 邮箱注册和登录
- 显示玩家信息

### 任务2：实现物品商店系统

使用CloudScript实现：
- 购买物品验证
- 库存更新
- 货币扣除

### 任务3：实现匹配系统

创建一个简单的1v1匹配流程：
- 创建匹配票据
- 轮询状态
- 连接到匹配的服务器

---

## 22.8 总结

本课学习了：
- PlayFab SDK集成与配置
- 多种用户认证方式
- 玩家数据存储与管理
- CloudScript云脚本使用
- PlayFab Matchmaking匹配系统
- 数据分析与事件追踪

**下一课预告**：容器化与编排 - Docker容器化游戏服务器和Kubernetes集群部署。

---

*课程版本：1.0*
*最后更新：2026-04-10*
