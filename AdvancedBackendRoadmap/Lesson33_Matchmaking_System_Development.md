# 第33课：匹配系统开发

> 学习目标：实现智能匹配算法、匹配队列管理和匹配状态追踪功能。

---

## 33.1 匹配系统架构

### 33.1.1 系统设计

```
匹配系统架构:

┌─────────────────────────────────────────────────────┐
│                    客户端                            │
│  ┌─────────────┐    ┌─────────────┐                │
│  │ 匹配请求UI  │    │ 匹配状态UI  │                │
│  └─────────────┘    └─────────────┘                │
└─────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│                匹配服务 (Match Service)              │
│  ┌─────────────┐    ┌─────────────┐                │
│  │ 匹配队列    │    │ 匹配算法    │                │
│  │ Manager     │    │ Matcher     │                │
│  └─────────────┘    └─────────────┘                │
│  ┌─────────────┐    ┌─────────────┐                │
│  │ 服务器分配  │    │ 匹配票据    │                │
│  │ Allocator   │    │ Tracker     │                │
│  └─────────────┘    └─────────────┘                │
└─────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│                   Redis缓存                         │
│  ┌─────────────┐    ┌─────────────┐                │
│  │ 匹配队列    │    │ 票据状态    │                │
│  └─────────────┘    └─────────────┘                │
└─────────────────────────────────────────────────────┘
```

---

## 33.2 匹配客户端子系统

### 33.2.1 匹配子系统实现

```csharp
// MatchmakingSubsystem.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "MatchmakingSubsystem.generated.h"

UENUM(BlueprintType)
enum class EMatchmakingMode : uint8
{
    Casual_1v1,
    Casual_2v2,
    Casual_3v3,
    Ranked_1v1,
    Ranked_2v2,
    Ranked_3v3,
    Custom
};

UENUM(BlueprintType)
enum class EMatchmakingStatus : uint8
{
    None,
    Searching,
    FoundMatch,
    Connecting,
    InMatch,
    Cancelled,
    Error
};

USTRUCT(BlueprintType)
struct FMatchmakingConfig
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    EMatchmakingMode Mode = EMatchmakingMode::Casual_1v1;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString Region = "auto";

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TMap<FString, FString> Preferences;
};

USTRUCT(BlueprintType)
struct FMatchmakingTicket
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString TicketId;

    UPROPERTY(BlueprintReadOnly)
    EMatchmakingStatus Status = EMatchmakingStatus::None;

    UPROPERTY(BlueprintReadOnly)
    FDateTime CreatedAt;

    UPROPERTY(BlueprintReadOnly)
    int32 EstimatedWaitSeconds = 0;

    UPROPERTY(BlueprintReadOnly)
    FString ErrorMessage;
};

USTRUCT(BlueprintType)
struct FMatchInfo
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString MatchId;

    UPROPERTY(BlueprintReadOnly)
    FString ServerAddress;

    UPROPERTY(BlueprintReadOnly)
    int32 ServerPort = 7777;

    UPROPERTY(BlueprintReadOnly)
    FString SessionToken;

    UPROPERTY(BlueprintReadOnly)
    TArray<FString> TeamMates;

    UPROPERTY(BlueprintReadOnly)
    TArray<FString> Opponents;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnTicketCreated, const FMatchmakingTicket&, Ticket);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnStatusChanged, EMatchmakingStatus, NewStatus);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnMatchFound, const FMatchInfo&, Match);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnMatchmakingError, const FString&, ErrorMessage);

UCLASS()
class ARENAMATCHMAKING_API UMatchmakingSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 开始匹配
    UFUNCTION(BlueprintCallable, Category = "Matchmaking")
    void StartMatchmaking(const FMatchmakingConfig& Config);

    // 取消匹配
    UFUNCTION(BlueprintCallable, Category = "Matchmaking")
    void CancelMatchmaking();

    // 接受匹配
    UFUNCTION(BlueprintCallable, Category = "Matchmaking")
    void AcceptMatch();

    // 拒绝匹配
    UFUNCTION(BlueprintCallable, Category = "Matchmaking")
    void DeclineMatch();

    // 状态查询
    UFUNCTION(BlueprintPure, Category = "Matchmaking")
    EMatchmakingStatus GetStatus() const { return CurrentStatus; }

    UFUNCTION(BlueprintPure, Category = "Matchmaking")
    FMatchmakingTicket GetCurrentTicket() const { return CurrentTicket; }

    UFUNCTION(BlueprintPure, Category = "Matchmaking")
    bool IsSearching() const { return CurrentStatus == EMatchmakingStatus::Searching; }

    UFUNCTION(BlueprintPure, Category = "Matchmaking")
    float GetElapsedWaitTime() const;

    // 配置
    UPROPERTY(EditDefaultsOnly, Category = "Matchmaking")
    float PollingInterval = 2.0f;

    UPROPERTY(EditDefaultsOnly, Category = "Matchmaking")
    float MaxWaitTime = 600.0f;

    // 事件
    UPROPERTY(BlueprintAssignable, Category = "Matchmaking")
    FOnTicketCreated OnTicketCreated;

    UPROPERTY(BlueprintAssignable, Category = "Matchmaking")
    FOnStatusChanged OnStatusChanged;

    UPROPERTY(BlueprintAssignable, Category = "Matchmaking")
    FOnMatchFound OnMatchFound;

    UPROPERTY(BlueprintAssignable, Category = "Matchmaking")
    FOnMatchmakingError OnError;

private:
    FString ApiBaseUrl;
    EMatchmakingStatus CurrentStatus = EMatchmakingStatus::None;
    FMatchmakingTicket CurrentTicket;
    FMatchInfo CurrentMatch;
    FMatchmakingConfig CurrentConfig;
    FDateTime SearchStartTime;

    FTimerHandle PollingTimerHandle;
    FTimerHandle TimeoutTimerHandle;

    void CreateTicket();
    void StartPolling();
    void StopPolling();
    void PollTicketStatus();
    void HandleTicketResponse(const FString& ResponseJson);
    void HandleMatchFoundResponse(const FString& ResponseJson);
    void SetStatus(EMatchmakingStatus NewStatus);
    void HandleTimeout();
    void HandleError(const FString& Message);
};
```

