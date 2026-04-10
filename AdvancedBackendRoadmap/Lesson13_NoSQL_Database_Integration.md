# 第13课：NoSQL数据库集成

> 本课深入讲解NoSQL数据库在UE5中的应用，包括Redis缓存、MongoDB文档存储和DynamoDB云数据库。

---

## 课程目标

- 掌握Redis集成与缓存策略
- 学会MongoDB文档存储应用
- 理解DynamoDB云数据库使用
- 掌握数据模型设计最佳实践
- 学会数据库性能优化

---

## 一、NoSQL数据库概述

### 1.1 NoSQL vs 关系型数据库

```
┌─────────────────────────────────────────────────────────────┐
│                    数据库类型对比                            │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                 关系型数据库 (SQL)                   │    │
│  │  特点：表格结构、ACID事务、SQL查询                   │    │
│  │  适用：复杂关系、事务要求高、数据结构稳定             │    │
│  │  示例：MySQL、PostgreSQL、SQLite                     │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   Redis (键值存储)                   │    │
│  │  特点：内存存储、超快速度、丰富数据类型               │    │
│  │  适用：缓存、会话、排行榜、实时数据                   │    │
│  │  优点：极低延迟、支持持久化                          │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                 MongoDB (文档存储)                   │    │
│  │  特点：JSON文档、灵活Schema、水平扩展                 │    │
│  │  适用：用户数据、日志、内容管理、多变结构             │    │
│  │  优点：Schema灵活、开发效率高                        │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                DynamoDB (云数据库)                   │    │
│  │  特点：全托管、自动扩展、低延迟                       │    │
│  │  适用：游戏后端、用户数据、全球部署                   │    │
│  │  优点：无需运维、高可用、按需付费                    │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、Redis集成

### 2.1 Redis客户端实现

```cpp
// MyRedisClient.h
#pragma once

#include "CoreMinimal.h"
#include "MyRedisClient.generated.h"

// Redis配置
USTRUCT(BlueprintType)
struct FRedisConfig
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString Host = TEXT("localhost");

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 Port = 6379;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString Password;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 Database = 0;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 ConnectionTimeout = 5000; // 毫秒

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 OperationTimeout = 3000; // 毫秒
};

// Redis数据类型
UENUM(BlueprintType)
enum class ERedisDataType : uint8
{
    String,
    Hash,
    List,
    Set,
    SortedSet
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnRedisConnected, bool, bSuccess);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnRedisError, const FString&, Operation, const FString&, Message);

UCLASS()
class UMyRedisClient : public UObject
{
    GENERATED_BODY()

public:
    // 连接管理
    UFUNCTION(BlueprintCallable, Category = "Redis")
    bool Connect(const FRedisConfig& Config);

    UFUNCTION(BlueprintCallable, Category = "Redis")
    void Disconnect();

    UFUNCTION(BlueprintPure, Category = "Redis")
    bool IsConnected() const;

    // 字符串操作
    UFUNCTION(BlueprintCallable, Category = "Redis")
    bool Set(const FString& Key, const FString& Value, int32 TTL = -1);

    UFUNCTION(BlueprintCallable, Category = "Redis")
    FString Get(const FString& Key);

    UFUNCTION(BlueprintCallable, Category = "Redis")
    bool Delete(const FString& Key);

    UFUNCTION(BlueprintCallable, Category = "Redis")
    bool Exists(const FString& Key);

    // 过期时间
    UFUNCTION(BlueprintCallable, Category = "Redis")
    bool Expire(const FString& Key, int32 Seconds);

    UFUNCTION(BlueprintCallable, Category = "Redis")
    int32 TTL(const FString& Key);

    // 哈希操作
    UFUNCTION(BlueprintCallable, Category = "Redis")
    bool HSet(const FString& Key, const FString& Field, const FString& Value);

    UFUNCTION(BlueprintCallable, Category = "Redis")
    FString HGet(const FString& Key, const FString& Field);

    UFUNCTION(BlueprintCallable, Category = "Redis")
    TMap<FString, FString> HGetAll(const FString& Key);

    UFUNCTION(BlueprintCallable, Category = "Redis")
    bool HDel(const FString& Key, const FString& Field);

