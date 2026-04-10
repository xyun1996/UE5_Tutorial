# 第10课：高级服务器功能

> 本课深入讲解UE5服务器的高级功能，包括控制台命令、远程管理、日志监控和热重载修复。

---

## 课程目标

- 掌握服务器控制台命令开发
- 学会实现远程管理接口
- 理解服务器日志与监控系统
- 掌握热重载与热修复技术
- 学会服务器配置管理系统

---

## 一、服务器控制台命令

### 1.1 控制台命令概述

```
┌─────────────────────────────────────────────────────────────┐
│                    控制台命令系统                            │
│                                                              │
│  输入方式：                                                  │
│  - 游戏内控制台（~ 键）                                      │
│  - 服务器命令行                                              │
│  - RCON远程控制                                              │
│  - HTTP API                                                  │
│                                                              │
│  命令类型：                                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  管理命令                                           │    │
│  │  - kick <player>      踢出玩家                      │    │
│  │  - ban <player>       封禁玩家                      │    │
│  │  - changelevel <map>  切换地图                      │    │
│  │  - restart            重启服务器                    │    │
│  └─────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  调试命令                                           │    │
│  │  - stat fps           显示帧率                      │    │
│  │  - stat net           网络统计                      │    │
│  │  - debug <type>       调试模式                      │    │
│  └─────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  游戏命令                                           │    │
│  │  - startmatch         开始比赛                      │    │
│  │  - endmatch           结束比赛                      │    │
│  │  - setscore <value>   设置分数                      │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 自定义控制台命令

```cpp
// MyConsoleCommands.h
#pragma once

#include "CoreMinimal.h"
#include "Console/Console.h"
#include "MyConsoleCommands.generated.h"

// 控制台命令管理器
UCLASS()
class UMyConsoleCommandManager : public UObject
{
    GENERATED_BODY()

public:
    static UMyConsoleCommandManager* Get(UWorld* World);

    void RegisterCommands();
    void UnregisterCommands();

private:
    // 注册的命令列表
    TArray<IConsoleCommand*> RegisteredCommands;

    // 命令处理函数
    static void HandleKickPlayer(const TArray<FString>& Args, UWorld* World);
    static void HandleBanPlayer(const TArray<FString>& Args, UWorld* World);
    static void HandleUnbanPlayer(const TArray<FString>& Args, UWorld* World);
    static void HandleListPlayers(const TArray<FString>& Args, UWorld* World);
    static void HandleChangeMap(const TArray<FString>& Args, UWorld* World);
    static void HandleRestartServer(const TArray<FString>& Args, UWorld* World);
    static void HandleServerInfo(const TArray<FString>& Args, UWorld* World);
    static void HandleSetMaxPlayers(const TArray<FString>& Args, UWorld* World);
    static void HandleBroadcastMessage(const TArray<FString>& Args, UWorld* World);
    static void HandlePauseMatch(const TArray<FString>& Args, UWorld* World);
    static void HandleDebugMode(const TArray<FString>& Args, UWorld* World);
};

// MyConsoleCommands.cpp
#include "MyConsoleCommands.h"
#include "GameFramework/PlayerController.h"
#include "GameFramework/GameModeBase.h"
#include "Engine/World.h"

static FAutoConsoleCommand KickCommand(
    TEXT("my.kick"),
    TEXT("Kick a player from the server"),
    FConsoleCommandWithWorldAndArgsDelegate::CreateStatic(
        &UMyConsoleCommandManager::HandleKickPlayer
    )
);

static FAutoConsoleCommand BanCommand(
    TEXT("my.ban"),
    TEXT("Ban a player from the server"),
    FConsoleCommandWithWorldAndArgsDelegate::CreateStatic(
        &UMyConsoleCommandManager::HandleBanPlayer
    )
);

static FAutoConsoleCommand ListPlayersCommand(
    TEXT("my.listplayers"),
    TEXT("List all connected players"),
    FConsoleCommandWithWorldAndArgsDelegate::CreateStatic(
        &UMyConsoleCommandManager::HandleListPlayers
    )
);

static FAutoConsoleCommand ChangeMapCommand(
    TEXT("my.changemap"),
    TEXT("Change to a different map"),
    FConsoleCommandWithWorldAndArgsDelegate::CreateStatic(
        &UMyConsoleCommandManager::HandleChangeMap
    )
);

