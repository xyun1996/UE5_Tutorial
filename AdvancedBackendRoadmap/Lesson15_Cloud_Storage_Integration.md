# 第15课：云存储集成

> 本课深入讲解如何在UE5中集成云存储服务，包括AWS S3、Azure Blob Storage和CDN加速策略。

---

## 课程目标

- 掌握AWS S3集成方法
- 学会Azure Blob Storage应用
- 理解游戏资源云存储策略
- 掌握用户生成内容(UGC)存储
- 学会CDN加速配置

---

## 一、云存储概述

### 1.1 云存储服务对比

| 服务 | 特点 | 适用场景 |
|------|------|---------|
| **AWS S3** | 高可靠、全球节点、丰富功能 | 游戏资源、用户数据、静态网站 |
| **Azure Blob** | 与Azure生态集成好、成本优化 | Windows生态、企业应用 |
| **Google Cloud Storage** | 大数据处理友好、AI集成 | 数据分析、机器学习 |
| **阿里云OSS** | 国内访问快、成本低 | 国内游戏市场 |
| **腾讯云COS** | CDN整合好、国内节点多 | 国内游戏市场 |

---

## 二、AWS S3集成

### 2.1 S3客户端实现

```cpp
// MyS3Client.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "MyS3Client.generated.h"

// S3配置
USTRUCT(BlueprintType)
struct FS3Config
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString Region = TEXT("us-east-1");

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString AccessKey;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString SecretKey;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString DefaultBucket;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 ConnectionTimeout = 30;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 RequestTimeout = 60;
};

// 上传结果
USTRUCT(BlueprintType)
struct FS3UploadResult
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    bool bSuccess = false;

    UPROPERTY(BlueprintReadOnly)
    FString ETag;

    UPROPERTY(BlueprintReadOnly)
    FString Location;

    UPROPERTY(BlueprintReadOnly)
    FString ErrorMessage;
};

// 下载结果
USTRUCT(BlueprintType)
struct FS3DownloadResult
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    bool bSuccess = false;

    UPROPERTY(BlueprintReadOnly)
    TArray<uint8> Data;

    UPROPERTY(BlueprintReadOnly)
    FString ContentType;

    UPROPERTY(BlueprintReadOnly)
    int64 ContentLength;

    UPROPERTY(BlueprintReadOnly)
    FString ErrorMessage;
};

// 对象信息
USTRUCT(BlueprintType)
struct FS3ObjectInfo
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString Key;

    UPROPERTY(BlueprintReadOnly)
    int64 Size;

    UPROPERTY(BlueprintReadOnly)
    FString ETag;

    UPROPERTY(BlueprintReadOnly)
    FDateTime LastModified;

    UPROPERTY(BlueprintReadOnly)
    FString ContentType;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnS3UploadComplete, const FS3UploadResult&, Result);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnS3DownloadComplete, const FS3DownloadResult&, Result);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnS3Progress, float, Percent);

UCLASS()
class UMyS3Client : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 配置
    UFUNCTION(BlueprintCallable, Category = "S3")
    void Configure(const FS3Config& Config);

    // 上传操作
    UFUNCTION(BlueprintCallable, Category = "S3")
    void UploadFile(const FString& Key, const FString& FilePath, const FString& ContentType = TEXT(""));

    UFUNCTION(BlueprintCallable, Category = "S3")
    void UploadData(const FString& Key, const TArray<uint8>& Data, const FString& ContentType = TEXT(""));

    void UploadFileAsync(const FString& Key, const FString& FilePath, TFunction<void(FS3UploadResult)> Callback);

    // 下载操作
    UFUNCTION(BlueprintCallable, Category = "S3")
    void DownloadFile(const FString& Key, const FString& SavePath);

    UFUNCTION(BlueprintCallable, Category = "S3")
    void DownloadData(const FString& Key);

    void DownloadDataAsync(const FString& Key, TFunction<void(FS3DownloadResult)> Callback);

    // 列表操作
    UFUNCTION(BlueprintCallable, Category = "S3")
    TArray<FS3ObjectInfo> ListObjects(const FString& Prefix = TEXT(""));

    UFUNCTION(BlueprintCallable, Category = "S3")
    bool ObjectExists(const FString& Key);

    // 删除操作
    UFUNCTION(BlueprintCallable, Category = "S3")
    bool DeleteObject(const FString& Key);

    UFUNCTION(BlueprintCallable, Category = "S3")
    bool DeleteObjects(const TArray<FString>& Keys);

    // 预签名URL
    UFUNCTION(BlueprintCallable, Category = "S3")
    FString GetPresignedUrl(const FString& Key, int32 ExpiresInSeconds = 3600);

    // 委托
    UPROPERTY(BlueprintAssignable)
    FOnS3UploadComplete OnUploadComplete;

    UPROPERTY(BlueprintAssignable)
    FOnS3DownloadComplete OnDownloadComplete;

    UPROPERTY(BlueprintAssignable)
    FOnS3Progress OnProgress;

protected:
    FS3Config CurrentConfig;
    bool bInitialized = false;

    // 内部方法
    FString BuildEndpoint() const;
    FString SignRequest(const FString& Method, const FString& Path, const FString& Payload) const;
};

// MyS3Client.cpp
#include "MyS3Client.h"
#include "HAL/PlatformFileManager.h"

void UMyS3Client::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    // 从配置加载默认值
    // CurrentConfig = LoadConfigFromIni();
}

void UMyS3Client::Deinitialize()
{
    Super::Deinitialize();
}

void UMyS3Client::Configure(const FS3Config& Config)
{
    CurrentConfig = Config;
    bInitialized = true;

    UE_LOG(LogTemp, Log, TEXT("S3 client configured for region: %s"), *Config.Region);
}

void UMyS3Client::UploadFile(const FString& Key, const FString& FilePath, const FString& ContentType)
{
    if (!bInitialized)
    {
        FS3UploadResult Result;
        Result.bSuccess = false;
        Result.ErrorMessage = TEXT("S3 client not configured");
        OnUploadComplete.Broadcast(Result);
        return;
    }

    // 读取文件
    TArray<uint8> FileData;
    if (!FFileHelper::LoadFileToArray(FileData, *FilePath))
    {
        FS3UploadResult Result;
        Result.bSuccess = false;
        Result.ErrorMessage = FString::Printf(TEXT("Failed to read file: %s"), *FilePath);
        OnUploadComplete.Broadcast(Result);
        return;
    }

    UploadData(Key, FileData, ContentType);
}

void UMyS3Client::UploadData(const FString& Key, const TArray<uint8>& Data, const FString& ContentType)
{
    if (!bInitialized)
    {
        FS3UploadResult Result;
        Result.bSuccess = false;
        Result.ErrorMessage = TEXT("S3 client not configured");
        OnUploadComplete.Broadcast(Result);
        return;
    }

    // 异步上传
    UploadFileAsync(Key, TEXT(""), [this](FS3UploadResult Result)
    {
        AsyncTask(ENamedThreads::GameThread, [this, Result]()
        {
            OnUploadComplete.Broadcast(Result);
        });
    });
}

void UMyS3Client::UploadFileAsync(const FString& Key, const FString& FilePath, TFunction<void(FS3UploadResult)> Callback)
{
    AsyncTask(ENamedThreads::AnyBackgroundThreadNormalTask, [this, Key, FilePath, Callback]()
    {
        FS3UploadResult Result;

        // 实际的S3上传实现
        // 这里需要使用HTTP客户端发送PUT请求到S3
        // 使用AWS Signature V4进行签名

        // 模拟上传
        FPlatformProcess::Sleep(2.0f);

        Result.bSuccess = true;
        Result.Location = FString::Printf(TEXT("https://%s.s3.%s.amazonaws.com/%s"),
            *CurrentConfig.DefaultBucket, *CurrentConfig.Region, *Key);

        if (Callback)
        {
            Callback(Result);
        }
    });
}

void UMyS3Client::DownloadDataAsync(const FString& Key, TFunction<void(FS3DownloadResult)> Callback)
{
    AsyncTask(ENamedThreads::AnyBackgroundThreadNormalTask, [this, Key, Callback]()
    {
        FS3DownloadResult Result;

        // 实际的S3下载实现
        // 发送GET请求到S3

        // 模拟下载
        FPlatformProcess::Sleep(1.0f);

        Result.bSuccess = true;
        Result.ContentLength = 0;

        if (Callback)
        {
            Callback(Result);
        }
    });
}

TArray<FS3ObjectInfo> UMyS3Client::ListObjects(const FString& Prefix)
{
    TArray<FS3ObjectInfo> Objects;

    if (!bInitialized)
    {
        return Objects;
    }

    // 发送LIST请求
    // 解析XML响应

    return Objects;
}

bool UMyS3Client::ObjectExists(const FString& Key)
{
    if (!bInitialized)
    {
        return false;
    }

    // 发送HEAD请求检查对象是否存在

    return true;
}

FString UMyS3Client::GetPresignedUrl(const FString& Key, int32 ExpiresInSeconds)
{
    if (!bInitialized)
    {
        return TEXT("");
    }

    // 生成预签名URL
    // 格式: https://bucket.s3.region.amazonaws.com/key?X-Amz-Algorithm=...&X-Amz-Credential=...

    FString Url = FString::Printf(TEXT("https://%s.s3.%s.amazonaws.com/%s"),
        *CurrentConfig.DefaultBucket, *CurrentConfig.Region, *Key);

    return Url;
}
```

