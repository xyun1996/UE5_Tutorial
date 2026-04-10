# 第25课：实时通信服务

> 学习目标：掌握WebSocket服务器集成、实时事件广播和跨平台推送通知系统。

---

## 25.1 WebSocket 服务器集成

### 25.1.1 UE5 WebSocket客户端

```csharp
// WebSocketClient.h
#pragma once

#include "CoreMinimal.h"
#include "IWebSocket.h"
#include "WebSocketsModule.h"

#include "WebSocketClient.generated.h"

USTRUCT(BlueprintType)
struct FWebSocketConfig
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString ServerUrl;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TMap<FString, FString> Headers;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float ReconnectInterval = 5.0f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 MaxReconnectAttempts = 5;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float PingInterval = 30.0f;
};

UENUM(BlueprintType)
enum class EWebSocketState : uint8
{
    Disconnected,
    Connecting,
    Connected,
    Reconnecting
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnWebSocketConnected, bool, bSuccess);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnWebSocketDisconnected, int32, StatusCode);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnWebSocketMessage, const FString&, Message, const FString&, Type);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnWebSocketError, const FString&, Error);

UCLASS()
class WEBSOCKETTUTORIAL_API UWebSocketClient : public UObject
{
    GENERATED_BODY()

public:
    virtual void BeginDestroy() override;

    // 连接服务器
    UFUNCTION(BlueprintCallable, Category = "WebSocket")
    void Connect(const FWebSocketConfig& Config);

    // 断开连接
    UFUNCTION(BlueprintCallable, Category = "WebSocket")
    void Disconnect();

    // 发送文本消息
    UFUNCTION(BlueprintCallable, Category = "WebSocket")
    bool SendText(const FString& Message);

    // 发送二进制消息
    UFUNCTION(BlueprintCallable, Category = "WebSocket")
    bool SendBinary(const TArray<uint8>& Data);

    // 发送JSON消息
    UFUNCTION(BlueprintCallable, Category = "WebSocket")
    bool SendJson(const TSharedPtr<FJsonObject>& JsonObject);

    // 获取连接状态
    UFUNCTION(BlueprintPure, Category = "WebSocket")
    EWebSocketState GetState() const { return CurrentState; }

    // 是否已连接
    UFUNCTION(BlueprintPure, Category = "WebSocket")
    bool IsConnected() const;

    // 订阅频道
    UFUNCTION(BlueprintCallable, Category = "WebSocket")
    void Subscribe(const FString& Channel);

    // 取消订阅
    UFUNCTION(BlueprintCallable, Category = "WebSocket")
    void Unsubscribe(const FString& Channel);

    UPROPERTY(BlueprintAssignable, Category = "WebSocket")
    FOnWebSocketConnected OnConnected;

    UPROPERTY(BlueprintAssignable, Category = "WebSocket")
    FOnWebSocketDisconnected OnDisconnected;

    UPROPERTY(BlueprintAssignable, Category = "WebSocket")
    FOnWebSocketMessage OnMessage;

    UPROPERTY(BlueprintAssignable, Category = "WebSocket")
    FOnWebSocketError OnError;

private:
    TSharedPtr<IWebSocket> WebSocket;
    EWebSocketState CurrentState = EWebSocketState::Disconnected;
    FWebSocketConfig CurrentConfig;
    int32 ReconnectAttempts = 0;
    FTimerHandle ReconnectTimerHandle;
    FTimerHandle PingTimerHandle;
    TSet<FString> SubscribedChannels;

    void HandleConnected();
    void HandleConnectionError(const FString& Error);
    void HandleClosed(int32 StatusCode, const FString& Reason, bool bWasClean);
    void HandleMessage(const FString& Message);
    void HandleRawMessage(const void* Data, SIZE_T Size, SIZE_T BytesRemaining);

    void AttemptReconnect();
    void StartPingTimer();
    void SendPing();

    // 消息处理
    void ProcessJsonMessage(const TSharedPtr<FJsonObject>& JsonObject);
};
```