static FAutoConsoleCommand ServerInfoCommand(
    TEXT("my.serverinfo"),
    TEXT("Display server information"),
    FConsoleCommandWithWorldAndArgsDelegate::CreateStatic(
        &UMyConsoleCommandManager::HandleServerInfo
    )
);

static FAutoConsoleCommand BroadcastCommand(
    TEXT("my.broadcast"),
    TEXT("Broadcast a message to all players"),
    FConsoleCommandWithWorldAndArgsDelegate::CreateStatic(
        &UMyConsoleCommandManager::HandleBroadcastMessage
    )
);

UMyConsoleCommandManager* UMyConsoleCommandManager::Get(UWorld* World)
{
    static TMap<UWorld*, UMyConsoleCommandManager*> Managers;
    if (!Managers.Contains(World))
    {
        Managers.Add(World, NewObject<UMyConsoleCommandManager>());
    }
    return Managers[World];
}

void UMyConsoleCommandManager::RegisterCommands()
{
    // 命令已在静态初始化时注册
    UE_LOG(LogTemp, Log, TEXT("Console commands registered"));
}

void UMyConsoleCommandManager::UnregisterCommands()
{
    // 清理
    UE_LOG(LogTemp, Log, TEXT("Console commands unregistered"));
}

// ========== 命令处理函数 ==========

void UMyConsoleCommandManager::HandleKickPlayer(const TArray<FString>& Args, UWorld* World)
{
    if (Args.Num() < 1)
    {
        UE_LOG(LogTemp, Warning, TEXT("Usage: my.kick <player_name_or_id> [reason]"));
        return;
    }

    FString PlayerIdentifier = Args[0];
    FString Reason = Args.Num() > 1 ? Args[1] : TEXT("Kicked by admin");

    // 查找玩家
    for (FConstPlayerControllerIterator It = World->GetPlayerControllerIterator(); It; ++It)
    {
        APlayerController* PC = It->Get();
        FString PlayerName = PC->GetPlayerState<APlayerState>() ?
            PC->GetPlayerState<APlayerState>()->GetPlayerName() : PC->GetName();

        if (PlayerName == PlayerIdentifier || PC->GetName() == PlayerIdentifier)
        {
            // 踢出玩家
            PC->ClientWasKicked(Reason);

            UE_LOG(LogTemp, Log, TEXT("Kicked player: %s, Reason: %s"), *PlayerName, *Reason);
            return;
        }
    }

    UE_LOG(LogTemp, Warning, TEXT("Player not found: %s"), *PlayerIdentifier);
}

void UMyConsoleCommandManager::HandleBanPlayer(const TArray<FString>& Args, UWorld* World)
{
    if (Args.Num() < 1)
    {
        UE_LOG(LogTemp, Warning, TEXT("Usage: my.ban <player_name_or_id> [reason] [duration]"));
        return;
    }

    FString PlayerIdentifier = Args[0];
    FString Reason = Args.Num() > 1 ? Args[1] : TEXT("Banned by admin");
    float Duration = Args.Num() > 2 ? FCString::Atof(*Args[2]) : 0.0f; // 0 = 永久

    // 添加到封禁列表
    // BanList.Add(FBanEntry{PlayerIdentifier, Reason, Duration, FDateTime::Now()});

    // 踢出玩家
    HandleKickPlayer(Args, World);

    UE_LOG(LogTemp, Log, TEXT("Banned player: %s, Reason: %s, Duration: %.0f"),
        *PlayerIdentifier, *Reason, Duration);
}

void UMyConsoleCommandManager::HandleListPlayers(const TArray<FString>& Args, UWorld* World)
{
    UE_LOG(LogTemp, Log, TEXT("=== Connected Players ==="));

    int32 Count = 0;
    for (FConstPlayerControllerIterator It = World->GetPlayerControllerIterator(); It; ++It)
    {
        APlayerController* PC = It->Get();
        FString PlayerName = PC->GetPlayerState<APlayerState>() ?
            PC->GetPlayerState<APlayerState>()->GetPlayerName() : TEXT("Unknown");

        FString NetAddr = PC->GetNetConnection() ?
            PC->GetNetConnection()->LowLevelGetRemoteAddress() : TEXT("Local");

        UE_LOG(LogTemp, Log, TEXT("  %d: %s [%s]"), Count + 1, *PlayerName, *NetAddr);
        Count++;
    }

    UE_LOG(LogTemp, Log, TEXT("Total: %d players"), Count);
}

