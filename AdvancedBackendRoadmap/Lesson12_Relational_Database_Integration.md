# 第12课：关系型数据库集成

> 本课深入讲解如何在UE5中集成关系型数据库，包括SQLite、MySQL和PostgreSQL的应用。

---

## 课程目标

- 掌握SQLite在UE5中的集成方法
- 学会MySQL/PostgreSQL连接方案
- 理解数据库连接池管理
- 掌握异步数据库操作
- 学会ORM思想与UE5结合

---

## 一、数据库集成概述

### 1.1 数据库选择指南

```
┌─────────────────────────────────────────────────────────────┐
│                    数据库选择指南                            │
│                                                              │
│  SQLite                                                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  特点：嵌入式、无服务器、单文件                       │    │
│  │  适用：单机游戏、本地存档、小型服务器                 │    │
│  │  优点：部署简单、零配置、体积小                       │    │
│  │  缺点：并发受限、无网络访问                          │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  MySQL / MariaDB                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  特点：客户端-服务器架构、高性能                      │    │
│  │  适用：中大型多人游戏、排行榜、用户系统               │    │
│  │  优点：高性能、丰富的工具生态                        │    │
│  │  缺点：需要独立服务器、配置复杂                      │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  PostgreSQL                                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  特点：高级SQL特性、扩展性强                         │    │
│  │  适用：需要复杂查询、JSON支持的项目                   │    │
│  │  优点：功能强大、ACID完整、JSON原生支持              │    │
│  │  缺点：资源占用较高、学习曲线陡峭                    │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、SQLite集成

### 2.1 SQLite插件配置

```cpp
// Build.cs 添加依赖
PublicDependencyModuleNames.AddRange(new string[]
{
    "Core",
    "CoreUObject",
    "Engine",
    "SQLiteSupport",  // SQLite支持
    "Database"        // 数据库抽象层
});

// SQLite配置
// DefaultEngine.ini
[/Script/SQLiteSupport.SQLiteSettings]
DatabasePath="Saved/SaveGames"
bReadOnly=false
```

### 2.2 SQLite数据库管理器

```cpp
// MySQLiteDatabase.h
#pragma once

#include "CoreMinimal.h"
#include "sqlite3.h"
#include "MySQLiteDatabase.generated.h"

// 查询结果行
USTRUCT(BlueprintType)
struct FDatabaseRow
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    TMap<FString, FString> Columns;
};

// 查询结果
USTRUCT(BlueprintType)
struct FDatabaseResult
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    bool bSuccess = false;

    UPROPERTY(BlueprintReadOnly)
    FString ErrorMessage;

    UPROPERTY(BlueprintReadOnly)
    TArray<FDatabaseRow> Rows;

    UPROPERTY(BlueprintReadOnly)
    int32 AffectedRows = 0;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnQueryComplete, const FDatabaseResult&, Result);

UCLASS()
class UMySQLiteManager : public UObject
{
    GENERATED_BODY()

public:
    UMySQLiteManager();
    ~UMySQLiteManager();

    // 数据库操作
    bool OpenDatabase(const FString& DatabasePath);
    void CloseDatabase();
    bool IsOpen() const { return Database != nullptr; }

    // 同步查询
    FDatabaseResult ExecuteQuery(const FString& Query);
    FDatabaseResult ExecuteQueryWithParams(const FString& Query, const TArray<FString>& Params);

    // 异步查询
    void ExecuteQueryAsync(const FString& Query, const FOnQueryComplete& Callback);

    // 事务
    bool BeginTransaction();
    bool CommitTransaction();
    bool RollbackTransaction();

    // 表操作
    bool CreateTable(const FString& TableName, const TMap<FString, FString>& Columns);
    bool DropTable(const FString& TableName);
    bool TableExists(const FString& TableName);