```csharp
// WebSocketClient.cpp
#include "WebSocketClient.h"
#include "JsonUtilities.h"
#include "TimerManager.h"

void UWebSocketClient::BeginDestroy()
{
    Disconnect();
    Super::BeginDestroy();
}

void UWebSocketClient::Connect(const FWebSocketConfig& Config)
{
    if (CurrentState == EWebSocketState::Connected || CurrentState == EWebSocketState::Connecting)
    {
        UE_LOG(LogTemp, Warning, TEXT("WebSocket already connected or connecting"));
        return;
    }

    CurrentConfig = Config;
    CurrentState = EWebSocketState::Connecting;

    // 创建WebSocket
    TMap<FString, FString> Headers = Config.Headers;

    // 添加认证头
    // Headers.Add("Authorization", "Bearer " + AuthToken);

    WebSocket = FWebSocketsModule::Get().CreateWebSocket(Config.ServerUrl, TEXT("wss"), Headers);

    // 绑定事件
    WebSocket->OnConnected().AddLambda([this]() { HandleConnected(); });
    WebSocket->OnConnectionError().AddLambda([this](const FString& Error) { HandleConnectionError(Error); });
    WebSocket->OnClosed().AddLambda([this](int32 StatusCode, const FString& Reason, bool bWasClean) { HandleClosed(StatusCode, Reason, bWasClean); });
    WebSocket->OnMessage().AddLambda([this](const FString& Message) { HandleMessage(Message); });
    WebSocket->OnRawMessage().AddLambda([this](const void* Data, SIZE_T Size, SIZE_T BytesRemaining) { HandleRawMessage(Data, Size, BytesRemaining); });

    // 连接
    WebSocket->Connect();
}

void UWebSocketClient::Disconnect()
{
    if (WebSocket.IsValid())
    {
        CurrentState = EWebSocketState::Disconnected;
        WebSocket->Close();
        WebSocket.Reset();
    }

    // 清除定时器
    if (UWorld* World = GetWorld())
    {
        World->GetTimerManager().ClearTimer(ReconnectTimerHandle);
        World->GetTimerManager().ClearTimer(PingTimerHandle);
    }
}

bool UWebSocketClient::IsConnected() const
{
    return CurrentState == EWebSocketState::Connected && WebSocket.IsValid() && WebSocket->IsConnected();
}

bool UWebSocketClient::SendText(const FString& Message)
{
    if (!IsConnected())
    {
        return false;
    }

    WebSocket->Send(Message);
    return true;
}

bool UWebSocketClient::SendBinary(const TArray<uint8>& Data)
{
    if (!IsConnected())
    {
        return false;
    }

    WebSocket->Send(Data.GetData(), Data.Num(), true);
    return true;
}

bool UWebSocketClient::SendJson(const TSharedPtr<FJsonObject>& JsonObject)
{
    FString JsonString;
    TJsonWriterFactory<>::Create(&JsonString)->WriteObject(JsonObject.ToSharedRef());
    return SendText(JsonString);
}

void UWebSocketClient::Subscribe(const FString& Channel)
{
    SubscribedChannels.Add(Channel);

    if (IsConnected())
    {
        TSharedPtr<FJsonObject> Message = MakeShareable(new FJsonObject);
        Message->SetStringField("action", "subscribe");
        Message->SetStringField("channel", Channel);
        SendJson(Message);
    }
}

void UWebSocketClient::Unsubscribe(const FString& Channel)
{
    SubscribedChannels.Remove(Channel);

    if (IsConnected())
    {
        TSharedPtr<FJsonObject> Message = MakeShareable(new FJsonObject);
        Message->SetStringField("action", "unsubscribe");
        Message->SetStringField("channel", Channel);
        SendJson(Message);
    }
}

void UWebSocketClient::HandleConnected()
{
    CurrentState = EWebSocketState::Connected;
    ReconnectAttempts = 0;

    UE_LOG(LogTemp, Log, TEXT("WebSocket connected to %s"), *CurrentConfig.ServerUrl);

    // 重新订阅频道
    for (const FString& Channel : SubscribedChannels)
    {
        Subscribe(Channel);
    }

    // 开始心跳
    StartPingTimer();

    OnConnected.Broadcast(true);
}

void UWebSocketClient::HandleConnectionError(const FString& Error)
{
    CurrentState = EWebSocketState::Disconnected;
    UE_LOG(LogTemp, Error, TEXT("WebSocket connection error: %s"), *Error);

    OnError.Broadcast(Error);
    OnConnected.Broadcast(false);

    // 尝试重连
    AttemptReconnect();
}

void UWebSocketClient::HandleClosed(int32 StatusCode, const FString& Reason, bool bWasClean)
{
    EWebSocketState PreviousState = CurrentState;
    CurrentState = EWebSocketState::Disconnected;

    UE_LOG(LogTemp, Log, TEXT("WebSocket closed: %d - %s"), StatusCode, *Reason);

    OnDisconnected.Broadcast(StatusCode);

    // 非正常断开尝试重连
    if (!bWasClean && PreviousState == EWebSocketState::Connected)
    {
        AttemptReconnect();
    }
}

void UWebSocketClient::HandleMessage(const FString& Message)
{
    // 尝试解析JSON
    TSharedPtr<FJsonObject> JsonObject;
    TJsonReaderFactory<TCHAR>::Create(Message) >> JsonObject;

    if (JsonObject.IsValid())
    {
        ProcessJsonMessage(JsonObject);
    }
    else
    {
        // 纯文本消息
        OnMessage.Broadcast(Message, TEXT("text"));
    }
}

void UWebSocketClient::ProcessJsonMessage(const TSharedPtr<FJsonObject>& JsonObject)
{
    FString Type = JsonObject->GetStringField("type");

    // 处理不同类型的消息
    if (Type == "ping")
    {
        // 响应pong
        TSharedPtr<FJsonObject> Pong = MakeShareable(new FJsonObject);
        Pong->SetStringField("type", "pong");
        SendJson(Pong);
    }
    else if (Type == "pong")
    {
        // 心跳响应
    }
    else
    {
        // 广播给业务层
        FString MessageData;
        TJsonWriterFactory<>::Create(&MessageData)->WriteObject(JsonObject.ToSharedRef());
        OnMessage.Broadcast(MessageData, Type);
    }
}

void UWebSocketClient::AttemptReconnect()
{
    if (ReconnectAttempts >= CurrentConfig.MaxReconnectAttempts)
    {
        UE_LOG(LogTemp, Error, TEXT("Max reconnect attempts reached"));
        return;
    }

    CurrentState = EWebSocketState::Reconnecting;
    ReconnectAttempts++;

    UE_LOG(LogTemp, Log, TEXT("Attempting reconnect (%d/%d) in %.1f seconds"),
        ReconnectAttempts, CurrentConfig.MaxReconnectAttempts, CurrentConfig.ReconnectInterval);

    if (UWorld* World = GetWorld())
    {
        World->GetTimerManager().SetTimer(
            ReconnectTimerHandle,
            [this]() { Connect(CurrentConfig); },
            CurrentConfig.ReconnectInterval,
            false
        );
    }
}

void UWebSocketClient::StartPingTimer()
{
    if (CurrentConfig.PingInterval > 0 && UWorld* World = GetWorld())
    {
        World->GetTimerManager().SetTimer(
            PingTimerHandle,
            this,
            &UWebSocketClient::SendPing,
            CurrentConfig.PingInterval,
            true
        );
    }
}

void UWebSocketClient::SendPing()
{
    if (IsConnected())
    {
        TSharedPtr<FJsonObject> Ping = MakeShareable(new FJsonObject);
        Ping->SetStringField("type", "ping");
        Ping->SetNumberField("timestamp", FPlatformTime::Seconds());
        SendJson(Ping);
    }
}
```

