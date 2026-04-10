# 第11课：SaveGame系统深入

> 本课深入讲解UE5的SaveGame系统架构，包括自定义序列化、异步加载和云存档集成。

---

## 课程目标

- 深入理解USaveGame架构原理
- 掌握自定义存档序列化方法
- 学会异步存档加载实现
- 理解存档版本管理策略
- 掌握云存档集成方法

---

## 一、USaveGame架构解析

### 1.1 SaveGame系统概述

```
┌─────────────────────────────────────────────────────────────┐
│                    SaveGame系统架构                          │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                  USaveGame对象                       │    │
│  │  ┌─────────────────────────────────────────────┐   │    │
│  │  │  可序列化属性                                │   │    │
│  │  │  - UPROPERTY(SaveGame)                     │   │    │
│  │  │  - 基本类型、容器、自定义结构               │   │    │
│  │  └─────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│                          ▼                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              UGameplayStatics                        │    │
│  │  - SaveGameToSlot()     保存到槽位                  │    │
│  │  - LoadGameFromSlot()   从槽位加载                  │    │
│  │  - DoesSaveGameExist()  检查是否存在                │    │
│  │  - DeleteGameInSlot()   删除存档                    │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│                          ▼                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              文件系统                                │    │
│  │  Saved/SaveGames/SlotName.sav                       │    │
│  │  - 二进制格式                                       │    │
│  │  - 支持加密                                         │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 基础SaveGame类

```cpp
// MySaveGame.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/SaveGame.h"
#include "MySaveGame.generated.h"

// 玩家统计数据
USTRUCT(BlueprintType)
struct FPlayerStatsSaveData
{
    GENERATED_BODY()

    UPROPERTY(SaveGame)
    int32 TotalKills = 0;

    UPROPERTY(SaveGame)
    int32 TotalDeaths = 0;

    UPROPERTY(SaveGame)
    float TotalScore = 0.0f;

    UPROPERTY(SaveGame)
    float TotalPlayTime = 0.0f;
};

// 物品数据
USTRUCT(BlueprintType)
struct FItemSaveData
{
    GENERATED_BODY()

    UPROPERTY(SaveGame)
    FName ItemId;

    UPROPERTY(SaveGame)
    int32 Quantity = 1;

    UPROPERTY(SaveGame)
    TArray<FString> Modifiers;
};

// 角色状态
USTRUCT(BlueprintType)
struct FCharacterSaveData
{
    GENERATED_BODY()

    UPROPERTY(SaveGame)
    FString CharacterName;

    UPROPERTY(SaveGame)
    FVector Location;

    UPROPERTY(SaveGame)
    FRotator Rotation;

    UPROPERTY(SaveGame)
    float Health = 100.0f;

    UPROPERTY(SaveGame)
    float Stamina = 100.0f;

    UPROPERTY(SaveGame)
    TArray<FItemSaveData> Inventory;
};

UCLASS()
class UMySaveGame : public USaveGame
{
    GENERATED_BODY()

public:
    // 存档元数据
    UPROPERTY(SaveGame)
    int32 SaveVersion = 1;

    UPROPERTY(SaveGame)
    FString SaveName;

    UPROPERTY(SaveGame)
    FString PlayerId;

    UPROPERTY(SaveGame)
    FString MapName;

    UPROPERTY(SaveGame)
    FDateTime SaveTimestamp;

    UPROPERTY(SaveGame)
    float TotalPlayTime = 0.0f;

    // 游戏数据
    UPROPERTY(SaveGame)
    FCharacterSaveData CharacterData;

    UPROPERTY(SaveGame)
    FPlayerStatsSaveData PlayerStats;

    UPROPERTY(SaveGame)
    TArray<FString> UnlockedAchievements;

    UPROPERTY(SaveGame)
    TMap<FString, int32> GameProgressFlags;

    UPROPERTY(SaveGame)
    TArray<FString> DiscoveredLocations;