    // CRUD操作
    FDatabaseResult Insert(const FString& TableName, const TMap<FString, FString>& Values);
    FDatabaseResult Select(const FString& TableName, const TArray<FString>& Columns = {},
                           const FString& WhereClause = TEXT(""), const FString& OrderBy = TEXT(""));
    FDatabaseResult Update(const FString& TableName, const TMap<FString, FString>& Values,
                           const FString& WhereClause);
    FDatabaseResult Delete(const FString& TableName, const FString& WhereClause);

    // 辅助函数
    FString EscapeString(const FString& Input);
    FString GetLastError() const;

private:
    sqlite3* Database;
    FString DatabasePath;

    // 回调函数
    static int QueryCallback(void* Data, int Argc, char** Argv, char** ColName);

    // 辅助方法
    FDatabaseResult ExecuteInternal(const FString& Query);
    TArray<FDatabaseRow> ParseResults(sqlite3_stmt* Statement);
};

// MySQLiteDatabase.cpp
#include "MySQLiteDatabase.h"
#include "HAL/PlatformFileManager.h"

UMySQLiteManager::UMySQLiteManager()
{
    Database = nullptr;
}

UMySQLiteManager::~UMySQLiteManager()
{
    CloseDatabase();
}

bool UMySQLiteManager::OpenDatabase(const FString& DatabasePath)
{
    if (Database != nullptr)
    {
        CloseDatabase();
    }

    // 确保目录存在
    FString Directory = FPaths::GetPath(DatabasePath);
    IPlatformFile& PlatformFile = FPlatformFileManager::Get().GetPlatformFile();
    if (!PlatformFile.DirectoryExists(*Directory))
    {
        PlatformFile.CreateDirectory(*Directory);
    }

    // 打开数据库
    int Result = sqlite3_open(TCHAR_TO_UTF8(*DatabasePath), &Database);

    if (Result != SQLITE_OK)
    {
        UE_LOG(LogTemp, Error, TEXT("Failed to open database: %s"), *DatabasePath);
        sqlite3_close(Database);
        Database = nullptr;
        return false;
    }

    this->DatabasePath = DatabasePath;
    UE_LOG(LogTemp, Log, TEXT("Database opened: %s"), *DatabasePath);
    return true;
}

void UMySQLiteManager::CloseDatabase()
{
    if (Database != nullptr)
    {
        sqlite3_close(Database);
        Database = nullptr;
        UE_LOG(LogTemp, Log, TEXT("Database closed"));
    }
}

FDatabaseResult UMySQLiteManager::ExecuteQuery(const FString& Query)
{
    return ExecuteInternal(Query);
}

FDatabaseResult UMySQLiteManager::ExecuteQueryWithParams(const FString& Query, const TArray<FString>& Params)
{
    if (!IsOpen())
    {
        FDatabaseResult Result;
        Result.bSuccess = false;
        Result.ErrorMessage = TEXT("Database not open");
        return Result;
    }

    // 准备语句
    sqlite3_stmt* Statement;
    int Result = sqlite3_prepare_v2(Database, TCHAR_TO_UTF8(*Query), -1, &Statement, nullptr);

    if (Result != SQLITE_OK)
    {
        FDatabaseResult DBResult;
        DBResult.bSuccess = false;
        DBResult.ErrorMessage = FString(UTF8_TO_TCHAR(sqlite3_errmsg(Database)));
        return DBResult;
    }

    // 绑定参数
    for (int32 i = 0; i < Params.Num(); i++)
    {
        sqlite3_bind_text(Statement, i + 1, TCHAR_TO_UTF8(*Params[i]), -1, SQLITE_TRANSIENT);
    }

    // 执行查询
    FDatabaseResult DBResult;
    DBResult.bSuccess = true;
    DBResult.Rows = ParseResults(Statement);
    DBResult.AffectedRows = sqlite3_changes(Database);

    sqlite3_finalize(Statement);
    return DBResult;
}

void UMySQLiteManager::ExecuteQueryAsync(const FString& Query, const FOnQueryComplete& Callback)
{
    AsyncTask(ENamedThreads::AnyBackgroundThreadNormalTask, [this, Query, Callback]()
    {
        FDatabaseResult Result = ExecuteQuery(Query);

        AsyncTask(ENamedThreads::GameThread, [Result, Callback]()
        {
            Callback.Broadcast(Result);
        });
    });
}

