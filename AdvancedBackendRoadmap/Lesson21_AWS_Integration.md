# 第21课：AWS云服务集成

> 学习目标：掌握AWS SDK在UE5中的集成，实现Amazon GameLift游戏服务器托管、Lambda函数调用和API Gateway集成。

---

## 21.1 AWS SDK for C++ 集成

### 21.1.1 安装与配置

**通过vcpkg安装AWS SDK：**

```bash
# 安装vcpkg（如果尚未安装）
git clone https://github.com/Microsoft/vcpkg.git
cd vcpkg
.\bootstrap-vcpkg.bat

# 安装AWS SDK核心模块
.\vcpkg install aws-sdk-cpp[core,s3,lambda,gamelift,apigateway]:x64-windows
```

**在Build.cs中配置依赖：**

```csharp
// ProjectName.Build.cs
using UnrealBuildTool;

public class ProjectName : ModuleRules
{
    public ProjectName(ReadOnlyTargetRules Target) : base(Target)
    {
        PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;

        PublicDependencyModuleNames.AddRange(new string[] {
            "Core", "CoreUObject", "Engine", "InputCore", "Networking"
        });

        // AWS SDK依赖
        if (Target.Platform == UnrealTargetPlatform.Win64)
        {
            string AWS_SDK_PATH = "C:/vcpkg/installed/x64-windows";

            // 添加包含路径
            PublicIncludePaths.Add(AWS_SDK_PATH + "/include");

            // 添加库路径
            PublicLibraryPaths.Add(AWS_SDK_PATH + "/lib");

            // 添加AWS库
            PublicAdditionalLibraries.AddRange(new string[] {
                "aws-cpp-sdk-core.lib",
                "aws-cpp-sdk-s3.lib",
                "aws-cpp-sdk-lambda.lib",
                "aws-cpp-sdk-gamelift.lib",
                "aws-cpp-sdk-apigateway.lib"
            });

            // 运行时依赖（DLL复制）
            RuntimeDependencies.Add("$(BinaryOutputDir)/aws-cpp-sdk-core.dll", AWS_SDK_PATH + "/bin/aws-cpp-sdk-core.dll");
            RuntimeDependencies.Add("$(BinaryOutputDir)/aws-cpp-sdk-s3.dll", AWS_SDK_PATH + "/bin/aws-cpp-sdk-s3.dll");
            RuntimeDependencies.Add("$(BinaryOutputDir)/aws-cpp-sdk-lambda.dll", AWS_SDK_PATH + "/bin/aws-cpp-sdk-lambda.dll");
            RuntimeDependencies.Add("$(BinaryOutputDir)/aws-cpp-sdk-gamelift.dll", AWS_SDK_PATH + "/bin/aws-cpp-sdk-gamelift.dll");
        }
    }
}
```

### 21.1.2 AWS凭证配置

**创建AWS配置子系统：**

```csharp
// AWSConfigSubsystem.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include <aws/core/Aws.h>
#include <aws/core/auth/AWSCredentialsProvider.h>
#include <aws/core/client/ClientConfiguration.h>

#include "AWSConfigSubsystem.generated.h"

USTRUCT(BlueprintType)
struct FAWSConfig
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AWS")
    FString AccessKeyID;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AWS")
    FString SecretAccessKey;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AWS")
    FString Region = "us-east-1";

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AWS")
    FString ProfileName = "default";

    // 使用环境变量或凭证文件
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AWS")
    bool bUseDefaultCredentialChain = true;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnAWSInitialized, bool, bSuccess);

UCLASS()
class AWSTUTORIAL_API UAWSConfigSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    UFUNCTION(BlueprintCallable, Category = "AWS")
    void InitializeAWS(const FAWSConfig& Config);

    UFUNCTION(BlueprintCallable, Category = "AWS")
    void ShutdownAWS();

    UFUNCTION(BlueprintPure, Category = "AWS")
    bool IsAWSInitialized() const { return bIsInitialized; }

    UPROPERTY(BlueprintAssignable, Category = "AWS")
    FOnAWSInitialized OnAWSInitialized;

    // 获取SDK配置（用于其他模块）
    Aws::Client::ClientConfiguration GetClientConfig() const;

private:
    Aws::SDKOptions SDKOptions;
    bool bIsInitialized = false;
    FString CurrentRegion;
};
```