    // 设置
    UPROPERTY(SaveGame)
    float MusicVolume = 1.0f;

    UPROPERTY(SaveGame)
    float SFXVolume = 1.0f;

    UPROPERTY(SaveGame)
    float MouseSensitivity = 1.0f;

    // 辅助函数
    void InitializeNewSave(const FString& InPlayerId);
    void UpdateSaveTimestamp();
};

// MySaveGame.cpp
#include "MySaveGame.h"
#include "Kismet/GameplayStatics.h"

void UMySaveGame::InitializeNewSave(const FString& InPlayerId)
{
    PlayerId = InPlayerId;
    SaveVersion = 1;
    SaveTimestamp = FDateTime::Now();
    TotalPlayTime = 0.0f;

    // 默认值
    CharacterData.Health = 100.0f;
    CharacterData.Stamina = 100.0f;
}

void UMySaveGame::UpdateSaveTimestamp()
{
    SaveTimestamp = FDateTime::Now();
}
```

---

## 二、自定义存档序列化

### 2.1 高级序列化方法

```cpp
// MyAdvancedSaveGame.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/SaveGame.h"
#include "MyAdvancedSaveGame.generated.h"

// 自定义序列化接口
UINTERFACE()
class UCustomSerializable : public UInterface
{
    GENERATED_BODY()
};

class ICustomSerializable
{
    GENERATED_BODY()

public:
    virtual void SerializeToSaveGame(TMap<FString, FString>& OutData) = 0;
    virtual void DeserializeFromSaveGame(const TMap<FString, FString>& InData) = 0;
};

// 高级存档类
UCLASS()
class UMyAdvancedSaveGame : public USaveGame
{
    GENERATED_BODY()

public:
    // 使用字节数组存储复杂数据
    UPROPERTY(SaveGame)
    TArray<uint8> SerializedActorData;

    UPROPERTY(SaveGame)
    TArray<uint8> SerializedWorldData;

    UPROPERTY(SaveGame)
    TArray<uint8> SerializedCustomData;

    // 自定义序列化方法
    template<typename T>
    void SerializeObject(T* Object, TArray<uint8>& OutData);

    template<typename T>
    bool DeserializeObject(T* OutObject, const TArray<uint8>& InData);

    // Actor序列化
    void SaveActorState(AActor* Actor);
    bool LoadActorState(AActor* Actor);

    // 世界状态序列化
    void SaveWorldState(UWorld* World);
    void LoadWorldState(UWorld* World);

    // 二进制序列化辅助
    void WriteToArchive(FArchive& Ar, const TArray<uint8>& Data);
    void ReadFromArchive(FArchive& Ar, TArray<uint8>& Data);

private:
    // Actor引用映射
    TMap<FString, FString> ActorGuidToName;
};

// 实现
template<typename T>
void UMyAdvancedSaveGame::SerializeObject(T* Object, TArray<uint8>& OutData)
{
    if (!Object)
    {
        return;
    }

    FMemoryWriter Writer(OutData, true);

    // 写入类型信息
    FString ClassName = Object->GetClass()->GetName();
    Writer << ClassName;

    // 序列化对象
    Object->Serialize(Writer);
}

template<typename T>
bool UMyAdvancedSaveGame::DeserializeObject(T* OutObject, const TArray<uint8>& InData)
{
    if (!OutObject || InData.Num() == 0)
    {
        return false;
    }

    FMemoryReader Reader(InData, true);

    // 读取类型信息
    FString ClassName;
    Reader << ClassName;

    // 反序列化对象
    OutObject->Serialize(Reader);

    return true;
}