    // 列表操作
    UFUNCTION(BlueprintCallable, Category = "Redis")
    int64 LPush(const FString& Key, const FString& Value);

    UFUNCTION(BlueprintCallable, Category = "Redis")
    int64 RPush(const FString& Key, const FString& Value);

    UFUNCTION(BlueprintCallable, Category = "Redis")
    FString LPop(const FString& Key);

    UFUNCTION(BlueprintCallable, Category = "Redis")
    FString RPop(const FString& Key);

    UFUNCTION(BlueprintCallable, Category = "Redis")
    TArray<FString> LRange(const FString& Key, int64 Start, int64 Stop);

    // 有序集合（排行榜）
    UFUNCTION(BlueprintCallable, Category = "Redis")
    bool ZAdd(const FString& Key, float Score, const FString& Member);

    UFUNCTION(BlueprintCallable, Category = "Redis")
    TArray<FString> ZRange(const FString& Key, int64 Start, int64 Stop, bool bReverse = false);

    UFUNCTION(BlueprintCallable, Category = "Redis")
    TArray<TPair<FString, float>> ZRangeWithScores(const FString& Key, int64 Start, int64 Stop, bool bReverse = false);

    UFUNCTION(BlueprintCallable, Category = "Redis")
    int64 ZRank(const FString& Key, const FString& Member, bool bReverse = false);

    UFUNCTION(BlueprintCallable, Category = "Redis")
    float ZScore(const FString& Key, const FString& Member);

    // 原子操作
    UFUNCTION(BlueprintCallable, Category = "Redis")
    int64 Increment(const FString& Key, int64 Amount = 1);

    UFUNCTION(BlueprintCallable, Category = "Redis")
    float IncrementFloat(const FString& Key, float Amount = 1.0f);

    // 异步操作
    void GetAsync(const FString& Key, TFunction<void(FString)> Callback);
    void SetAsync(const FString& Key, const FString& Value, TFunction<void(bool)> Callback, int32 TTL = -1);

    // 委托
    UPROPERTY(BlueprintAssignable)
    FOnRedisConnected OnConnected;

    UPROPERTY(BlueprintAssignable)
    FOnRedisError OnError;

private:
    void* RedisContext; // redisContext*
    FRedisConfig CurrentConfig;

    FString BuildCommand(const TArray<FString>& Args);
    bool ExecuteCommand(const FString& Command);
};

// MyRedisClient.cpp
#include "MyRedisClient.h"
#include "hiredis.h"

bool UMyRedisClient::Connect(const FRedisConfig& Config)
{
    if (RedisContext != nullptr)
    {
        Disconnect();
    }

    CurrentConfig = Config;

    // 创建连接
    struct timeval Timeout = {Config.ConnectionTimeout / 1000, (Config.ConnectionTimeout % 1000) * 1000};
    RedisContext = redisConnectWithTimeout(TCHAR_TO_UTF8(*Config.Host), Config.Port, Timeout);

    if (RedisContext == nullptr || reinterpret_cast<redisContext*>(RedisContext)->err)
    {
        FString ErrorMsg = RedisContext ?
            FString(UTF8_TO_TCHAR(reinterpret_cast<redisContext*>(RedisContext)->errstr)) :
            TEXT("Failed to allocate redis context");

        UE_LOG(LogTemp, Error, TEXT("Redis connection failed: %s"), *ErrorMsg);
        OnError.Broadcast(TEXT("Connect"), ErrorMsg);
        OnConnected.Broadcast(false);

        if (RedisContext)
        {
            redisFree(reinterpret_cast<redisContext*>(RedisContext));
            RedisContext = nullptr;
        }
        return false;
    }

    // 认证
    if (!Config.Password.IsEmpty())
    {
        redisReply* Reply = reinterpret_cast<redisReply*>(
            redisCommand(reinterpret_cast<redisContext*>(RedisContext), "AUTH %s", TCHAR_TO_UTF8(*Config.Password))
        );

        if (Reply == nullptr || Reply->type == REDIS_REPLY_ERROR)
        {
            OnError.Broadcast(TEXT("Auth"), TEXT("Authentication failed"));
            OnConnected.Broadcast(false);
            Disconnect();
            return false;
        }

        freeReplyObject(Reply);
    }

    // 选择数据库
    if (Config.Database > 0)
    {
        redisReply* Reply = reinterpret_cast<redisReply*>(
            redisCommand(reinterpret_cast<redisContext*>(RedisContext), "SELECT %d", Config.Database)
        );
        if (Reply)
        {
            freeReplyObject(Reply);
        }
    }

    OnConnected.Broadcast(true);
    UE_LOG(LogTemp, Log, TEXT("Redis connected to %s:%d"), *Config.Host, Config.Port);
    return true;
}

