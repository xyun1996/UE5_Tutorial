# 第32课：用户系统实现

> 学习目标：实现完整的用户系统，包括注册、登录、认证和用户数据管理功能。

---

## 32.1 用户系统架构

### 32.1.1 系统组成

```
用户系统组件:
├── 认证模块 (Authentication)
│   ├── 账号密码登录
│   ├── 第三方登录
│   ├── 设备登录
│   └── Token管理
├── 用户数据模块 (User Data)
│   ├── 基础信息
│   ├── 角色数据
│   ├── 统计数据
│   └── 设置数据
├── 会话管理模块 (Session)
│   ├── 会话创建
│   ├── 会话验证
│   └── 会话过期
└── 安全模块 (Security)
    ├── 密码加密
    ├── 防暴力破解
    └── 设备绑定
```

---

## 32.2 认证子系统

### 32.2.1 认证子系统实现

```csharp
// AuthSubsystem.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "HttpModule.h"

#include "AuthSubsystem.generated.h"

UENUM(BlueprintType)
enum class EAuthType : uint8
{
    EmailPassword,
    DeviceId,
    Steam,
    Epic,
    Google,
    Apple
};

USTRUCT(BlueprintType)
struct FAuthCredentials
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString Email;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString Password;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString DeviceId;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString PlatformToken;  // 第三方平台Token

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    EAuthType AuthType = EAuthType::EmailPassword;
};

USTRUCT(BlueprintType)
struct FAuthResult
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    bool bSuccess = false;

    UPROPERTY(BlueprintReadOnly)
    FString AccessToken;

    UPROPERTY(BlueprintReadOnly)
    FString RefreshToken;

    UPROPERTY(BlueprintReadOnly)
    int32 ExpiresIn = 0;

    UPROPERTY(BlueprintReadOnly)
    FString ErrorMessage;

    UPROPERTY(BlueprintReadOnly)
    FString UserId;
};

USTRUCT(BlueprintType)
struct FUserInfo
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString UserId;

    UPROPERTY(BlueprintReadOnly)
    FString Email;

    UPROPERTY(BlueprintReadOnly)
    FString DisplayName;

    UPROPERTY(BlueprintReadOnly)
    FString AvatarUrl;

    UPROPERTY(BlueprintReadOnly)
    int32 Level = 1;

    UPROPERTY(BlueprintReadOnly)
    int32 Experience = 0;

    UPROPERTY(BlueprintReadOnly)
    FString Region;

    UPROPERTY(BlueprintReadOnly)
    FDateTime CreatedAt;

    UPROPERTY(BlueprintReadOnly)
    FDateTime LastLoginAt;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnAuthComplete, const FAuthResult&, Result);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnUserLoaded, const FUserInfo&, UserInfo);
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnLogoutComplete);

UCLASS()
class ARENAAUTH_API UAuthSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 登录方法
    UFUNCTION(BlueprintCallable, Category = "Auth")
    void LoginWithEmailAndPassword(const FString& Email, const FString& Password);

    UFUNCTION(BlueprintCallable, Category = "Auth")
    void LoginWithDeviceId();

    UFUNCTION(BlueprintCallable, Category = "Auth")
    void LoginWithSteam(const FString& SteamToken);

    UFUNCTION(BlueprintCallable, Category = "Auth")
    void LoginWithPlatform(EAuthType Platform, const FString& Token);

    // 注册
    UFUNCTION(BlueprintCallable, Category = "Auth")
    void Register(const FString& Email, const FString& Password, const FString& DisplayName);

    // 登出
    UFUNCTION(BlueprintCallable, Category = "Auth")
    void Logout();

    // Token管理
    UFUNCTION(BlueprintCallable, Category = "Auth")
    void RefreshAccessToken();

    UFUNCTION(BlueprintPure, Category = "Auth")
    bool HasValidToken() const;

    UFUNCTION(BlueprintPure, Category = "Auth")
    FString GetAccessToken() const { return CurrentAccessToken; }

    // 用户信息
    UFUNCTION(BlueprintCallable, Category = "Auth")
    void LoadUserInfo();

    UFUNCTION(BlueprintPure, Category = "Auth")
    FUserInfo GetUserInfo() const { return CurrentUser; }

    UFUNCTION(BlueprintPure, Category = "Auth")
    bool IsLoggedIn() const { return bIsLoggedIn; }

    UFUNCTION(BlueprintPure, Category = "Auth")
    FString GetCurrentUserId() const { return CurrentUser.UserId; }

    // 事件
    UPROPERTY(BlueprintAssignable, Category = "Auth")
    FOnAuthComplete OnLoginComplete;

    UPROPERTY(BlueprintAssignable, Category = "Auth")
    FOnAuthComplete OnRegisterComplete;

    UPROPERTY(BlueprintAssignable, Category = "Auth")
    FOnUserLoaded OnUserLoaded;

    UPROPERTY(BlueprintAssignable, Category = "Auth")
    FOnLogoutComplete OnLogoutComplete;

    // 配置
    UPROPERTY(EditDefaultsOnly, Category = "Auth")
    FString AuthApiEndpoint = "/auth";

private:
    FString ApiBaseUrl;
    FString CurrentAccessToken;
    FString CurrentRefreshToken;
    FDateTime TokenExpirationTime;
    FUserInfo CurrentUser;
    bool bIsLoggedIn = false;

    // Token刷新定时器
    FTimerHandle TokenRefreshTimer;

    // HTTP请求
    void MakeAuthRequest(const FString& Endpoint, const FString& Method, const FString& Body);
    void HandleLoginResponse(const FString& ResponseJson);
    void HandleRegisterResponse(const FString& ResponseJson);
    void HandleUserInfoResponse(const FString& ResponseJson);
    void HandleAuthError(const FString& Error);

    // 持久化
    void SaveTokens();
    bool LoadSavedTokens();
    void ClearSavedTokens();

    // Token管理
    void ScheduleTokenRefresh();
    void CheckTokenExpiration();
    FString GenerateDeviceId();
};
```