---

## 三、Azure Blob Storage

### 3.1 Azure Blob客户端

```cpp
// MyAzureBlobClient.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "MyAzureBlobClient.generated.h"

// Azure配置
USTRUCT(BlueprintType)
struct FAzureBlobConfig
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString StorageAccountName;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString AccessKey;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString ContainerName;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FString SasToken; // 可选的共享访问签名
};

UCLASS()
class UMyAzureBlobClient : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 配置
    UFUNCTION(BlueprintCallable, Category = "Azure")
    void Configure(const FAzureBlobConfig& Config);

    // 基本操作
    UFUNCTION(BlueprintCallable, Category = "Azure")
    void UploadBlob(const FString& BlobName, const TArray<uint8>& Data);

    UFUNCTION(BlueprintCallable, Category = "Azure")
    void DownloadBlob(const FString& BlobName);

    UFUNCTION(BlueprintCallable, Category = "Azure")
    bool DeleteBlob(const FString& BlobName);

    UFUNCTION(BlueprintCallable, Category = "Azure")
    TArray<FString> ListBlobs(const FString& Prefix = TEXT(""));

    // 块操作（大文件）
    UFUNCTION(BlueprintCallable, Category = "Azure")
    FString StartBlockUpload(const FString& BlobName);

    UFUNCTION(BlueprintCallable, Category = "Azure")
    void UploadBlock(const FString& UploadId, int64 BlockIndex, const TArray<uint8>& Data);

    UFUNCTION(BlueprintCallable, Category = "Azure")
    void CompleteBlockUpload(const FString& UploadId);

private:
    FAzureBlobConfig CurrentConfig;
    FString BuildBlobUrl(const FString& BlobName) const;
    FString SignRequest(const FString& Method, const FString& Resource, const FString& Content) const;
};
```