```csharp
// AWSConfigSubsystem.cpp
#include "AWSConfigSubsystem.h"

void UAWSConfigSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
    // 延迟初始化，等待配置加载
}

void UAWSConfigSubsystem::Deinitialize()
{
    ShutdownAWS();
    Super::Deinitialize();
}

void UAWSConfigSubsystem::InitializeAWS(const FAWSConfig& Config)
{
    if (bIsInitialized)
    {
        UE_LOG(LogTemp, Warning, TEXT("AWS already initialized"));
        OnAWSInitialized.Broadcast(true);
        return;
    }

    // 设置日志级别
    SDKOptions.loggingOptions.logLevel = Aws::Utils::Logging::LogLevel::Info;

    // 初始化AWS SDK
    Aws::InitAPI(SDKOptions);

    // 配置区域
    CurrentRegion = TCHAR_TO_UTF8(*Config.Region);

    bIsInitialized = true;
    UE_LOG(LogTemp, Log, TEXT("AWS SDK initialized successfully in region: %s"), *Config.Region);

    OnAWSInitialized.Broadcast(true);
}

void UAWSConfigSubsystem::ShutdownAWS()
{
    if (bIsInitialized)
    {
        Aws::ShutdownAPI(SDKOptions);
        bIsInitialized = false;
        UE_LOG(LogTemp, Log, TEXT("AWS SDK shutdown complete"));
    }
}

Aws::Client::ClientConfiguration UAWSConfigSubsystem::GetClientConfig() const
{
    Aws::Client::ClientConfiguration Config;
    Config.region = TCHAR_TO_UTF8(*CurrentRegion);
    return Config;
}
```

---

## 21.2 Amazon S3 存储集成

### 21.2.1 S3客户端封装

```csharp
// S3Client.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include <aws/s3/S3Client.h>
#include <aws/s3/model/PutObjectRequest.h>
#include <aws/s3/model/GetObjectRequest.h>
#include <aws/s3/model/DeleteObjectRequest.h>
#include <aws/s3/model/ListObjectsRequest.h>

#include "S3Client.generated.h"

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnS3UploadComplete, bool, bSuccess, FString, ErrorMessage);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnS3DownloadComplete, bool, bSuccess, TArray<uint8>, Data);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnS3ListComplete, bool, bSuccess, TArray<FString>, ObjectKeys);

UCLASS()
class AWSTUTORIAL_API US3Client : public UObject
{
    GENERATED_BODY()

public:
    void Initialize(TSharedPtr<Aws::S3::S3Client> InClient);

    // 上传文件
    UFUNCTION(BlueprintCallable, Category = "S3")
    void UploadFile(const FString& BucketName, const FString& ObjectKey, const TArray<uint8>& Data);

    // 下载文件
    UFUNCTION(BlueprintCallable, Category = "S3")
    void DownloadFile(const FString& BucketName, const FString& ObjectKey);

    // 删除文件
    UFUNCTION(BlueprintCallable, Category = "S3")
    void DeleteFile(const FString& BucketName, const FString& ObjectKey);

    // 列出存储桶内容
    UFUNCTION(BlueprintCallable, Category = "S3")
    void ListObjects(const FString& BucketName, const FString& Prefix = "");

    UPROPERTY(BlueprintAssignable, Category = "S3")
    FOnS3UploadComplete OnUploadComplete;

    UPROPERTY(BlueprintAssignable, Category = "S3")
    FOnS3DownloadComplete OnDownloadComplete;

    UPROPERTY(BlueprintAssignable, Category = "S3")
    FOnS3ListComplete OnListComplete;

private:
    TSharedPtr<Aws::S3::S3Client> S3Client;

    // 异步任务
    void UploadFileAsync(const FString& BucketName, const FString& ObjectKey, const TArray<uint8>& Data);
    void DownloadFileAsync(const FString& BucketName, const FString& ObjectKey);
};
```