```csharp
// MatchmakingSubsystem.cpp
#include "MatchmakingSubsystem.h"
#include "JsonUtilities.h"
#include "TimerManager.h"
#include "HttpModule.h"

void UMatchmakingSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
    ApiBaseUrl = "http://localhost:8080/api";
}

void UMatchmakingSubsystem::Deinitialize()
{
    StopPolling();
    Super::Deinitialize();
}

void UMatchmakingSubsystem::StartMatchmaking(const FMatchmakingConfig& Config)
{
    if (CurrentStatus == EMatchmakingStatus::Searching)
    {
        OnError.Broadcast(TEXT("Already searching for match"));
        return;
    }

    CurrentConfig = Config;
    SearchStartTime = FDateTime::UtcNow();

    SetStatus(EMatchmakingStatus::Searching);
    CreateTicket();
}

void UMatchmakingSubsystem::CreateTicket()
{
    TSharedPtr<FJsonObject> JsonRequest = MakeShareable(new FJsonObject);

    // 转换模式到字符串
    FString ModeStr;
    switch (CurrentConfig.Mode)
    {
        case EMatchmakingMode::Casual_1v1: ModeStr = TEXT("casual_1v1"); break;
        case EMatchmakingMode::Casual_2v2: ModeStr = TEXT("casual_2v2"); break;
        case EMatchmakingMode::Casual_3v3: ModeStr = TEXT("casual_3v3"); break;
        case EMatchmakingMode::Ranked_1v1: ModeStr = TEXT("ranked_1v1"); break;
        case EMatchmakingMode::Ranked_2v2: ModeStr = TEXT("ranked_2v2"); break;
        case EMatchmakingMode::Ranked_3v3: ModeStr = TEXT("ranked_3v3"); break;
        default: ModeStr = TEXT("casual_1v1");
    }

    JsonRequest->SetStringField("mode", ModeStr);
    JsonRequest->SetStringField("region", CurrentConfig.Region);

    // 添加偏好
    TSharedPtr<FJsonObject> PreferencesObj = MakeShareable(new FJsonObject);
    for (const auto& Pair : CurrentConfig.Preferences)
    {
        PreferencesObj->SetStringField(Pair.Key, Pair.Value);
    }
    JsonRequest->SetObjectField("preferences", PreferencesObj);

    FString Body;
    TJsonWriterFactory<>::Create(&Body)->WriteObject(JsonRequest.ToSharedRef());

    TSharedRef<IHttpRequest> Request = FHttpModule::Get().CreateRequest();
    Request->SetURL(ApiBaseUrl + "/matchmaking/tickets");
    Request->SetVerb("POST");
    Request->SetHeader("Content-Type", "application/json");
    Request->SetHeader("Authorization", "Bearer " + GetAccessToken());
    Request->SetContentAsString(Body);

    Request->OnProcessRequestComplete().BindLambda([this](FHttpRequestPtr Req, FHttpResponsePtr Resp, bool bSuccess)
    {
        if (bSuccess && Resp.IsValid() && Resp->GetResponseCode() == 201)
        {
            HandleTicketResponse(Resp->GetContentAsString());
        }
        else
        {
            HandleError(TEXT("Failed to create matchmaking ticket"));
        }
    });

    Request->ProcessRequest();
}

void UMatchmakingSubsystem::HandleTicketResponse(const FString& ResponseJson)
{
    TSharedPtr<FJsonObject> JsonObject;
    TJsonReaderFactory<TCHAR>::Create(ResponseJson) >> JsonObject;

    if (!JsonObject.IsValid())
    {
        HandleError(TEXT("Invalid ticket response"));
        return;
    }

    CurrentTicket.TicketId = JsonObject->GetStringField("ticketId");
    CurrentTicket.Status = EMatchmakingStatus::Searching;
    CurrentTicket.CreatedAt = FDateTime::UtcNow();
    CurrentTicket.EstimatedWaitSeconds = JsonObject->GetNumberField("estimatedWait");

    OnTicketCreated.Broadcast(CurrentTicket);

    // 开始轮询
    StartPolling();

    // 设置超时
    if (UWorld* World = GetWorld())
    {
        World->GetTimerManager().SetTimer(
            TimeoutTimerHandle,
            this,
            &UMatchmakingSubsystem::HandleTimeout,
            MaxWaitTime,
            false
        );
    }
}

void UMatchmakingSubsystem::StartPolling()
{
    if (UWorld* World = GetWorld())
    {
        World->GetTimerManager().SetTimer(
            PollingTimerHandle,
            this,
            &UMatchmakingSubsystem::PollTicketStatus,
            PollingInterval,
            true
        );
    }
}

void UMatchmakingSubsystem::StopPolling()
{
    if (UWorld* World = GetWorld())
    {
        World->GetTimerManager().ClearTimer(PollingTimerHandle);
        World->GetTimerManager().ClearTimer(TimeoutTimerHandle);
    }
}

void UMatchmakingSubsystem::PollTicketStatus()
{
    if (CurrentTicket.TicketId.IsEmpty())
    {
        return;
    }

    TSharedRef<IHttpRequest> Request = FHttpModule::Get().CreateRequest();
    Request->SetURL(ApiBaseUrl + "/matchmaking/tickets/" + CurrentTicket.TicketId);
    Request->SetVerb("GET");
    Request->SetHeader("Authorization", "Bearer " + GetAccessToken());

    Request->OnProcessRequestComplete().BindLambda([this](FHttpRequestPtr Req, FHttpResponsePtr Resp, bool bSuccess)
    {
        if (bSuccess && Resp.IsValid())
        {
            if (Resp->GetResponseCode() == 200)
            {
                FString ResponseJson = Resp->GetContentAsString();

                TSharedPtr<FJsonObject> JsonObject;
                TJsonReaderFactory<TCHAR>::Create(ResponseJson) >> JsonObject;

                if (JsonObject.IsValid())
                {
                    FString StatusStr = JsonObject->GetStringField("status");

                    if (StatusStr == "matched")
                    {
                        StopPolling();
                        HandleMatchFoundResponse(ResponseJson);
                    }
                    else if (StatusStr == "cancelled")
                    {
                        StopPolling();
                        SetStatus(EMatchmakingStatus::Cancelled);
                    }
                }
            }
        }
    });

    Request->ProcessRequest();
}

void UMatchmakingSubsystem::HandleMatchFoundResponse(const FString& ResponseJson)
{
    TSharedPtr<FJsonObject> JsonObject;
    TJsonReaderFactory<TCHAR>::Create(ResponseJson) >> JsonObject;

    if (!JsonObject.IsValid())
    {
        HandleError(TEXT("Invalid match response"));
        return;
    }

    // 解析匹配信息
    TSharedPtr<FJsonObject> MatchObj = JsonObject->GetObjectField("match");

    CurrentMatch.MatchId = MatchObj->GetStringField("matchId");
    CurrentMatch.ServerAddress = MatchObj->GetStringField("serverAddress");
    CurrentMatch.ServerPort = MatchObj->GetNumberField("serverPort");
    CurrentMatch.SessionToken = MatchObj->GetStringField("sessionToken");

    // 解析队伍
    const TArray<TSharedPtr<FJsonValue>>* TeamMates;
    if (MatchObj->TryGetArrayField("teamMates", TeamMates))
    {
        for (const auto& TeamMate : *TeamMates)
        {
            CurrentMatch.TeamMates.Add(TeamMate->AsString());
        }
    }

    const TArray<TSharedPtr<FJsonValue>>* Opponents;
    if (MatchObj->TryGetArrayField("opponents", Opponents))
    {
        for (const auto& Opponent : *Opponents)
        {
            CurrentMatch.Opponents.Add(Opponent->AsString());
        }
    }

    SetStatus(EMatchmakingStatus::FoundMatch);
    OnMatchFound.Broadcast(CurrentMatch);
}

void UMatchmakingSubsystem::CancelMatchmaking()
{
    if (CurrentTicket.TicketId.IsEmpty())
    {
        return;
    }

    StopPolling();

    TSharedRef<IHttpRequest> Request = FHttpModule::Get().CreateRequest();
    Request->SetURL(ApiBaseUrl + "/matchmaking/tickets/" + CurrentTicket.TicketId + "/cancel");
    Request->SetVerb("POST");
    Request->SetHeader("Authorization", "Bearer " + GetAccessToken());

    Request->OnProcessRequestComplete().BindLambda([this](FHttpRequestPtr Req, FHttpResponsePtr Resp, bool bSuccess)
    {
        CurrentTicket = FMatchmakingTicket();
        SetStatus(EMatchmakingStatus::Cancelled);
    });

    Request->ProcessRequest();
}

void UMatchmakingSubsystem::AcceptMatch()
{
    if (CurrentMatch.MatchId.IsEmpty())
    {
        return;
    }

    TSharedRef<IHttpRequest> Request = FHttpModule::Get().CreateRequest();
    Request->SetURL(ApiBaseUrl + "/matchmaking/matches/" + CurrentMatch.MatchId + "/accept");
    Request->SetVerb("POST");
    Request->SetHeader("Authorization", "Bearer " + GetAccessToken());

    Request->OnProcessRequestComplete().BindLambda([this](FHttpRequestPtr Req, FHttpResponsePtr Resp, bool bSuccess)
    {
        if (bSuccess && Resp.IsValid() && Resp->GetResponseCode() == 200)
        {
            SetStatus(EMatchmakingStatus::Connecting);
            // 开始连接服务器
            ConnectToMatchServer();
        }
        else
        {
            HandleError(TEXT("Failed to accept match"));
        }
    });

    Request->ProcessRequest();
}

void UMatchmakingSubsystem::DeclineMatch()
{
    if (CurrentMatch.MatchId.IsEmpty())
    {
        return;
    }

    TSharedRef<IHttpRequest> Request = FHttpModule::Get().CreateRequest();
    Request->SetURL(ApiBaseUrl + "/matchmaking/matches/" + CurrentMatch.MatchId + "/decline");
    Request->SetVerb("POST");
    Request->SetHeader("Authorization", "Bearer " + GetAccessToken());

    Request->OnProcessRequestComplete().BindLambda([this](FHttpRequestPtr Req, FHttpResponsePtr Resp, bool bSuccess)
    {
        CurrentMatch = FMatchInfo();
        CurrentTicket = FMatchmakingTicket();
        SetStatus(EMatchmakingStatus::Cancelled);
    });

    Request->ProcessRequest();
}

void UMatchmakingSubsystem::SetStatus(EMatchmakingStatus NewStatus)
{
    if (CurrentStatus != NewStatus)
    {
        CurrentStatus = NewStatus;
        OnStatusChanged.Broadcast(NewStatus);
    }
}

float UMatchmakingSubsystem::GetElapsedWaitTime() const
{
    if (CurrentStatus != EMatchmakingStatus::Searching)
    {
        return 0.0f;
    }

    FTimespan Elapsed = FDateTime::UtcNow() - SearchStartTime;
    return (float)Elapsed.GetTotalSeconds();
}

void UMatchmakingSubsystem::HandleTimeout()
{
    StopPolling();
    CancelMatchmaking();
    HandleError(TEXT("Matchmaking timed out"));
}

void UMatchmakingSubsystem::HandleError(const FString& Message)
{
    CurrentTicket.ErrorMessage = Message;
    SetStatus(EMatchmakingStatus::Error);
    OnError.Broadcast(Message);
}

void UMatchmakingSubsystem::ConnectToMatchServer()
{
    // 使用UE的网络连接
    FString ServerAddress = FString::Printf(TEXT("%s:%d"),
        *CurrentMatch.ServerAddress, CurrentMatch.ServerPort);

    // 打开服务器连接
    UGameplayStatics::OpenLevel(GetWorld(), FName(*ServerAddress), true);
}
```