### 25.1.2 WebSocket服务器（Node.js示例）

```javascript
// websocket-server.js
const WebSocket = require('ws');
const Redis = require('ioredis');

class GameWebSocketServer {
    constructor(port) {
        this.port = port;
        this.wss = new WebSocket.Server({ port });
        this.clients = new Map();
        this.channels = new Map();

        // Redis订阅（用于跨服务器广播）
        this.redisPub = new Redis();
        this.redisSub = new Redis();

        this.setupRedis();
        this.setupWebSocket();
        this.setupHeartbeat();

        console.log(`WebSocket server started on port ${port}`);
    }

    setupRedis() {
        this.redisSub.subscribe('game:broadcast', 'game:channel:*');
        this.redisSub.on('message', (channel, message) => {
            this.handleRedisMessage(channel, message);
        });
    }

    setupWebSocket() {
        this.wss.on('connection', (ws, req) => {
            const clientId = this.generateId();
            const clientIp = req.socket.remoteAddress;

            // 存储客户端信息
            ws.clientId = clientId;
            ws.isAlive = true;
            ws.channels = new Set();

            this.clients.set(clientId, ws);

            console.log(`Client connected: ${clientId} from ${clientIp}`);

            // 发送欢迎消息
            this.sendToClient(ws, {
                type: 'connected',
                clientId: clientId
            });

            // 消息处理
            ws.on('message', (data) => {
                this.handleMessage(ws, data);
            });

            // 断开处理
            ws.on('close', () => {
                this.handleDisconnect(ws);
            });

            // 错误处理
            ws.on('error', (error) => {
                console.error(`WebSocket error for ${clientId}:`, error);
            });

            // 心跳检测
            ws.on('pong', () => {
                ws.isAlive = true;
            });
        });
    }

    setupHeartbeat() {
        setInterval(() => {
            this.wss.clients.forEach((ws) => {
                if (!ws.isAlive) {
                    console.log(`Client ${ws.clientId} timeout, terminating`);
                    this.handleDisconnect(ws);
                    return ws.terminate();
                }

                ws.isAlive = false;
                ws.ping();
            });
        }, 30000);
    }

    handleMessage(ws, data) {
        try {
            const message = JSON.parse(data);

            switch (message.action) {
                case 'subscribe':
                    this.subscribeChannel(ws, message.channel);
                    break;
                case 'unsubscribe':
                    this.unsubscribeChannel(ws, message.channel);
                    break;
                case 'broadcast':
                    this.broadcastToChannel(message.channel, message.data);
                    break;
                case 'direct':
                    this.sendDirectMessage(message.targetId, message.data);
                    break;
                default:
                    // 自定义消息处理
                    this.handleCustomMessage(ws, message);
            }
        } catch (error) {
            console.error('Failed to parse message:', error);
        }
    }

    handleCustomMessage(ws, message) {
        switch (message.type) {
            case 'chat':
                this.handleChatMessage(ws, message);
                break;
            case 'game_event':
                this.handleGameEvent(ws, message);
                break;
            case 'player_update':
                this.handlePlayerUpdate(ws, message);
                break;
        }
    }

    subscribeChannel(ws, channel) {
        if (!ws.channels.has(channel)) {
            ws.channels.add(channel);

            if (!this.channels.has(channel)) {
                this.channels.set(channel, new Set());
            }
            this.channels.get(channel).add(ws.clientId);

            this.sendToClient(ws, {
                type: 'subscribed',
                channel: channel
            });

            console.log(`Client ${ws.clientId} subscribed to ${channel}`);
        }
    }

    unsubscribeChannel(ws, channel) {
        if (ws.channels.has(channel)) {
            ws.channels.delete(channel);

            if (this.channels.has(channel)) {
                this.channels.get(channel).delete(ws.clientId);
            }

            this.sendToClient(ws, {
                type: 'unsubscribed',
                channel: channel
            });
        }
    }

    broadcastToChannel(channel, data, excludeClient = null) {
        const message = JSON.stringify({
            type: 'broadcast',
            channel: channel,
            data: data,
            timestamp: Date.now()
        });

        if (this.channels.has(channel)) {
            this.channels.get(channel).forEach((clientId) => {
                if (clientId !== excludeClient) {
                    const client = this.clients.get(clientId);
                    if (client && client.readyState === WebSocket.OPEN) {
                        client.send(message);
                    }
                }
            });
        }

        // 发布到Redis（跨服务器广播）
        this.redisPub.publish(`game:channel:${channel}`, message);
    }

    sendDirectMessage(targetId, data) {
        const client = this.clients.get(targetId);
        if (client && client.readyState === WebSocket.OPEN) {
            this.sendToClient(client, {
                type: 'direct',
                data: data
            });
        }
    }

    handleRedisMessage(channel, message) {
        // 处理来自其他服务器的消息
        const parsedMessage = JSON.parse(message);

        if (channel.startsWith('game:channel:')) {
            const channelName = channel.replace('game:channel:', '');
            this.broadcastToChannel(channelName, parsedMessage.data);
        } else if (channel === 'game:broadcast') {
            // 全局广播
            this.broadcastAll(parsedMessage.data);
        }
    }

    handleDisconnect(ws) {
        const clientId = ws.clientId;

        // 从所有频道移除
        ws.channels.forEach((channel) => {
            if (this.channels.has(channel)) {
                this.channels.get(channel).delete(clientId);
            }
        });

        // 从客户端列表移除
        this.clients.delete(clientId);

        console.log(`Client disconnected: ${clientId}`);

        // 通知其他服务
        this.redisPub.publish('game:events', JSON.stringify({
            type: 'player_disconnect',
            playerId: clientId
        }));
    }

    handleChatMessage(ws, message) {
        const { channel, content, sender } = message;

        // 验证消息
        if (!content || content.length > 500) {
            this.sendToClient(ws, {
                type: 'error',
                message: 'Invalid message'
            });
            return;
        }

        // 广播聊天消息
        this.broadcastToChannel(channel, {
            type: 'chat',
            sender: sender,
            content: content,
            timestamp: Date.now()
        });
    }

    handleGameEvent(ws, message) {
        const { gameId, event, data } = message;

        // 广播到游戏房间频道
        this.broadcastToChannel(`game:${gameId}`, {
            type: 'game_event',
            event: event,
            data: data,
            timestamp: Date.now()
        });
    }

    handlePlayerUpdate(ws, message) {
        const { gameId, playerData } = message;

        // 更新玩家状态并广播
        this.broadcastToChannel(`game:${gameId}:players`, {
            type: 'player_update',
            playerId: ws.clientId,
            data: playerData
        }, ws.clientId);
    }

    sendToClient(ws, data) {
        if (ws.readyState === WebSocket.OPEN) {
            ws.send(JSON.stringify(data));
        }
    }

    broadcastAll(data) {
        const message = JSON.stringify(data);
        this.wss.clients.forEach((client) => {
            if (client.readyState === WebSocket.OPEN) {
                client.send(message);
            }
        });
    }

    generateId() {
        return Math.random().toString(36).substr(2, 9) + Date.now().toString(36);
    }
}

// 启动服务器
const server = new GameWebSocketServer(8080);
```