---

## 四、游戏资源云存储

### 4.1 资源管理器

```cpp
// MyCloudResourceManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "MyCloudResourceManager.generated.h"

// 资源类型
UENUM(BlueprintType)
enum class ECloudResourceType : uint8
{
    AssetBundle,
    ConfigFile,
    Localization,
    UpdatePatch,
    UserGeneratedContent
};

// 资源信息
USTRUCT(BlueprintType)
struct FCloudResourceInfo
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString ResourceId;

    UPROPERTY(BlueprintReadOnly)
    FString DisplayName;

    UPROPERTY(BlueprintReadOnly)
    ECloudResourceType Type;

    UPROPERTY(BlueprintReadOnly)
    FString Version;

    UPROPERTY(BlueprintReadOnly)
    int64 Size;

    UPROPERTY(BlueprintReadOnly)
    FString MD5Hash;

    UPROPERTY(BlueprintReadOnly)
    FString DownloadUrl;

    UPROPERTY(BlueprintReadOnly)
    TArray<FString> Dependencies;
};

// 下载状态
UENUM(BlueprintType)
enum class EDownloadStatus : uint8
{
    NotStarted,
    Queued,
    Downloading,
    Completed,
    Failed,
    Paused
};

// 下载任务
USTRUCT(BlueprintType)
struct FDownloadTask
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString TaskId;

    UPROPERTY(BlueprintReadOnly)
    FString ResourceId;

    UPROPERTY(BlueprintReadOnly)
    EDownloadStatus Status;

    UPROPERTY(BlueprintReadOnly)
    float Progress;

    UPROPERTY(BlueprintReadOnly)
    int64 BytesDownloaded;

    UPROPERTY(BlueprintReadOnly)
    int64 TotalBytes;

    UPROPERTY(BlueprintReadOnly)
    float DownloadSpeed; // bytes/sec
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnResourceDownloadComplete, const FString&, ResourceId, bool, bSuccess);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnDownloadProgress, const FString&, TaskId, float, Progress);

UCLASS()
class UMyCloudResourceManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Tick(float DeltaTime) override;

    // 资源查询
    UFUNCTION(BlueprintCallable, Category = "Cloud Resources")
    TArray<FCloudResourceInfo> GetAvailableResources(ECloudResourceType Type);

    UFUNCTION(BlueprintCallable, Category = "Cloud Resources")
    FCloudResourceInfo GetResourceInfo(const FString& ResourceId);

    UFUNCTION(BlueprintPure, Category = "Cloud Resources")
    bool IsResourceDownloaded(const FString& ResourceId);

    // 下载管理
    UFUNCTION(BlueprintCallable, Category = "Cloud Resources")
    FString StartDownload(const FString& ResourceId);

    UFUNCTION(BlueprintCallable, Category = "Cloud Resources")
    void PauseDownload(const FString& TaskId);

    UFUNCTION(BlueprintCallable, Category = "Cloud Resources")
    void ResumeDownload(const FString& TaskId);

    UFUNCTION(BlueprintCallable, Category = "Cloud Resources")
    void CancelDownload(const FString& TaskId);

    UFUNCTION(BlueprintCallable, Category = "Cloud Resources")
    void DownloadAllRequired();

    // 任务状态
    UFUNCTION(BlueprintPure, Category = "Cloud Resources")
    TArray<FDownloadTask> GetActiveDownloads();

    UFUNCTION(BlueprintPure, Category = "Cloud Resources")
    FDownloadTask GetDownloadTask(const FString& TaskId);

    // 本地管理
    UFUNCTION(BlueprintCallable, Category = "Cloud Resources")
    bool DeleteLocalResource(const FString& ResourceId);

    UFUNCTION(BlueprintCallable, Category = "Cloud Resources")
    int64 GetLocalCacheSize();

    UFUNCTION(BlueprintCallable, Category = "Cloud Resources")
    void ClearLocalCache();

    // 委托
    UPROPERTY(BlueprintAssignable)
    FOnResourceDownloadComplete OnDownloadComplete;

    UPROPERTY(BlueprintAssignable)
    FOnDownloadProgress OnDownloadProgress;

protected:
    TMap<FString, FDownloadTask> ActiveTasks;
    TArray<FString> DownloadQueue;

    FString LocalCachePath;

    void ProcessDownloadQueue();
    void UpdateDownloadTask(const FString& TaskId);
    void CompleteDownload(const FString& TaskId, bool bSuccess);

    // 文件操作
    bool VerifyDownload(const FString& FilePath, const FString& ExpectedHash);
    FString GetLocalPath(const FString& ResourceId) const;
};

// MyCloudResourceManager.cpp
#include "MyCloudResourceManager.h"
#include "HAL/PlatformFileManager.h"

void UMyCloudResourceManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    LocalCachePath = FPaths::ProjectSavedDir() / TEXT("CloudCache");

    IPlatformFile& PlatformFile = FPlatformFileManager::Get().GetPlatformFile();
    if (!PlatformFile.DirectoryExists(*LocalCachePath))
    {
        PlatformFile.CreateDirectory(*LocalCachePath);
    }
}

void UMyCloudResourceManager::Tick(float DeltaTime)
{
    ProcessDownloadQueue();
}

TArray<FCloudResourceInfo> UMyCloudResourceManager::GetAvailableResources(ECloudResourceType Type)
{
    TArray<FCloudResourceInfo> Resources;

    // 从服务器获取资源列表
    // ...

    return Resources;
}

FString UMyCloudResourceManager::StartDownload(const FString& ResourceId)
{
    FString TaskId = FGuid::NewGuid().ToString();

    FDownloadTask Task;
    Task.TaskId = TaskId;
    Task.ResourceId = ResourceId;
    Task.Status = EDownloadStatus::Queued;
    Task.Progress = 0.0f;

    ActiveTasks.Add(TaskId, Task);
    DownloadQueue.Add(TaskId);

    return TaskId;
}

void UMyCloudResourceManager::ProcessDownloadQueue()
{
    if (DownloadQueue.Num() == 0)
    {
        return;
    }

    // 获取下一个待处理的任务
    FString TaskId = DownloadQueue[0];
    FDownloadTask* Task = ActiveTasks.Find(TaskId);

    if (!Task)
    {
        DownloadQueue.RemoveAt(0);
        return;
    }

    if (Task->Status == EDownloadStatus::Queued)
    {
        Task->Status = EDownloadStatus::Downloading;
    }

    // 更新下载进度
    UpdateDownloadTask(TaskId);
}

void UMyCloudResourceManager::UpdateDownloadTask(const FString& TaskId)
{
    FDownloadTask* Task = ActiveTasks.Find(TaskId);
    if (!Task || Task->Status != EDownloadStatus::Downloading)
    {
        return;
    }

    // 实际的下载逻辑
    // 从S3/Azure下载文件块

    // 更新进度
    // Task->Progress = ...
    // Task->BytesDownloaded = ...

    OnDownloadProgress.Broadcast(TaskId, Task->Progress);

    // 检查是否完成
    if (Task->Progress >= 1.0f)
    {
        CompleteDownload(TaskId, true);
    }
}

void UMyCloudResourceManager::CompleteDownload(const FString& TaskId, bool bSuccess)
{
    FDownloadTask* Task = ActiveTasks.Find(TaskId);
    if (!Task)
    {
        return;
    }

    Task->Status = bSuccess ? EDownloadStatus::Completed : EDownloadStatus::Failed;
    Task->Progress = bSuccess ? 1.0f : Task->Progress;

    // 从队列移除
    DownloadQueue.Remove(TaskId);

    OnDownloadComplete.Broadcast(Task->ResourceId, bSuccess);
}

bool UMyCloudResourceManager::IsResourceDownloaded(const FString& ResourceId)
{
    FString LocalPath = GetLocalPath(ResourceId);
    return FPaths::FileExists(LocalPath);
}

FString UMyCloudResourceManager::GetLocalPath(const FString& ResourceId) const
{
    return LocalCachePath / ResourceId + TEXT(".pak");
}
```