```csharp
// AuthSubsystem.cpp
#include "AuthSubsystem.h"
#include "JsonUtilities.h"
#include "Misc/FileHelper.h"
#include "Misc/Paths.h"
#include "HAL/PlatformProcess.h"
#include "TimerManager.h"

void UAuthSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    // 从GameInstance获取API基础URL
    ApiBaseUrl = "http://localhost:8080/api";

    // 尝试加载保存的Token
    if (LoadSavedTokens())
    {
        // 验证Token是否仍然有效
        if (HasValidToken())
        {
            bIsLoggedIn = true;
            LoadUserInfo();
            ScheduleTokenRefresh();
        }
    }
}

void UAuthSubsystem::Deinitialize()
{
    if (UWorld* World = GetWorld())
    {
        World->GetTimerManager().ClearTimer(TokenRefreshTimer);
    }

    Super::Deinitialize();
}

void UAuthSubsystem::LoginWithEmailAndPassword(const FString& Email, const FString& Password)
{
    TSharedPtr<FJsonObject> JsonRequest = MakeShareable(new FJsonObject);
    JsonRequest->SetStringField("email", Email);
    JsonRequest->SetStringField("password", Password);

    FString Body;
    TJsonWriterFactory<>::Create(&Body)->WriteObject(JsonRequest.ToSharedRef());

    MakeAuthRequest("/login", "POST", Body);
}

void UAuthSubsystem::LoginWithDeviceId()
{
    FString DeviceId = GenerateDeviceId();

    TSharedPtr<FJsonObject> JsonRequest = MakeShareable(new FJsonObject);
    JsonRequest->SetStringField("deviceId", DeviceId);
    JsonRequest->SetStringField("platform", UGameplayStatics::GetPlatformName());

    FString Body;
    TJsonWriterFactory<>::Create(&Body)->WriteObject(JsonRequest.ToSharedRef());

    MakeAuthRequest("/login/device", "POST", Body);
}

void UAuthSubsystem::LoginWithSteam(const FString& SteamToken)
{
    LoginWithPlatform(EAuthType::Steam, SteamToken);
}

void UAuthSubsystem::LoginWithPlatform(EAuthType Platform, const FString& Token)
{
    FString PlatformStr;
    switch (Platform)
    {
        case EAuthType::Steam: PlatformStr = "steam"; break;
        case EAuthType::Epic: PlatformStr = "epic"; break;
        case EAuthType::Google: PlatformStr = "google"; break;
        case EAuthType::Apple: PlatformStr = "apple"; break;
        default: return;
    }

    TSharedPtr<FJsonObject> JsonRequest = MakeShareable(new FJsonObject);
    JsonRequest->SetStringField("platform", PlatformStr);
    JsonRequest->SetStringField("token", Token);

    FString Body;
    TJsonWriterFactory<>::Create(&Body)->WriteObject(JsonRequest.ToSharedRef());

    MakeAuthRequest("/login/platform", "POST", Body);
}

void UAuthSubsystem::Register(const FString& Email, const FString& Password, const FString& DisplayName)
{
    TSharedPtr<FJsonObject> JsonRequest = MakeShareable(new FJsonObject);
    JsonRequest->SetStringField("email", Email);
    JsonRequest->SetStringField("password", Password);
    JsonRequest->SetStringField("displayName", DisplayName);

    FString Body;
    TJsonWriterFactory<>::Create(&Body)->WriteObject(JsonRequest.ToSharedRef());

    TSharedRef<IHttpRequest> Request = FHttpModule::Get().CreateRequest();
    Request->SetURL(ApiBaseUrl + AuthApiEndpoint + "/register");
    Request->SetVerb("POST");
    Request->SetHeader("Content-Type", "application/json");
    Request->SetContentAsString(Body);

    Request->OnProcessRequestComplete().BindLambda([this](FHttpRequestPtr Req, FHttpResponsePtr Resp, bool bSuccess)
    {
        FAuthResult Result;

        if (bSuccess && Resp.IsValid() && Resp->GetResponseCode() == 201)
        {
            HandleRegisterResponse(Resp->GetContentAsString());
            Result.bSuccess = true;
        }
        else
        {
            Result.bSuccess = false;
            Result.ErrorMessage = "Registration failed";
        }

        OnRegisterComplete.Broadcast(Result);
    });

    Request->ProcessRequest();
}

void UAuthSubsystem::Logout()
{
    // 调用后端登出
    TSharedRef<IHttpRequest> Request = FHttpModule::Get().CreateRequest();
    Request->SetURL(ApiBaseUrl + AuthApiEndpoint + "/logout");
    Request->SetVerb("POST");
    Request->SetHeader("Authorization", "Bearer " + CurrentAccessToken);

    Request->OnProcessRequestComplete().BindLambda([this](FHttpRequestPtr Req, FHttpResponsePtr Resp, bool bSuccess)
    {
        // 无论成功与否都清除本地状态
        CurrentAccessToken.Empty();
        CurrentRefreshToken.Empty();
        CurrentUser = FUserInfo();
        bIsLoggedIn = false;

        ClearSavedTokens();

        if (UWorld* World = GetWorld())
        {
            World->GetTimerManager().ClearTimer(TokenRefreshTimer);
        }

        OnLogoutComplete.Broadcast();
    });

    Request->ProcessRequest();
}

void UAuthSubsystem::RefreshAccessToken()
{
    if (CurrentRefreshToken.IsEmpty())
    {
        return;
    }

    TSharedPtr<FJsonObject> JsonRequest = MakeShareable(new FJsonObject);
    JsonRequest->SetStringField("refreshToken", CurrentRefreshToken);

    FString Body;
    TJsonWriterFactory<>::Create(&Body)->WriteObject(JsonRequest.ToSharedRef());

    TSharedRef<IHttpRequest> Request = FHttpModule::Get().CreateRequest();
    Request->SetURL(ApiBaseUrl + AuthApiEndpoint + "/refresh");
    Request->SetVerb("POST");
    Request->SetHeader("Content-Type", "application/json");
    Request->SetContentAsString(Body);

    Request->OnProcessRequestComplete().BindLambda([this](FHttpRequestPtr Req, FHttpResponsePtr Resp, bool bSuccess)
    {
        if (bSuccess && Resp.IsValid() && Resp->GetResponseCode() == 200)
        {
            HandleLoginResponse(Resp->GetContentAsString());
        }
        else
        {
            // Token刷新失败，需要重新登录
            bIsLoggedIn = false;
            OnLogoutComplete.Broadcast();
        }
    });

    Request->ProcessRequest();
}

bool UAuthSubsystem::HasValidToken() const
{
    if (CurrentAccessToken.IsEmpty())
    {
        return false;
    }

    return FDateTime::UtcNow() < TokenExpirationTime;
}

void UAuthSubsystem::LoadUserInfo()
{
    if (!HasValidToken())
    {
        return;
    }

    TSharedRef<IHttpRequest> Request = FHttpModule::Get().CreateRequest();
    Request->SetURL(ApiBaseUrl + "/user/me");
    Request->SetVerb("GET");
    Request->SetHeader("Authorization", "Bearer " + CurrentAccessToken);

    Request->OnProcessRequestComplete().BindLambda([this](FHttpRequestPtr Req, FHttpResponsePtr Resp, bool bSuccess)
    {
        if (bSuccess && Resp.IsValid() && Resp->GetResponseCode() == 200)
        {
            HandleUserInfoResponse(Resp->GetContentAsString());
        }
    });

    Request->ProcessRequest();
}

void UAuthSubsystem::MakeAuthRequest(const FString& Endpoint, const FString& Method, const FString& Body)
{
    TSharedRef<IHttpRequest> Request = FHttpModule::Get().CreateRequest();
    Request->SetURL(ApiBaseUrl + AuthApiEndpoint + Endpoint);
    Request->SetVerb(Method);
    Request->SetHeader("Content-Type", "application/json");

    if (!Body.IsEmpty())
    {
        Request->SetContentAsString(Body);
    }

    Request->OnProcessRequestComplete().BindLambda([this](FHttpRequestPtr Req, FHttpResponsePtr Resp, bool bSuccess)
    {
        FAuthResult Result;

        if (bSuccess && Resp.IsValid())
        {
            int32 StatusCode = Resp->GetResponseCode();

            if (StatusCode == 200 || StatusCode == 201)
            {
                HandleLoginResponse(Resp->GetContentAsString());
                Result.bSuccess = true;
            }
            else
            {
                Result.bSuccess = false;
                HandleAuthError(Resp->GetContentAsString());
                Result.ErrorMessage = "Authentication failed";
            }
        }
        else
        {
            Result.bSuccess = false;
            Result.ErrorMessage = "Network error";
        }

        OnLoginComplete.Broadcast(Result);
    });

    Request->ProcessRequest();
}

void UAuthSubsystem::HandleLoginResponse(const FString& ResponseJson)
{
    TSharedPtr<FJsonObject> JsonObject;
    TJsonReaderFactory<TCHAR>::Create(ResponseJson) >> JsonObject;

    if (!JsonObject.IsValid())
    {
        return;
    }

    // 解析Token
    CurrentAccessToken = JsonObject->GetStringField("accessToken");
    CurrentRefreshToken = JsonObject->GetStringField("refreshToken");
    int32 ExpiresIn = JsonObject->GetNumberField("expiresIn");

    // 计算过期时间
    TokenExpirationTime = FDateTime::UtcNow() + FTimespan::FromSeconds(ExpiresIn);

    bIsLoggedIn = true;

    // 保存Token
    SaveTokens();

    // 调度Token刷新
    ScheduleTokenRefresh();

    // 加载用户信息
    LoadUserInfo();
}

void UAuthSubsystem::HandleUserInfoResponse(const FString& ResponseJson)
{
    TSharedPtr<FJsonObject> JsonObject;
    TJsonReaderFactory<TCHAR>::Create(ResponseJson) >> JsonObject;

    if (!JsonObject.IsValid())
    {
        return;
    }

    CurrentUser.UserId = JsonObject->GetStringField("userId");
    CurrentUser.Email = JsonObject->GetStringField("email");
    CurrentUser.DisplayName = JsonObject->GetStringField("displayName");

    if (JsonObject->HasField("avatarUrl"))
    {
        CurrentUser.AvatarUrl = JsonObject->GetStringField("avatarUrl");
    }

    if (JsonObject->HasField("level"))
    {
        CurrentUser.Level = JsonObject->GetNumberField("level");
    }

    if (JsonObject->HasField("experience"))
    {
        CurrentUser.Experience = JsonObject->GetNumberField("experience");
    }

    if (JsonObject->HasField("region"))
    {
        CurrentUser.Region = JsonObject->GetStringField("region");
    }

    OnUserLoaded.Broadcast(CurrentUser);
}

void UAuthSubsystem::HandleAuthError(const FString& ErrorJson)
{
    UE_LOG(LogTemp, Error, TEXT("Auth error: %s"), *ErrorJson);
}

void UAuthSubsystem::SaveTokens()
{
    FString TokenFilePath = FPaths::Combine(FPaths::ProjectSavedDir(), TEXT("auth_tokens.json"));

    TSharedPtr<FJsonObject> JsonObject = MakeShareable(new FJsonObject);
    JsonObject->SetStringField("accessToken", CurrentAccessToken);
    JsonObject->SetStringField("refreshToken", CurrentRefreshToken);
    JsonObject->SetNumberField("expiresAt", TokenExpirationTime.ToUnixTimestamp());

    FString JsonString;
    TJsonWriterFactory<>::Create(&JsonString)->WriteObject(JsonObject.ToSharedRef());

    FFileHelper::SaveStringToFile(JsonString, *TokenFilePath);
}

bool UAuthSubsystem::LoadSavedTokens()
{
    FString TokenFilePath = FPaths::Combine(FPaths::ProjectSavedDir(), TEXT("auth_tokens.json"));

    if (!FPaths::FileExists(TokenFilePath))
    {
        return false;
    }

    FString JsonString;
    FFileHelper::LoadFileToString(JsonString, *TokenFilePath);

    TSharedPtr<FJsonObject> JsonObject;
    TJsonReaderFactory<TCHAR>::Create(JsonString) >> JsonObject;

    if (!JsonObject.IsValid())
    {
        return false;
    }

    CurrentAccessToken = JsonObject->GetStringField("accessToken");
    CurrentRefreshToken = JsonObject->GetStringField("refreshToken");
    int64 ExpiresAt = JsonObject->GetNumberField("expiresAt");

    TokenExpirationTime = FDateTime::FromUnixTimestamp(ExpiresAt);

    return true;
}

void UAuthSubsystem::ClearSavedTokens()
{
    FString TokenFilePath = FPaths::Combine(FPaths::ProjectSavedDir(), TEXT("auth_tokens.json"));
    IFileManager::Get().Delete(*TokenFilePath);
}

void UAuthSubsystem::ScheduleTokenRefresh()
{
    if (UWorld* World = GetWorld())
    {
        // 在过期前5分钟刷新
        FTimespan TimeUntilRefresh = TokenExpirationTime - FDateTime::UtcNow() - FTimespan::FromMinutes(5);

        if (TimeUntilRefresh.GetTotalSeconds() > 0)
        {
            World->GetTimerManager().SetTimer(
                TokenRefreshTimer,
                this,
                &UAuthSubsystem::RefreshAccessToken,
                TimeUntilRefresh.GetTotalSeconds(),
                false
            );
        }
    }
}

FString UAuthSubsystem::GenerateDeviceId()
{
    FString DeviceId = FPlatformMisc::GetDeviceId();

    if (DeviceId.IsEmpty())
    {
        // 生成并保存设备ID
        FString StoredIdPath = FPaths::Combine(FPaths::ProjectSavedDir(), TEXT("device_id.txt"));

        if (FPaths::FileExists(StoredIdPath))
        {
            FFileHelper::LoadFileToString(DeviceId, *StoredIdPath);
        }
        else
        {
            DeviceId = FGuid::NewGuid().ToString();
            FFileHelper::SaveStringToFile(DeviceId, *StoredIdPath);
        }
    }

    return DeviceId;
}
```