void UMyRedisClient::Disconnect()
{
    if (RedisContext != nullptr)
    {
        redisFree(reinterpret_cast<redisContext*>(RedisContext));
        RedisContext = nullptr;
        UE_LOG(LogTemp, Log, TEXT("Redis disconnected"));
    }
}

bool UMyRedisClient::IsConnected() const
{
    return RedisContext != nullptr;
}

bool UMyRedisClient::Set(const FString& Key, const FString& Value, int32 TTL)
{
    if (!IsConnected()) return false;

    redisReply* Reply;

    if (TTL > 0)
    {
        Reply = reinterpret_cast<redisReply*>(
            redisCommand(reinterpret_cast<redisContext*>(RedisContext),
                "SET %s %s EX %d",
                TCHAR_TO_UTF8(*Key),
                TCHAR_TO_UTF8(*Value),
                TTL)
        );
    }
    else
    {
        Reply = reinterpret_cast<redisReply*>(
            redisCommand(reinterpret_cast<redisContext*>(RedisContext),
                "SET %s %s",
                TCHAR_TO_UTF8(*Key),
                TCHAR_TO_UTF8(*Value))
        );
    }

    bool bSuccess = (Reply != nullptr && Reply->type == REDIS_REPLY_STATUS);
    if (Reply)
    {
        freeReplyObject(Reply);
    }

    return bSuccess;
}

FString UMyRedisClient::Get(const FString& Key)
{
    if (!IsConnected()) return TEXT("");

    redisReply* Reply = reinterpret_cast<redisReply*>(
        redisCommand(reinterpret_cast<redisContext*>(RedisContext), "GET %s", TCHAR_TO_UTF8(*Key))
    );

    FString Result;
    if (Reply != nullptr && Reply->type == REDIS_REPLY_STRING)
    {
        Result = FString(UTF8_TO_TCHAR(Reply->str));
    }

    if (Reply)
    {
        freeReplyObject(Reply);
    }

    return Result;
}

bool UMyRedisClient::ZAdd(const FString& Key, float Score, const FString& Member)
{
    if (!IsConnected()) return false;

    redisReply* Reply = reinterpret_cast<redisReply*>(
        redisCommand(reinterpret_cast<redisContext*>(RedisContext),
            "ZADD %s %f %s",
            TCHAR_TO_UTF8(*Key),
            Score,
            TCHAR_TO_UTF8(*Member))
    );

    bool bSuccess = (Reply != nullptr);
    if (Reply)
    {
        freeReplyObject(Reply);
    }

    return bSuccess;
}

TArray<TPair<FString, float>> UMyRedisClient::ZRangeWithScores(const FString& Key, int64 Start, int64 Stop, bool bReverse)
{
    TArray<TPair<FString, float>> Results;

    if (!IsConnected()) return Results;

    FString Command = bReverse ?
        FString::Printf(TEXT("ZREVRANGE %s %lld %lld WITHSCORES"), *Key, Start, Stop) :
        FString::Printf(TEXT("ZRANGE %s %lld %lld WITHSCORES"), *Key, Start, Stop);

    redisReply* Reply = reinterpret_cast<redisReply*>(
        redisCommand(reinterpret_cast<redisContext*>(RedisContext), TCHAR_TO_UTF8(*Command))
    );

    if (Reply != nullptr && Reply->type == REDIS_REPLY_ARRAY)
    {
        for (size_t i = 0; i < Reply->elements; i += 2)
        {
            TPair<FString, float> Item;
            Item.Key = FString(UTF8_TO_TCHAR(Reply->element[i]->str));
            Item.Value = FCString::Atof(UTF8_TO_TCHAR(Reply->element[i + 1]->str));
            Results.Add(Item);
        }
    }

    if (Reply)
    {
        freeReplyObject(Reply);
    }

    return Results;
}