```csharp
// S3Client.cpp
#include "S3Client.h"
#include "Async/Async.h"

void US3Client::Initialize(TSharedPtr<Aws::S3::S3Client> InClient)
{
    S3Client = InClient;
}

void US3Client::UploadFile(const FString& BucketName, const FString& ObjectKey, const TArray<uint8>& Data)
{
    AsyncTask(ENamedThreads::AnyThread, [this, BucketName, ObjectKey, Data]()
    {
        UploadFileAsync(BucketName, ObjectKey, Data);
    });
}

void US3Client::UploadFileAsync(const FString& BucketName, const FString& ObjectKey, const TArray<uint8>& Data)
{
    using namespace Aws::S3::Model;

    PutObjectRequest Request;
    Request.SetBucket(TCHAR_TO_UTF8(*BucketName));
    Request.SetKey(TCHAR_TO_UTF8(*ObjectKey));

    // 创建输入流
    auto DataStream = Aws::MakeShared<Aws::StringStream>("UploadStream");
    DataStream->write(reinterpret_cast<const char*>(Data.GetData()), Data.Num());
    DataStream->flush();

    Request.SetBody(DataStream);
    Request.SetContentLength(Data.Num());

    auto Outcome = S3Client->PutObject(Request);

    AsyncTask(ENamedThreads::GameThread, [this, Outcome]()
    {
        if (Outcome.IsSuccess())
        {
            OnUploadComplete.Broadcast(true, TEXT(""));
        }
        else
        {
            FString ErrorMsg = UTF8_TO_TCHAR(Outcome.GetError().GetMessage().c_str());
            OnUploadComplete.Broadcast(false, ErrorMsg);
        }
    });
}

void US3Client::DownloadFile(const FString& BucketName, const FString& ObjectKey)
{
    AsyncTask(ENamedThreads::AnyThread, [this, BucketName, ObjectKey]()
    {
        DownloadFileAsync(BucketName, ObjectKey);
    });
}

void US3Client::DownloadFileAsync(const FString& BucketName, const FString& ObjectKey)
{
    using namespace Aws::S3::Model;

    GetObjectRequest Request;
    Request.SetBucket(TCHAR_TO_UTF8(*BucketName));
    Request.SetKey(TCHAR_TO_UTF8(*ObjectKey));

    auto Outcome = S3Client->GetObject(Request);

    AsyncTask(ENamedThreads::GameThread, [this, Outcome]()
    {
        if (Outcome.IsSuccess())
        {
            auto& Body = Outcome.GetResult().GetBody();
            Body.seekg(0, std::ios::end);
            size_t Size = Body.tellg();
            Body.seekg(0, std::ios::beg);

            TArray<uint8> Data;
            Data.SetNumUninitialized(Size);
            Body.read(reinterpret_cast<char*>(Data.GetData()), Size);

            OnDownloadComplete.Broadcast(true, Data);
        }
        else
        {
            TArray<uint8> EmptyData;
            OnDownloadComplete.Broadcast(false, EmptyData);
        }
    });
}

void US3Client::ListObjects(const FString& BucketName, const FString& Prefix)
{
    AsyncTask(ENamedThreads::AnyThread, [this, BucketName, Prefix]()
    {
        using namespace Aws::S3::Model;

        ListObjectsRequest Request;
        Request.SetBucket(TCHAR_TO_UTF8(*BucketName));
        if (!Prefix.IsEmpty())
        {
            Request.SetPrefix(TCHAR_TO_UTF8(*Prefix));
        }

        auto Outcome = S3Client->ListObjects(Request);

        AsyncTask(ENamedThreads::GameThread, [this, Outcome]()
        {
            if (Outcome.IsSuccess())
            {
                TArray<FString> Keys;
                for (const auto& Object : Outcome.GetResult().GetContents())
                {
                    Keys.Add(UTF8_TO_TCHAR(Object.GetKey().c_str()));
                }
                OnListComplete.Broadcast(true, Keys);
            }
            else
            {
                TArray<FString> EmptyKeys;
                OnListComplete.Broadcast(false, EmptyKeys);
            }
        });
    });
}
```

---

## 21.3 Amazon GameLift 集成

### 21.3.1 GameLift服务器SDK集成

**服务器端初始化：**

```csharp
// GameLiftServerManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include <aws/gamelift/server/GameLiftServerAPI.h>
#include <aws/gamelift/server/ProcessParameters.h>
#include <aws/gamelift/server/model/GameSession.h>
#include <aws/gamelift/server/model/PlayerSession.h>

#include "GameLiftServerManager.generated.h"

USTRUCT(BlueprintType)
struct FGameLiftServerConfig
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 Port = 7777;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString LogPath;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TArray<FString> LogPaths;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnGameSessionStart, const FString&, GameSessionId);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnGameSessionEnd, const FString&, GameSessionId);

UCLASS()
class AWSTUTORIAL_API UGameLiftServerManager : public UObject
{
    GENERATED_BODY()

public:
    bool InitializeGameLift(const FGameLiftServerConfig& Config);
    void ProcessReady();
    void ProcessEnding();
    void ActivateGameSession(const FString& GameSessionId);
    void TerminateGameSession(const FString& GameSessionId);

    // 玩家会话管理
    bool AcceptPlayerSession(const FString& PlayerSessionId);
    void RemovePlayerSession(const FString& PlayerSessionId);

    UPROPERTY(BlueprintAssignable)
    FOnGameSessionStart OnGameSessionStart;

    UPROPERTY(BlueprintAssignable)
    FOnGameSessionEnd OnGameSessionEnd;

private:
    bool bIsInitialized = false;
    FString CurrentGameSessionId;

    // GameLift回调
    void OnStartGameSession(const Aws::GameLift::Server::Model::GameSession& GameSession);
    void OnUpdateGameSession(const Aws::GameLift::Server::Model::UpdateGameSession& UpdateGameSession);
    void OnProcessTerminate();
    bool OnHealthCheck();
};
```