bool UMySQLiteManager::BeginTransaction()
{
    if (!IsOpen()) return false;

    char* ErrMsg = nullptr;
    int Result = sqlite3_exec(Database, "BEGIN TRANSACTION", nullptr, nullptr, &ErrMsg);

    if (ErrMsg)
    {
        sqlite3_free(ErrMsg);
    }

    return Result == SQLITE_OK;
}

bool UMySQLiteManager::CommitTransaction()
{
    if (!IsOpen()) return false;

    char* ErrMsg = nullptr;
    int Result = sqlite3_exec(Database, "COMMIT", nullptr, nullptr, &ErrMsg);

    if (ErrMsg)
    {
        sqlite3_free(ErrMsg);
    }

    return Result == SQLITE_OK;
}

bool UMySQLiteManager::RollbackTransaction()
{
    if (!IsOpen()) return false;

    char* ErrMsg = nullptr;
    int Result = sqlite3_exec(Database, "ROLLBACK", nullptr, nullptr, &ErrMsg);

    if (ErrMsg)
    {
        sqlite3_free(ErrMsg);
    }

    return Result == SQLITE_OK;
}

bool UMySQLiteManager::CreateTable(const FString& TableName, const TMap<FString, FString>& Columns)
{
    if (!IsOpen() || Columns.Num() == 0) return false;

    FString ColumnDefs;
    for (const auto& Pair : Columns)
    {
        if (!ColumnDefs.IsEmpty())
        {
            ColumnDefs += TEXT(", ");
        }
        ColumnDefs += FString::Printf(TEXT("%s %s"), *Pair.Key, *Pair.Value);
    }

    FString Query = FString::Printf(TEXT("CREATE TABLE IF NOT EXISTS %s (%s)"),
        *TableName, *ColumnDefs);

    FDatabaseResult Result = ExecuteQuery(Query);
    return Result.bSuccess;
}

bool UMySQLiteManager::TableExists(const FString& TableName)
{
    FString Query = FString::Printf(
        TEXT("SELECT name FROM sqlite_master WHERE type='table' AND name='%s'"),
        *TableName
    );

    FDatabaseResult Result = ExecuteQuery(Query);
    return Result.Rows.Num() > 0;
}

FDatabaseResult UMySQLiteManager::Insert(const FString& TableName, const TMap<FString, FString>& Values)
{
    if (!IsOpen() || Values.Num() == 0)
    {
        FDatabaseResult Result;
        Result.bSuccess = false;
        return Result;
    }

    TArray<FString> Columns;
    TArray<FString> ValuePlaceholders;
    TArray<FString> ParamValues;

    for (const auto& Pair : Values)
    {
        Columns.Add(Pair.Key);
        ValuePlaceholders.Add(TEXT("?"));
        ParamValues.Add(Pair.Value);
    }

    FString Query = FString::Printf(TEXT("INSERT INTO %s (%s) VALUES (%s)"),
        *TableName,
        *FString::Join(Columns, TEXT(", ")),
        *FString::Join(ValuePlaceholders, TEXT(", "))
    );

    return ExecuteQueryWithParams(Query, ParamValues);
}

FDatabaseResult UMySQLiteManager::Select(const FString& TableName, const TArray<FString>& Columns,
                                          const FString& WhereClause, const FString& OrderBy)
{
    FString ColumnList = Columns.Num() > 0 ? FString::Join(Columns, TEXT(", ")) : TEXT("*");

    FString Query = FString::Printf(TEXT("SELECT %s FROM %s"), *ColumnList, *TableName);

    if (!WhereClause.IsEmpty())
    {
        Query += FString::Printf(TEXT(" WHERE %s"), *WhereClause);
    }

    if (!OrderBy.IsEmpty())
    {
        Query += FString::Printf(TEXT(" ORDER BY %s"), *OrderBy);
    }

    return ExecuteQuery(Query);
}