---

## 32.3 用户数据管理

### 32.3.1 玩家数据子系统

```csharp
// PlayerDataSubsystem.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "PlayerDataSubsystem.generated.h"

USTRUCT(BlueprintType)
struct FPlayerStats
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    int32 TotalMatches = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 Wins = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 Losses = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 Kills = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 Deaths = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 Assists = 0;

    UPROPERTY(BlueprintReadOnly)
    float WinRate = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    float KDRatio = 0.0f;
};

USTRUCT(BlueprintType)
struct FPlayerRank
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    int32 RankPoints = 0;

    UPROPERTY(BlueprintReadOnly)
    FString RankTier;  // Bronze, Silver, Gold, etc.

    UPROPERTY(BlueprintReadOnly)
    int32 RankPosition = 0;

    UPROPERTY(BlueprintReadOnly)
    FString SeasonId;
};

USTRUCT(BlueprintType)
struct FPlayerSettings
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadWrite)
    float MouseSensitivity = 1.0f;

    UPROPERTY(BlueprintReadWrite)
    float AudioMasterVolume = 1.0f;

    UPROPERTY(BlueprintReadWrite)
    float AudioMusicVolume = 0.5f;

    UPROPERTY(BlueprintReadWrite)
    float AudioSFXVolume = 0.8f;

    UPROPERTY(BlueprintReadWrite)
    FString PreferredRegion = "auto";

    UPROPERTY(BlueprintReadWrite)
    bool bShowFPS = false;

    UPROPERTY(BlueprintReadWrite)
    FString Language = "en";
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnPlayerStatsUpdated, const FPlayerStats&, Stats);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnPlayerSettingsSaved, bool, bSuccess);

UCLASS()
class ARENADATA_API UPlayerDataSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 加载玩家数据
    UFUNCTION(BlueprintCallable, Category = "PlayerData")
    void LoadAllData();

    // 统计数据
    UFUNCTION(BlueprintCallable, Category = "PlayerData")
    void LoadPlayerStats();

    UFUNCTION(BlueprintPure, Category = "PlayerData")
    FPlayerStats GetPlayerStats() const { return CurrentStats; }

    UFUNCTION(BlueprintCallable, Category = "PlayerData")
    void UpdateStatsAfterMatch(bool bWon, int32 Kills, int32 Deaths, int32 Assists);

    // 排名数据
    UFUNCTION(BlueprintCallable, Category = "PlayerData")
    void LoadPlayerRank();

    UFUNCTION(BlueprintPure, Category = "PlayerData")
    FPlayerRank GetPlayerRank() const { return CurrentRank; }

    // 设置
    UFUNCTION(BlueprintCallable, Category = "PlayerData")
    void LoadPlayerSettings();

    UFUNCTION(BlueprintCallable, Category = "PlayerData")
    void SavePlayerSettings(const FPlayerSettings& Settings);

    UFUNCTION(BlueprintPure, Category = "PlayerData")
    FPlayerSettings GetPlayerSettings() const { return CurrentSettings; }

    // 事件
    UPROPERTY(BlueprintAssignable, Category = "PlayerData")
    FOnPlayerStatsUpdated OnStatsUpdated;

    UPROPERTY(BlueprintAssignable, Category = "PlayerData")
    FOnPlayerSettingsSaved OnSettingsSaved;

private:
    FString ApiBaseUrl;
    FPlayerStats CurrentStats;
    FPlayerRank CurrentRank;
    FPlayerSettings CurrentSettings;

    void ParseStatsJson(const TSharedPtr<FJsonObject>& JsonObject);
    void ParseRankJson(const TSharedPtr<FJsonObject>& JsonObject);
    void ParseSettingsJson(const TSharedPtr<FJsonObject>& JsonObject);
    FString SettingsToJson() const;
};
```