int64 UMyRedisClient::Increment(const FString& Key, int64 Amount)
{
    if (!IsConnected()) return 0;

    redisReply* Reply = reinterpret_cast<redisReply*>(
        redisCommand(reinterpret_cast<redisContext*>(RedisContext),
            "INCRBY %s %lld",
            TCHAR_TO_UTF8(*Key),
            Amount)
    );

    int64 Result = 0;
    if (Reply != nullptr && Reply->type == REDIS_REPLY_INTEGER)
    {
        Result = Reply->integer;
    }

    if (Reply)
    {
        freeReplyObject(Reply);
    }

    return Result;
}

void UMyRedisClient::GetAsync(const FString& Key, TFunction<void(FString)> Callback)
{
    AsyncTask(ENamedThreads::AnyBackgroundThreadNormalTask, [this, Key, Callback]()
    {
        FString Value = Get(Key);

        AsyncTask(ENamedThreads::GameThread, [Callback, Value]()
        {
            Callback(Value);
        });
    });
}
```

### 2.2 缓存策略实现

```cpp
// MyCacheManager.h
#pragma once

#include "CoreMinimal.h"
#include "MyCacheManager.generated.h"

// 缓存配置
USTRUCT()
struct FCacheConfig
{
    GENERATED_BODY()

    int32 DefaultTTL = 3600;           // 默认1小时
    int32 MaxMemoryMB = 256;           // 最大内存256MB
    bool bEnableCompression = true;    // 启用压缩
    int32 CompressionThreshold = 1024; // 压缩阈值（字节）
};

UCLASS()
class UMyCacheManager : public UObject
{
    GENERATED_BODY()

public:
    void Initialize(UMyRedisClient* InRedisClient, const FCacheConfig& Config);

    // 基本缓存操作
    bool SetCache(const FString& Key, const FString& Value, int32 TTL = -1);
    FString GetCache(const FString& Key);
    bool DeleteCache(const FString& Key);
    bool CacheExists(const FString& Key);

    // 对象缓存（自动序列化）
    template<typename T>
    bool SetObjectCache(const FString& Key, const T& Object, int32 TTL = -1);

    template<typename T>
    bool GetObjectCache(const FString& Key, T& OutObject);

    // 缓存模式
    FString GetOrSet(const FString& Key, TFunction<FString()> ValueFactory, int32 TTL = -1);

    // 批量操作
    TMap<FString, FString> MultiGet(const TArray<FString>& Keys);
    void MultiSet(const TMap<FString, FString>& KeyValues, int32 TTL = -1);

    // 缓存标签（用于批量失效）
    void TagKey(const FString& Key, const FString& Tag);
    void InvalidateTag(const FString& Tag);

    // 缓存预热
    void WarmupCache(const TArray<FString>& Keys);

    // 统计
    int64 GetCacheSize();
    float GetHitRate();

private:
    UPROPERTY()
    UMyRedisClient* RedisClient;

    FCacheConfig Config;

    int64 HitCount = 0;
    int64 MissCount = 0;

    FString BuildTagKey(const FString& Tag);
    FString Compress(const FString& Data);
    FString Decompress(const FString& Data);
};

// 实现
void UMyCacheManager::Initialize(UMyRedisClient* InRedisClient, const FCacheConfig& InConfig)
{
    RedisClient = InRedisClient;
    Config = InConfig;
}

bool UMyCacheManager::SetCache(const FString& Key, const FString& Value, int32 TTL)
{
    if (!RedisClient) return false;

    int32 EffectiveTTL = TTL > 0 ? TTL : Config.DefaultTTL;

    FString DataToStore = Value;
    if (Config.bEnableCompression && Value.Len() > Config.CompressionThreshold)
    {
        DataToStore = Compress(Value);
    }

    return RedisClient->Set(Key, DataToStore, EffectiveTTL);
}

FString UMyCacheManager::GetCache(const FString& Key)
{
    if (!RedisClient)
    {
        MissCount++;
        return TEXT("");
    }

    FString Value = RedisClient->Get(Key);

    if (Value.IsEmpty())
    {
        MissCount++;
        return TEXT("");
    }

    HitCount++;

    // 尝试解压
    FString Decompressed = Decompress(Value);
    if (!Decompressed.IsEmpty())
    {
        return Decompressed;
    }

    return Value;
}