FDatabaseResult UMySQLiteManager::Update(const FString& TableName, const TMap<FString, FString>& Values,
                                          const FString& WhereClause)
{
    if (!IsOpen() || Values.Num() == 0)
    {
        FDatabaseResult Result;
        Result.bSuccess = false;
        return Result;
    }

    TArray<FString> SetClauses;
    TArray<FString> ParamValues;

    for (const auto& Pair : Values)
    {
        SetClauses.Add(FString::Printf(TEXT("%s = ?"), *Pair.Key));
        ParamValues.Add(Pair.Value);
    }

    FString Query = FString::Printf(TEXT("UPDATE %s SET %s"),
        *TableName,
        *FString::Join(SetClauses, TEXT(", "))
    );

    if (!WhereClause.IsEmpty())
    {
        Query += FString::Printf(TEXT(" WHERE %s"), *WhereClause);
    }

    return ExecuteQueryWithParams(Query, ParamValues);
}

FDatabaseResult UMySQLiteManager::Delete(const FString& TableName, const FString& WhereClause)
{
    FString Query = FString::Printf(TEXT("DELETE FROM %s"), *TableName);

    if (!WhereClause.IsEmpty())
    {
        Query += FString::Printf(TEXT(" WHERE %s"), *WhereClause);
    }

    return ExecuteQuery(Query);
}

FString UMySQLiteManager::EscapeString(const FString& Input)
{
    FString Escaped;
    Escaped.Reserve(Input.Len() * 2);

    for (const TCHAR& Char : Input)
    {
        if (Char == '\'')
        {
            Escaped += TEXT("''");
        }
        else if (Char == '\\')
        {
            Escaped += TEXT("\\\\");
        }
        else
        {
            Escaped += Char;
        }
    }

    return Escaped;
}

FString UMySQLiteManager::GetLastError() const
{
    if (Database)
    {
        return FString(UTF8_TO_TCHAR(sqlite3_errmsg(Database)));
    }
    return TEXT("Database not open");
}

FDatabaseResult UMySQLiteManager::ExecuteInternal(const FString& Query)
{
    FDatabaseResult Result;

    if (!IsOpen())
    {
        Result.bSuccess = false;
        Result.ErrorMessage = TEXT("Database not open");
        return Result;
    }

    char* ErrMsg = nullptr;
    int SQLiteResult = sqlite3_exec(Database, TCHAR_TO_UTF8(*Query), QueryCallback, &Result, &ErrMsg);

    if (SQLiteResult != SQLITE_OK)
    {
        Result.bSuccess = false;
        Result.ErrorMessage = ErrMsg ? FString(UTF8_TO_TCHAR(ErrMsg)) : TEXT("Unknown error");
        if (ErrMsg)
        {
            sqlite3_free(ErrMsg);
        }
    }
    else
    {
        Result.bSuccess = true;
        Result.AffectedRows = sqlite3_changes(Database);
    }

    return Result;
}

TArray<FDatabaseRow> UMySQLiteManager::ParseResults(sqlite3_stmt* Statement)
{
    TArray<FDatabaseRow> Rows;

    while (sqlite3_step(Statement) == SQLITE_ROW)
    {
        FDatabaseRow Row;
        int ColumnCount = sqlite3_column_count(Statement);

        for (int i = 0; i < ColumnCount; i++)
        {
            FString ColumnName = FString(UTF8_TO_TCHAR(sqlite3_column_name(Statement, i)));
            FString ColumnValue;

            const char* Value = reinterpret_cast<const char*>(sqlite3_column_text(Statement, i));
            if (Value)
            {
                ColumnValue = FString(UTF8_TO_TCHAR(Value));
            }

            Row.Columns.Add(ColumnName, ColumnValue);
        }

        Rows.Add(Row);
    }

    return Rows;
}