---

## 五、UGC存储

### 5.1 UGC管理器

```cpp
// MyUGCManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "MyUGCManager.generated.h"

// UGC类型
UENUM(BlueprintType)
enum class EUGCType : uint8
{
    Image,
    Video,
    Audio,
    Blueprint,
    Map,
    Mod
};

// UGC内容信息
USTRUCT(BlueprintType)
struct FUGCContentInfo
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString ContentId;

    UPROPERTY(BlueprintReadOnly)
    FString Title;

    UPROPERTY(BlueprintReadOnly)
    FString Description;

    UPROPERTY(BlueprintReadOnly)
    FString AuthorId;

    UPROPERTY(BlueprintReadOnly)
    FString AuthorName;

    UPROPERTY(BlueprintReadOnly)
    EUGCType ContentType;

    UPROPERTY(BlueprintReadOnly)
    FString CloudUrl;

    UPROPERTY(BlueprintReadOnly)
    FString ThumbnailUrl;

    UPROPERTY(BlueprintReadOnly)
    int64 FileSize;

    UPROPERTY(BlueprintReadOnly)
    int32 Downloads;

    UPROPERTY(BlueprintReadOnly)
    int32 Likes;

    UPROPERTY(BlueprintReadOnly)
    FDateTime UploadTime;

    UPROPERTY(BlueprintReadOnly)
    TArray<FString> Tags;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnUGCUploadComplete, bool, bSuccess, const FString&, ContentId);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnUGCDownloadComplete, bool, bSuccess, const FString&, ContentId);

UCLASS()
class UMyUGCManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    // 上传
    UFUNCTION(BlueprintCallable, Category = "UGC")
    void UploadContent(const FString& FilePath, const FString& Title, const FString& Description,
                       const TArray<FString>& Tags, EUGCType ContentType);

    void UploadContentAsync(const FString& FilePath, TFunction<void(bool, FString)> Callback);

    // 下载
    UFUNCTION(BlueprintCallable, Category = "UGC")
    void DownloadContent(const FString& ContentId);

    // 搜索
    UFUNCTION(BlueprintCallable, Category = "UGC")
    TArray<FUGCContentInfo> SearchContent(const FString& Query, const TArray<FString>& Tags);

    UFUNCTION(BlueprintCallable, Category = "UGC")
    TArray<FUGCContentInfo> GetContentByAuthor(const FString& AuthorId);

    UFUNCTION(BlueprintCallable, Category = "UGC")
    TArray<FUGCContentInfo> GetPopularContent(EUGCType Type, int32 Limit = 20);

    // 评价
    UFUNCTION(BlueprintCallable, Category = "UGC")
    void LikeContent(const FString& ContentId);

    UFUNCTION(BlueprintCallable, Category = "UGC")
    void ReportContent(const FString& ContentId, const FString& Reason);

    // 管理
    UFUNCTION(BlueprintCallable, Category = "UGC")
    bool DeleteMyContent(const FString& ContentId);

    UFUNCTION(BlueprintPure, Category = "UGC")
    TArray<FUGCContentInfo> GetMyContent();

    // 委托
    UPROPERTY(BlueprintAssignable)
    FOnUGCUploadComplete OnUploadComplete;

    UPROPERTY(BlueprintAssignable)
    FOnUGCDownloadComplete OnDownloadComplete;

protected:
    FString UGCLocalPath;
    TArray<FUGCContentInfo> CachedContentList;

    FString GenerateContentId();
    bool ValidateContent(const FString& FilePath, EUGCType Type);
    FString GetLocalContentPath(const FString& ContentId) const;
};
```