void UMyConsoleCommandManager::HandleChangeMap(const TArray<FString>& Args, UWorld* World)
{
    if (Args.Num() < 1)
    {
        UE_LOG(LogTemp, Warning, TEXT("Usage: my.changemap <map_name>"));
        return;
    }

    FString MapName = Args[0];
    FString TravelURL = FString::Printf(TEXT("%s?Listen"), *MapName);

    World->ServerTravel(TravelURL);

    UE_LOG(LogTemp, Log, TEXT("Changing map to: %s"), *MapName);
}

void UMyConsoleCommandManager::HandleRestartServer(const TArray<FString>& Args, UWorld* World)
{
    FString CurrentMap = World->GetMapName();
    FString TravelURL = FString::Printf(TEXT("%s?Listen"), *CurrentMap);

    World->ServerTravel(TravelURL);

    UE_LOG(LogTemp, Log, TEXT("Server restarting..."));
}

void UMyConsoleCommandManager::HandleServerInfo(const TArray<FString>& Args, UWorld* World)
{
    UE_LOG(LogTemp, Log, TEXT("=== Server Information ==="));
    UE_LOG(LogTemp, Log, TEXT("  Map: %s"), *World->GetMapName());
    UE_LOG(LogTemp, Log, TEXT("  Players: %d"), World->GetPlayerControllerIterator().Num());
    UE_LOG(LogTemp, Log, TEXT("  Time: %s"), *FDateTime::Now().ToString());

    if (AGameModeBase* GM = World->GetAuthGameMode())
    {
        UE_LOG(LogTemp, Log, TEXT("  Game Mode: %s"), *GM->GetName());
    }

    // 网络信息
    if (UNetDriver* NetDriver = World->GetNetDriver())
    {
        UE_LOG(LogTemp, Log, TEXT("  Port: %d"), NetDriver->GetPort());
        UE_LOG(LogTemp, Log, TEXT("  Connections: %d"), NetDriver->ClientConnections.Num());
    }
}

void UMyConsoleCommandManager::HandleSetMaxPlayers(const TArray<FString>& Args, UWorld* World)
{
    if (Args.Num() < 1)
    {
        UE_LOG(LogTemp, Warning, TEXT("Usage: my.setmaxplayers <number>"));
        return;
    }

    int32 NewMax = FCString::Atoi(*Args[0]);

    if (AGameModeBase* GM = World->GetAuthGameMode())
    {
        // GM->MaxPlayers = NewMax;
        UE_LOG(LogTemp, Log, TEXT("Max players set to: %d"), NewMax);
    }
}

void UMyConsoleCommandManager::HandleBroadcastMessage(const TArray<FString>& Args, UWorld* World)
{
    if (Args.Num() < 1)
    {
        UE_LOG(LogTemp, Warning, TEXT("Usage: my.broadcast <message>"));
        return;
    }

    FString Message;
    for (const FString& Arg : Args)
    {
        Message += Arg + TEXT(" ");
    }

    // 广播消息
    for (FConstPlayerControllerIterator It = World->GetPlayerControllerIterator(); It; ++It)
    {
        if (APlayerController* PC = It->Get())
        {
            PC->ClientMessage(Message);
        }
    }

    UE_LOG(LogTemp, Log, TEXT("Broadcast: %s"), *Message);
}

void UMyConsoleCommandManager::HandlePauseMatch(const TArray<FString>& Args, UWorld* World)
{
    bool bPause = true;
    if (Args.Num() > 0)
    {
        bPause = Args[0].ToLower() == TEXT("true") || Args[0] == TEXT("1");
    }

    if (AGameModeBase* GM = World->GetAuthGameMode())
    {
        GM->SetPause(nullptr); // 或 GM->ClearPause()
        UE_LOG(LogTemp, Log, TEXT("Match %s"), bPause ? TEXT("paused") : TEXT("resumed"));
    }
}

void UMyConsoleCommandManager::HandleDebugMode(const TArray<FString>& Args, UWorld* World)
{
    if (Args.Num() < 1)
    {
        UE_LOG(LogTemp, Warning, TEXT("Usage: my.debug <on|off>"));
        return;
    }

    bool bEnable = Args[0].ToLower() == TEXT("on") || Args[0].ToLower() == TEXT("true");

    // 切换调试模式
    // SetDebugMode(bEnable);

    UE_LOG(LogTemp, Log, TEXT("Debug mode: %s"), bEnable ? TEXT("enabled") : TEXT("disabled"));
}
```

---

## 二、远程管理接口

### 2.1 RCON实现

```cpp
// MyRCONManager.h
#pragma once