void UMyAdvancedSaveGame::SaveActorState(AActor* Actor)
{
    if (!Actor)
    {
        return;
    }

    FMemoryWriter Writer(SerializedActorData, true);

    // 保存Actor基本信息
    FString ActorName = Actor->GetName();
    FString ActorClass = Actor->GetClass()->GetPathName();
    FVector Location = Actor->GetActorLocation();
    FRotator Rotation = Actor->GetActorRotation();

    Writer << ActorName;
    Writer << ActorClass;
    Writer << Location;
    Writer << Rotation;

    // 保存组件状态
    TArray<UActorComponent*> Components;
    Actor->GetComponents(Components);

    int32 ComponentCount = Components.Num();
    Writer << ComponentCount;

    for (UActorComponent* Component : Components)
    {
        FString ComponentClass = Component->GetClass()->GetPathName();
        Writer << ComponentClass;

        // 可以扩展保存更多组件数据
    }

    UE_LOG(LogTemp, Log, TEXT("Saved actor state: %s"), *ActorName);
}

bool UMyAdvancedSaveGame::LoadActorState(AActor* Actor)
{
    if (!Actor || SerializedActorData.Num() == 0)
    {
        return false;
    }

    FMemoryReader Reader(SerializedActorData, true);

    FString ActorName, ActorClass;
    FVector Location;
    FRotator Rotation;

    Reader << ActorName;
    Reader << ActorClass;
    Reader << Location;
    Reader << Rotation;

    // 恢复Actor状态
    Actor->SetActorLocation(Location);
    Actor->SetActorRotation(Rotation);

    return true;
}
```

### 2.2 复杂数据类型序列化

```cpp
// 复杂数据类型的序列化处理

// 枚举序列化
template<typename EnumType>
FString EnumToString(EnumType Value)
{
    return StaticEnum<EnumType>()->GetNameStringByValue(static_cast<int64>(Value));
}

template<typename EnumType>
EnumType StringToEnum(const FString& String)
{
    return static_cast<EnumType>(StaticEnum<EnumType>()->GetValueByNameString(String));
}

// TMap序列化
template<typename KeyType, typename ValueType>
void SerializeMap(const TMap<KeyType, ValueType>& Map, TArray<uint8>& OutData)
{
    FMemoryWriter Writer(OutData, true);

    int32 Count = Map.Num();
    Writer << Count;

    for (const auto& Pair : Map)
    {
        Writer << Pair.Key;
        Writer << Pair.Value;
    }
}

template<typename KeyType, typename ValueType>
void DeserializeMap(TMap<KeyType, ValueType>& OutMap, const TArray<uint8>& InData)
{
    FMemoryReader Reader(InData, true);

    int32 Count;
    Reader << Count;

    OutMap.Empty();
    OutMap.Reserve(Count);

    for (int32 i = 0; i < Count; i++)
    {
        KeyType Key;
        ValueType Value;
        Reader << Key;
        Reader << Value;
        OutMap.Add(Key, Value);
    }
}

// TSet序列化
template<typename ElementType>
void SerializeSet(const TSet<ElementType>& Set, TArray<uint8>& OutData)
{
    FMemoryWriter Writer(OutData, true);

    int32 Count = Set.Num();
    Writer << Count;

    for (const ElementType& Element : Set)
    {
        Writer << Element;
    }
}

// 对象引用序列化
USTRUCT()
struct FObjectReferenceSaveData
{
    GENERATED_BODY()

    UPROPERTY(SaveGame)
    FString ObjectPath;

    UPROPERTY(SaveGame)
    FString ObjectName;

    // 运行时恢复引用
    UObject* ResolveObject() const
    {
        if (!ObjectPath.IsEmpty())
        {
            return LoadObject<UObject>(nullptr, *ObjectPath);
        }
        return nullptr;
    }
};
```

---

## 三、异步存档加载

### 3.1 异步存档管理器

```cpp
// MyAsyncSaveManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "MyAsyncSaveManager.generated.h"

// 存档操作结果
UENUM(BlueprintType)
enum class ESaveResult : uint8
{
    Success,
    Failed,
    Cancelled,
    Corrupted,
    VersionMismatch
};