int UMySQLiteManager::QueryCallback(void* Data, int Argc, char** Argv, char** ColName)
{
    FDatabaseResult* Result = static_cast<FDatabaseResult*>(Data);
    FDatabaseRow Row;

    for (int i = 0; i < Argc; i++)
    {
        FString ColumnName = FString(UTF8_TO_TCHAR(ColName[i]));
        FString ColumnValue = Argv[i] ? FString(UTF8_TO_TCHAR(Argv[i])) : TEXT("");
        Row.Columns.Add(ColumnName, ColumnValue);
    }

    Result->Rows.Add(Row);
    return 0;
}
```

---

## 三、MySQL/PostgreSQL集成

### 3.1 通用数据库接口

```cpp
// MyDatabaseInterface.h
#pragma once

#include "CoreMinimal.h"
#include "MyDatabaseInterface.generated.h"

// 数据库配置
USTRUCT(BlueprintType)
struct FDatabaseConfig
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString Host = TEXT("localhost");

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 Port = 3306;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString Database;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString Username;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString Password;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 ConnectionPoolSize = 10;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 ConnectionTimeout = 30;
};

// 数据库连接接口
UINTERFACE()
class UDatabaseConnection : public UInterface
{
    GENERATED_BODY()
};

class IDatabaseConnection
{
    GENERATED_BODY()

public:
    virtual bool Connect(const FDatabaseConfig& Config) = 0;
    virtual void Disconnect() = 0;
    virtual bool IsConnected() const = 0;
    virtual FDatabaseResult ExecuteQuery(const FString& Query) = 0;
    virtual void ExecuteQueryAsync(const FString& Query, TFunction<void(const FDatabaseResult&)> Callback) = 0;
};

// MySQL连接实现
UCLASS()
class UMySQLConnection : public UObject, public IDatabaseConnection
{
    GENERATED_BODY()

public:
    virtual bool Connect(const FDatabaseConfig& Config) override;
    virtual void Disconnect() override;
    virtual bool IsConnected() const override;
    virtual FDatabaseResult ExecuteQuery(const FString& Query) override;
    virtual void ExecuteQueryAsync(const FString& Query, TFunction<void(const FDatabaseResult&)> Callback) override;

private:
    // MYSQL* Connection;
    FDatabaseConfig Config;
};

// PostgreSQL连接实现
UCLASS()
class UPostgreSQLConnection : public UObject, public IDatabaseConnection
{
    GENERATED_BODY()

public:
    virtual bool Connect(const FDatabaseConfig& Config) override;
    virtual void Disconnect() override;
    virtual bool IsConnected() const override;
    virtual FDatabaseResult ExecuteQuery(const FString& Query) override;
    virtual void ExecuteQueryAsync(const FString& Query, TFunction<void(const FDatabaseResult&)> Callback) override;

private:
    // PGconn* Connection;
    FDatabaseConfig Config;
};
```

### 3.2 连接池管理

```cpp
// MyDatabaseConnectionPool.h
#pragma once

#include "CoreMinimal.h"
#include "MyDatabaseInterface.h"
#include "MyDatabaseConnectionPool.generated.h"

UCLASS()
class UDatabaseConnectionPool : public UObject
{
    GENERATED_BODY()

public:
    // 初始化连接池
    bool Initialize(const FDatabaseConfig& Config, int32 PoolSize);

    // 获取连接
    TScriptInterface<IDatabaseConnection> AcquireConnection();

    // 释放连接
    void ReleaseConnection(TScriptInterface<IDatabaseConnection> Connection);

    // 关闭所有连接
    void Shutdown();

    // 状态查询
    int32 GetAvailableConnections() const;
    int32 GetTotalConnections() const;
    bool IsHealthy() const;

private:
    TArray<TScriptInterface<IDatabaseConnection>> AvailableConnections;
    TArray<TScriptInterface<IDatabaseConnection>> UsedConnections;
    FDatabaseConfig Config;
    FCriticalSection PoolLock;
    int32 MaxPoolSize = 10;

    TScriptInterface<IDatabaseConnection> CreateNewConnection();
};