#include "CoreMinimal.h"
#include "Sockets.h"
#include "SocketSubsystem.h"
#include "MyRCONManager.generated.h"

// RCON配置
USTRUCT()
struct FRCONConfig
{
    GENERATED_BODY()

    UPROPERTY(Config)
    bool bEnabled = true;

    UPROPERTY(Config)
    int32 Port = 27015;

    UPROPERTY(Config)
    FString Password = TEXT("admin");

    UPROPERTY(Config)
    int32 MaxConnections = 5;
};

// RCON客户端连接
struct FRCONConnection
{
    FSocket* Socket;
    FString Address;
    bool bAuthenticated;
    float LastActivity;
};

// RCON数据包
struct FRCONPacket
{
    int32 Size;
    int32 Id;
    int32 Type;
    FString Body;
};

UCLASS()
class UMyRCONManager : public UObject
{
    GENERATED_BODY()

public:
    UMyRCONManager();
    ~UMyRCONManager();

    void Initialize();
    void Shutdown();
    void Tick(float DeltaTime);

    bool IsRunning() const { return bIsRunning; }

private:
    FRCONConfig Config;
    FSocket* ListenSocket;
    TArray<FRCONConnection> Connections;
    bool bIsRunning;

    // 网络处理
    void AcceptNewConnections();
    void ReadIncomingData();
    void ProcessPacket(const FRCONPacket& Packet, FRCONConnection& Connection);
    void SendResponse(FSocket* Socket, int32 Id, int32 Type, const FString& Body);

    // 命令处理
    FString ExecuteCommand(const FString& Command);

    // 数据包解析
    bool ParsePacket(const TArray<uint8>& Data, FRCONPacket& OutPacket);
    TArray<uint8> BuildPacket(const FRCONPacket& Packet);
};

// MyRCONManager.cpp
#include "MyRCONManager.h"
#include "Common/TcpSocketBuilder.h"

UMyRCONManager::UMyRCONManager()
{
    ListenSocket = nullptr;
    bIsRunning = false;
}

UMyRCONManager::~UMyRCONManager()
{
    Shutdown();
}

void UMyRCONManager::Initialize()
{
    // 加载配置
    // Config = LoadConfig();

    if (!Config.bEnabled)
    {
        UE_LOG(LogTemp, Log, TEXT("RCON is disabled"));
        return;
    }

    // 创建监听Socket
    FString SocketName = FString::Printf(TEXT("RCON_%d"), Config.Port);

    ListenSocket = FTcpSocketBuilder(*SocketName)
        .AsReusable()
        .BoundToPort(Config.Port)
        .Listening(8);

    if (!ListenSocket)
    {
        UE_LOG(LogTemp, Error, TEXT("Failed to create RCON socket on port %d"), Config.Port);
        return;
    }

    ListenSocket->SetNonBlocking(true);

    bIsRunning = true;
    UE_LOG(LogTemp, Log, TEXT("RCON server started on port %d"), Config.Port);
}

void UMyRCONManager::Shutdown()
{
    if (ListenSocket)
    {
        ListenSocket->Close();
        ISocketSubsystem::Get(PLATFORM_SOCKETSUBSYSTEM)->DestroySocket(ListenSocket);
        ListenSocket = nullptr;
    }

    for (FRCONConnection& Conn : Connections)
    {
        if (Conn.Socket)
        {
            Conn.Socket->Close();
            ISocketSubsystem::Get(PLATFORM_SOCKETSUBSYSTEM)->DestroySocket(Conn.Socket);
        }
    }
    Connections.Empty();

    bIsRunning = false;
    UE_LOG(LogTemp, Log, TEXT("RCON server stopped"));
}

void UMyRCONManager::Tick(float DeltaTime)
{
    if (!bIsRunning)
    {
        return;
    }

    AcceptNewConnections();
    ReadIncomingData();
}