---

## 25.2 实时事件广播

### 25.2.1 事件广播系统

```csharp
// EventBroadcastSystem.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "WebSocketClient.h"

#include "EventBroadcastSystem.generated.h"

USTRUCT(BlueprintType)
struct FGameEvent
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadWrite)
    FString EventType;

    UPROPERTY(BlueprintReadWrite)
    FString EventId;

    UPROPERTY(BlueprintReadWrite)
    int64 Timestamp = 0;

    UPROPERTY(BlueprintReadWrite)
    TMap<FString, FString> Payload;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnGameEvent, const FGameEvent&, Event);

UCLASS()
class EVENTBROADCAST_API UEventBroadcastSystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 初始化广播系统
    UFUNCTION(BlueprintCallable, Category = "EventBroadcast")
    void InitializeBroadcast(const FString& WebSocketUrl);

    // 发送事件
    UFUNCTION(BlueprintCallable, Category = "EventBroadcast")
    void SendEvent(const FString& EventType, const TMap<FString, FString>& Payload);

    // 发送事件到特定频道
    UFUNCTION(BlueprintCallable, Category = "EventBroadcast")
    void SendEventToChannel(const FString& Channel, const FString& EventType, const TMap<FString, FString>& Payload);

    // 订阅事件类型
    UFUNCTION(BlueprintCallable, Category = "EventBroadcast")
    void SubscribeEvent(const FString& EventType);

    // 取消订阅
    UFUNCTION(BlueprintCallable, Category = "EventBroadcast")
    void UnsubscribeEvent(const FString& EventType);

    // 通用事件委托
    UPROPERTY(BlueprintAssignable, Category = "EventBroadcast")
    FOnGameEvent OnGameEvent;

    // 特定类型事件委托（动态添加）
    TMap<FString, FOnGameEvent> EventTypeDelegates;

private:
    UPROPERTY()
    UWebSocketClient* WebSocketClient;

    FString ClientId;
    TSet<FString> SubscribedEvents;

    void HandleWebSocketMessage(const FString& Message, const FString& Type);
    FGameEvent ParseEvent(const TSharedPtr<FJsonObject>& JsonObject);
};
```