---

## 33.3 匹配服务后端

### 33.3.1 匹配算法实现

```javascript
// match-service/src/matcher/MatchmakingEngine.js
const { v4: uuidv4 } = require('uuid');
const Redis = require('ioredis');

class MatchmakingEngine {
    constructor() {
        this.redis = new Redis();
        this.matchInterval = 1000; // 1秒
        this.ratingRange = 100; // 初始Rating范围
        this.maxRatingRange = 500; // 最大Rating范围
        this.rangeIncreasePerSecond = 10; // 每秒范围增加

        this.startMatchmakingLoop();
    }

    // 创建匹配票据
    async createTicket(userId, mode, region, preferences) {
        const ticketId = uuidv4();
        const now = Date.now();

        // 获取玩家Rating
        const rating = await this.getPlayerRating(userId, mode);

        const ticket = {
            ticketId,
            userId,
            mode,
            region: region === 'auto' ? await this.determineBestRegion(userId) : region,
            rating,
            preferences,
            createdAt: now,
            ratingRange: this.ratingRange
        };

        // 添加到匹配队列
        await this.addToQueue(ticket);

        return {
            ticketId,
            estimatedWait: await this.estimateWaitTime(mode, rating)
        };
    }

    // 取消匹配
    async cancelTicket(ticketId) {
        const ticket = await this.redis.hgetall(`ticket:${ticketId}`);
        if (ticket) {
            await this.redis.lrem(`queue:${ticket.mode}`, 0, ticketId);
            await this.redis.del(`ticket:${ticketId}`);
        }
    }

    // 添加到队列
    async addToQueue(ticket) {
        const key = `queue:${ticket.mode}`;

        await this.redis.hset(`ticket:${ticket.ticketId}`, {
            ticketId: ticket.ticketId,
            userId: ticket.userId,
            mode: ticket.mode,
            region: ticket.region,
            rating: ticket.rating,
            createdAt: ticket.createdAt,
            ratingRange: ticket.ratingRange
        });

        await this.redis.rpush(key, ticket.ticketId);
    }

    // 匹配循环
    startMatchmakingLoop() {
        setInterval(async () => {
            const modes = ['casual_1v1', 'casual_2v2', 'casual_3v3', 'ranked_1v1', 'ranked_2v2', 'ranked_3v3'];

            for (const mode of modes) {
                await this.processQueue(mode);
            }
        }, this.matchInterval);
    }

    // 处理队列
    async processQueue(mode) {
        const queueKey = `queue:${mode}`;
        const teamSize = this.getTeamSize(mode);

        // 获取队列中所有票据
        const ticketIds = await this.redis.lrange(queueKey, 0, -1);

        if (ticketIds.length < teamSize * 2) {
            return; // 人数不足
        }

        // 获取票据详情
        const tickets = [];
        for (const ticketId of ticketIds) {
            const ticket = await this.redis.hgetall(`ticket:${ticketId}`);
            if (ticket && ticket.ticketId) {
                tickets.push(ticket);
            }
        }

        // 更新Rating范围
        await this.updateRatingRanges(tickets);

        // 尝试匹配
        const match = await this.tryMatch(tickets, teamSize);

        if (match) {
            await this.createMatch(match, mode);
        }
    }

    // 尝试匹配
    async tryMatch(tickets, teamSize) {
        const totalPlayers = teamSize * 2;

        // 按Rating排序
        tickets.sort((a, b) => parseInt(a.rating) - parseInt(b.rating));

        // 滑动窗口匹配
        for (let i = 0; i <= tickets.length - totalPlayers; i++) {
            const window = tickets.slice(i, i + totalPlayers);

            // 检查是否都在彼此的Rating范围内
            if (this.isWithinRange(window)) {
                // 检查区域
                const regions = window.map(t => t.region);
                const commonRegion = this.getCommonRegion(regions);

                if (commonRegion) {
                    return window;
                }
            }
        }

        return null;
    }

    // 检查Rating范围
    isWithinRange(tickets) {
        const minRating = parseInt(tickets[0].rating);
        const maxRating = parseInt(tickets[tickets.length - 1].rating);

        for (const ticket of tickets) {
            const range = parseInt(ticket.ratingRange);
            const rating = parseInt(ticket.rating);

            if (maxRating - rating > range || rating - minRating > range) {
                return false;
            }
        }

        return true;
    }

    // 更新Rating范围
    async updateRatingRanges(tickets) {
        const now = Date.now();

        for (const ticket of tickets) {
            const createdAt = parseInt(ticket.createdAt);
            const waitTime = (now - createdAt) / 1000;

            // 根据等待时间扩大范围
            const newRange = Math.min(
                this.ratingRange + waitTime * this.rangeIncreasePerSecond,
                this.maxRatingRange
            );

            if (newRange > parseInt(ticket.ratingRange)) {
                ticket.ratingRange = newRange;
                await this.redis.hset(`ticket:${ticket.ticketId}`, 'ratingRange', newRange);
            }
        }
    }

    // 创建匹配
    async createMatch(tickets, mode) {
        const matchId = uuidv4();
        const teamSize = this.getTeamSize(mode);

        // 分配队伍
        const team1 = tickets.slice(0, teamSize).map(t => t.userId);
        const team2 = tickets.slice(teamSize, teamSize * 2).map(t => t.userId);

        // 分配服务器
        const server = await this.allocateServer(tickets[0].region, mode);

        // 创建匹配记录
        const match = {
            matchId,
            mode,
            team1,
            team2,
            serverAddress: server.address,
            serverPort: server.port,
            sessionToken: this.generateSessionToken(matchId),
            createdAt: Date.now(),
            status: 'pending'
        };

        // 存储匹配
        await this.redis.hset(`match:${matchId}`, match);
        await this.redis.expire(`match:${matchId}`, 300); // 5分钟过期

        // 从队列移除票据
        for (const ticket of tickets) {
            await this.redis.lrem(`queue:${mode}`, 0, ticket.ticketId);
            await this.redis.del(`ticket:${ticket.ticketId}`);

            // 通知玩家
            await this.notifyPlayer(ticket.userId, matchId, match);
        }

        return match;
    }

    // 获取队伍大小
    getTeamSize(mode) {
        const sizes = {
            'casual_1v1': 1,
            'casual_2v2': 2,
            'casual_3v3': 3,
            'ranked_1v1': 1,
            'ranked_2v2': 2,
            'ranked_3v3': 3
        };
        return sizes[mode] || 1;
    }

    // 分配服务器
    async allocateServer(region, mode) {
        // 查找可用服务器
        // 这里简化实现，实际应该从服务器池中选择
        const servers = {
            'us-east': { address: 'game-server-us-east.example.com', port: 7777 },
            'us-west': { address: 'game-server-us-west.example.com', port: 7777 },
            'eu': { address: 'game-server-eu.example.com', port: 7777 },
            'asia': { address: 'game-server-asia.example.com', port: 7777 }
        };

        return servers[region] || servers['us-east'];
    }

    // 获取玩家Rating
    async getPlayerRating(userId, mode) {
        // 从数据库或缓存获取
        const key = `rating:${userId}:${mode}`;
        let rating = await this.redis.get(key);

        if (!rating) {
            // 默认Rating
            rating = 1000;
        }

        return parseInt(rating);
    }

    // 预估等待时间
    async estimateWaitTime(mode, rating) {
        // 基于历史数据估算
        // 简化实现
        return 60; // 60秒
    }

    // 确定最佳区域
    async determineBestRegion(userId) {
        // 基于IP或历史记录
        return 'us-east';
    }

    // 获取共同区域
    getCommonRegion(regions) {
        const counts = {};
        for (const region of regions) {
            counts[region] = (counts[region] || 0) + 1;
        }

        // 返回最常见的区域
        let maxCount = 0;
        let commonRegion = null;
        for (const [region, count] of Object.entries(counts)) {
            if (count > maxCount) {
                maxCount = count;
                commonRegion = region;
            }
        }

        return commonRegion;
    }

    // 生成会话Token
    generateSessionToken(matchId) {
        const crypto = require('crypto');
        return crypto.randomBytes(32).toString('hex');
    }

    // 通知玩家
    async notifyPlayer(userId, matchId, match) {
        // 通过WebSocket或推送通知
        const notificationKey = `notification:${userId}`;
        await this.redis.publish('match_events', JSON.stringify({
            type: 'match_found',
            userId,
            matchId,
            match: {
                matchId: match.matchId,
                serverAddress: match.serverAddress,
                serverPort: match.serverPort,
                sessionToken: match.sessionToken
            }
        }));
    }
}

module.exports = new MatchmakingEngine();
```