FString UMyCacheManager::GetOrSet(const FString& Key, TFunction<FString()> ValueFactory, int32 TTL)
{
    FString CachedValue = GetCache(Key);

    if (!CachedValue.IsEmpty())
    {
        return CachedValue;
    }

    // 缓存未命中，执行工厂函数
    FString NewValue = ValueFactory();

    if (!NewValue.IsEmpty())
    {
        SetCache(Key, NewValue, TTL);
    }

    return NewValue;
}

void UMyCacheManager::TagKey(const FString& Key, const FString& Tag)
{
    if (!RedisClient) return;

    // 将key添加到标签集合
    FString TagKey = BuildTagKey(Tag);
    RedisClient->ExecuteCommand(FString::Printf(TEXT("SADD %s %s"), *TagKey, *Key));
}

void UMyCacheManager::InvalidateTag(const FString& Tag)
{
    if (!RedisClient) return;

    FString TagKey = BuildTagKey(Tag);

    // 获取所有关联的key
    TArray<FString> Keys = RedisClient->SMembers(TagKey);

    // 删除所有key
    for (const FString& Key : Keys)
    {
        RedisClient->Delete(Key);
    }

    // 删除标签集合
    RedisClient->Delete(TagKey);
}

float UMyCacheManager::GetHitRate()
{
    int64 Total = HitCount + MissCount;
    if (Total == 0) return 0.0f;
    return static_cast<float>(HitCount) / Total;
}
```

---

## 三、MongoDB集成

### 3.1 MongoDB客户端实现

```cpp
// MyMongoClient.h
#pragma once

#include "CoreMinimal.h"
#include "MyMongoClient.generated.h"

// MongoDB配置
USTRUCT(BlueprintType)
struct FMongoConfig
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString ConnectionString = TEXT("mongodb://localhost:27017");

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString DatabaseName = TEXT("gamedb");

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 ConnectionTimeout = 5000;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 SocketTimeout = 30000;
};

// 文档（BSON）
USTRUCT(BlueprintType)
struct FMongoDocument
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    TMap<FString, FString> StringFields;

    UPROPERTY(BlueprintReadOnly)
    TMap<FString, int64> IntFields;

    UPROPERTY(BlueprintReadOnly)
    TMap<FString, float> FloatFields;

    UPROPERTY(BlueprintReadOnly)
    TMap<FString, bool> BoolFields;

    UPROPERTY(BlueprintReadOnly)
    TArray<FMongoDocument> ArrayFields;

    FString GetId() const { return StringFields.Contains("_id") ? StringFields["_id"] : TEXT(""); }
};

// 查询选项
USTRUCT()
struct FMongoFindOptions
{
    GENERATED_BODY()

    TMap<FString, int32> Sort;
    int32 Limit = 0;
    int32 Skip = 0;
    TArray<FString> Projection;
};

UCLASS()
class UMyMongoClient : public UObject
{
    GENERATED_BODY()

public:
    bool Connect(const FMongoConfig& Config);
    void Disconnect();
    bool IsConnected() const;

    // 集合操作
    void CreateCollection(const FString& Name);
    void DropCollection(const FString& Name);
    TArray<FString> ListCollections();

    // CRUD操作
    FString InsertOne(const FString& Collection, const FMongoDocument& Document);
    void InsertMany(const FString& Collection, const TArray<FMongoDocument>& Documents);

    FMongoDocument FindOne(const FString& Collection, const TMap<FString, FString>& Filter);
    TArray<FMongoDocument> Find(const FString& Collection, const TMap<FString, FString>& Filter,
                                 const FMongoFindOptions& Options = FMongoFindOptions());

    bool UpdateOne(const FString& Collection, const TMap<FString, FString>& Filter,
                   const TMap<FString, FMongoDocument>& Update, bool bUpsert = false);
    int32 UpdateMany(const FString& Collection, const TMap<FString, FString>& Filter,
                     const TMap<FString, FMongoDocument>& Update);

    bool DeleteOne(const FString& Collection, const TMap<FString, FString>& Filter);
    int32 DeleteMany(const FString& Collection, const TMap<FString, FString>& Filter);

