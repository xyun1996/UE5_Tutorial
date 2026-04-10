# 第24课：微服务架构设计

> 学习目标：掌握游戏后端微服务拆分策略，学习服务间通信、API网关设计和服务发现机制。

---

## 24.1 微服务架构概述

### 24.1.1 单体架构 vs 微服务架构

**单体架构特点：**
- 所有功能在一个进程中
- 共享数据库
- 简单部署
- 扩展受限

**微服务架构特点：**
- 服务独立部署
- 独立数据库
- 技术栈灵活
- 高可用可扩展

### 24.1.2 游戏后端服务拆分

```
┌─────────────────────────────────────────────────────────────────┐
│                        API Gateway                               │
│                     (Kong / AWS API Gateway)                     │
└─────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  Auth Service │   │ Match Service │   │  Game Service │
│   (认证服务)   │   │   (匹配服务)   │   │  (游戏服务)   │
└───────────────┘   └───────────────┘   └───────────────┘
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ User Database │   │ Match Redis   │   │ Game Database │
│   (PostgreSQL) │   │    (Redis)    │   │  (MongoDB)    │
└───────────────┘   └───────────────┘   └───────────────┘

┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│Payment Service│   │Inventory Svc  │   │ Analytics Svc │
│   (支付服务)   │   │  (库存服务)    │   │  (分析服务)   │
└───────────────┘   └───────────────┘   └───────────────┘
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│Payment Gateway│   │Inventory DB   │   │Analytics DB   │
│               │   │  (PostgreSQL) │   │ (TimescaleDB) │
└───────────────┘   └───────────────┘   └───────────────┘
```

---

## 24.2 服务间通信

### 24.2.1 REST API通信

**服务间HTTP客户端：**

```csharp
// MicroserviceClient.h
#pragma once

#include "CoreMinimal.h"
#include "HttpModule.h"
#include "Interfaces/IHttpRequest.h"
#include "Interfaces/IHttpResponse.h"
#include "JsonUtilities.h"

#include "MicroserviceClient.generated.h"

USTRUCT(BlueprintType)
struct FServiceResponse
{
    GENERATED_BODY()

    UPROPERTY()
    int32 StatusCode = 0;

    UPROPERTY()
    FString Body;

    UPROPERTY()
    bool bSuccess = false;

    UPROPERTY()
    FString ErrorMessage;
};

DECLARE_DYNAMIC_DELEGATE_OneParam(FOnServiceResponse, const FServiceResponse&, Response);

UCLASS()
class MICROSERVICES_API UMicroserviceClient : public UObject
{
    GENERATED_BODY()

public:
    UMicroserviceClient();

    // 配置服务端点
    void Configure(const FString& InServiceName, const FString& InBaseUrl);

    // GET请求
    void Get(const FString& Endpoint, const TMap<FString, FString>& Headers, FOnServiceResponse Callback);

    // POST请求
    void Post(const FString& Endpoint, const FString& Body, const TMap<FString, FString>& Headers, FOnServiceResponse Callback);

    // PUT请求
    void Put(const FString& Endpoint, const FString& Body, const TMap<FString, FString>& Headers, FOnServiceResponse Callback);

    // DELETE请求
    void Delete(const FString& Endpoint, const TMap<FString, FString>& Headers, FOnServiceResponse Callback);

    // 设置认证令牌
    void SetAuthToken(const FString& Token);

    // 设置超时
    void SetTimeout(float Seconds);

private:
    FString ServiceName;
    FString BaseUrl;
    FString AuthToken;
    float TimeoutSeconds = 30.0f;

    void MakeRequest(const FString& Verb, const FString& Endpoint, const FString& Body, const TMap<FString, FString>& Headers, FOnServiceResponse Callback);
    void HandleResponse(FHttpRequestPtr Request, FHttpResponsePtr Response, bool bSuccess, FOnServiceResponse Callback);
};
```