---

## 32.4 后端认证服务

### 32.4.1 Node.js认证服务

```javascript
// auth-service/src/controllers/authController.js
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const { v4: uuidv4 } = require('uuid');
const database = require('../database');
const redis = require('../redis');

const SALT_ROUNDS = 12;
const ACCESS_TOKEN_EXPIRY = 3600; // 1小时
const REFRESH_TOKEN_EXPIRY = 604800; // 7天

class AuthController {
    // 注册
    async register(req, res) {
        try {
            const { email, password, displayName } = req.body;

            // 验证输入
            if (!email || !password || !displayName) {
                return res.status(400).json({ error: 'Missing required fields' });
            }

            // 检查邮箱是否已存在
            const existingUser = await database.query(
                'SELECT id FROM users WHERE email = $1',
                [email]
            );

            if (existingUser.rows.length > 0) {
                return res.status(409).json({ error: 'Email already registered' });
            }

            // 加密密码
            const hashedPassword = await bcrypt.hash(password, SALT_ROUNDS);

            // 创建用户
            const userId = uuidv4();
            const result = await database.query(
                `INSERT INTO users (id, email, password_hash, display_name, created_at, last_login_at)
                 VALUES ($1, $2, $3, $4, NOW(), NOW())
                 RETURNING id, email, display_name`,
                [userId, email, hashedPassword, displayName]
            );

            // 创建初始统计数据
            await database.query(
                `INSERT INTO player_stats (user_id) VALUES ($1)`,
                [userId]
            );

            // 生成Token
            const tokens = await this.generateTokens(userId);

            // 记录审计日志
            await this.logAuthEvent(userId, 'register', req.ip);

            res.status(201).json({
                ...tokens,
                userId,
                user: result.rows[0]
            });
        } catch (error) {
            console.error('Register error:', error);
            res.status(500).json({ error: 'Internal server error' });
        }
    }

    // 登录
    async login(req, res) {
        try {
            const { email, password } = req.body;

            // 查找用户
            const result = await database.query(
                'SELECT id, email, password_hash, display_name FROM users WHERE email = $1',
                [email]
            );

            if (result.rows.length === 0) {
                return res.status(401).json({ error: 'Invalid credentials' });
            }

            const user = result.rows[0];

            // 验证密码
            const validPassword = await bcrypt.compare(password, user.password_hash);

            if (!validPassword) {
                // 记录失败尝试
                await this.recordFailedAttempt(user.id, req.ip);
                return res.status(401).json({ error: 'Invalid credentials' });
            }

            // 更新最后登录时间
            await database.query(
                'UPDATE users SET last_login_at = NOW() WHERE id = $1',
                [user.id]
            );

            // 生成Token
            const tokens = await this.generateTokens(user.id);

            // 清除失败尝试记录
            await redis.del(`failed_attempts:${user.id}`);

            // 记录审计日志
            await this.logAuthEvent(user.id, 'login', req.ip);

            res.json({
                ...tokens,
                userId: user.id
            });
        } catch (error) {
            console.error('Login error:', error);
            res.status(500).json({ error: 'Internal server error' });
        }
    }

    // 设备登录
    async loginWithDevice(req, res) {
        try {
            const { deviceId, platform } = req.body;

            if (!deviceId) {
                return res.status(400).json({ error: 'Device ID required' });
            }

            // 查找或创建设备用户
            let result = await database.query(
                'SELECT user_id FROM device_auths WHERE device_id = $1',
                [deviceId]
            );

            let userId;

            if (result.rows.length > 0) {
                userId = result.rows[0].user_id;
            } else {
                // 创建新用户
                userId = uuidv4();
                const displayName = `Player_${userId.substring(0, 8)}`;

                await database.query(
                    `INSERT INTO users (id, display_name, created_at, last_login_at)
                     VALUES ($1, $2, NOW(), NOW())`,
                    [userId, displayName]
                );

                await database.query(
                    `INSERT INTO device_auths (device_id, user_id, platform, created_at)
                     VALUES ($1, $2, $3, NOW())`,
                    [deviceId, userId, platform]
                );

                await database.query(
                    `INSERT INTO player_stats (user_id) VALUES ($1)`,
                    [userId]
                );
            }

            // 更新最后登录
            await database.query(
                'UPDATE users SET last_login_at = NOW() WHERE id = $1',
                [userId]
            );

            const tokens = await this.generateTokens(userId);

            await this.logAuthEvent(userId, 'device_login', req.ip);

            res.json({
                ...tokens,
                userId
            });
        } catch (error) {
            console.error('Device login error:', error);
            res.status(500).json({ error: 'Internal server error' });
        }
    }

    // 刷新Token
    async refresh(req, res) {
        try {
            const { refreshToken } = req.body;

            if (!refreshToken) {
                return res.status(400).json({ error: 'Refresh token required' });
            }

            // 验证刷新Token
            const decoded = jwt.verify(refreshToken, process.env.JWT_REFRESH_SECRET);

            // 检查是否在黑名单中
            const isBlacklisted = await redis.get(`blacklist:${refreshToken}`);
            if (isBlacklisted) {
                return res.status(401).json({ error: 'Token revoked' });
            }

            // 生成新Token
            const tokens = await this.generateTokens(decoded.userId);

            // 将旧刷新Token加入黑名单
            await redis.setex(
                `blacklist:${refreshToken}`,
                REFRESH_TOKEN_EXPIRY,
                '1'
            );

            res.json(tokens);
        } catch (error) {
            console.error('Refresh error:', error);
            res.status(401).json({ error: 'Invalid refresh token' });
        }
    }

    // 登出
    async logout(req, res) {
        try {
            const authHeader = req.headers.authorization;
            const token = authHeader?.split(' ')[1];

            if (token) {
                // 将访问Token加入黑名单
                const decoded = jwt.decode(token);
                const expiresIn = decoded.exp - Math.floor(Date.now() / 1000);

                if (expiresIn > 0) {
                    await redis.setex(`blacklist:${token}`, expiresIn, '1');
                }
            }

            await this.logAuthEvent(req.user?.id, 'logout', req.ip);

            res.json({ success: true });
        } catch (error) {
            console.error('Logout error:', error);
            res.status(500).json({ error: 'Internal server error' });
        }
    }

    // 生成Token对
    async generateTokens(userId) {
        const accessToken = jwt.sign(
            { userId },
            process.env.JWT_SECRET,
            { expiresIn: ACCESS_TOKEN_EXPIRY }
        );

        const refreshToken = jwt.sign(
            { userId, type: 'refresh' },
            process.env.JWT_REFRESH_SECRET,
            { expiresIn: REFRESH_TOKEN_EXPIRY }
        );

        // 存储刷新Token
        await redis.setex(
            `refresh_token:${userId}`,
            REFRESH_TOKEN_EXPIRY,
            refreshToken
        );

        return {
            accessToken,
            refreshToken,
            expiresIn: ACCESS_TOKEN_EXPIRY
        };
    }

    // 记录失败尝试
    async recordFailedAttempt(userId, ip) {
        const key = `failed_attempts:${userId}`;
        const attempts = await redis.incr(key);
        await redis.expire(key, 300); // 5分钟过期

        if (attempts >= 5) {
            // 锁定账户5分钟
            await redis.setex(`locked:${userId}`, 300, '1');
            await this.logAuthEvent(userId, 'account_locked', ip);
        }
    }

    // 审计日志
    async logAuthEvent(userId, event, ip) {
        await database.query(
            `INSERT INTO auth_audit_log (user_id, event, ip_address, created_at)
             VALUES ($1, $2, $3, NOW())`,
            [userId, event, ip]
        );
    }
}

module.exports = new AuthController();
```