    // 聚合查询
    TArray<FMongoDocument> Aggregate(const FString& Collection, const TArray<TMap<FString, FString>>& Pipeline);

    // 索引操作
    void CreateIndex(const FString& Collection, const FString& Field, bool bUnique = false, bool bDescending = false);
    void DropIndex(const FString& Collection, const FString& IndexName);

    // 异步操作
    void FindAsync(const FString& Collection, const TMap<FString, FString>& Filter,
                   TFunction<void(TArray<FMongoDocument>)> Callback);

private:
    void* MongoClient;    // mongoc_client_t*
    void* MongoDatabase;  // mongoc_database_t*
    FMongoConfig CurrentConfig;
};
```

---

## 四、DynamoDB集成

### 4.1 DynamoDB客户端实现

```cpp
// MyDynamoDBClient.h
#pragma once

#include "CoreMinimal.h"
#include "MyDynamoDBClient.generated.h"

// DynamoDB配置
USTRUCT(BlueprintType)
struct FDynamoDBConfig
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString Region = TEXT("us-east-1");

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString AccessKey;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString SecretKey;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString Endpoint; // 用于本地开发
};

// DynamoDB属性
USTRUCT(BlueprintType)
struct FDynamoDBAttribute
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadWrite)
    FString Name;

    UPROPERTY(BlueprintReadWrite)
    FString StringValue;

    UPROPERTY(BlueprintReadWrite)
    int64 NumberValue = 0;

    UPROPERTY(BlueprintReadWrite)
    bool BoolValue = false;

    UPROPERTY(BlueprintReadWrite)
    TArray<uint8> BinaryValue;

    UPROPERTY(BlueprintReadWrite)
    FString Type; // S, N, B, BOOL, NULL, L, M
};

// 项目
USTRUCT(BlueprintType)
struct FDynamoDBItem
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    TMap<FString, FDynamoDBAttribute> Attributes;

    void SetString(const FString& Key, const FString& Value);
    void SetNumber(const FString& Key, int64 Value);
    void SetBool(const FString& Key, bool Value);
    FString GetString(const FString& Key) const;
    int64 GetNumber(const FString& Key) const;
    bool GetBool(const FString& Key) const;
};

// 查询结果
USTRUCT(BlueprintType)
struct FDynamoDBQueryResult
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    TArray<FDynamoDBItem> Items;

    UPROPERTY(BlueprintReadOnly)
    int32 Count = 0;

    UPROPERTY(BlueprintReadOnly)
    FString LastEvaluatedKey;
};

UCLASS()
class UMyDynamoDBClient : public UObject
{
    GENERATED_BODY()

public:
    bool Initialize(const FDynamoDBConfig& Config);
    void Shutdown();

    // 表操作
    bool CreateTable(const FString& TableName, const FString& PartitionKey, const FString& SortKey = TEXT(""));
    bool DeleteTable(const FString& TableName);
    bool TableExists(const FString& TableName);

    // CRUD操作
    bool PutItem(const FString& TableName, const FDynamoDBItem& Item);
    FDynamoDBItem GetItem(const FString& TableName, const FDynamoDBItem& Key);
    bool UpdateItem(const FString& TableName, const FDynamoDBItem& Key,
                    const TMap<FString, FString>& UpdateExpressions);
    bool DeleteItem(const FString& TableName, const FDynamoDBItem& Key);

    // 查询操作
    FDynamoDBQueryResult Query(const FString& TableName, const FString& KeyConditionExpression,
                                const TMap<FString, FDynamoDBAttribute>& ExpressionValues);
    FDynamoDBQueryResult Scan(const FString& TableName, const FString& FilterExpression,
                               const TMap<FString, FDynamoDBAttribute>& ExpressionValues);

    // 批量操作
    bool BatchWrite(const FString& TableName, const TArray<FDynamoDBItem>& Items);
    TArray<FDynamoDBItem> BatchGet(const FString& TableName, const TArray<FDynamoDBItem>& Keys);