```csharp
// MicroserviceClient.cpp
#include "MicroserviceClient.h"

UMicroserviceClient::UMicroserviceClient()
{
}

void UMicroserviceClient::Configure(const FString& InServiceName, const FString& InBaseUrl)
{
    ServiceName = InServiceName;
    BaseUrl = InBaseUrl;

    // 移除末尾斜杠
    if (BaseUrl.EndsWith(TEXT("/")))
    {
        BaseUrl.RemoveAt(BaseUrl.Len() - 1);
    }
}

void UMicroserviceClient::SetAuthToken(const FString& Token)
{
    AuthToken = Token;
}

void UMicroserviceClient::SetTimeout(float Seconds)
{
    TimeoutSeconds = Seconds;
}

void UMicroserviceClient::Get(const FString& Endpoint, const TMap<FString, FString>& Headers, FOnServiceResponse Callback)
{
    MakeRequest(TEXT("GET"), Endpoint, TEXT(""), Headers, Callback);
}

void UMicroserviceClient::Post(const FString& Endpoint, const FString& Body, const TMap<FString, FString>& Headers, FOnServiceResponse Callback)
{
    MakeRequest(TEXT("POST"), Endpoint, Body, Headers, Callback);
}

void UMicroserviceClient::Put(const FString& Endpoint, const FString& Body, const TMap<FString, FString>& Headers, FOnServiceResponse Callback)
{
    MakeRequest(TEXT("PUT"), Endpoint, Body, Headers, Callback);
}

void UMicroserviceClient::Delete(const FString& Endpoint, const TMap<FString, FString>& Headers, FOnServiceResponse Callback)
{
    MakeRequest(TEXT("DELETE"), Endpoint, TEXT(""), Headers, Callback);
}

void UMicroserviceClient::MakeRequest(const FString& Verb, const FString& Endpoint, const FString& Body, const TMap<FString, FString>& Headers, FOnServiceResponse Callback)
{
    TSharedRef<IHttpRequest> Request = FHttpModule::Get().CreateRequest();

    // 构建完整URL
    FString FullUrl = BaseUrl + Endpoint;
    Request->SetURL(FullUrl);
    Request->SetVerb(Verb);

    // 设置默认头
    Request->SetHeader(TEXT("Content-Type"), TEXT("application/json"));
    Request->SetHeader(TEXT("X-Service-Name"), ServiceName);

    // 设置认证头
    if (!AuthToken.IsEmpty())
    {
        Request->SetHeader(TEXT("Authorization"), TEXT("Bearer ") + AuthToken);
    }

    // 设置自定义头
    for (const auto& Header : Headers)
    {
        Request->SetHeader(Header.Key, Header.Value);
    }

    // 设置请求体
    if (!Body.IsEmpty())
    {
        Request->SetContentAsString(Body);
    }

    // 设置超时
    Request->SetTimeout(TimeoutSeconds);

    // 绑定回调
    Request->OnProcessRequestComplete().BindLambda([this, Callback](FHttpRequestPtr Req, FHttpResponsePtr Resp, bool bSuccess)
    {
        HandleResponse(Req, Resp, bSuccess, Callback);
    });

    Request->ProcessRequest();
}

void UMicroserviceClient::HandleResponse(FHttpRequestPtr Request, FHttpResponsePtr Response, bool bSuccess, FOnServiceResponse Callback)
{
    FServiceResponse ServiceResponse;

    if (bSuccess && Response.IsValid())
    {
        ServiceResponse.StatusCode = Response->GetResponseCode();
        ServiceResponse.Body = Response->GetContentAsString();
        ServiceResponse.bSuccess = ServiceResponse.StatusCode >= 200 && ServiceResponse.StatusCode < 300;

        if (!ServiceResponse.bSuccess)
        {
            ServiceResponse.ErrorMessage = FString::Printf(TEXT("HTTP Error: %d"), ServiceResponse.StatusCode);
        }
    }
    else
    {
        ServiceResponse.bSuccess = false;
        ServiceResponse.ErrorMessage = Request.IsValid() ? Request->GetErrorMessage() : TEXT("Unknown error");
    }

    Callback.ExecuteIfBound(ServiceResponse);
}
```

### 24.2.2 gRPC通信

**gRPC服务定义（Proto）：**

```protobuf
// match_service.proto
syntax = "proto3";

package game.match;

// 匹配服务定义
service MatchService {
    // 创建匹配请求
    rpc CreateMatchRequest (CreateMatchRequest) returns (MatchResponse);
    // 取消匹配
    rpc CancelMatchRequest (CancelMatchRequest) returns (CancelResponse);
    // 获取匹配状态
    rpc GetMatchStatus (GetMatchStatusRequest) returns (MatchStatusResponse);
    // 流式匹配状态更新
    rpc StreamMatchStatus (StreamMatchStatusRequest) returns (stream MatchStatusUpdate);
}

message CreateMatchRequest {
    string player_id = 1;
    string game_mode = 2;
    map<string, string> attributes = 3;
    int32 max_wait_seconds = 4;
}

message MatchResponse {
    bool success = 1;
    string ticket_id = 2;
    string error_message = 3;
}

message CancelMatchRequest {
    string ticket_id = 1;
    string player_id = 2;
}

message CancelResponse {
    bool success = 1;
    string error_message = 2;
}

message GetMatchStatusRequest {
    string ticket_id = 1;
}

message MatchStatusResponse {
    string status = 1; // PENDING, MATCHING, MATCHED, TIMEOUT, CANCELLED
    string match_id = 2;
    string server_address = 3;
    int32 server_port = 4;
    repeated PlayerInfo players = 5;
}

message PlayerInfo {
    string player_id = 1;
    string display_name = 2;
    int32 team_id = 3;
}

message StreamMatchStatusRequest {
    string ticket_id = 1;
}

message MatchStatusUpdate {
    string status = 1;
    int32 players_found = 2;
    int32 players_needed = 3;
    int32 estimated_wait_seconds = 4;
}
```