```csharp
// GameLiftServerManager.cpp
#include "GameLiftServerManager.h"
#include "Engine/World.h"
#include "Kismet/GameplayStatics.h"

bool UGameLiftServerManager::InitializeGameLift(const FGameLiftServerConfig& Config)
{
    using namespace Aws::GameLift::Server;

    // 初始化GameLift Server SDK
    auto Outcome = GameLiftServerAPI::InitSDK();

    if (!Outcome.IsSuccess())
    {
        UE_LOG(LogTemp, Error, TEXT("GameLift InitSDK failed: %s"),
            UTF8_TO_TCHAR(Outcome.GetError().GetErrorMessage().c_str()));
        return false;
    }

    // 设置进程参数
    ProcessParameters Params;

    // 端口配置
    Params.SetPort(Config.Port);

    // 游戏会话回调
    Params.SetOnStartGameSession([this](const Model::GameSession& GameSession)
    {
        OnStartGameSession(GameSession);
    });

    // 更新游戏会话回调
    Params.SetOnUpdateGameSession([this](const Model::UpdateGameSession& UpdateGameSession)
    {
        OnUpdateGameSession(UpdateGameSession);
    });

    // 进程终止回调
    Params.SetOnProcessTerminate([this]()
    {
        OnProcessTerminate();
    });

    // 健康检查回调
    Params.SetOnHealthCheck([this]()
    {
        return OnHealthCheck();
    });

    // 日志路径
    if (!Config.LogPath.IsEmpty())
    {
        std::vector<std::string> LogPathsVec;
        for (const FString& Path : Config.LogPaths)
        {
            LogPathsVec.push_back(TCHAR_TO_UTF8(*Path));
        }
        Params.SetLogParameters(LogPathsVec);
    }

    // 通知GameLift进程就绪
    auto ReadyOutcome = GameLiftServerAPI::ProcessReady(Params);

    if (!ReadyOutcome.IsSuccess())
    {
        UE_LOG(LogTemp, Error, TEXT("GameLift ProcessReady failed: %s"),
            UTF8_TO_TCHAR(ReadyOutcome.GetError().GetErrorMessage().c_str()));
        return false;
    }

    bIsInitialized = true;
    UE_LOG(LogTemp, Log, TEXT("GameLift Server SDK initialized successfully"));
    return true;
}

void UGameLiftServerManager::OnStartGameSession(const Aws::GameLift::Server::Model::GameSession& GameSession)
{
    CurrentGameSessionId = UTF8_TO_TCHAR(GameSession.GetGameSessionId().c_str());
    UE_LOG(LogTemp, Log, TEXT("Game session started: %s"), *CurrentGameSessionId);

    // 激活游戏会话
    ActivateGameSession(CurrentGameSessionId);

    // 广播事件
    OnGameSessionStart.Broadcast(CurrentGameSessionId);
}

void UGameLiftServerManager::OnUpdateGameSession(const Aws::GameLift::Server::Model::UpdateGameSession& UpdateGameSession)
{
    UE_LOG(LogTemp, Log, TEXT("Game session update received"));

    // 处理回话更新，如匹配回填等
    const auto& GameSession = UpdateGameSession.GetGameSession();
    // 根据更新类型处理
}

void UGameLiftServerManager::OnProcessTerminate()
{
    UE_LOG(LogTemp, Log, TEXT("Process termination requested by GameLift"));

    // 清理游戏会话
    if (!CurrentGameSessionId.IsEmpty())
    {
        TerminateGameSession(CurrentGameSessionId);
        OnGameSessionEnd.Broadcast(CurrentGameSessionId);
    }

    // 通知GameLift进程结束
    ProcessEnding();
}

bool UGameLiftServerManager::OnHealthCheck()
{
    // 返回服务器健康状态
    return true;
}

void UGameLiftServerManager::ActivateGameSession(const FString& GameSessionId)
{
    using namespace Aws::GameLift::Server;

    auto Outcome = GameLiftServerAPI::ActivateGameSession();
    if (!Outcome.IsSuccess())
    {
        UE_LOG(LogTemp, Error, TEXT("Failed to activate game session: %s"),
            UTF8_TO_TCHAR(Outcome.GetError().GetErrorMessage().c_str()));
    }
}

void UGameLiftServerManager::TerminateGameSession(const FString& GameSessionId)
{
    using namespace Aws::GameLift::Server;

    auto Outcome = GameLiftServerAPI::TerminateGameSession();
    if (!Outcome.IsSuccess())
    {
        UE_LOG(LogTemp, Error, TEXT("Failed to terminate game session: %s"),
            UTF8_TO_TCHAR(Outcome.GetError().GetErrorMessage().c_str()));
    }
}

bool UGameLiftServerManager::AcceptPlayerSession(const FString& PlayerSessionId)
{
    using namespace Aws::GameLift::Server;

    auto Outcome = GameLiftServerAPI::AcceptPlayerSession(
        TCHAR_TO_UTF8(*PlayerSessionId));

    if (Outcome.IsSuccess())
    {
        UE_LOG(LogTemp, Log, TEXT("Player session accepted: %s"), *PlayerSessionId);
        return true;
    }
    else
    {
        UE_LOG(LogTemp, Error, TEXT("Failed to accept player session: %s"),
            UTF8_TO_TCHAR(Outcome.GetError().GetErrorMessage().c_str()));
        return false;
    }
}

void UGameLiftServerManager::RemovePlayerSession(const FString& PlayerSessionId)
{
    using namespace Aws::GameLift::Server;

    GameLiftServerAPI::RemovePlayerSession(TCHAR_TO_UTF8(*PlayerSessionId));
    UE_LOG(LogTemp, Log, TEXT("Player session removed: %s"), *PlayerSessionId);
}

void UGameLiftServerManager::ProcessEnding()
{
    Aws::GameLift::Server::GameLiftServerAPI::ProcessEnding();
}
```