    // 异步操作
    void PutItemAsync(const FString& TableName, const FDynamoDBItem& Item, TFunction<void(bool)> Callback);
    void QueryAsync(const FString& TableName, const FString& KeyConditionExpression,
                    const TMap<FString, FDynamoDBAttribute>& ExpressionValues,
                    TFunction<void(FDynamoDBQueryResult)> Callback);

private:
    FDynamoDBConfig Config;
    bool bInitialized;
};
```

---

## 五、数据模型设计最佳实践

### 5.1 游戏数据模型设计

```cpp
// 玩家数据模型示例

// MySQL - 关系型数据
/*
CREATE TABLE players (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    steam_id VARCHAR(64) UNIQUE NOT NULL,
    player_name VARCHAR(64) NOT NULL,
    level INT DEFAULT 1,
    experience BIGINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login TIMESTAMP,
    INDEX idx_steam_id (steam_id),
    INDEX idx_level (level)
);
*/

// Redis - 缓存层
// Key: player:{steam_id}
// Value: JSON序列化的玩家数据
// TTL: 1小时

// MongoDB - 灵活数据
/*
{
    _id: ObjectId("..."),
    steam_id: "76561198...",
    player_name: "Player1",
    stats: {
        kills: 1000,
        deaths: 500,
        score: 15000
    },
    inventory: [
        { item_id: "weapon_rifle", quantity: 1 },
        { item_id: "ammo_small", quantity: 500 }
    ],
    achievements: ["first_kill", "veteran"],
    metadata: {
        created_at: ISODate("..."),
        last_login: ISODate("...")
    }
}
*/

// DynamoDB - 云数据
/*
Table: Players
Partition Key: steam_id (S)
Attributes: player_name, level, experience, stats (M), inventory (L)
*/

// 数据访问层
UCLASS()
class UPlayerDataAccess : public UObject
{
    GENERATED_BODY()

public:
    // 多层缓存策略
    UFUNCTION(BlueprintCallable, Category = "Player")
    FPlayerData GetPlayer(const FString& SteamId)
    {
        // 1. 检查Redis缓存
        FString CacheKey = FString::Printf(TEXT("player:%s"), *SteamId);
        FString CachedData = CacheManager->GetCache(CacheKey);

        if (!CachedData.IsEmpty())
        {
            return DeserializePlayerData(CachedData);
        }

        // 2. 从数据库加载
        FPlayerData PlayerData;

        if (bUseMongoDB)
        {
            PlayerData = LoadFromMongoDB(SteamId);
        }
        else
        {
            PlayerData = LoadFromMySQL(SteamId);
        }

        // 3. 写入缓存
        CacheManager->SetCache(CacheKey, SerializePlayerData(PlayerData), 3600);

        return PlayerData;
    }

    UFUNCTION(BlueprintCallable, Category = "Player")
    void SavePlayer(const FString& SteamId, const FPlayerData& Data)
    {
        // 1. 更新数据库
        if (bUseMongoDB)
        {
            SaveToMongoDB(SteamId, Data);
        }
        else
        {
            SaveToMySQL(SteamId, Data);
        }

        // 2. 更新缓存
        FString CacheKey = FString::Printf(TEXT("player:%s"), *SteamId);
        CacheManager->SetCache(CacheKey, SerializePlayerData(Data), 3600);
    }

private:
    UPROPERTY()
    UMyCacheManager* CacheManager;

    UPROPERTY()
    UMyMongoClient* MongoClient;

    UPROPERTY()
    UMySQLManager* SQLManager;

    bool bUseMongoDB = true;

    FPlayerData LoadFromMongoDB(const FString& SteamId);
    FPlayerData LoadFromMySQL(const FString& SteamId);
    void SaveToMongoDB(const FString& SteamId, const FPlayerData& Data);
    void SaveToMySQL(const FString& SteamId, const FPlayerData& Data);
    FString SerializePlayerData(const FPlayerData& Data);
    FPlayerData DeserializePlayerData(const FString& JsonString);
};
```

---

## 六、总结

本课我们学习了：

1. **Redis集成**：键值存储、缓存策略、排行榜实现
2. **MongoDB集成**：文档存储、灵活Schema
3. **DynamoDB集成**：云数据库、自动扩展
4. **数据模型设计**：多数据库协作、缓存策略

---

## 七、下节预告

**第14课：用户数据管理**

将深入学习：
- 用户认证系统设计
- 账号数据持久化
- 跨设备数据同步

---

*课程版本：1.0*
*最后更新：2026-04-10*