// 存档信息
USTRUCT(BlueprintType)
struct FSaveSlotInfo
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString SlotName;

    UPROPERTY(BlueprintReadOnly)
    FString DisplayName;

    UPROPERTY(BlueprintReadOnly)
    FDateTime SaveTime;

    UPROPERTY(BlueprintReadOnly)
    FString MapName;

    UPROPERTY(BlueprintReadOnly)
    float PlayTime;

    UPROPERTY(BlueprintReadOnly)
    int64 FileSize;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnSaveComplete, ESaveResult, Result, const FString&, SlotName);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_ThreeParams(FOnLoadComplete, ESaveResult, Result, USaveGame*, SaveGame, const FString&, SlotName);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnDeleteComplete, bool, bSuccess, const FString&, SlotName);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnSaveSlotsRetrieved, const TArray<FSaveSlotInfo>&, Slots);

UCLASS()
class UMyAsyncSaveManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 异步保存
    UFUNCTION(BlueprintCallable, Category = "Save")
    void SaveGameAsync(USaveGame* SaveGame, const FString& SlotName);

    // 异步加载
    UFUNCTION(BlueprintCallable, Category = "Save")
    void LoadGameAsync(const FString& SlotName);

    // 异步删除
    UFUNCTION(BlueprintCallable, Category = "Save")
    void DeleteSaveAsync(const FString& SlotName);

    // 获取存档槽列表
    UFUNCTION(BlueprintCallable, Category = "Save")
    void GetSaveSlotsAsync();

    // 同步操作（阻塞）
    UFUNCTION(BlueprintCallable, Category = "Save")
    bool SaveGameSync(USaveGame* SaveGame, const FString& SlotName);

    UFUNCTION(BlueprintCallable, Category = "Save")
    USaveGame* LoadGameSync(const FString& SlotName);

    // 检查存档是否存在
    UFUNCTION(BlueprintPure, Category = "Save")
    bool DoesSaveExist(const FString& SlotName) const;

    // 委托
    UPROPERTY(BlueprintAssignable)
    FOnSaveComplete OnSaveComplete;

    UPROPERTY(BlueprintAssignable)
    FOnLoadComplete OnLoadComplete;

    UPROPERTY(BlueprintAssignable)
    FOnDeleteComplete OnDeleteComplete;

    UPROPERTY(BlueprintAssignable)
    FOnSaveSlotsRetrieved OnSaveSlotsRetrieved;

protected:
    // 存档目录
    FString SaveDirectory;

private:
    // 后台任务
    FGraphEventRef SaveTaskHandle;
    FGraphEventRef LoadTaskHandle;

    // 生成存档槽名称
    FString GenerateSlotPath(const FString& SlotName) const;
};

// MyAsyncSaveManager.cpp
#include "MyAsyncSaveGame.h"
#include "Kismet/GameplayStatics.h"
#include "Async/Async.h"
#include "HAL/PlatformFileManager.h"

void UMyAsyncSaveManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    SaveDirectory = FPaths::ProjectSavedDir() / TEXT("SaveGames");

    // 确保目录存在
    IPlatformFile& PlatformFile = FPlatformFileManager::Get().GetPlatformFile();
    if (!PlatformFile.DirectoryExists(*SaveDirectory))
    {
        PlatformFile.CreateDirectory(*SaveDirectory);
    }
}

void UMyAsyncSaveManager::SaveGameAsync(USaveGame* SaveGame, const FString& SlotName)
{
    if (!SaveGame)
    {
        OnSaveComplete.Broadcast(ESaveResult::Failed, SlotName);
        return;
    }

    // 在后台线程保存
    SaveTaskHandle = FFunctionGraphTask::CreateAndDispatchWhenReady(
        [this, SaveGame, SlotName]()
        {
            bool bSuccess = UGameplayStatics::SaveGameToSlot(
                SaveGame,
                SlotName,
                0
            );

            // 回到游戏线程广播结果
            AsyncTask(ENamedThreads::GameThread, [this, bSuccess, SlotName]()
            {
                ESaveResult Result = bSuccess ? ESaveResult::Success : ESaveResult::Failed;
                OnSaveComplete.Broadcast(Result, SlotName);
            });
        },
        TStatId(),
        nullptr,
        ENamedThreads::AnyBackgroundThreadNormalTask
    );
}