void UMyRCONManager::AcceptNewConnections()
{
    if (!ListenSocket)
    {
        return;
    }

    // 检查是否达到最大连接数
    if (Connections.Num() >= Config.MaxConnections)
    {
        return;
    }

    TSharedRef<FInternetAddr> RemoteAddr = ISocketSubsystem::Get(PLATFORM_SOCKETSUBSYSTEM)->CreateInternetAddr();
    FSocket* NewSocket = ListenSocket->Accept(*RemoteAddr, TEXT("RCON Client"));

    if (NewSocket)
    {
        NewSocket->SetNonBlocking(true);

        FRCONConnection Connection;
        Connection.Socket = NewSocket;
        Connection.Address = RemoteAddr->ToString(false);
        Connection.bAuthenticated = false;
        Connection.LastActivity = FPlatformTime::Seconds();

        Connections.Add(Connection);

        UE_LOG(LogTemp, Log, TEXT("RCON connection from: %s"), *Connection.Address);
    }
}

void UMyRCONManager::ReadIncomingData()
{
    TArray<uint8> Buffer;
    Buffer.SetNumUninitialized(4096);

    for (int32 i = Connections.Num() - 1; i >= 0; i--)
    {
        FRCONConnection& Conn = Connections[i];

        uint32 PendingDataSize = 0;
        if (Conn.Socket->HasPendingData(PendingDataSize))
        {
            int32 BytesRead = 0;
            if (Conn.Socket->Recv(Buffer.GetData(), Buffer.Num(), BytesRead))
            {
                Conn.LastActivity = FPlatformTime::Seconds();

                // 解析数据包
                FRCONPacket Packet;
                if (ParsePacket(Buffer, Packet))
                {
                    ProcessPacket(Packet, Conn);
                }
            }
        }

        // 检查超时
        if (FPlatformTime::Seconds() - Conn.LastActivity > 60.0f)
        {
            UE_LOG(LogTemp, Warning, TEXT("RCON connection timeout: %s"), *Conn.Address);
            Conn.Socket->Close();
            ISocketSubsystem::Get(PLATFORM_SOCKETSUBSYSTEM)->DestroySocket(Conn.Socket);
            Connections.RemoveAt(i);
        }
    }
}

void UMyRCONManager::ProcessPacket(const FRCONPacket& Packet, FRCONConnection& Connection)
{
    // 检查认证
    if (!Connection.bAuthenticated)
    {
        if (Packet.Type == 3) // SERVERDATA_AUTH
        {
            if (Packet.Body == Config.Password)
            {
                Connection.bAuthenticated = true;
                SendResponse(Connection.Socket, Packet.Id, 2, TEXT("")); // SERVERDATA_AUTH_RESPONSE
                UE_LOG(LogTemp, Log, TEXT("RCON client authenticated: %s"), *Connection.Address);
            }
            else
            {
                SendResponse(Connection.Socket, Packet.Id, 2, TEXT("")); // 认证失败
                UE_LOG(LogTemp, Warning, TEXT("RCON auth failed: %s"), *Connection.Address);
            }
        }
        return;
    }

    // 处理命令
    switch (Packet.Type)
    {
        case 2: // SERVERDATA_EXECCOMMAND
        {
            FString Result = ExecuteCommand(Packet.Body);
            SendResponse(Connection.Socket, Packet.Id, 0, Result); // SERVERDATA_RESPONSE_VALUE
            break;
        }
    }
}

void UMyRCONManager::SendResponse(FSocket* Socket, int32 Id, int32 Type, const FString& Body)
{
    FRCONPacket Response;
    Response.Id = Id;
    Response.Type = Type;
    Response.Body = Body;

    TArray<uint8> Data = BuildPacket(Response);

    int32 BytesSent = 0;
    Socket->Send(Data.GetData(), Data.Num(), BytesSent);
}

FString UMyRCONManager::ExecuteCommand(const FString& Command)
{
    // 执行控制台命令
    FString Output;
    FOutputDeviceRedirector OutputRedirect;

    OutputRedirect.AddLambda([&Output](ELogVerbosity::Type Verbosity, const FName& Category, const FString& Message)
    {
        Output += Message + LINE_TERMINATOR;
    });

    GLog->AddOutputDevice(&OutputRedirect);
    GEngine->Exec(nullptr, *Command);
    GLog->RemoveOutputDevice(&OutputRedirect);

    return Output;
}