---

## 33.4 匹配状态UI

### 33.4.1 匹配界面控件

```csharp
// UIMatchmakingWidget.h
#pragma once

#include "CoreMinimal.h"
#include "Blueprint/UserWidget.h"
#include "MatchmakingSubsystem.h"

#include "UIMatchmakingWidget.generated.h"

UCLASS()
class ARENAUI_API UUIMatchmakingWidget : public UUserWidget
{
    GENERATED_BODY()

public:
    virtual void NativeConstruct() override;
    virtual void NativeDestruct() override;
    virtual void NativeTick(const FGeometry& MyGeometry, float InDeltaTime) override;

protected:
    // 绑定UI元素
    UPROPERTY(meta = (BindWidget))
    class UTextBlock* StatusText;

    UPROPERTY(meta = (BindWidget))
    class UTextBlock* WaitTimeText;

    UPROPERTY(meta = (BindWidget))
    class UTextBlock* EstimatedTimeText;

    UPROPERTY(meta = (BindWidget))
    class UButton* CancelButton;

    UPROPERTY(meta = (BindWidget))
    class UButton* AcceptButton;

    UPROPERTY(meta = (BindWidget))
    class UButton* DeclineButton;

    UPROPERTY(meta = (BindWidget))
    class UWidgetSwitcher* StateSwitcher;

    UPROPERTY(meta = (BindWidget))
    class UWidget* SearchingPanel;

    UPROPERTY(meta = (BindWidget))
    class UWidget* MatchFoundPanel;

    UFUNCTION()
    void OnCancelButtonClicked();

    UFUNCTION()
    void OnAcceptButtonClicked();

    UFUNCTION()
    void OnDeclineButtonClicked();

    UFUNCTION()
    void OnMatchmakingStatusChanged(EMatchmakingStatus NewStatus);

    UFUNCTION()
    void OnMatchFound(const FMatchInfo& Match);

    UFUNCTION()
    void OnMatchmakingError(const FString& ErrorMessage);

    void UpdateWaitTime();
    void ShowSearchingPanel();
    void ShowMatchFoundPanel();
    FString FormatTime(float Seconds) const;

private:
    UMatchmakingSubsystem* MatchmakingSubsystem;
    float AcceptCountdown = 30.0f;
};
```