**UE5 gRPC客户端封装：**

```csharp
// GrpcClient.h
#pragma once

#include "CoreMinimal.h"
#include "grpcpp/grpcpp.h"
#include "match_service.grpc.pb.h"

#include "GrpcClient.generated.h"

USTRUCT(BlueprintType)
struct FMatchTicket
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString TicketId;

    UPROPERTY(BlueprintReadOnly)
    FString Status;

    UPROPERTY(BlueprintReadOnly)
    FString MatchId;

    UPROPERTY(BlueprintReadOnly)
    FString ServerAddress;

    UPROPERTY(BlueprintReadOnly)
    int32 ServerPort = 0;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnMatchCreated, const FMatchTicket&, Ticket);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnMatchStatusUpdate, const FString&, Status);

UCLASS()
class GRPCTUTORIAL_API UGrpcMatchClient : public UObject
{
    GENERATED_BODY()

public:
    void Initialize(const FString& ServerAddress);

    // 创建匹配请求
    UFUNCTION(BlueprintCallable, Category = "gRPC|Match")
    void CreateMatchRequest(const FString& PlayerId, const FString& GameMode, int32 MaxWaitSeconds);

    // 取消匹配
    UFUNCTION(BlueprintCallable, Category = "gRPC|Match")
    void CancelMatchRequest(const FString& TicketId);

    // 获取匹配状态
    UFUNCTION(BlueprintCallable, Category = "gRPC|Match")
    void GetMatchStatus(const FString& TicketId);

    // 开始流式监听
    UFUNCTION(BlueprintCallable, Category = "gRPC|Match")
    void StartStatusStream(const FString& TicketId);

    // 停止流式监听
    UFUNCTION(BlueprintCallable, Category = "gRPC|Match")
    void StopStatusStream();

    UPROPERTY(BlueprintAssignable, Category = "gRPC|Match")
    FOnMatchCreated OnMatchCreated;

    UPROPERTY(BlueprintAssignable, Category = "gRPC|Match")
    FOnMatchStatusUpdate OnMatchStatusUpdate;

private:
    std::unique_ptr<game::match::MatchService::Stub> Stub;
    std::unique_ptr<grpc::ClientContext> StreamContext;
    std::unique_ptr<grpc::ClientReader<game::match::MatchStatusUpdate>> StreamReader;
    bool bIsStreaming = false;

    void CreateMatchRequestAsync(const FString& PlayerId, const FString& GameMode, int32 MaxWaitSeconds);
    void StreamStatusAsync(const FString& TicketId);
};
```

```csharp
// GrpcClient.cpp
#include "GrpcClient.h"
#include "Async/Async.h"

void UGrpcMatchClient::Initialize(const FString& ServerAddress)
{
    // 创建gRPC通道
    grpc::ChannelArguments Args;
    Args.SetInt(GRPC_ARG_KEEPALIVE_TIME_MS, 10000);
    Args.SetInt(GRPC_ARG_KEEPALIVE_TIMEOUT_MS, 5000);
    Args.SetInt(GRPC_ARG_KEEPALIVE_PERMIT_WITHOUT_CALLS, 1);

    auto Channel = grpc::CreateCustomChannel(
        TCHAR_TO_UTF8(*ServerAddress),
        grpc::InsecureChannelCredentials(),
        Args
    );

    Stub = game::match::MatchService::NewStub(Channel);
}

void UGrpcMatchClient::CreateMatchRequest(const FString& PlayerId, const FString& GameMode, int32 MaxWaitSeconds)
{
    AsyncTask(ENamedThreads::AnyThread, [this, PlayerId, GameMode, MaxWaitSeconds]()
    {
        CreateMatchRequestAsync(PlayerId, GameMode, MaxWaitSeconds);
    });
}

void UGrpcMatchClient::CreateMatchRequestAsync(const FString& PlayerId, const FString& GameMode, int32 MaxWaitSeconds)
{
    game::match::CreateMatchRequest Request;
    Request.set_player_id(TCHAR_TO_UTF8(*PlayerId));
    Request.set_game_mode(TCHAR_TO_UTF8(*GameMode));
    Request.set_max_wait_seconds(MaxWaitSeconds);

    grpc::ClientContext Context;
    game::match::MatchResponse Response;

    grpc::Status Status = Stub->CreateMatchRequest(&Context, Request, &Response);

    AsyncTask(ENamedThreads::GameThread, [this, Status, Response]()
    {
        FMatchTicket Ticket;
        if (Status.ok() && Response.success())
        {
            Ticket.TicketId = UTF8_TO_TCHAR(Response.ticket_id().c_str());
            Ticket.Status = TEXT("PENDING");
            OnMatchCreated.Broadcast(Ticket);
        }
        else
        {
            Ticket.Status = TEXT("ERROR");
            OnMatchCreated.Broadcast(Ticket);
        }
    });
}

void UGrpcMatchClient::StartStatusStream(const FString& TicketId)
{
    if (bIsStreaming)
    {
        return;
    }

    bIsStreaming = true;
    StreamContext = std::make_unique<grpc::ClientContext>();

    game::match::StreamMatchStatusRequest Request;
    Request.set_ticket_id(TCHAR_TO_UTF8(*TicketId));

    StreamReader = Stub->StreamMatchStatus(StreamContext.get(), Request);

    AsyncTask(ENamedThreads::AnyThread, [this]()
    {
        StreamStatusAsync(TicketId);
    });
}

void UGrpcMatchClient::StreamStatusAsync(const FString& TicketId)
{
    game::match::MatchStatusUpdate Update;

    while (bIsStreaming && StreamReader->Read(&Update))
    {
        FString Status = UTF8_TO_TCHAR(Update.status().c_str());

        AsyncTask(ENamedThreads::GameThread, [this, Status]()
        {
            OnMatchStatusUpdate.Broadcast(Status);
        });
    }
}

void UGrpcMatchClient::StopStatusStream()
{
    bIsStreaming = false;
    if (StreamContext)
    {
        StreamContext->TryCancel();
        StreamContext.reset();
    }
    if (StreamReader)
    {
        StreamReader.reset();
    }
}
```