---

## 32.5 数据库设计

### 32.5.1 用户相关表

```sql
-- 用户表
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE,
    password_hash VARCHAR(255),
    display_name VARCHAR(64) NOT NULL,
    avatar_url VARCHAR(512),
    level INTEGER DEFAULT 1,
    experience INTEGER DEFAULT 0,
    region VARCHAR(16) DEFAULT 'auto',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_login_at TIMESTAMP WITH TIME ZONE,
    is_banned BOOLEAN DEFAULT FALSE,
    ban_reason TEXT
);

-- 设备认证表
CREATE TABLE device_auths (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    device_id VARCHAR(255) NOT NULL UNIQUE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    platform VARCHAR(32),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_used_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 玩家统计表
CREATE TABLE player_stats (
    user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    total_matches INTEGER DEFAULT 0,
    wins INTEGER DEFAULT 0,
    losses INTEGER DEFAULT 0,
    kills INTEGER DEFAULT 0,
    deaths INTEGER DEFAULT 0,
    assists INTEGER DEFAULT 0,
    highest_kill_streak INTEGER DEFAULT 0,
    total_play_time_seconds BIGINT DEFAULT 0,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 排名表
CREATE TABLE player_ranks (
    user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    season_id VARCHAR(32) NOT NULL,
    rank_points INTEGER DEFAULT 0,
    rank_tier VARCHAR(16) DEFAULT 'Unranked',
    wins INTEGER DEFAULT 0,
    losses INTEGER DEFAULT 0,
    highest_rank_tier VARCHAR(16),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(user_id, season_id)
);

-- 用户设置表
CREATE TABLE user_settings (
    user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    mouse_sensitivity FLOAT DEFAULT 1.0,
    audio_master_volume FLOAT DEFAULT 1.0,
    audio_music_volume FLOAT DEFAULT 0.5,
    audio_sfx_volume FLOAT DEFAULT 0.8,
    preferred_region VARCHAR(16) DEFAULT 'auto',
    show_fps BOOLEAN DEFAULT FALSE,
    language VARCHAR(8) DEFAULT 'en',
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 审计日志表
CREATE TABLE auth_audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID,
    event VARCHAR(32) NOT NULL,
    ip_address VARCHAR(45),
    user_agent TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 索引
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_display_name ON users(display_name);
CREATE INDEX idx_device_auths_device_id ON device_auths(device_id);
CREATE INDEX idx_player_ranks_season ON player_ranks(season_id);
CREATE INDEX idx_auth_audit_user ON auth_audit_log(user_id);
CREATE INDEX idx_auth_audit_event ON auth_audit_log(event);
```

---

## 32.6 实践任务

### 任务1：实现登录界面

创建完整的登录UI：
- 邮箱密码登录表单
- 设备登录按钮
- 注册表单
- 错误提示

### 任务2：实现Token管理

完善Token生命周期：
- 自动刷新
- 过期处理
- 安全存储

### 任务3：创建后端API

实现完整的认证API：
- 注册接口
- 登录接口
- Token刷新接口
- 登出接口

---

## 32.7 总结

本课完成了：
- 用户系统架构设计
- 认证子系统实现
- 用户数据管理
- 后端认证服务开发
- 数据库表设计

**下一课预告**：匹配系统开发 - 实现智能匹配算法和队列管理。

---

*课程版本：1.0*
*最后更新：2026-04-10*