---

## 六、CDN加速策略

### 6.1 CDN配置

```cpp
// MyCDNManager.h
#pragma once

#include "CoreMinimal.h"
#include "MyCDNManager.generated.h"

UCLASS(Config = Game)
class UCDNConfig : public UDeveloperSettings
{
    GENERATED_BODY()

public:
    // CDN端点
    UPROPERTY(Config, EditAnywhere, Category = "CDN")
    FString PrimaryCDNEndpoint;

    UPROPERTY(Config, EditAnywhere, Category = "CDN")
    TArray<FString> FallbackCDNEndpoints;

    // 区域映射
    UPROPERTY(Config, EditAnywhere, Category = "CDN")
    TMap<FString, FString> RegionToCDNMapping;

    // 缓存策略
    UPROPERTY(Config, EditAnywhere, Category = "CDN")
    int32 CacheTimeSeconds = 3600;

    UPROPERTY(Config, EditAnywhere, Category = "CDN")
    bool bEnableCompression = true;
};

UCLASS()
class UMyCDNManager : public UObject
{
    GENERATED_BODY()

public:
    // 获取最优CDN URL
    FString GetOptimalCDNUrl(const FString& ResourcePath);

    // 刷新CDN缓存
    void RefreshCache(const TArray<FString>& ResourcePaths);

    // 预热CDN
    void PreloadResources(const TArray<FString>& ResourcePaths);

    // 区域检测
    FString DetectRegion();

private:
    FString CurrentRegion;
    FString CurrentCDNEndpoint;

    void InitializeCDN();
    FString SelectBestEndpoint();
    float TestEndpointLatency(const FString& Endpoint);
};
```

---

## 七、总结

本课我们学习了：

1. **AWS S3集成**：上传、下载、预签名URL
2. **Azure Blob**：块上传、大文件处理
3. **资源管理**：下载队列、本地缓存
4. **UGC存储**：用户内容上传和分享
5. **CDN加速**：区域优化、缓存策略

---

## 八、下节预告

**第16课：映射系统（Map Transfer）**

将深入学习：
- 关卡切换与服务器旅行
- 无缝旅行实现
- 多关卡服务器架构

---

*课程版本：1.0*
*最后更新：2026-04-10*