---

## 24.3 API 网关设计

### 24.3.1 网关架构

```csharp
// APIGateway.h
#pragma once

#include "CoreMinimal.h"
#include "MicroserviceClient.h"

#include "APIGateway.generated.h"

USTRUCT(BlueprintType)
struct FServiceEndpoint
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere)
    FString ServiceName;

    UPROPERTY(EditAnywhere)
    FString BaseUrl;

    UPROPERTY(EditAnywhere)
    FString HealthCheckPath = "/health";

    UPROPERTY(EditAnywhere)
    int32 TimeoutSeconds = 30;

    UPROPERTY(EditAnywhere)
    int32 RetryCount = 3;
};

UCLASS()
class MICROSERVICES_API UAPIGateway : public UObject
{
    GENERATED_BODY()

public:
    // 注册服务
    UFUNCTION(BlueprintCallable, Category = "API Gateway")
    void RegisterService(const FString& ServiceName, const FString& BaseUrl);

    // 路由请求
    UFUNCTION(BlueprintCallable, Category = "API Gateway")
    void RouteRequest(const FString& ServiceName, const FString& Endpoint, const FString& Method, const FString& Body, FOnServiceResponse Callback);

    // 健康检查
    UFUNCTION(BlueprintCallable, Category = "API Gateway")
    void HealthCheck(const FString& ServiceName);

    // 负载均衡
    FString GetNextEndpoint(const FString& ServiceName);

    // 设置全局认证
    void SetGlobalAuthToken(const FString& Token);

private:
    TMap<FString, TArray<FString>> ServiceEndpoints;
    TMap<FString, int32> RoundRobinIndex;
    FString GlobalAuthToken;

    TMap<FString, UMicroserviceClient*> Clients;
};
```

### 24.3.2 Kong网关配置

```yaml
# Kong配置 - decK格式
_format_version: "3.0"

services:
  # 认证服务
  - name: auth-service
    url: http://auth-service:8001
    routes:
      - name: auth-route
        paths:
          - /api/v1/auth
    plugins:
      - name: rate-limiting
        config:
          minute: 100
          policy: local
      - name: jwt
        config:
          secret_is_base64: false

  # 匹配服务
  - name: match-service
    url: http://match-service:8002
    routes:
      - name: match-route
        paths:
          - /api/v1/match
    plugins:
      - name: rate-limiting
        config:
          minute: 50
          policy: local
      - name: request-transformer
        config:
          add:
            headers:
              - X-Forwarded-For:$(remote_addr)

  # 游戏服务
  - name: game-service
    url: http://game-service:8003
    routes:
      - name: game-route
        paths:
          - /api/v1/game
    plugins:
      - name: rate-limiting
        config:
          minute: 200
          policy: local

  # 支付服务
  - name: payment-service
    url: http://payment-service:8004
    routes:
      - name: payment-route
        paths:
          - /api/v1/payment
    plugins:
      - name: rate-limiting
        config:
          minute: 20
          policy: local
      - name: bot-detection
        config:
          deny:
            - ^/api/v1/payment/batch

# 全局插件
plugins:
  - name: cors
    config:
      origins:
        - "*"
      methods:
        - GET
        - POST
        - PUT
        - DELETE
      headers:
        - Accept
        - Accept-Version
        - Content-Length
        - Content-MD5
        - Content-Type
        - Date
        - Authorization
      exposed_headers:
        - X-Auth-Token
      credentials: true
      max_age: 3600

  - name: request-size-limiting
    config:
      allowed_payload_size: 10  # MB

  - name: response-transformer
    config:
      add:
        headers:
          - X-Response-Time:$(time())
```