### 21.3.2 客户端GameLift集成

```csharp
// GameLiftClientManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include <aws/gamelift/GameLiftClient.h>
#include <aws/gamelift/model/CreateGameSessionRequest.h>
#include <aws/gamelift/model/DescribeGameSessionRequest.h>
#include <aws/gamelift/model/CreatePlayerSessionRequest.h>

#include "GameLiftClientManager.generated.h"

USTRUCT(BlueprintType)
struct FGameSessionInfo
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString GameSessionId;

    UPROPERTY(BlueprintReadOnly)
    FString FleetId;

    UPROPERTY(BlueprintReadOnly)
    FString IpAddress;

    UPROPERTY(BlueprintReadOnly)
    int32 Port = 0;

    UPROPERTY(BlueprintReadOnly)
    FString Status;

    UPROPERTY(BlueprintReadOnly)
    int32 CurrentPlayerCount = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 MaximumPlayerCount = 0;
};

USTRUCT(BlueprintType)
struct FPlayerSessionInfo
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString PlayerSessionId;

    UPROPERTY(BlueprintReadOnly)
    FString PlayerId;

    UPROPERTY(BlueprintReadOnly)
    FString IpAddress;

    UPROPERTY(BlueprintReadOnly)
    int32 Port = 0;

    UPROPERTY(BlueprintReadOnly)
    FString Status;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnCreateGameSession, bool, bSuccess, FGameSessionInfo, SessionInfo);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnJoinGameSession, bool, bSuccess, FPlayerSessionInfo, PlayerSession);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnSearchSessions, bool, bSuccess, TArray<FGameSessionInfo>, Sessions);

UCLASS()
class AWSTUTORIAL_API UGameLiftClientManager : public UObject
{
    GENERATED_BODY()

public:
    void Initialize(const FString& Region, const FString& AccessKey, const FString& SecretKey);

    // 创建游戏会话
    UFUNCTION(BlueprintCallable, Category = "GameLift")
    void CreateGameSession(const FString& FleetId, int32 MaxPlayers, const TMap<FString, FString>& GameProperties);

    // 搜索游戏会话
    UFUNCTION(BlueprintCallable, Category = "GameLift")
    void SearchGameSessions(const FString& FleetId, const FString& FilterExpression = TEXT(""));

    // 加入游戏会话
    UFUNCTION(BlueprintCallable, Category = "GameLift")
    void JoinGameSession(const FString& GameSessionId, const FString& PlayerId, const FString& PlayerData = TEXT(""));

    // 开始匹配
    UFUNCTION(BlueprintCallable, Category = "GameLift")
    void StartMatchmaking(const FString& ConfigurationName, const FString& PlayerId, int32 LatencyMs);

    // 取消匹配
    UFUNCTION(BlueprintCallable, Category = "GameLift")
    void StopMatchmaking(const FString& TicketId);

    UPROPERTY(BlueprintAssignable, Category = "GameLift")
    FOnCreateGameSession OnCreateGameSession;

    UPROPERTY(BlueprintAssignable, Category = "GameLift")
    FOnJoinGameSession OnJoinGameSession;

    UPROPERTY(BlueprintAssignable, Category = "GameLift")
    FOnSearchSessions OnSearchSessions;

private:
    TSharedPtr<Aws::GameLift::GameLiftClient> GameLiftClient;
};
```

---

## 21.4 AWS Lambda 函数调用

### 21.4.1 Lambda客户端封装

```csharp
// LambdaClient.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include <aws/lambda/LambdaClient.h>
#include <aws/lambda/model/InvokeRequest.h>
#include <aws/lambda/model/InvokeResult.h>

#include "LambdaClient.generated.h"

USTRUCT(BlueprintType)
struct FLambdaInvokeResult
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    int32 StatusCode = 0;

    UPROPERTY(BlueprintReadOnly)
    FString Payload;

    UPROPERTY(BlueprintReadOnly)
    FString FunctionError;

    UPROPERTY(BlueprintReadOnly)
    bool bSuccess = false;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnLambdaInvoke, FLambdaInvokeResult, Result);

UCLASS()
class AWSTUTORIAL_API ULambdaClient : public UObject
{
    GENERATED_BODY()

public:
    void Initialize(TSharedPtr<Aws::Lambda::LambdaClient> InClient);

    // 同步调用Lambda
    UFUNCTION(BlueprintCallable, Category = "Lambda")
    FLambdaInvokeResult InvokeSync(const FString& FunctionName, const FString& Payload);

    // 异步调用Lambda
    UFUNCTION(BlueprintCallable, Category = "Lambda")
    void InvokeAsync(const FString& FunctionName, const FString& Payload);

    // 事件异步调用（Fire and Forget）
    UFUNCTION(BlueprintCallable, Category = "Lambda")
    bool InvokeEvent(const FString& FunctionName, const FString& Payload);

    UPROPERTY(BlueprintAssignable, Category = "Lambda")
    FOnLambdaInvoke OnLambdaInvoke;

private:
    TSharedPtr<Aws::Lambda::LambdaClient> LambdaClient;

    FLambdaInvokeResult ParseInvokeResult(const Aws::Lambda::Model::InvokeResult& Result);
};
```