bool UMyRCONManager::ParsePacket(const TArray<uint8>& Data, FRCONPacket& OutPacket)
{
    if (Data.Num() < 12) // 最小包大小
    {
        return false;
    }

    // RCON协议: Size(4) + Id(4) + Type(4) + Body + Null(2)
    uint8* Ptr = const_cast<uint8*>(Data.GetData());

    OutPacket.Size = *reinterpret_cast<int32*>(Ptr); Ptr += 4;
    OutPacket.Id = *reinterpret_cast<int32*>(Ptr); Ptr += 4;
    OutPacket.Type = *reinterpret_cast<int32*>(Ptr); Ptr += 4;

    // Body (以null结尾)
    OutPacket.Body = FString(UTF8_TO_TCHAR(reinterpret_cast<char*>(Ptr)));

    return true;
}

TArray<uint8> UMyRCONManager::BuildPacket(const FRCONPacket& Packet)
{
    TArray<uint8> Data;

    FTCHARToUTF8 BodyUTF8(*Packet.Body);
    int32 BodyLength = BodyUTF8.Length();

    int32 TotalSize = 4 + 4 + BodyLength + 2; // Id + Type + Body + Nulls

    Data.SetNumUninitialized(4 + TotalSize);

    uint8* Ptr = Data.GetData();
    *reinterpret_cast<int32*>(Ptr) = TotalSize; Ptr += 4;
    *reinterpret_cast<int32*>(Ptr) = Packet.Id; Ptr += 4;
    *reinterpret_cast<int32*>(Ptr) = Packet.Type; Ptr += 4;
    FMemory::Memcpy(Ptr, (const uint8*)BodyUTF8.Get(), BodyLength); Ptr += BodyLength;
    *reinterpret_cast<int16*>(Ptr) = 0; // 双null结尾

    return Data;
}
```

---

## 三、服务器日志与监控

### 3.1 日志系统

```cpp
// MyServerLogger.h
#pragma once

#include "CoreMinimal.h"
#include "MyServerLogger.generated.h"

// 日志级别
UENUM(BlueprintType)
enum class EServerLogLevel : uint8
{
    Debug,
    Info,
    Warning,
    Error,
    Critical
};

// 日志条目
USTRUCT()
struct FServerLogEntry
{
    GENERATED_BODY()

    FDateTime Timestamp;
    EServerLogLevel Level;
    FString Category;
    FString Message;
    FString Source;
    int32 LineNumber;
};

UCLASS()
class UMyServerLogger : public UObject
{
    GENERATED_BODY()

public:
    static UMyServerLogger* Get();

    void Initialize();
    void Shutdown();

    void Log(EServerLogLevel Level, const FString& Category, const FString& Message);
    void Logf(EServerLogLevel Level, const FString& Category, const FString& Format, ...);

    // 日志查询
    TArray<FServerLogEntry> GetRecentLogs(int32 Count = 100);
    TArray<FServerLogEntry> GetLogsByLevel(EServerLogLevel Level, int32 Count = 100);
    TArray<FServerLogEntry> GetLogsByCategory(const FString& Category, int32 Count = 100);

    // 日志管理
    void ClearLogs();
    void FlushToFile();
    void RotateLogFiles();

    // 配置
    UPROPERTY(Config)
    bool bLogToFile = true;

    UPROPERTY(Config)
    FString LogDirectory = TEXT("Saved/Logs");

    UPROPERTY(Config)
    int32 MaxLogFiles = 10;

    UPROPERTY(Config)
    int64 MaxLogFileSize = 10 * 1024 * 1024; // 10MB

private:
    TArray<FServerLogEntry> LogBuffer;
    FCriticalSection LogLock;
    FString CurrentLogFile;
    IFileHandle* LogFileHandle;

    FString LevelToString(EServerLogLevel Level);
    void WriteToConsole(const FServerLogEntry& Entry);
    void WriteToFile(const FServerLogEntry& Entry);
    FString FormatLogEntry(const FServerLogEntry& Entry);
};