---

## 24.4 服务发现与注册

### 24.4.1 Consul服务发现

```csharp
// ConsulClient.h
#pragma once

#include "CoreMinimal.h"
#include "HttpModule.h"

#include "ConsulClient.generated.h"

USTRUCT(BlueprintType)
struct FServiceInstance
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString ServiceId;

    UPROPERTY(BlueprintReadOnly)
    FString ServiceName;

    UPROPERTY(BlueprintReadOnly)
    FString Address;

    UPROPERTY(BlueprintReadOnly)
    int32 Port = 0;

    UPROPERTY(BlueprintReadOnly)
    TMap<FString, FString> Tags;
};

USTRUCT(BlueprintType)
struct FHealthCheckResult
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString Status; // passing, warning, critical

    UPROPERTY(BlueprintReadOnly)
    FString Output;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnServiceDiscovered, const TArray<FServiceInstance>&, Instances);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnHealthCheckComplete, const FHealthCheckResult&, Result);

UCLASS()
class CONSUL_API UConsulClient : public UObject
{
    GENERATED_BODY()

public:
    // 初始化
    void Initialize(const FString& ConsulAddress);

    // 注册服务
    UFUNCTION(BlueprintCallable, Category = "Consul")
    void RegisterService(const FString& ServiceName, const FString& ServiceId, const FString& Address, int32 Port, const TMap<FString, FString>& Tags);

    // 注销服务
    UFUNCTION(BlueprintCallable, Category = "Consul")
    void DeregisterService(const FString& ServiceId);

    // 发现服务
    UFUNCTION(BlueprintCallable, Category = "Consul")
    void DiscoverService(const FString& ServiceName);

    // 健康检查
    UFUNCTION(BlueprintCallable, Category = "Consul")
    void HealthCheck(const FString& ServiceName);

    // 监听服务变化
    UFUNCTION(BlueprintCallable, Category = "Consul")
    void WatchService(const FString& ServiceName);

    UPROPERTY(BlueprintAssignable, Category = "Consul")
    FOnServiceDiscovered OnServiceDiscovered;

    UPROPERTY(BlueprintAssignable, Category = "Consul")
    FOnHealthCheckComplete OnHealthCheckComplete;

private:
    FString ConsulUrl;
    FTimerHandle WatchTimerHandle;

    void MakeConsulRequest(const FString& Path, const FString& Method, const FString& Body, TFunction<void(bool, FString)> Callback);
};
```