// 实现
bool UDatabaseConnectionPool::Initialize(const FDatabaseConfig& InConfig, int32 PoolSize)
{
    Config = InConfig;
    MaxPoolSize = PoolSize;

    // 预创建连接
    for (int32 i = 0; i < PoolSize; i++)
    {
        TScriptInterface<IDatabaseConnection> Connection = CreateNewConnection();
        if (Connection)
        {
            AvailableConnections.Add(Connection);
        }
        else
        {
            UE_LOG(LogTemp, Warning, TEXT("Failed to create connection %d"), i);
        }
    }

    UE_LOG(LogTemp, Log, TEXT("Connection pool initialized with %d connections"), AvailableConnections.Num());
    return AvailableConnections.Num() > 0;
}

TScriptInterface<IDatabaseConnection> UDatabaseConnectionPool::AcquireConnection()
{
    FScopeLock Lock(&PoolLock);

    if (AvailableConnections.Num() > 0)
    {
        TScriptInterface<IDatabaseConnection> Connection = AvailableConnections.Pop();
        UsedConnections.Add(Connection);
        return Connection;
    }

    // 如果没有可用连接且未达到最大值，创建新连接
    if (UsedConnections.Num() < MaxPoolSize)
    {
        TScriptInterface<IDatabaseConnection> Connection = CreateNewConnection();
        if (Connection)
        {
            UsedConnections.Add(Connection);
            return Connection;
        }
    }

    UE_LOG(LogTemp, Warning, TEXT("No available connections in pool"));
    return nullptr;
}

void UDatabaseConnectionPool::ReleaseConnection(TScriptInterface<IDatabaseConnection> Connection)
{
    FScopeLock Lock(&PoolLock);

    if (UsedConnections.Remove(Connection) > 0)
    {
        AvailableConnections.Add(Connection);
    }
}

TScriptInterface<IDatabaseConnection> UDatabaseConnectionPool::CreateNewConnection()
{
    // 根据配置创建对应类型的连接
    UMySQLConnection* MySQLConn = NewObject<UMySQLConnection>();
    if (MySQLConn->Connect(Config))
    {
        return TScriptInterface<IDatabaseConnection>(MySQLConn);
    }

    return nullptr;
}

void UDatabaseConnectionPool::Shutdown()
{
    FScopeLock Lock(&PoolLock);

    for (auto& Connection : AvailableConnections)
    {
        Connection->Disconnect();
    }
    AvailableConnections.Empty();

    for (auto& Connection : UsedConnections)
    {
        Connection->Disconnect();
    }
    UsedConnections.Empty();
}
```

---

## 四、异步数据库操作

### 4.1 异步查询系统

```cpp
// MyAsyncDatabaseManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "MyAsyncDatabaseManager.generated.h"

// 查询优先级
enum class EQueryPriority : uint8
{
    Low,
    Normal,
    High,
    Critical
};

// 待执行查询
struct FPendingQuery
{
    FString Query;
    EQueryPriority Priority;
    TFunction<void(const FDatabaseResult&)> Callback;
    float Timestamp;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnDatabaseError, const FString&, ErrorMessage);

UCLASS()
class UMyAsyncDatabaseManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 配置
    UFUNCTION(BlueprintCallable, Category = "Database")
    void Configure(const FDatabaseConfig& Config, int32 PoolSize = 10);

    // 异步查询
    void QueryAsync(const FString& Query, TFunction<void(const FDatabaseResult&)> Callback,
                    EQueryPriority Priority = EQueryPriority::Normal);

    // 批量查询
    void BatchQueryAsync(const TArray<FString>& Queries, TFunction<void(const TArray<FDatabaseResult>&)> Callback);

    // 事务
    void TransactionAsync(const TArray<FString>& Queries, TFunction<void(bool, const FString&)> Callback);

    // 状态
    UFUNCTION(BlueprintPure, Category = "Database")
    bool IsConnected() const;

    UFUNCTION(BlueprintPure, Category = "Database")
    int32 GetPendingQueries() const;

    // 委托
    UPROPERTY(BlueprintAssignable)
    FOnDatabaseError OnDatabaseError;