```csharp
// EventBroadcastSystem.cpp
#include "EventBroadcastSystem.h"
#include "JsonUtilities.h"

void UEventBroadcastSystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    WebSocketClient = NewObject<UWebSocketClient>(this);
}

void UEventBroadcastSystem::Deinitialize()
{
    if (WebSocketClient)
    {
        WebSocketClient->Disconnect();
    }

    Super::Deinitialize();
}

void UEventBroadcastSystem::InitializeBroadcast(const FString& WebSocketUrl)
{
    FWebSocketConfig Config;
    Config.ServerUrl = WebSocketUrl;
    Config.ReconnectInterval = 5.0f;
    Config.MaxReconnectAttempts = 10;
    Config.PingInterval = 30.0f;

    WebSocketClient->OnMessage.AddDynamic(this, &UEventBroadcastSystem::HandleWebSocketMessage);
    WebSocketClient->Connect(Config);
}

void UEventBroadcastSystem::SendEvent(const FString& EventType, const TMap<FString, FString>& Payload)
{
    if (!WebSocketClient->IsConnected())
    {
        UE_LOG(LogTemp, Warning, TEXT("WebSocket not connected, cannot send event"));
        return;
    }

    TSharedPtr<FJsonObject> Message = MakeShareable(new FJsonObject);
    Message->SetStringField("action", "broadcast");
    Message->SetStringField("eventType", EventType);
    Message->SetStringField("eventId", FGuid::NewGuid().ToString());
    Message->SetNumberField("timestamp", FDateTime::UtcNow().ToUnixTimestamp());

    // 添加负载
    TSharedPtr<FJsonObject> PayloadObj = MakeShareable(new FJsonObject);
    for (const auto& Pair : Payload)
    {
        PayloadObj->SetStringField(Pair.Key, Pair.Value);
    }
    Message->SetObjectField("payload", PayloadObj);

    WebSocketClient->SendJson(Message);
}

void UEventBroadcastSystem::SendEventToChannel(const FString& Channel, const FString& EventType, const TMap<FString, FString>& Payload)
{
    if (!WebSocketClient->IsConnected())
    {
        return;
    }

    TSharedPtr<FJsonObject> Message = MakeShareable(new FJsonObject);
    Message->SetStringField("action", "broadcast");
    Message->SetStringField("channel", Channel);
    Message->SetStringField("eventType", EventType);
    Message->SetStringField("eventId", FGuid::NewGuid().ToString());
    Message->SetNumberField("timestamp", FDateTime::UtcNow().ToUnixTimestamp());

    TSharedPtr<FJsonObject> PayloadObj = MakeShareable(new FJsonObject);
    for (const auto& Pair : Payload)
    {
        PayloadObj->SetStringField(Pair.Key, Pair.Value);
    }
    Message->SetObjectField("payload", PayloadObj);

    WebSocketClient->SendJson(Message);
}

void UEventBroadcastSystem::SubscribeEvent(const FString& EventType)
{
    SubscribedEvents.Add(EventType);

    // 如果需要，订阅特定频道
    FString Channel = FString::Printf(TEXT("events:%s"), *EventType);
    WebSocketClient->Subscribe(Channel);
}

void UEventBroadcastSystem::UnsubscribeEvent(const FString& EventType)
{
    SubscribedEvents.Remove(EventType);

    FString Channel = FString::Printf(TEXT("events:%s"), *EventType);
    WebSocketClient->Unsubscribe(Channel);
}

void UEventBroadcastSystem::HandleWebSocketMessage(const FString& Message, const FString& Type)
{
    TSharedPtr<FJsonObject> JsonObject;
    TJsonReaderFactory<TCHAR>::Create(Message) >> JsonObject;

    if (!JsonObject.IsValid())
    {
        return;
    }

    FGameEvent Event = ParseEvent(JsonObject);

    // 检查是否订阅了此事件类型
    if (SubscribedEvents.Contains(Event.EventType) || SubscribedEvents.Contains("*"))
    {
        OnGameEvent.Broadcast(Event);

        // 触发特定类型的委托
        if (FOnGameEvent* TypeDelegate = EventTypeDelegates.Find(Event.EventType))
        {
            TypeDelegate->Broadcast(Event);
        }
    }
}

FGameEvent UEventBroadcastSystem::ParseEvent(const TSharedPtr<FJsonObject>& JsonObject)
{
    FGameEvent Event;

    Event.EventType = JsonObject->GetStringField("eventType");
    Event.EventId = JsonObject->GetStringField("eventId");
    Event.Timestamp = JsonObject->GetNumberField("timestamp");

    const TSharedPtr<FJsonObject>* PayloadObj;
    if (JsonObject->TryGetObjectField("payload", PayloadObj))
    {
        for (const auto& Pair : (*PayloadObj)->Values)
        {
            Event.Payload.Add(Pair.Key, Pair.Value->AsString());
        }
    }

    return Event;
}
```