```csharp
// ConsulClient.cpp
#include "ConsulClient.h"
#include "JsonUtilities.h"

void UConsulClient::Initialize(const FString& ConsulAddress)
{
    ConsulUrl = ConsulAddress;
    if (!ConsulUrl.EndsWith(TEXT("/")))
    {
        ConsulUrl += TEXT("/");
    }
}

void UConsulClient::RegisterService(const FString& ServiceName, const FString& ServiceId, const FString& Address, int32 Port, const TMap<FString, FString>& Tags)
{
    TSharedPtr<FJsonObject> JsonRequest = MakeShareable(new FJsonObject);
    JsonRequest->SetStringField("Name", ServiceName);
    JsonRequest->SetStringField("ID", ServiceId);
    JsonRequest->SetStringField("Address", Address);
    JsonRequest->SetNumberField("Port", Port);

    // 添加标签
    TArray<TSharedPtr<FJsonValue>> TagsArray;
    for (const auto& Tag : Tags)
    {
        TagsArray.Add(MakeShareable(new FJsonValueString(Tag.Key + ":" + Tag.Value)));
    }
    JsonRequest->SetArrayField("Tags", TagsArray);

    // 添加健康检查
    TSharedPtr<FJsonObject> Check = MakeShareable(new FJsonObject);
    Check->SetStringField("HTTP", FString::Printf(TEXT("http://%s:%d/health"), *Address, Port));
    Check->SetStringField("Interval", "10s");
    JsonRequest->SetObjectField("Check", Check);

    FString RequestBody;
    TJsonWriterFactory<>::Create(&RequestBody)->WriteObject(JsonRequest.ToSharedRef());

    MakeConsulRequest(TEXT("v1/agent/service/register"), TEXT("PUT"), RequestBody, [](bool bSuccess, FString Response)
    {
        if (bSuccess)
        {
            UE_LOG(LogTemp, Log, TEXT("Service registered successfully"));
        }
    });
}

void UConsulClient::DeregisterService(const FString& ServiceId)
{
    MakeConsulRequest(FString::Printf(TEXT("v1/agent/service/deregister/%s"), *ServiceId), TEXT("PUT"), TEXT(""), [](bool bSuccess, FString Response)
    {
        if (bSuccess)
        {
            UE_LOG(LogTemp, Log, TEXT("Service deregistered successfully"));
        }
    });
}

void UConsulClient::DiscoverService(const FString& ServiceName)
{
    MakeConsulRequest(FString::Printf(TEXT("v1/catalog/service/%s"), *ServiceName), TEXT("GET"), TEXT(""),
        [this](bool bSuccess, FString Response)
        {
            if (!bSuccess)
            {
                OnServiceDiscovered.Broadcast(TArray<FServiceInstance>());
                return;
            }

            TArray<FServiceInstance> Instances;
            TArray<TSharedPtr<FJsonValue>> JsonArray;
            TJsonReaderFactory<TCHAR>::Create(Response) >> JsonArray;

            for (const auto& Value : JsonArray)
            {
                TSharedPtr<FJsonObject> Obj = Value->AsObject();
                FServiceInstance Instance;
                Instance.ServiceId = Obj->GetStringField("ServiceID");
                Instance.ServiceName = Obj->GetStringField("ServiceName");
                Instance.Address = Obj->GetStringField("ServiceAddress");
                Instance.Port = Obj->GetNumberField("ServicePort");

                Instances.Add(Instance);
            }

            OnServiceDiscovered.Broadcast(Instances);
        });
}

void UConsulClient::MakeConsulRequest(const FString& Path, const FString& Method, const FString& Body, TFunction<void(bool, FString)> Callback)
{
    TSharedRef<IHttpRequest> Request = FHttpModule::Get().CreateRequest();
    Request->SetURL(ConsulUrl + Path);
    Request->SetVerb(Method);

    if (!Body.IsEmpty())
    {
        Request->SetHeader(TEXT("Content-Type"), TEXT("application/json"));
        Request->SetContentAsString(Body);
    }

    Request->OnProcessRequestComplete().BindLambda([Callback](FHttpRequestPtr Req, FHttpResponsePtr Resp, bool bSuccess)
    {
        if (bSuccess && Resp.IsValid())
        {
            Callback(true, Resp->GetContentAsString());
        }
        else
        {
            Callback(false, TEXT(""));
        }
    });

    Request->ProcessRequest();
}
```

---

## 24.5 分布式配置管理

### 24.5.1 配置中心实现

```csharp
// ConfigClient.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "ConfigClient.generated.h"

USTRUCT(BlueprintType)
struct FServiceConfig
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadWrite)
    FString Key;

    UPROPERTY(BlueprintReadWrite)
    FString Value;

    UPROPERTY(BlueprintReadWrite)
    FString Version;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnConfigUpdated);

UCLASS()
class CONFIGCENTER_API UConfigClient : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 连接配置中心
    UFUNCTION(BlueprintCallable, Category = "Config")
    void Connect(const FString& ServerUrl, const FString& ServiceName, const FString& Environment);

    // 获取配置
    UFUNCTION(BlueprintCallable, Category = "Config")
    FString GetConfig(const FString& Key, const FString& DefaultValue = TEXT(""));

    // 获取所有配置
    UFUNCTION(BlueprintCallable, Category = "Config")
    TMap<FString, FString> GetAllConfigs();

    // 刷新配置
    UFUNCTION(BlueprintCallable, Category = "Config")
    void RefreshConfigs();

    // 监听配置变化
    UFUNCTION(BlueprintCallable, Category = "Config")
    void WatchConfig(const FString& Key);

    UPROPERTY(BlueprintAssignable, Category = "Config")
    FOnConfigUpdated OnConfigUpdated;

private:
    FString ConfigServerUrl;
    FString CurrentService;
    FString CurrentEnvironment;
    TMap<FString, FServiceConfig> CachedConfigs;
    FTimerHandle RefreshTimerHandle;

    void FetchConfigs();
    void StartPolling(float IntervalSeconds = 30.0f);
};
```