// 全局日志宏
#define SERVER_LOG(Level, Category, Message) \
    UMyServerLogger::Get()->Log(EServerLogLevel::Level, TEXT(#Category), Message)

#define SERVER_LOGF(Level, Category, Format, ...) \
    UMyServerLogger::Get()->Logf(EServerLogLevel::Level, TEXT(#Category), Format, __VA_ARGS__)
```

### 3.2 性能监控

```cpp
// MyServerMonitor.h
#pragma once

#include "CoreMinimal.h"
#include "MyServerMonitor.generated.h"

// 性能指标
USTRUCT(BlueprintType)
struct FServerPerformanceMetrics
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    float FrameRate;

    UPROPERTY(BlueprintReadOnly)
    float FrameTime;

    UPROPERTY(BlueprintReadOnly)
    float ServerTickTime;

    UPROPERTY(BlueprintReadOnly)
    float CPUUsage;

    UPROPERTY(BlueprintReadOnly)
    int64 MemoryUsed;

    UPROPERTY(BlueprintReadOnly)
    int64 MemoryAvailable;

    UPROPERTY(BlueprintReadOnly)
    int32 PlayerCount;

    UPROPERTY(BlueprintReadOnly)
    int32 ActorCount;

    UPROPERTY(BlueprintReadOnly)
    float NetworkInKBps;

    UPROPERTY(BlueprintReadOnly)
    float NetworkOutKBps;

    UPROPERTY(BlueprintReadOnly)
    int32 RPCCount;

    UPROPERTY(BlueprintReadOnly)
    int32 ReplicatedActorCount;
};

// 告警配置
USTRUCT()
struct FAlertConfig
{
    GENERATED_BODY()

    float MinFrameRate = 20.0f;
    float MaxFrameTime = 0.05f;
    float MaxCPUUsage = 0.9f;
    int64 MaxMemoryMB = 4096;
    int32 MaxPlayers = 100;
};

// 告警类型
UENUM(BlueprintType)
enum class EAlertType : uint8
{
    LowFrameRate,
    HighFrameTime,
    HighCPU,
    HighMemory,
    HighPlayerCount,
    NetworkIssue
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnAlertTriggered, EAlertType, AlertType, const FString&, Message);

UCLASS()
class UMyServerMonitor : public UObject
{
    GENERATED_BODY()

public:
    static UMyServerMonitor* Get();

    void Initialize();
    void Shutdown();
    void Tick(float DeltaTime);

    // 获取指标
    UFUNCTION(BlueprintPure, Category = "Monitor")
    FServerPerformanceMetrics GetCurrentMetrics() const;

    UFUNCTION(BlueprintPure, Category = "Monitor")
    TArray<FServerPerformanceMetrics> GetMetricsHistory() const;

    // 告警
    UPROPERTY(BlueprintAssignable, Category = "Monitor")
    FOnAlertTriggered OnAlertTriggered;

    void CheckAlerts();
    void SetAlertConfig(const FAlertConfig& Config);

private:
    FServerPerformanceMetrics CurrentMetrics;
    TArray<FServerPerformanceMetrics> MetricsHistory;
    FAlertConfig AlertConfig;

    int32 MetricsHistorySize = 300; // 5分钟历史（每秒记录）
    float LastMetricsTime;

    void CollectMetrics();
    void CollectMemoryMetrics();
    void CollectNetworkMetrics();
    void CollectGameMetrics();
};
```

---

## 四、热重载与热修复

### 4.1 热重载系统

```cpp
// MyHotReloadSystem.h
#pragma once

#include "CoreMinimal.h"
#include "MyHotReloadSystem.generated.h"

// 热更新状态
UENUM(BlueprintType)
enum class EHotReloadState : uint8
{
    Idle,
    Checking,
    Downloading,
    Applying,
    Complete,
    Failed
};

// 热更新信息
USTRUCT(BlueprintType)
struct FHotReloadInfo
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString Version;

    UPROPERTY(BlueprintReadOnly)
    FString UpdateURL;

    UPROPERTY(BlueprintReadOnly)
    int64 TotalSize;

    UPROPERTY(BlueprintReadOnly)
    int64 DownloadedSize;

    UPROPERTY(BlueprintReadOnly)
    FString Checksum;

    UPROPERTY(BlueprintReadOnly)
    TArray<FString> ChangedFiles;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnHotReloadProgress, float, Progress);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnHotReloadComplete, bool, bSuccess, const FString&, Message);

UCLASS()
class UMyHotReloadSystem : public UObject
{
    GENERATED_BODY()

public:
    // 检查更新
    UFUNCTION(BlueprintCallable, Category = "HotReload")
    void CheckForUpdates(const FString& VersionURL);

    // 应用更新
    UFUNCTION(BlueprintCallable, Category = "HotReload")
    void ApplyUpdate();

    // 状态查询
    UFUNCTION(BlueprintPure, Category = "HotReload")
    EHotReloadState GetState() const { return CurrentState; }

    UFUNCTION(BlueprintPure, Category = "HotReload")
    FHotReloadInfo GetUpdateInfo() const { return UpdateInfo; }

    // 委托
    UPROPERTY(BlueprintAssignable)
    FOnHotReloadProgress OnProgress;

    UPROPERTY(BlueprintAssignable)
    FOnHotReloadComplete OnComplete;

private:
    EHotReloadState CurrentState = EHotReloadState::Idle;
    FHotReloadInfo UpdateInfo;

    void SetState(EHotReloadState NewState);
    void DownloadUpdate();
    void ApplyDownloadedFiles();
    bool VerifyChecksum(const FString& FilePath, const FString& ExpectedChecksum);
};

// 热重载Actor数据
UCLASS()
class AMyHotReloadableActor : public AActor
{
    GENERATED_BODY()

public:
    // 保存状态用于热重载后恢复
    virtual void SaveHotReloadState(TMap<FString, FString>& OutState);
    virtual void LoadHotReloadState(const TMap<FString, FString>& InState);

protected:
    UPROPERTY()
    TMap<FString, FString> SavedState;
};

void AMyHotReloadableActor::SaveHotReloadState(TMap<FString, FString>& OutState)
{
    OutState.Add(TEXT("Location"), GetActorLocation().ToString());
    OutState.Add(TEXT("Rotation"), GetActorRotation().ToString());
    OutState.Add(TEXT("Scale"), GetActorScale3D().ToString());
}

void AMyHotReloadableActor::LoadHotReloadState(const TMap<FString, FString>& InState)
{
    if (InState.Contains(TEXT("Location")))
    {
        FVector Location;
        Location.InitFromString(InState[TEXT("Location")]);
        SetActorLocation(Location);
    }

    if (InState.Contains(TEXT("Rotation")))
    {
        FRotator Rotation;
        Rotation.InitFromString(InState[TEXT("Rotation")]);
        SetActorRotation(Rotation);
    }
}
```

### 4.2 配置热更新

```cpp
// 运行时配置更新
UCLASS()
class UMyConfigManager : public UObject
{
    GENERATED_BODY()

public:
    // 热更新配置
    UFUNCTION(BlueprintCallable, Category = "Config")
    void ReloadConfig(const FString& ConfigName);

    UFUNCTION(BlueprintCallable, Category = "Config")
    void UpdateConfigValue(const FString& ConfigName, const FString& Section, const FString& Key, const FString& Value);

    UFUNCTION(BlueprintPure, Category = "Config")
    FString GetConfigValue(const FString& ConfigName, const FString& Section, const FString& Key);

    // 监听配置变化
    DECLARE_MULTICAST_DELEGATE_ThreeParams(FOnConfigChanged, const FString&, const FString&, const FString&);
    FOnConfigChanged OnConfigChanged;

private:
    void BroadcastConfigChange(const FString& ConfigName, const FString& Section, const FString& Key);
};

void UMyConfigManager::ReloadConfig(const FString& ConfigName)
{
    // 重新加载配置文件
    FString ConfigPath = FPaths::ProjectConfigDir() / ConfigName + TEXT(".ini");

    if (FPaths::FileExists(ConfigPath))
    {
        GConfig->LoadFile(*ConfigPath);
        UE_LOG(LogTemp, Log, TEXT("Config reloaded: %s"), *ConfigName);
    }
}

void UMyConfigManager::UpdateConfigValue(const FString& ConfigName, const FString& Section, const FString& Key, const FString& Value)
{
    FString ConfigPath = ConfigName;

    GConfig->SetString(*Section, *Key, *Value, *ConfigPath);
    GConfig->Flush(false, *ConfigPath);

    BroadcastConfigChange(ConfigName, Section, Key);

    UE_LOG(LogTemp, Log, TEXT("Config updated: [%s] %s=%s"), *Section, *Key, *Value);
}
```

---

## 五、总结

本课我们学习了：

1. **控制台命令**：自定义命令开发和注册
2. **远程管理**：RCON协议实现
3. **日志监控**：服务器日志和性能监控
4. **热重载**：运行时更新和配置管理

---

## 六、下节预告

**第11课：SaveGame系统深入**

将深入学习：
- USaveGame架构解析
- 自定义存档序列化
- 异步存档加载
- 云存档集成

---

*课程版本：1.0*
*最后更新：2026-04-10*