```csharp
// LambdaClient.cpp
#include "LambdaClient.h"

void ULambdaClient::Initialize(TSharedPtr<Aws::Lambda::LambdaClient> InClient)
{
    LambdaClient = InClient;
}

FLambdaInvokeResult ULambdaClient::InvokeSync(const FString& FunctionName, const FString& Payload)
{
    using namespace Aws::Lambda::Model;

    InvokeRequest Request;
    Request.SetFunctionName(TCHAR_TO_UTF8(*FunctionName));
    Request.SetInvocationType(InvocationType::RequestResponse);

    // 设置Payload
    auto PayloadStream = Aws::MakeShared<Aws::StringStream>("PayloadStream");
    *PayloadStream << TCHAR_TO_UTF8(*Payload);
    Request.SetBody(PayloadStream);

    auto Outcome = LambdaClient->Invoke(Request);
    return ParseInvokeResult(Outcome.GetResult());
}

void ULambdaClient::InvokeAsync(const FString& FunctionName, const FString& Payload)
{
    AsyncTask(ENamedThreads::AnyThread, [this, FunctionName, Payload]()
    {
        using namespace Aws::Lambda::Model;

        InvokeRequest Request;
        Request.SetFunctionName(TCHAR_TO_UTF8(*FunctionName));
        Request.SetInvocationType(InvocationType::RequestResponse);

        auto PayloadStream = Aws::MakeShared<Aws::StringStream>("PayloadStream");
        *PayloadStream << TCHAR_TO_UTF8(*Payload);
        Request.SetBody(PayloadStream);

        auto Outcome = LambdaClient->Invoke(Request);
        FLambdaInvokeResult Result = ParseInvokeResult(Outcome.GetResult());

        AsyncTask(ENamedThreads::GameThread, [this, Result]()
        {
            OnLambdaInvoke.Broadcast(Result);
        });
    });
}

bool ULambdaClient::InvokeEvent(const FString& FunctionName, const FString& Payload)
{
    using namespace Aws::Lambda::Model;

    InvokeRequest Request;
    Request.SetFunctionName(TCHAR_TO_UTF8(*FunctionName));
    Request.SetInvocationType(InvocationType::Event);

    auto PayloadStream = Aws::MakeShared<Aws::StringStream>("PayloadStream");
    *PayloadStream << TCHAR_TO_UTF8(*Payload);
    Request.SetBody(PayloadStream);

    auto Outcome = LambdaClient->Invoke(Request);
    return Outcome.IsSuccess();
}

FLambdaInvokeResult ULambdaClient::ParseInvokeResult(const Aws::Lambda::Model::InvokeResult& Result)
{
    FLambdaInvokeResult OutResult;
    OutResult.StatusCode = Result.GetStatusCode();
    OutResult.FunctionError = UTF8_TO_TCHAR(Result.GetFunctionError().c_str());
    OutResult.bSuccess = Result.GetStatusCode() >= 200 && Result.GetStatusCode() < 300;

    // 读取Payload
    auto& PayloadStream = Result.GetPayload();
    std::string PayloadStr((std::istreambuf_iterator<char>(PayloadStream)),
                           std::istreambuf_iterator<char>());
    OutResult.Payload = UTF8_TO_TCHAR(PayloadStr.c_str());

    return OutResult;
}
```

### 21.4.2 常用Lambda场景示例

**用户数据验证Lambda（Python）：**

```python
# AWS Lambda函数 - 用户数据验证
import json

def lambda_handler(event, context):
    """
    验证用户数据并返回验证结果
    """
    user_id = event.get('userId')
    action = event.get('action')

    if not user_id:
        return {
            'statusCode': 400,
            'body': json.dumps({'error': 'Missing userId'})
        }

    # 根据操作类型处理
    if action == 'validate_purchase':
        return validate_purchase(event)
    elif action == 'sync_inventory':
        return sync_inventory(event)
    elif action == 'check_ban_status':
        return check_ban_status(user_id)

    return {
        'statusCode': 200,
        'body': json.dumps({'status': 'unknown_action'})
    }

def validate_purchase(event):
    # 实现购买验证逻辑
    return {
        'statusCode': 200,
        'body': json.dumps({
            'valid': True,
            'transactionId': event.get('transactionId')
        })
    }

def sync_inventory(event):
    # 同步库存逻辑
    return {
        'statusCode': 200,
        'body': json.dumps({
            'items': [],
            'synced': True
        })
    }

def check_ban_status(user_id):
    # 检查封禁状态
    return {
        'statusCode': 200,
        'body': json.dumps({
            'banned': False,
            'reason': None
        })
    }
```