void UMyAsyncSaveManager::LoadGameAsync(const FString& SlotName)
{
    if (!DoesSaveExist(SlotName))
    {
        OnLoadComplete.Broadcast(ESaveResult::Failed, nullptr, SlotName);
        return;
    }

    // 在后台线程加载
    LoadTaskHandle = FFunctionGraphTask::CreateAndDispatchWhenReady(
        [this, SlotName]()
        {
            USaveGame* LoadedGame = UGameplayStatics::LoadGameFromSlot(SlotName, 0);

            // 回到游戏线程广播结果
            AsyncTask(ENamedThreads::GameThread, [this, LoadedGame, SlotName]()
            {
                ESaveResult Result = LoadedGame ? ESaveResult::Success : ESaveResult::Corrupted;
                OnLoadComplete.Broadcast(Result, LoadedGame, SlotName);
            });
        },
        TStatId(),
        nullptr,
        ENamedThreads::AnyBackgroundThreadNormalTask
    );
}

void UMyAsyncSaveManager::DeleteSaveAsync(const FString& SlotName)
{
    FFunctionGraphTask::CreateAndDispatchWhenReady(
        [this, SlotName]()
        {
            bool bSuccess = UGameplayStatics::DeleteGameInSlot(SlotName, 0);

            AsyncTask(ENamedThreads::GameThread, [this, bSuccess, SlotName]()
            {
                OnDeleteComplete.Broadcast(bSuccess, SlotName);
            });
        },
        TStatId(),
        nullptr,
        ENamedThreads::AnyBackgroundThreadNormalTask
    );
}

void UMyAsyncSaveManager::GetSaveSlotsAsync()
{
    FFunctionGraphTask::CreateAndDispatchWhenReady(
        [this]()
        {
            TArray<FSaveSlotInfo> Slots;

            IFileManager& FileManager = IFileManager::Get();
            TArray<FString> FoundFiles;
            FileManager.FindFiles(FoundFiles, *SaveDirectory, TEXT(".sav"));

            for (const FString& File : FoundFiles)
            {
                FString FullPath = SaveDirectory / File;
                FSaveSlotInfo Info;
                Info.SlotName = FPaths::GetBaseFilename(File);
                Info.FileSize = FileManager.FileSize(*FullPath);

                // 读取存档元数据
                FFileStatData StatData = FileManager.GetStatData(*FullPath);
                Info.SaveTime = StatData.ModificationTime;

                // 尝试读取存档获取更多信息
                USaveGame* TempSave = UGameplayStatics::LoadGameFromSlot(Info.SlotName, 0);
                if (UMySaveGame* MySave = Cast<UMySaveGame>(TempSave))
                {
                    Info.MapName = MySave->MapName;
                    Info.PlayTime = MySave->TotalPlayTime;
                    Info.DisplayName = MySave->SaveName;
                }

                Slots.Add(Info);
            }

            AsyncTask(ENamedThreads::GameThread, [this, Slots]()
            {
                OnSaveSlotsRetrieved.Broadcast(Slots);
            });
        },
        TStatId(),
        nullptr,
        ENamedThreads::AnyBackgroundThreadNormalTask
    );
}

bool UMyAsyncSaveManager::SaveGameSync(USaveGame* SaveGame, const FString& SlotName)
{
    if (!SaveGame)
    {
        return false;
    }

    return UGameplayStatics::SaveGameToSlot(SaveGame, SlotName, 0);
}