private:
    UPROPERTY()
    UDatabaseConnectionPool* ConnectionPool;

    TArray<FPendingQuery> QueryQueue;
    FCriticalSection QueueLock;
    FTimerHandle ProcessTimer;
    bool bIsProcessing;

    void ProcessQueue();
    void ExecuteQuery(const FPendingQuery& PendingQuery);
    void SortQueue();
};

// 实现
void UMyAsyncDatabaseManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    ConnectionPool = NewObject<UDatabaseConnectionPool>(this);
    bIsProcessing = false;

    // 启动查询处理定时器
    GetWorld()->GetTimerManager().SetTimer(
        ProcessTimer,
        this,
        &UMyAsyncDatabaseManager::ProcessQueue,
        0.016f, // 约60fps
        true
    );
}

void UMyAsyncDatabaseManager::Deinitialize()
{
    GetWorld()->GetTimerManager().ClearTimer(ProcessTimer);

    if (ConnectionPool)
    {
        ConnectionPool->Shutdown();
    }

    Super::Deinitialize();
}

void UMyAsyncDatabaseManager::Configure(const FDatabaseConfig& Config, int32 PoolSize)
{
    if (ConnectionPool)
    {
        ConnectionPool->Initialize(Config, PoolSize);
    }
}

void UMyAsyncDatabaseManager::QueryAsync(const FString& Query,
                                          TFunction<void(const FDatabaseResult&)> Callback,
                                          EQueryPriority Priority)
{
    FPendingQuery PendingQuery;
    PendingQuery.Query = Query;
    PendingQuery.Priority = Priority;
    PendingQuery.Callback = Callback;
    PendingQuery.Timestamp = FPlatformTime::Seconds();

    {
        FScopeLock Lock(&QueueLock);
        QueryQueue.Add(PendingQuery);
    }
}

void UMyAsyncDatabaseManager::ProcessQueue()
{
    if (bIsProcessing || !ConnectionPool)
    {
        return;
    }

    FScopeLock Lock(&QueueLock);

    if (QueryQueue.Num() == 0)
    {
        return;
    }

    // 按优先级排序
    SortQueue();

    // 获取连接并执行
    TScriptInterface<IDatabaseConnection> Connection = ConnectionPool->AcquireConnection();
    if (!Connection)
    {
        return;
    }

    bIsProcessing = true;

    FPendingQuery Query = QueryQueue.Pop();

    // 在后台线程执行
    AsyncTask(ENamedThreads::AnyBackgroundThreadNormalTask,
        [this, Connection, Query]()
        {
            FDatabaseResult Result = Connection->ExecuteQuery(Query.Query);

            // 回到游戏线程调用回调
            AsyncTask(ENamedThreads::GameThread, [this, Connection, Query, Result]()
            {
                Query.Callback(Result);
                ConnectionPool->ReleaseConnection(Connection);
                bIsProcessing = false;
            });
        }
    );
}

void UMyAsyncDatabaseManager::SortQueue()
{
    QueryQueue.Sort([](const FPendingQuery& A, const FPendingQuery& B)
    {
        if (A.Priority != B.Priority)
        {
            return static_cast<int32>(A.Priority) > static_cast<int32>(B.Priority);
        }
        return A.Timestamp < B.Timestamp;
    });
}
```

---

## 五、ORM思想与UE5结合

### 5.1 简易ORM实现

```cpp
// MyORM.h
#pragma once

#include "CoreMinimal.h"
#include "MyORM.generated.h"

// ORM属性映射
USTRUCT()
struct FORMFieldMapping
{
    GENERATED_BODY()

    FString ColumnName;
    FString ColumnType;
    bool bPrimaryKey = false;
    bool bAutoIncrement = false;
    bool bNullable = false;
};

// ORM模型基类
UCLASS(Abstract)
class UORMModel : public UObject
{
    GENERATED_BODY()

public:
    // 表名
    virtual FString GetTableName() const { return TEXT(""); }

    // 主键
    UPROPERTY()
    int64 Id = 0;

    // 序列化到数据库格式
    virtual TMap<FString, FString> ToDatabase() const { return {}; }