---

## 21.5 API Gateway 集成

### 21.5.1 REST API调用封装

```csharp
// APIGatewayClient.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include <aws/apigateway/APIGatewayClient.h>
#include "HttpModule.h"
#include "Interfaces/IHttpRequest.h"
#include "Interfaces/IHttpResponse.h"

#include "APIGatewayClient.generated.h"

USTRUCT(BlueprintType)
struct FAPIResponse
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    int32 StatusCode = 0;

    UPROPERTY(BlueprintReadOnly)
    FString Body;

    UPROPERTY(BlueprintReadOnly)
    TMap<FString, FString> Headers;

    UPROPERTY(BlueprintReadOnly)
    bool bSuccess = false;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnAPIResponse, FAPIResponse, Response);

UCLASS()
class AWSTUTORIAL_API UAPIGatewayClient : public UObject
{
    GENERATED_BODY()

public:
    void Initialize(const FString& InBaseUrl, const FString& InApiKey = TEXT(""));

    // GET请求
    UFUNCTION(BlueprintCallable, Category = "API Gateway")
    void Get(const FString& Path, const TMap<FString, FString>& QueryParams = TMap<FString, FString>());

    // POST请求
    UFUNCTION(BlueprintCallable, Category = "API Gateway")
    void Post(const FString& Path, const FString& Body);

    // PUT请求
    UFUNCTION(BlueprintCallable, Category = "API Gateway")
    void Put(const FString& Path, const FString& Body);

    // DELETE请求
    UFUNCTION(BlueprintCallable, Category = "API Gateway")
    void Delete(const FString& Path);

    // 设置认证头
    UFUNCTION(BlueprintCallable, Category = "API Gateway")
    void SetAuthToken(const FString& Token);

    UPROPERTY(BlueprintAssignable, Category = "API Gateway")
    FOnAPIResponse OnResponse;

private:
    FString BaseUrl;
    FString ApiKey;
    FString AuthToken;

    void MakeRequest(const FString& Verb, const FString& Path, const FString& Body = TEXT(""));
    FString BuildUrl(const FString& Path, const TMap<FString, FString>& QueryParams);
    void OnRequestComplete(FHttpRequestPtr Request, FHttpResponsePtr Response, bool bSuccess);
};
```

```csharp
// APIGatewayClient.cpp
#include "APIGatewayClient.h"

void UAPIGatewayClient::Initialize(const FString& InBaseUrl, const FString& InApiKey)
{
    BaseUrl = InBaseUrl;
    ApiKey = InApiKey;
}

void UAPIGatewayClient::SetAuthToken(const FString& Token)
{
    AuthToken = Token;
}

void UAPIGatewayClient::Get(const FString& Path, const TMap<FString, FString>& QueryParams)
{
    FString Url = BuildUrl(Path, QueryParams);
    MakeRequest(TEXT("GET"), Url);
}

void UAPIGatewayClient::Post(const FString& Path, const FString& Body)
{
    MakeRequest(TEXT("POST"), Path, Body);
}

void UAPIGatewayClient::Put(const FString& Path, const FString& Body)
{
    MakeRequest(TEXT("PUT"), Path, Body);
}

void UAPIGatewayClient::Delete(const FString& Path)
{
    MakeRequest(TEXT("DELETE"), Path);
}

FString UAPIGatewayClient::BuildUrl(const FString& Path, const TMap<FString, FString>& QueryParams)
{
    FString Url = BaseUrl + Path;

    if (QueryParams.Num() > 0)
    {
        Url += TEXT("?");
        bool bFirst = true;
        for (const auto& Pair : QueryParams)
        {
            if (!bFirst) Url += TEXT("&");
            Url += FPlatformHttp::UrlEncode(Pair.Key) + TEXT("=") + FPlatformHttp::UrlEncode(Pair.Value);
            bFirst = false;
        }
    }

    return Url;
}

void UAPIGatewayClient::MakeRequest(const FString& Verb, const FString& Path, const FString& Body)
{
    TSharedRef<IHttpRequest> Request = FHttpModule::Get().CreateRequest();
    Request->SetVerb(Verb);
    Request->SetURL(BaseUrl + Path);
    Request->SetHeader(TEXT("Content-Type"), TEXT("application/json"));

    // API Key
    if (!ApiKey.IsEmpty())
    {
        Request->SetHeader(TEXT("x-api-key"), ApiKey);
    }

    // Auth Token
    if (!AuthToken.IsEmpty())
    {
        Request->SetHeader(TEXT("Authorization"), TEXT("Bearer ") + AuthToken);
    }

    if (!Body.IsEmpty())
    {
        Request->SetContentAsString(Body);
    }

    Request->OnProcessRequestComplete().BindUObject(this, &UAPIGatewayClient::OnRequestComplete);
    Request->ProcessRequest();
}

void UAPIGatewayClient::OnRequestComplete(FHttpRequestPtr Request, FHttpResponsePtr Response, bool bSuccess)
{
    FAPIResponse APIResponse;
    APIResponse.bSuccess = bSuccess && Response.IsValid();

    if (Response.IsValid())
    {
        APIResponse.StatusCode = Response->GetResponseCode();
        APIResponse.Body = Response->GetContentAsString();

        // 解析头部
        for (const auto& Header : Response->GetAllHeaders())
        {
            FString Key, Value;
            if (Header.Split(TEXT(": "), &Key, &Value))
            {
                APIResponse.Headers.Add(Key, Value);
            }
        }
    }

    OnResponse.Broadcast(APIResponse);
}
```