---

## 25.3 Push通知系统

### 25.3.1 跨平台推送封装

```csharp
// PushNotificationManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "PushNotificationManager.generated.h"

UENUM(BlueprintType)
enum class EPushPlatform : uint8
{
    None,
    FCM,        // Firebase Cloud Messaging (Android/iOS/Web)
    APNS,       // Apple Push Notification Service (iOS)
    HMS,        // Huawei Mobile Services
    OneSignal   // 跨平台推送服务
};

USTRUCT(BlueprintType)
struct FPushNotification
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadWrite)
    FString Title;

    UPROPERTY(BlueprintReadWrite)
    FString Body;

    UPROPERTY(BlueprintReadWrite)
    FString Icon;

    UPROPERTY(BlueprintReadWrite)
    FString Image;

    UPROPERTY(BlueprintReadWrite)
    FString Sound = "default";

    UPROPERTY(BlueprintReadWrite)
    FString ClickAction;

    UPROPERTY(BlueprintReadWrite)
    TMap<FString, FString> Data;

    UPROPERTY(BlueprintReadWrite)
    int32 Badge = 0;

    UPROPERTY(BlueprintReadWrite)
    int64 ExpireTime = 0;
};

USTRUCT(BlueprintType)
struct FPushToken
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString Token;

    UPROPERTY(BlueprintReadOnly)
    EPushPlatform Platform;

    UPROPERTY(BlueprintReadOnly)
    bool bIsValid = false;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnPushTokenReceived, const FPushToken&, Token);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnPushReceived, const FPushNotification&, Notification);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnPushOpened, const FPushNotification&, Notification);

UCLASS()
class PUSHNOTIFICATIONS_API UPushNotificationManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 初始化推送服务
    UFUNCTION(BlueprintCallable, Category = "Push")
    void InitializePush(EPushPlatform Platform, const FString& AppId, const FString& SenderId);

    // 请求推送权限
    UFUNCTION(BlueprintCallable, Category = "Push")
    void RequestPermission();

    // 获取推送令牌
    UFUNCTION(BlueprintCallable, Category = "Push")
    void GetPushToken();

    // 注册令牌到服务器
    UFUNCTION(BlueprintCallable, Category = "Push")
    void RegisterTokenToServer(const FString& UserId);

    // 取消注册
    UFUNCTION(BlueprintCallable, Category = "Push")
    void UnregisterToken();

    // 发送本地通知
    UFUNCTION(BlueprintCallable, Category = "Push")
    void SendLocalNotification(const FPushNotification& Notification, int64 DelaySeconds = 0);

    // 取消本地通知
    UFUNCTION(BlueprintCallable, Category = "Push")
    void CancelLocalNotification(const FString& NotificationId);

    // 清除所有通知
    UFUNCTION(BlueprintCallable, Category = "Push")
    void ClearAllNotifications();

    // 设置角标数字（iOS）
    UFUNCTION(BlueprintCallable, Category = "Push")
    void SetBadgeCount(int32 Count);

    // 获取待处理通知
    UFUNCTION(BlueprintPure, Category = "Push")
    TArray<FPushNotification> GetPendingNotifications();

    UPROPERTY(BlueprintAssignable, Category = "Push")
    FOnPushTokenReceived OnPushTokenReceived;

    UPROPERTY(BlueprintAssignable, Category = "Push")
    FOnPushReceived OnPushReceived;

    UPROPERTY(BlueprintAssignable, Category = "Push")
    FOnPushOpened OnPushOpened;

private:
    EPushPlatform CurrentPlatform = EPushPlatform::None;
    FPushToken CurrentToken;
    FString AppId;
    FString SenderId;

    void OnTokenReceived(const FString& Token);
    void OnNotificationReceived(const FString& Payload);
    void OnNotificationOpened(const FString& Payload);

    // 平台特定实现
#if PLATFORM_ANDROID
    void InitializeAndroid();
#elif PLATFORM_IOS
    void InitializeIOS();
#endif
};
```