```csharp
// ConfigClient.cpp
#include "ConfigClient.h"
#include "HttpModule.h"
#include "JsonUtilities.h"

void UConfigClient::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
}

void UConfigClient::Connect(const FString& ServerUrl, const FString& ServiceName, const FString& Environment)
{
    ConfigServerUrl = ServerUrl;
    CurrentService = ServiceName;
    CurrentEnvironment = Environment;

    FetchConfigs();
    StartPolling();
}

void UConfigClient::FetchConfigs()
{
    TSharedRef<IHttpRequest> Request = FHttpModule::Get().CreateRequest();

    FString Url = FString::Printf(TEXT("%s/configs/%s/%s"),
        *ConfigServerUrl, *CurrentService, *CurrentEnvironment);

    Request->SetURL(Url);
    Request->SetVerb(TEXT("GET"));

    Request->OnProcessRequestComplete().BindLambda([this](FHttpRequestPtr Req, FHttpResponsePtr Resp, bool bSuccess)
    {
        if (bSuccess && Resp.IsValid())
        {
            TArray<TSharedPtr<FJsonValue>> JsonArray;
            TJsonReaderFactory<TCHAR>::Create(Resp->GetContentAsString()) >> JsonArray;

            bool bConfigChanged = false;

            for (const auto& Value : JsonArray)
            {
                TSharedPtr<FJsonObject> Obj = Value->AsObject();
                FString Key = Obj->GetStringField("key");
                FString NewValue = Obj->GetStringField("value");
                FString Version = Obj->GetStringField("version");

                if (!CachedConfigs.Contains(Key) || CachedConfigs[Key].Version != Version)
                {
                    FServiceConfig& Config = CachedConfigs.FindOrAdd(Key);
                    Config.Key = Key;
                    Config.Value = NewValue;
                    Config.Version = Version;
                    bConfigChanged = true;
                }
            }

            if (bConfigChanged)
            {
                OnConfigUpdated.Broadcast();
            }
        }
    });

    Request->ProcessRequest();
}

FString UConfigClient::GetConfig(const FString& Key, const FString& DefaultValue)
{
    if (CachedConfigs.Contains(Key))
    {
        return CachedConfigs[Key].Value;
    }
    return DefaultValue;
}

TMap<FString, FString> UConfigClient::GetAllConfigs()
{
    TMap<FString, FString> Result;
    for (const auto& Pair : CachedConfigs)
    {
        Result.Add(Pair.Key, Pair.Value.Value);
    }
    return Result;
}

void UConfigClient::StartPolling(float IntervalSeconds)
{
    GetWorld()->GetTimerManager().SetTimer(
        RefreshTimerHandle,
        this,
        &UConfigClient::FetchConfigs,
        IntervalSeconds,
        true
    );
}

void UConfigClient::RefreshConfigs()
{
    FetchConfigs();
}
```

---

## 24.6 分布式追踪

### 24.6.1 OpenTelemetry集成

```csharp
// TelemetryManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "TelemetryManager.generated.h"

USTRUCT(BlueprintType)
struct FSpanContext
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString TraceId;

    UPROPERTY(BlueprintReadOnly)
    FString SpanId;

    UPROPERTY(BlueprintReadOnly)
    FString ParentSpanId;
};

UCLASS()
class TELEMETRY_API UTelemetryManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 开始追踪Span
    UFUNCTION(BlueprintCallable, Category = "Telemetry")
    FString StartSpan(const FString& OperationName, const FString& ParentSpanId = TEXT(""));

    // 结束Span
    UFUNCTION(BlueprintCallable, Category = "Telemetry")
    void EndSpan(const FString& SpanId);

    // 添加Span事件
    UFUNCTION(BlueprintCallable, Category = "Telemetry")
    void AddSpanEvent(const FString& SpanId, const FString& EventName, const TMap<FString, FString>& Attributes);

    // 添加Span属性
    UFUNCTION(BlueprintCallable, Category = "Telemetry")
    void SetSpanAttribute(const FString& SpanId, const FString& Key, const FString& Value);

    // 记录异常
    UFUNCTION(BlueprintCallable, Category = "Telemetry")
    void RecordException(const FString& SpanId, const FString& ExceptionType, const FString& Message, const FString& StackTrace);

    // 设置服务信息
    void SetServiceInfo(const FString& ServiceName, const FString& Version);

    // 获取当前追踪上下文
    FSpanContext GetCurrentContext();

    // 注入追踪头到HTTP请求
    void InjectTraceHeaders(TMap<FString, FString>& Headers);

private:
    FString ServiceName;
    FString ServiceVersion;
    FString CurrentTraceId;

    struct FSpanData
    {
        FString SpanId;
        FString ParentSpanId;
        FString OperationName;
        double StartTime;
        double EndTime;
        TMap<FString, FString> Attributes;
        TArray<TPair<FString, TMap<FString, FString>>> Events;
    };

    TMap<FString, FSpanData> ActiveSpans;

    FString GenerateTraceId();
    FString GenerateSpanId();
    void ExportSpan(const FSpanData& Span);
};
```