---

## 21.6 AWS安全最佳实践

### 21.6.1 IAM角色与策略

**最小权限原则示例：**

```json
// GameLift服务角色策略
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "gamelift:CreateGameSession",
                "gamelift:DescribeGameSessions",
                "gamelift:CreatePlayerSession",
                "gamelift:DescribePlayerSessions"
            ],
            "Resource": "arn:aws:gamelift:*:*:gamesession/*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject"
            ],
            "Resource": "arn:aws:s3:::game-assets-bucket/*"
        }
    ]
}
```

### 21.6.2 凭证安全存储

```csharp
// AWSSecurityManager.h
#pragma once

#include "CoreMinimal.h"
#include "AWSConfigSubsystem.h"

UCLASS()
class AWSTUTORIAL_API UAWSSecurityManager : public UObject
{
    GENERATED_BODY()

public:
    // 从环境变量获取凭证
    static FAWSConfig GetCredentialsFromEnv();

    // 从AWS凭证文件获取
    static FAWSConfig GetCredentialsFromFile(const FString& ProfileName = TEXT("default"));

    // 从EC2实例元数据获取（适用于EC2部署）
    static FAWSConfig GetCredentialsFromIMDS();

    // 验证凭证有效性
    static bool ValidateCredentials(const FAWSConfig& Config);
};
```

```csharp
// AWSSecurityManager.cpp
#include "AWSSecurityManager.h"
#include "Misc/FileHelper.h"
#include "Misc/Paths.h"

FAWSConfig UAWSSecurityManager::GetCredentialsFromEnv()
{
    FAWSConfig Config;

    // 从环境变量读取
    Config.AccessKeyID = FPlatformMisc::GetEnvironmentVariable(TEXT("AWS_ACCESS_KEY_ID"));
    Config.SecretAccessKey = FPlatformMisc::GetEnvironmentVariable(TEXT("AWS_SECRET_ACCESS_KEY"));
    Config.Region = FPlatformMisc::GetEnvironmentVariable(TEXT("AWS_DEFAULT_REGION"));

    if (Config.Region.IsEmpty())
    {
        Config.Region = TEXT("us-east-1");
    }

    Config.bUseDefaultCredentialChain = false;
    return Config;
}

FAWSConfig UAWSSecurityManager::GetCredentialsFromFile(const FString& ProfileName)
{
    FAWSConfig Config;
    Config.ProfileName = ProfileName;
    Config.bUseDefaultCredentialChain = true;

    // AWS SDK会自动从~/.aws/credentials读取
    return Config;
}

FAWSConfig UAWSSecurityManager::GetCredentialsFromIMDS()
{
    FAWSConfig Config;

    // 在EC2实例上，使用IAM角色
    Config.bUseDefaultCredentialChain = true;

    // 从IMDSv2获取区域信息
    // 注意：这是简化示例，实际应使用HTTP请求获取

    return Config;
}
```

---

## 21.7 实践任务

### 任务1：实现用户存档云备份

创建一个系统，将用户SaveGame数据上传到S3，并支持跨设备恢复。

### 任务2：集成GameLift匹配

实现一个完整的匹配流程：
1. 客户端请求匹配
2. GameLift创建会话
3. 客户端加入会话

### 任务3：Lambda函数集成

创建一个物品商店验证系统：
1. 客户端发起购买请求
2. Lambda验证购买合法性
3. 返回验证结果

---

## 21.8 总结

本课学习了：
- AWS SDK for C++的集成与配置
- Amazon S3存储服务的使用
- Amazon GameLift服务器托管集成
- AWS Lambda函数调用
- API Gateway REST API集成
- AWS安全最佳实践

**下一课预告**：PlayFab集成 - 微软的完整游戏后端服务平台。

---

*课程版本：1.0*
*最后更新：2026-04-10*