```csharp
// UIMatchmakingWidget.cpp
#include "UIMatchmakingWidget.h"
#include "Components/TextBlock.h"
#include "Components/Button.h"
#include "Components/WidgetSwitcher.h"
#include "Kismet/KismetSystemLibrary.h"

void UUIMatchmakingWidget::NativeConstruct()
{
    Super::NativeConstruct();

    // 获取匹配子系统
    if (UGameInstance* GI = GetGameInstance())
    {
        MatchmakingSubsystem = GI->GetSubsystem<UMatchmakingSubsystem>();
    }

    // 绑定事件
    if (MatchmakingSubsystem)
    {
        MatchmakingSubsystem->OnStatusChanged.AddDynamic(this, &UUIMatchmakingWidget::OnMatchmakingStatusChanged);
        MatchmakingSubsystem->OnMatchFound.AddDynamic(this, &UUIMatchmakingWidget::OnMatchFound);
        MatchmakingSubsystem->OnError.AddDynamic(this, &UUIMatchmakingWidget::OnMatchmakingError);
    }

    // 绑定按钮
    if (CancelButton)
    {
        CancelButton->OnClicked.AddDynamic(this, &UUIMatchmakingWidget::OnCancelButtonClicked);
    }

    if (AcceptButton)
    {
        AcceptButton->OnClicked.AddDynamic(this, &UUIMatchmakingWidget::OnAcceptButtonClicked);
    }

    if (DeclineButton)
    {
        DeclineButton->OnClicked.AddDynamic(this, &UUIMatchmakingWidget::OnDeclineButtonClicked);
    }

    // 初始显示搜索面板
    ShowSearchingPanel();
}

void UUIMatchmakingWidget::NativeDestruct()
{
    if (MatchmakingSubsystem)
    {
        MatchmakingSubsystem->OnStatusChanged.RemoveDynamic(this, &UUIMatchmakingWidget::OnMatchmakingStatusChanged);
        MatchmakingSubsystem->OnMatchFound.RemoveDynamic(this, &UUIMatchmakingWidget::OnMatchFound);
        MatchmakingSubsystem->OnError.RemoveDynamic(this, &UUIMatchmakingWidget::OnMatchmakingError);
    }

    Super::NativeDestruct();
}

void UUIMatchmakingWidget::NativeTick(const FGeometry& MyGeometry, float InDeltaTime)
{
    Super::NativeTick(MyGeometry, InDeltaTime);

    if (MatchmakingSubsystem && MatchmakingSubsystem->IsSearching())
    {
        UpdateWaitTime();
    }

    // 匹配接受倒计时
    if (MatchmakingSubsystem && MatchmakingSubsystem->GetStatus() == EMatchmakingStatus::FoundMatch)
    {
        AcceptCountdown -= InDeltaTime;
        if (WaitTimeText)
        {
            WaitTimeText->SetText(FText::FromString(FString::Printf(TEXT("Accept in: %s"), *FormatTime(AcceptCountdown))));
        }

        if (AcceptCountdown <= 0)
        {
            OnDeclineButtonClicked();
        }
    }
}

void UUIMatchmakingWidget::OnCancelButtonClicked()
{
    if (MatchmakingSubsystem)
    {
        MatchmakingSubsystem->CancelMatchmaking();
    }

    RemoveFromParent();
}

void UUIMatchmakingWidget::OnAcceptButtonClicked()
{
    if (MatchmakingSubsystem)
    {
        MatchmakingSubsystem->AcceptMatch();
    }
}

void UUIMatchmakingWidget::OnDeclineButtonClicked()
{
    if (MatchmakingSubsystem)
    {
        MatchmakingSubsystem->DeclineMatch();
    }

    RemoveFromParent();
}

void UUIMatchmakingWidget::OnMatchmakingStatusChanged(EMatchmakingStatus NewStatus)
{
    switch (NewStatus)
    {
        case EMatchmakingStatus::Searching:
            ShowSearchingPanel();
            break;

        case EMatchmakingStatus::FoundMatch:
            ShowMatchFoundPanel();
            AcceptCountdown = 30.0f;
            break;

        case EMatchmakingStatus::Cancelled:
            RemoveFromParent();
            break;

        default:
            break;
    }

    if (StatusText)
    {
        FString StatusString;
        switch (NewStatus)
        {
            case EMatchmakingStatus::Searching: StatusString = TEXT("Searching for match..."); break;
            case EMatchmakingStatus::FoundMatch: StatusString = TEXT("Match found!"); break;
            case EMatchmakingStatus::Connecting: StatusString = TEXT("Connecting to server..."); break;
            default: StatusString = TEXT("");
        }
        StatusText->SetText(FText::FromString(StatusString));
    }
}

void UUIMatchmakingWidget::OnMatchFound(const FMatchInfo& Match)
{
    ShowMatchFoundPanel();
}

void UUIMatchmakingWidget::OnMatchmakingError(const FString& ErrorMessage)
{
    if (StatusText)
    {
        StatusText->SetText(FText::FromString(ErrorMessage));
    }
}

void UUIMatchmakingWidget::UpdateWaitTime()
{
    if (WaitTimeText && MatchmakingSubsystem)
    {
        float WaitTime = MatchmakingSubsystem->GetElapsedWaitTime();
        WaitTimeText->SetText(FText::FromString(FormatTime(WaitTime)));
    }

    if (EstimatedTimeText && MatchmakingSubsystem)
    {
        FMatchmakingTicket Ticket = MatchmakingSubsystem->GetCurrentTicket();
        int32 Estimated = Ticket.EstimatedWaitSeconds;
        EstimatedTimeText->SetText(FText::FromString(FString::Printf(TEXT("Estimated: %s"), *FormatTime(Estimated))));
    }
}

void UUIMatchmakingWidget::ShowSearchingPanel()
{
    if (StateSwitcher && SearchingPanel)
    {
        StateSwitcher->SetActiveWidget(SearchingPanel);
    }
}

void UUIMatchmakingWidget::ShowMatchFoundPanel()
{
    if (StateSwitcher && MatchFoundPanel)
    {
        StateSwitcher->SetActiveWidget(MatchFoundPanel);
    }
}

FString UUIMatchmakingWidget::FormatTime(float Seconds) const
{
    int32 Minutes = FMath::FloorToInt(Seconds / 60.0f);
    int32 Secs = FMath::FloorToInt(Seconds) % 60;

    return FString::Printf(TEXT("%02d:%02d"), Minutes, Secs);
}
```

---

## 33.5 实践任务

### 任务1：实现匹配队列

创建匹配队列系统：
- 队列管理
- 玩家入队/出队
- 队列状态监控

### 任务2：优化匹配算法

改进匹配算法：
- 平衡匹配速度和质量
- 添加偏好匹配
- 实现预组队匹配

### 任务3：创建匹配UI

设计匹配界面：
- 搜索动画
- 状态显示
- 取消/接受按钮

---

## 33.6 总结

本课完成了：
- 匹配系统架构设计
- 客户端匹配子系统
- 服务端匹配算法
- 匹配状态UI

**下一课预告**：游戏房间逻辑 - 房间创建、管理和游戏流程控制。

---

*课程版本：1.0*
*最后更新：2026-04-10*