### 25.3.2 推送服务器端

```csharp
// PushServerClient.h
#pragma once

#include "CoreMinimal.h"
#include "HttpModule.h"

#include "PushServerClient.generated.h"

USTRUCT(BlueprintType)
struct FPushTarget
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadWrite)
    FString UserId;

    UPROPERTY(BlueprintReadWrite)
    FString PushToken;

    UPROPERTY(BlueprintReadWrite)
    EPushPlatform Platform;
};

UCLASS()
class PUSHNOTIFICATIONS_API UPushServerClient : public UObject
{
    GENERATED_BODY()

public:
    void Initialize(const FString& ServerUrl, const FString& ApiKey);

    // 发送推送给单个用户
    UFUNCTION(BlueprintCallable, Category = "Push")
    void SendToUser(const FString& UserId, const FPushNotification& Notification);

    // 发送推送给多个用户
    UFUNCTION(BlueprintCallable, Category = "Push")
    void SendToUsers(const TArray<FString>& UserIds, const FPushNotification& Notification);

    // 发送给主题订阅者
    UFUNCTION(BlueprintCallable, Category = "Push")
    void SendToTopic(const FString& Topic, const FPushNotification& Notification);

    // 订阅主题
    UFUNCTION(BlueprintCallable, Category = "Push")
    void SubscribeToTopic(const FString& UserId, const FString& Topic);

    // 取消订阅
    UFUNCTION(BlueprintCallable, Category = "Push")
    void UnsubscribeFromTopic(const FString& UserId, const FString& Topic);

    // 注册设备令牌
    UFUNCTION(BlueprintCallable, Category = "Push")
    void RegisterDeviceToken(const FString& UserId, const FString& Token, EPushPlatform Platform);

private:
    FString ServerUrl;
    FString ApiKey;

    void MakeRequest(const FString& Endpoint, const FString& Method, const FString& Body);
    FString NotificationToJson(const FPushNotification& Notification);
};
```

```csharp
// PushServerClient.cpp
#include "PushServerClient.h"
#include "JsonUtilities.h"

void UPushServerClient::Initialize(const FString& InServerUrl, const FString& InApiKey)
{
    ServerUrl = InServerUrl;
    ApiKey = InApiKey;
}

void UPushServerClient::SendToUser(const FString& UserId, const FPushNotification& Notification)
{
    TSharedPtr<FJsonObject> JsonRequest = MakeShareable(new FJsonObject);
    JsonRequest->SetStringField("userId", UserId);

    TSharedPtr<FJsonObject> NotificationObj = MakeShareable(new FJsonObject);
    NotificationObj->SetStringField("title", Notification.Title);
    NotificationObj->SetStringField("body", Notification.Body);
    NotificationObj->SetStringField("icon", Notification.Icon);
    NotificationObj->SetStringField("sound", Notification.Sound);
    NotificationObj->SetNumberField("badge", Notification.Badge);

    // 添加数据
    TSharedPtr<FJsonObject> DataObj = MakeShareable(new FJsonObject);
    for (const auto& Pair : Notification.Data)
    {
        DataObj->SetStringField(Pair.Key, Pair.Value);
    }
    NotificationObj->SetObjectField("data", DataObj);

    JsonRequest->SetObjectField("notification", NotificationObj);

    FString Body;
    TJsonWriterFactory<>::Create(&Body)->WriteObject(JsonRequest.ToSharedRef());

    MakeRequest(TEXT("/send/user"), TEXT("POST"), Body);
}

void UPushServerClient::SendToUsers(const TArray<FString>& UserIds, const FPushNotification& Notification)
{
    TSharedPtr<FJsonObject> JsonRequest = MakeShareable(new FJsonObject);

    TArray<TSharedPtr<FJsonValue>> UserIdsArray;
    for (const FString& UserId : UserIds)
    {
        UserIdsArray.Add(MakeShareable(new FJsonValueString(UserId)));
    }
    JsonRequest->SetArrayField("userIds", UserIdsArray);

    // ... 同上设置notification

    FString Body;
    TJsonWriterFactory<>::Create(&Body)->WriteObject(JsonRequest.ToSharedRef());

    MakeRequest(TEXT("/send/users"), TEXT("POST"), Body);
}

void UPushServerClient::SendToTopic(const FString& Topic, const FPushNotification& Notification)
{
    TSharedPtr<FJsonObject> JsonRequest = MakeShareable(new FJsonObject);
    JsonRequest->SetStringField("topic", Topic);

    // ... 同上设置notification

    FString Body;
    TJsonWriterFactory<>::Create(&Body)->WriteObject(JsonRequest.ToSharedRef());

    MakeRequest(TEXT("/send/topic"), TEXT("POST"), Body);
}

void UPushServerClient::RegisterDeviceToken(const FString& UserId, const FString& Token, EPushPlatform Platform)
{
    TSharedPtr<FJsonObject> JsonRequest = MakeShareable(new FJsonObject);
    JsonRequest->SetStringField("userId", UserId);
    JsonRequest->SetStringField("pushToken", Token);
    JsonRequest->SetStringField("platform", Platform == EPushPlatform::FCM ? TEXT("fcm") :
                                 Platform == EPushPlatform::APNS ? TEXT("apns") : TEXT("unknown"));

    FString Body;
    TJsonWriterFactory<>::Create(&Body)->WriteObject(JsonRequest.ToSharedRef());

    MakeRequest(TEXT("/register"), TEXT("POST"), Body);
}

void UPushServerClient::MakeRequest(const FString& Endpoint, const FString& Method, const FString& Body)
{
    TSharedRef<IHttpRequest> Request = FHttpModule::Get().CreateRequest();
    Request->SetURL(ServerUrl + Endpoint);
    Request->SetVerb(Method);
    Request->SetHeader(TEXT("Content-Type"), TEXT("application/json"));
    Request->SetHeader(TEXT("Authorization"), TEXT("Bearer ") + ApiKey);

    if (!Body.IsEmpty())
    {
        Request->SetContentAsString(Body);
    }

    Request->ProcessRequest();
}
```