USaveGame* UMyAsyncSaveManager::LoadGameSync(const FString& SlotName)
{
    if (!DoesSaveExist(SlotName))
    {
        return nullptr;
    }

    return UGameplayStatics::LoadGameFromSlot(SlotName, 0);
}

bool UMyAsyncSaveManager::DoesSaveExist(const FString& SlotName) const
{
    return UGameplayStatics::DoesSaveGameExist(SlotName, 0);
}

FString UMyAsyncSaveManager::GenerateSlotPath(const FString& SlotName) const
{
    return SaveDirectory / SlotName + TEXT(".sav");
}
```

---

## 四、存档版本管理

### 4.1 版本迁移系统

```cpp
// MySaveMigration.h
#pragma once

#include "CoreMinimal.h"
#include "MySaveMigration.generated.h"

// 版本迁移器基类
UCLASS(Abstract, Blueprintable)
class USaveMigration : public UObject
{
    GENERATED_BODY()

public:
    // 源版本
    UPROPERTY(EditDefaultsOnly)
    int32 FromVersion = 0;

    // 目标版本
    UPROPERTY(EditDefaultsOnly)
    int32 ToVersion = 1;

    // 执行迁移
    UFUNCTION(BlueprintNativeEvent)
    bool Migrate(USaveGame* SaveGame);
    virtual bool Migrate_Implementation(USaveGame* SaveGame) { return true; }
};

// 版本管理器
UCLASS()
class USaveVersionManager : public UObject
{
    GENERATED_BODY()

public:
    // 注册迁移器
    void RegisterMigration(USaveMigration* Migration);

    // 获取最新版本
    int32 GetLatestVersion() const { return LatestVersion; }

    // 检查是否需要迁移
    bool NeedsMigration(int32 CurrentVersion) const;

    // 执行迁移
    bool PerformMigration(USaveGame* SaveGame, int32 FromVersion);

private:
    TMap<int32, TArray<USaveMigration*>> MigrationMap;
    int32 LatestVersion = 1;

    int32 FindMigrationPath(int32 FromVersion, TArray<int32>& OutPath);
};

// 具体迁移示例
UCLASS()
class UMigration_V0_to_V1 : public USaveMigration
{
    GENERATED_BODY()

public:
    UMigration_V0_to_V1()
    {
        FromVersion = 0;
        ToVersion = 1;
    }

    virtual bool Migrate_Implementation(USaveGame* SaveGame) override
    {
        UMySaveGame* MySave = Cast<UMySaveGame>(SaveGame);
        if (!MySave)
        {
            return false;
        }

        // V0 -> V1: 添加新字段默认值
        // 新版本添加了 PlayerStats 结构
        MySave->PlayerStats.TotalKills = 0;
        MySave->PlayerStats.TotalDeaths = 0;
        MySave->SaveVersion = 1;

        UE_LOG(LogTemp, Log, TEXT("Migrated save from V0 to V1"));
        return true;
    }
};

UCLASS()
class UMigration_V1_to_V2 : public USaveMigration
{
    GENERATED_BODY()

public:
    UMigration_V1_to_V2()
    {
        FromVersion = 1;
        ToVersion = 2;
    }