```csharp
// TelemetryManager.cpp
#include "TelemetryManager.h"
#include "HttpModule.h"
#include "JsonUtilities.h"
#include "Misc/DateTime.h"

void UTelemetryManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
}

void UTelemetryManager::SetServiceInfo(const FString& InServiceName, const FString& InVersion)
{
    ServiceName = InServiceName;
    ServiceVersion = InVersion;
}

FString UTelemetryManager::GenerateTraceId()
{
    FGuid Guid = FGuid::NewGuid();
    return FString::Printf(TEXT("%016llx%016llx"), Guid.A, Guid.B);
}

FString UTelemetryManager::GenerateSpanId()
{
    FGuid Guid = FGuid::NewGuid();
    return FString::Printf(TEXT("%016llx"), Guid.A);
}

FString UTelemetryManager::StartSpan(const FString& OperationName, const FString& ParentSpanId)
{
    FString SpanId = GenerateSpanId();

    // 如果没有父Span，创建新的Trace
    if (CurrentTraceId.IsEmpty())
    {
        CurrentTraceId = GenerateTraceId();
    }

    FSpanData Span;
    Span.SpanId = SpanId;
    Span.ParentSpanId = ParentSpanId;
    Span.OperationName = OperationName;
    Span.StartTime = FPlatformTime::Seconds();

    // 添加默认属性
    Span.Attributes.Add("service.name", ServiceName);
    Span.Attributes.Add("service.version", ServiceVersion);

    ActiveSpans.Add(SpanId, Span);

    return SpanId;
}

void UTelemetryManager::EndSpan(const FString& SpanId)
{
    if (FSpanData* Span = ActiveSpans.Find(SpanId))
    {
        Span->EndTime = FPlatformTime::Seconds();
        ExportSpan(*Span);
        ActiveSpans.Remove(SpanId);
    }
}

void UTelemetryManager::AddSpanEvent(const FString& SpanId, const FString& EventName, const TMap<FString, FString>& Attributes)
{
    if (FSpanData* Span = ActiveSpans.Find(SpanId))
    {
        Span->Events.Add(TPair<FString, TMap<FString, FString>>(EventName, Attributes));
    }
}

void UTelemetryManager::SetSpanAttribute(const FString& SpanId, const FString& Key, const FString& Value)
{
    if (FSpanData* Span = ActiveSpans.Find(SpanId))
    {
        Span->Attributes.Add(Key, Value);
    }
}

void UTelemetryManager::RecordException(const FString& SpanId, const FString& ExceptionType, const FString& Message, const FString& StackTrace)
{
    if (FSpanData* Span = ActiveSpans.Find(SpanId))
    {
        TMap<FString, FString> ExceptionAttrs;
        ExceptionAttrs.Add("exception.type", ExceptionType);
        ExceptionAttrs.Add("exception.message", Message);
        ExceptionAttrs.Add("exception.stacktrace", StackTrace);

        Span->Events.Add(TPair<FString, TMap<FString, FString>>("exception", ExceptionAttrs));
        Span->Attributes.Add("error", "true");
    }
}

void UTelemetryManager::InjectTraceHeaders(TMap<FString, FString>& Headers)
{
    // W3C Trace Context格式
    Headers.Add("traceparent", FString::Printf(TEXT("00-%s-%s-01"), *CurrentTraceId, *ActiveSpans.begin()->Key));
}

void UTelemetryManager::ExportSpan(const FSpanData& Span)
{
    TSharedPtr<FJsonObject> JsonSpan = MakeShareable(new FJsonObject);

    JsonSpan->SetStringField("traceId", CurrentTraceId);
    JsonSpan->SetStringField("spanId", Span.SpanId);
    JsonSpan->SetStringField("parentSpanId", Span.ParentSpanId);
    JsonSpan->SetStringField("operationName", Span.OperationName);
    JsonSpan->SetNumberField("startTime", Span.StartTime * 1000000); // 转换为纳秒
    JsonSpan->SetNumberField("endTime", Span.EndTime * 1000000);

    // 添加属性
    TSharedPtr<FJsonObject> AttrsObj = MakeShareable(new FJsonObject);
    for (const auto& Attr : Span.Attributes)
    {
        AttrsObj->SetStringField(Attr.Key, Attr.Value);
    }
    JsonSpan->SetObjectField("attributes", AttrsObj);

    // 序列化并发送
    FString JsonString;
    TJsonWriterFactory<>::Create(&JsonString)->WriteObject(JsonSpan.ToSharedRef());

    // 发送到收集器（如Jaeger、Zipkin）
    // 这里简化为日志输出
    UE_LOG(LogTemp, Verbose, TEXT("Telemetry Span: %s"), *JsonString);
}
```

---

## 24.7 实践任务

### 任务1：设计微服务架构

为一个多人游戏设计微服务架构：
- 用户服务
- 匹配服务
- 游戏房间服务
- 排行榜服务

### 任务2：实现服务发现

使用Consul实现：
- 服务注册
- 健康检查
- 服务发现

### 任务3：配置分布式追踪

集成OpenTelemetry：
- 创建追踪Span
- 记录关键事件
- 可视化追踪链路

---

## 24.8 总结

本课学习了：
- 微服务架构设计原则
- REST和gRPC服务间通信
- API网关设计与配置
- Consul服务发现机制
- 分布式配置管理
- OpenTelemetry分布式追踪

**下一课预告**：实时通信服务 - WebSocket集成与Push通知系统。

---

*课程版本：1.0*
*最后更新：2026-04-10*