---

## 25.4 跨平台推送集成

### 25.4.1 Firebase Cloud Messaging (FCM)

```javascript
// push-server-fcm.js
const admin = require('firebase-admin');

class FCMPushService {
    constructor(serviceAccount) {
        this.app = admin.initializeApp({
            credential: admin.credential.cert(serviceAccount)
        });
        this.messaging = admin.messaging();
    }

    async sendToToken(token, notification, data = {}) {
        const message = {
            token: token,
            notification: {
                title: notification.title,
                body: notification.body,
                image: notification.image
            },
            data: data,
            android: {
                notification: {
                    icon: notification.icon || 'ic_notification',
                    sound: notification.sound || 'default',
                    clickAction: notification.clickAction
                }
            },
            apns: {
                payload: {
                    aps: {
                        sound: notification.sound || 'default',
                        badge: notification.badge || 0
                    }
                }
            }
        };

        try {
            const response = await this.messaging.send(message);
            console.log('Successfully sent message:', response);
            return { success: true, messageId: response };
        } catch (error) {
            console.error('Error sending message:', error);
            return { success: false, error: error.message };
        }
    }

    async sendToMultipleTokens(tokens, notification, data = {}) {
        const message = {
            tokens: tokens,
            notification: {
                title: notification.title,
                body: notification.body
            },
            data: data
        };

        try {
            const response = await this.messaging.sendMulticast(message);
            console.log(`${response.successCount} messages were sent successfully`);
            return {
                success: true,
                successCount: response.successCount,
                failureCount: response.failureCount
            };
        } catch (error) {
            return { success: false, error: error.message };
        }
    }

    async sendToTopic(topic, notification, data = {}) {
        const message = {
            topic: topic,
            notification: {
                title: notification.title,
                body: notification.body
            },
            data: data
        };

        try {
            const response = await this.messaging.send(message);
            return { success: true, messageId: response };
        } catch (error) {
            return { success: false, error: error.message };
        }
    }

    async subscribeToTopic(tokens, topic) {
        try {
            const response = await this.messaging.subscribeToTopic(tokens, topic);
            return { success: true, response };
        } catch (error) {
            return { success: false, error: error.message };
        }
    }
}

module.exports = FCMPushService;
```

---

## 25.5 实践任务

### 任务1：实现实时聊天系统

创建一个实时聊天功能：
- WebSocket连接管理
- 频道订阅/取消订阅
- 消息广播
- 断线重连

### 任务2：实现游戏事件推送

创建游戏事件推送系统：
- 玩家上线/下线通知
- 好友状态更新
- 公告推送
- 系统维护通知

### 任务3：集成FCM推送

实现跨平台推送：
- 配置FCM项目
- 集成Android/iOS SDK
- 后台推送服务器
- 前台通知处理

---

## 25.6 总结

本课学习了：
- WebSocket客户端/服务器实现
- 实时事件广播系统
- 跨平台推送通知
- Firebase Cloud Messaging集成
- 推送服务器架构

**下一课预告**：网络性能优化 - 带宽分析、包大小优化和压缩算法选择。

---

*课程版本：1.0*
*最后更新：2026-04-10*