    // 从数据库格式反序列化
    virtual void FromDatabase(const TMap<FString, FString>& Data) {}

    // CRUD操作
    static TArray<UORMModel*> All(TSubclassOf<UORMModel> ModelClass);
    static UORMModel* Find(TSubclassOf<UORMModel> ModelClass, int64 Id);
    bool Save();
    bool Delete();
};

// 示例模型
UCLASS()
class UPlayerModel : public UORMModel
{
    GENERATED_BODY()

public:
    virtual FString GetTableName() const override { return TEXT("players"); }

    UPROPERTY()
    FString PlayerName;

    UPROPERTY()
    FString SteamId;

    UPROPERTY()
    int32 Level = 1;

    UPROPERTY()
    float Experience = 0.0f;

    UPROPERTY()
    FString LastLoginTime;

    virtual TMap<FString, FString> ToDatabase() const override
    {
        TMap<FString, FString> Data;
        Data.Add(TEXT("id"), FString::FromInt(Id));
        Data.Add(TEXT("player_name"), PlayerName);
        Data.Add(TEXT("steam_id"), SteamId);
        Data.Add(TEXT("level"), FString::FromInt(Level));
        Data.Add(TEXT("experience"), FString::SanitizeFloat(Experience));
        Data.Add(TEXT("last_login_time"), LastLoginTime);
        return Data;
    }

    virtual void FromDatabase(const TMap<FString, FString>& Data) override
    {
        if (Data.Contains(TEXT("id")))
        {
            Id = FCString::Atoi64(*Data[TEXT("id")]);
        }
        if (Data.Contains(TEXT("player_name")))
        {
            PlayerName = Data[TEXT("player_name")];
        }
        if (Data.Contains(TEXT("steam_id")))
        {
            SteamId = Data[TEXT("steam_id")];
        }
        if (Data.Contains(TEXT("level")))
        {
            Level = FCString::Atoi(*Data[TEXT("level")]);
        }
        if (Data.Contains(TEXT("experience")))
        {
            Experience = FCString::Atof(*Data[TEXT("experience")]);
        }
        if (Data.Contains(TEXT("last_login_time")))
        {
            LastLoginTime = Data[TEXT("last_login_time")];
        }
    }
};

// ORM查询构建器
UCLASS()
class UORMQueryBuilder : public UObject
{
    GENERATED_BODY()

public:
    UORMQueryBuilder(TSubclassOf<UORMModel> InModelClass);

    // 条件
    UORMQueryBuilder* Where(const FString& Column, const FString& Operator, const FString& Value);
    UORMQueryBuilder* WhereIn(const FString& Column, const TArray<FString>& Values);
    UORMQueryBuilder* WhereBetween(const FString& Column, const FString& Min, const FString& Max);

    // 排序和限制
    UORMQueryBuilder* OrderBy(const FString& Column, bool bDescending = false);
    UORMQueryBuilder* Limit(int32 Count);
    UORMQueryBuilder* Offset(int32 Count);

    // 执行
    TArray<UORMModel*> Get();
    UORMModel* First();
    int64 Count();
    bool Exists();

    // 聚合
    float Sum(const FString& Column);
    float Avg(const FString& Column);
    float Min(const FString& Column);
    float Max(const FString& Column);

private:
    TSubclassOf<UORMModel> ModelClass;
    TArray<FString> WhereClauses;
    TArray<FString> OrderClauses;
    int32 LimitCount = -1;
    int32 OffsetCount = 0;

    FString BuildQuery() const;
};
```

---

## 六、总结

本课我们学习了：

1. **SQLite集成**：嵌入式数据库的应用
2. **MySQL/PostgreSQL**：客户端-服务器数据库连接
3. **连接池**：高效的数据库连接管理
4. **异步操作**：非阻塞数据库查询
5. **ORM思想**：对象关系映射的实现

---

## 七、下节预告

**第13课：NoSQL数据库集成**

将深入学习：
- Redis集成与缓存策略
- MongoDB文档存储
- DynamoDB云数据库

---

*课程版本：1.0*
*最后更新：2026-04-10*