    virtual bool Migrate_Implementation(USaveGame* SaveGame) override
    {
        UMySaveGame* MySave = Cast<UMySaveGame>(SaveGame);
        if (!MySave)
        {
            return false;
        }

        // V1 -> V2: 数据结构调整
        // 例如：将单独的成就列表改为结构化数据
        // 旧数据迁移逻辑...

        MySave->SaveVersion = 2;
        UE_LOG(LogTemp, Log, TEXT("Migrated save from V1 to V2"));
        return true;
    }
};
```

### 4.2 自动版本检测

```cpp
// 在加载存档时自动检测并迁移
UCLASS()
class UMySaveSystem : public UObject
{
    GENERATED_BODY()

public:
    UFUNCTION(BlueprintCallable, Category = "Save")
    USaveGame* LoadWithMigration(const FString& SlotName)
    {
        USaveGame* LoadedSave = UGameplayStatics::LoadGameFromSlot(SlotName, 0);
        if (!LoadedSave)
        {
            return nullptr;
        }

        // 检查版本
        if (UMySaveGame* MySave = Cast<UMySaveGame>(LoadedSave))
        {
            int32 SaveVersion = MySave->SaveVersion;
            int32 CurrentVersion = VersionManager->GetLatestVersion();

            if (SaveVersion < CurrentVersion)
            {
                // 执行迁移
                if (VersionManager->PerformMigration(MySave, SaveVersion))
                {
                    // 保存迁移后的数据
                    UGameplayStatics::SaveGameToSlot(MySave, SlotName, 0);
                    UE_LOG(LogTemp, Log, TEXT("Save migrated from V%d to V%d"),
                        SaveVersion, CurrentVersion);
                }
                else
                {
                    UE_LOG(LogTemp, Error, TEXT("Save migration failed!"));
                    return nullptr;
                }
            }
            else if (SaveVersion > CurrentVersion)
            {
                // 存档版本太新，无法读取
                UE_LOG(LogTemp, Error, TEXT("Save version %d is newer than supported %d"),
                    SaveVersion, CurrentVersion);
                return nullptr;
            }
        }

        return LoadedSave;
    }

private:
    UPROPERTY()
    USaveVersionManager* VersionManager;
};
```

---

## 五、云存档集成

### 5.1 云存档接口

```cpp
// MyCloudSaveManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "Interfaces/OnlineCloudStorageInterface.h"
#include "MyCloudSaveManager.generated.h"

// 云存档状态
UENUM(BlueprintType)
enum class ECloudSaveState : uint8
{
    NotInitialized,
    Ready,
    Syncing,
    Error
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnCloudSyncComplete, bool, bSuccess);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnCloudStateChange, ECloudSaveState, NewState);

UCLASS()
class UMyCloudSaveManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // 同步操作
    UFUNCTION(BlueprintCallable, Category = "Cloud Save")
    void SyncToCloud();

    UFUNCTION(BlueprintCallable, Category = "Cloud Save")
    void SyncFromCloud();

    // 状态查询
    UFUNCTION(BlueprintPure, Category = "Cloud Save")
    ECloudSaveState GetState() const { return CurrentState; }

    UFUNCTION(BlueprintPure, Category = "Cloud Save")
    bool IsCloudAvailable() const;

    UFUNCTION(BlueprintPure, Category = "Cloud Save")
    int64 GetCloudStorageUsed() const;

    UFUNCTION(BlueprintPure, Category = "Cloud Save")
    int64 GetCloudStorageTotal() const;

    // 委托
    UPROPERTY(BlueprintAssignable)
    FOnCloudSyncComplete OnSyncComplete;

    UPROPERTY(BlueprintAssignable)
    FOnCloudStateChange OnStateChanged;

private:
    ECloudSaveState CurrentState = ECloudSaveState::NotInitialized;
    IOnlineCloudStoragePtr CloudStorageInterface;

    void SetState(ECloudSaveState NewState);
    void OnUserCloudDataReadComplete(bool bSuccess, const FUniqueNetId& UserId, const FString& FileName);
    void OnUserCloudDataWriteComplete(bool bSuccess, const FUniqueNetId& UserId, const FString& FileName);
};

// MyCloudSaveManager.cpp
#include "MyCloudSaveManager.h"
#include "OnlineSubsystem.h"
#include "Kismet/GameplayStatics.h"

void UMyCloudSaveManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    IOnlineSubsystem* OnlineSubsystem = IOnlineSubsystem::Get();
    if (OnlineSubsystem)
    {
        CloudStorageInterface = OnlineSubsystem->GetCloudStorageInterface();

        if (CloudStorageInterface.IsValid())
        {
            SetState(ECloudSaveState::Ready);
            UE_LOG(LogTemp, Log, TEXT("Cloud save manager initialized"));
        }
    }
}

void UMyCloudSaveManager::Deinitialize()
{
    Super::Deinitialize();
    CloudStorageInterface.Reset();
}

bool UMyCloudSaveManager::IsCloudAvailable() const
{
    return CloudStorageInterface.IsValid() &&
           CurrentState == ECloudSaveState::Ready;
}

void UMyCloudSaveManager::SyncToCloud()
{
    if (!IsCloudAvailable())
    {
        OnSyncComplete.Broadcast(false);
        return;
    }

    SetState(ECloudSaveState::Syncing);

    // 读取本地存档
    FString SlotName = TEXT("MainSave");
    USaveGame* LocalSave = UGameplayStatics::LoadGameFromSlot(SlotName, 0);

    if (!LocalSave)
    {
        SetState(ECloudSaveState::Error);
        OnSyncComplete.Broadcast(false);
        return;
    }

    // 序列化为字节数组
    TArray<uint8> SaveData;
    FObjectWriter Writer(SaveData);
    LocalSave->Serialize(Writer);

    // 上传到云
    IOnlineSubsystem* OnlineSubsystem = IOnlineSubsystem::Get();
    if (OnlineSubsystem)
    {
        TSharedPtr<const FUniqueNetId> UserId = OnlineSubsystem->GetIdentityInterface()->GetUniquePlayerId(0);
        if (UserId.IsValid())
        {
            FCloudStorageFileContents Contents;
            Contents.Data = MakeShareable(new TArray<uint8>(MoveTemp(SaveData)));
            Contents.FileName = SlotName;

            CloudStorageInterface->WriteUserFile(
                *UserId,
                SlotName,
                MoveTemp(Contents.Data)
            );
        }
    }
}

void UMyCloudSaveManager::SyncFromCloud()
{
    if (!IsCloudAvailable())
    {
        OnSyncComplete.Broadcast(false);
        return;
    }

    SetState(ECloudSaveState::Syncing);

    IOnlineSubsystem* OnlineSubsystem = IOnlineSubsystem::Get();
    if (OnlineSubsystem)
    {
        TSharedPtr<const FUniqueNetId> UserId = OnlineSubsystem->GetIdentityInterface()->GetUniquePlayerId(0);
        if (UserId.IsValid())
        {
            CloudStorageInterface->ReadUserFile(*UserId, TEXT("MainSave"));
        }
    }
}

void UMyCloudSaveManager::SetState(ECloudSaveState NewState)
{
    if (CurrentState != NewState)
    {
        CurrentState = NewState;
        OnStateChanged.Broadcast(NewState);
    }
}
```

---

## 六、实践任务

### 任务：实现完整的存档系统

```cpp
// 完整的存档系统示例

UCLASS()
class AMyGameMode : public AGameModeBase
{
    // 自动存档
    void AutoSave();

    // 存档验证
    bool ValidateSave(UMySaveGame* SaveGame);

    // 应用存档
    void ApplySave(UMySaveGame* SaveGame);

    // 创建新存档
    UMySaveGame* CreateNewSave(const FString& PlayerId);

private:
    FTimerHandle AutoSaveTimer;
    float AutoSaveInterval = 60.0f; // 每分钟自动存档
};
```

---

## 七、总结

本课我们学习了：

1. **SaveGame架构**：理解USaveGame的工作原理
2. **自定义序列化**：处理复杂数据类型
3. **异步操作**：非阻塞的存档加载
4. **版本管理**：存档迁移和兼容性
5. **云存档**：在线存储集成

---

## 八、下节预告

**第12课：关系型数据库集成**

将深入学习：
- SQLite在UE5中的应用
- MySQL/PostgreSQL集成方案
- 数据库连接池管理

---

*课程版本：1.0*
*最后更新：2026-04-10*