# 第3课：软引用与延迟加载

本课深入讲解软引用的实现原理和延迟加载的最佳实践。

---

## 3.1 软引用深入解析

### FSoftObjectPath 结构

```cpp
// FSoftObjectPath 是软引用的核心数据结构
// 位于 SoftObjectPath.h

struct FSoftObjectPath
{
private:
    // 资产路径，格式：/Game/Path/Asset.AssetName
    FName AssetPathName;

    // 子对象路径（可选），格式：AssetName:SubObjectName
    FString SubPathString;

public:
    // ========== 构造函数 ==========

    // 默认构造
    FSoftObjectPath() : AssetPathName(NAME_None) {}

    // 从字符串构造
    explicit FSoftObjectPath(const FString& PathString);

    // 从FName和子路径构造
    FSoftObjectPath(const FName& InAssetPath, const FString& InSubPath);

    // ========== 路径解析 ==========

    // 获取完整路径字符串
    FString ToString() const;

    // 获取资产路径（不含子对象）
    FName GetAssetPathName() const { return AssetPathName; }

    // 获取资产名称（最后一个组件）
    FString GetAssetName() const;

    // 获取子对象路径
    FString GetSubPathString() const { return SubPathString; }

    // 获取包名（不含资产名）
    FString GetLongPackageName() const;

    // ========== 对象解析 ==========

    // 解析为已加载的对象（不触发加载）
    UObject* ResolveObject() const;

    // 尝试加载对象
    UObject* TryLoad() const;

    // ========== 状态检查 ==========

    // 是否有效（路径非空）
    bool IsValid() const;

    // 是否等待加载
    bool IsPending() const;

    // 是否已加载
    bool IsAssetLoaded() const;
};
```

### 路径格式详解

```
┌─────────────────────────────────────────────────────────────────┐
│                    FSoftObjectPath 路径格式                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   完整路径示例：                                                 │
│   /Game/Characters/Hero/M_Hero_Body.M_Hero_Body:MaterialLayer1 │
│   └────┬──────────────────────┘ └──────┬──────┘ └──────┬─────┘ │
│        │                           │                │          │
│        │                           │                └── 子对象  │
│        │                           │                            │
│        │                           └── 资产名称                  │
│        │                                                        │
│        └── 包路径                                                │
│                                                                 │
│   分解：                                                        │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  AssetPathName = "/Game/Characters/Hero/M_Hero_Body"    │  │
│   │  SubPathString = "MaterialLayer1"                        │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   常见路径示例：                                                 │
│   ─────────────                                                 │
│   普通资产：  /Game/Meshes/SM_Cube.SM_Cube                      │
│   蓝图类：    /Game/BP_Actor.BP_Actor_C（注意_C后缀）           │
│   子对象：    /Game/Maps/MainMap.MainMap:PlayerStart           │
│   插件资产：  /PluginName/Assets/T_Icon.T_Icon                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### TSoftObjectPtr 结构

```cpp
// TSoftObjectPtr 是类型安全的软引用包装器
// 位于 SoftObjectPtr.h

template<class T>
class TSoftObjectPtr
{
private:
    // 内部的软对象路径
    FSoftObjectPath SoftObjectPath;

    // 缓存的弱指针（加载后使用）
    mutable TWeakObjectPtr<T> CachedWeakPtr;

public:
    // ========== 构造函数 ==========

    // 默认构造
    TSoftObjectPtr() {}

    // 从路径构造
    explicit TSoftObjectPtr(const FSoftObjectPath& InPath)
        : SoftObjectPath(InPath) {}

    // 从字符串构造
    explicit TSoftObjectPtr(const FString& PathString)
        : SoftObjectPath(PathString) {}

    // 从对象构造（提取路径）
    TSoftObjectPtr(T* Object);

    // ========== 状态检查 ==========

    // 是否有效（路径非空）
    bool IsValid() const { return SoftObjectPath.IsValid(); }

    // 是否等待加载（路径有效但对象未加载）
    bool IsPending() const
    {
        return IsValid() && !CachedWeakPtr.IsValid();
    }

    // 是否为空（路径无效）
    bool IsNull() const { return !IsValid(); }

    // 是否已加载
    bool IsLoaded() const;

    // ========== 获取对象 ==========

    // 获取对象（如果已加载）
    T* Get() const;

    // 同步加载并返回
    T* LoadSynchronous() const;

    // ========== 路径操作 ==========

    // 获取软对象路径
    const FSoftObjectPath& ToSoftObjectPath() const
    {
        return SoftObjectPath;
    }

    // 转换为字符串
    FString ToString() const { return SoftObjectPath.ToString(); }

    // 重置
    void Reset()
    {
        SoftObjectPath.Reset();
        CachedWeakPtr.Reset();
    }

    // ========== 操作符 ==========

    // 隐式转换为bool
    operator bool() const { return IsValid(); }

    // 比较操作符
    bool operator==(const TSoftObjectPtr& Other) const;
    bool operator!=(const TSoftObjectPtr& Other) const;
};
```

### TSoftClassPtr

```cpp
// TSoftClassPtr 用于引用蓝图类
template<class T>
class TSoftClassPtr
{
private:
    FSoftObjectPath SoftClassPath;
    mutable TWeakObjectPtr<UClass> CachedClass;

public:
    // 获取类（如果已加载）
    UClass* Get() const;

    // 同步加载类
    UClass* LoadSynchronous() const;

    // 获取路径
    const FSoftObjectPath& ToSoftObjectPath() const;

    // 检查状态
    bool IsValid() const;
    bool IsPending() const;
};
```

---

## 3.2 延迟加载模式

### 模式概述

```
┌─────────────────────────────────────────────────────────────────┐
│                    延迟加载生命周期                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   阶段1：配置期                                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  UPROPERTY(EditDefaultsOnly)                           │  │
│   │  TSoftObjectPtr<UStaticMesh> WeaponMesh;               │  │
│   │                                                         │  │
│   │  // 在蓝图编辑器中设置路径                               │  │
│   │  WeaponMesh = "/Game/Weapons/Rifle.Rifle"              │  │
│   │                                                         │  │
│   │  状态：仅存储路径字符串，资产未加载                       │  │
│   │  内存：极小（仅路径字符串）                               │  │
│   └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│   阶段2：等待期                                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  // 资产尚未需要，保持未加载状态                         │  │
│   │                                                         │  │
│   │  if (WeaponMesh.IsPending())                           │  │
│   │  {                                                      │  │
│   │      UE_LOG("Asset not loaded, waiting...");           │  │
│   │  }                                                      │  │
│   └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│   阶段3：加载期                                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  // 当需要使用资产时                                     │  │
│   │  void EquipWeapon()                                    │  │
│   │  {                                                      │  │
│   │      if (WeaponMesh.IsPending())                       │  │
│   │      {                                                  │  │
│   │          // 异步加载                                    │  │
│   │          AsyncLoad(WeaponMesh, OnLoaded);              │  │
│   │          ShowLoadingIndicator();                       │  │
│   │      }                                                  │  │
│   │  }                                                      │  │
│   │                                                         │  │
│   │  状态：正在加载资产                                      │  │
│   │  内存：逐渐增加                                          │  │
│   └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│   阶段4：使用期                                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  void OnWeaponLoaded()                                 │  │
│   │  {                                                      │  │
│   │      UStaticMesh* Mesh = WeaponMesh.Get();             │  │
│   │      MeshComponent->SetStaticMesh(Mesh);               │  │
│   │      HideLoadingIndicator();                           │  │
│   │  }                                                      │  │
│   │                                                         │  │
│   │  状态：资产已加载并使用                                  │  │
│   │  内存：完整占用                                          │  │
│   └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│   阶段5：释放期                                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  void UnequipWeapon()                                  │  │
│   │  {                                                      │  │
│   │      MeshComponent->SetStaticMesh(nullptr);            │  │
│   │      WeaponMesh.Reset();  // 释放引用                   │  │
│   │  }                                                      │  │
│   │                                                         │  │
│   │  // 后续GC会回收该资产（如果没有其他引用）               │  │
│   │                                                         │  │
│   │  状态：资产可被GC回收                                   │  │
│   │  内存：逐渐减少                                          │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.3 完整示例：武器皮肤系统

```cpp
// WeaponSkinSystem.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "WeaponSkinSystem.generated.h"

// 皮肤数据结构
USTRUCT(BlueprintType)
struct FWeaponSkin
{
    GENERATED_BODY()

    // 皮肤名称
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FName SkinId;

    // 皮肤显示名称
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FText DisplayName;

    // 皮肤网格（软引用）
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TSoftObjectPtr<UStaticMesh> SkinMesh;

    // 皮肤材质（软引用）
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TSoftObjectPtr<UMaterialInterface> SkinMaterial;

    // 预览图标（软引用）
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TSoftObjectPtr<UTexture2D> PreviewIcon;

    // 是否已解锁
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bUnlocked = false;
};

// 皮肤加载状态
UENUM(BlueprintType)
enum class ESkinLoadState : uint8
{
    NotLoaded,      // 未加载
    Loading,        // 加载中
    Loaded,         // 已加载
    Failed          // 加载失败
};

// 皮肤加载完成委托
DECLARE_DYNAMIC_MULTICAST_DELEGATE_ThreeParams(FOnSkinLoaded,
    FName, SkinId, UStaticMesh*, Mesh, UMaterialInterface*, Material);

// 皮肤加载失败委托
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnSkinLoadFailed,
    FName, SkinId, const FString&, ErrorReason);

UCLASS()
class AWeaponSkinSystem : public AActor
{
    GENERATED_BODY()

public:
    AWeaponSkinSystem();

    // 可用的皮肤列表
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Skins")
    TArray<FWeaponSkin> AvailableSkins;

    // 默认皮肤索引
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Skins")
    int32 DefaultSkinIndex = 0;

    // 当前激活的皮肤索引
    UPROPERTY(BlueprintReadOnly, Category = "Skins")
    int32 CurrentSkinIndex = -1;

    // 当前加载状态
    UPROPERTY(BlueprintReadOnly, Category = "Skins")
    ESkinLoadState CurrentLoadState = ESkinLoadState::NotLoaded;

    // 加载完成委托
    UPROPERTY(BlueprintAssignable, Category = "Skins")
    FOnSkinLoaded OnSkinLoaded;

    // 加载失败委托
    UPROPERTY(BlueprintAssignable, Category = "Skins")
    FOnSkinLoadFailed OnSkinLoadFailed;

    // 初始化系统
    UFUNCTION(BlueprintCallable, Category = "Skins")
    void Initialize();

    // 同步切换皮肤
    UFUNCTION(BlueprintCallable, Category = "Skins")
    bool ChangeSkinSync(int32 SkinIndex);

    // 异步切换皮肤
    UFUNCTION(BlueprintCallable, Category = "Skins")
    void ChangeSkinAsync(int32 SkinIndex);

    // 预加载皮肤
    UFUNCTION(BlueprintCallable, Category = "Skins")
    void PreloadSkin(int32 SkinIndex);

    // 卸载皮肤
    UFUNCTION(BlueprintCallable, Category = "Skins")
    void UnloadSkin(int32 SkinIndex);

    // 获取皮肤加载进度
    UFUNCTION(BlueprintPure, Category = "Skins")
    float GetLoadProgress() const;

    // 获取当前皮肤
    UFUNCTION(BlueprintPure, Category = "Skins")
    const FWeaponSkin& GetCurrentSkin() const;

protected:
    virtual void BeginPlay() override;

private:
    // 当前加载句柄
    TSharedPtr<FStreamableHandle> CurrentLoadHandle;

    // 已加载的资产缓存
    UPROPERTY()
    TMap<FName, UStaticMesh*> LoadedMeshes;

    UPROPERTY()
    TMap<FName, UMaterialInterface*> LoadedMaterials;

    // 待加载的皮肤索引
    int32 PendingSkinIndex = -1;

    // 加载单个皮肤的资产
    void LoadSkinAssets(FWeaponSkin& Skin);

    // 皮肤加载完成回调
    void OnSkinAssetsLoaded(FName SkinId);

    // 应用皮肤
    void ApplySkin(const FWeaponSkin& Skin);

    // 获取StreamableManager
    FStreamableManager& GetStreamableManager();
};
```

```cpp
// WeaponSkinSystem.cpp
#include "WeaponSkinSystem.h"
#include "AssetManager.h"

AWeaponSkinSystem::AWeaponSkinSystem()
{
    PrimaryActorTick.bCanEverTick = false;
}

void AWeaponSkinSystem::BeginPlay()
{
    Super::BeginPlay();

    // 加载默认皮肤
    if (AvailableSkins.IsValidIndex(DefaultSkinIndex))
    {
        ChangeSkinAsync(DefaultSkinIndex);
    }
}

void AWeaponSkinSystem::Initialize()
{
    // 初始化皮肤系统
    CurrentSkinIndex = -1;
    CurrentLoadState = ESkinLoadState::NotLoaded;
}

bool AWeaponSkinSystem::ChangeSkinSync(int32 SkinIndex)
{
    if (!AvailableSkins.IsValidIndex(SkinIndex))
    {
        UE_LOG(LogTemp, Warning, TEXT("Invalid skin index: %d"), SkinIndex);
        return false;
    }

    FWeaponSkin& Skin = AvailableSkins[SkinIndex];

    // 检查是否解锁
    if (!Skin.bUnlocked)
    {
        UE_LOG(LogTemp, Warning, TEXT("Skin not unlocked: %s"), *Skin.SkinId.ToString());
        return false;
    }

    // 取消正在进行的加载
    if (CurrentLoadHandle.IsValid())
    {
        CurrentLoadHandle->CancelHandle();
        CurrentLoadHandle.Reset();
    }

    // 同步加载资产
    UStaticMesh* Mesh = Skin.SkinMesh.LoadSynchronous();
    UMaterialInterface* Material = Skin.SkinMaterial.LoadSynchronous();

    if (!Mesh)
    {
        UE_LOG(LogTemp, Error, TEXT("Failed to load mesh for skin: %s"), *Skin.SkinId.ToString());
        return false;
    }

    // 缓存加载的资产
    LoadedMeshes.Add(Skin.SkinId, Mesh);
    if (Material)
    {
        LoadedMaterials.Add(Skin.SkinId, Material);
    }

    // 应用皮肤
    CurrentSkinIndex = SkinIndex;
    CurrentLoadState = ESkinLoadState::Loaded;
    ApplySkin(Skin);

    return true;
}

void AWeaponSkinSystem::ChangeSkinAsync(int32 SkinIndex)
{
    if (!AvailableSkins.IsValidIndex(SkinIndex))
    {
        OnSkinLoadFailed.Broadcast(NAME_None, TEXT("Invalid skin index"));
        return;
    }

    FWeaponSkin& Skin = AvailableSkins[SkinIndex];

    // 检查是否解锁
    if (!Skin.bUnlocked)
    {
        OnSkinLoadFailed.Broadcast(Skin.SkinId, TEXT("Skin not unlocked"));
        return;
    }

    // 如果资产已加载，直接应用
    if (!Skin.SkinMesh.IsPending() && Skin.SkinMesh.Get())
    {
        CurrentSkinIndex = SkinIndex;
        CurrentLoadState = ESkinLoadState::Loaded;
        ApplySkin(Skin);
        return;
    }

    // 取消之前的加载
    if (CurrentLoadHandle.IsValid())
    {
        CurrentLoadHandle->CancelHandle();
    }

    // 设置加载状态
    CurrentLoadState = ESkinLoadState::Loading;
    PendingSkinIndex = SkinIndex;

    // 收集要加载的资产
    TArray<FSoftObjectPath> AssetsToLoad;
    AssetsToLoad.Add(Skin.SkinMesh.ToSoftObjectPath());

    if (Skin.SkinMaterial.IsValid() && Skin.SkinMaterial.IsPending())
    {
        AssetsToLoad.Add(Skin.SkinMaterial.ToSoftObjectPath());
    }

    // 开始异步加载
    CurrentLoadHandle = GetStreamableManager().RequestAsyncLoad(
        AssetsToLoad,
        FStreamableDelegate::CreateUObject(this, &AWeaponSkinSystem::OnSkinAssetsLoaded, Skin.SkinId),
        FStreamableManager::DefaultAsyncLoadPriority,
        true  // 管理句柄，保持资产引用
    );

    if (!CurrentLoadHandle.IsValid())
    {
        CurrentLoadState = ESkinLoadState::Failed;
        OnSkinLoadFailed.Broadcast(Skin.SkinId, TEXT("Failed to start loading"));
    }
}

void AWeaponSkinSystem::PreloadSkin(int32 SkinIndex)
{
    if (!AvailableSkins.IsValidIndex(SkinIndex))
    {
        return;
    }

    FWeaponSkin& Skin = AvailableSkins[SkinIndex];

    if (!Skin.bUnlocked)
    {
        return;
    }

    // 如果已加载或正在加载，跳过
    if (!Skin.SkinMesh.IsPending())
    {
        return;
    }

    // 后台加载，不阻塞
    TArray<FSoftObjectPath> AssetsToLoad;
    AssetsToLoad.Add(Skin.SkinMesh.ToSoftObjectPath());

    if (Skin.SkinMaterial.IsValid())
    {
        AssetsToLoad.Add(Skin.SkinMaterial.ToSoftObjectPath());
    }

    GetStreamableManager().RequestAsyncLoad(
        AssetsToLoad,
        FStreamableDelegate::CreateLambda([Skin]()
        {
            UE_LOG(LogTemp, Log, TEXT("Preloaded skin: %s"), *Skin.SkinId.ToString());
        }),
        -100  // 低优先级
    );
}

void AWeaponSkinSystem::UnloadSkin(int32 SkinIndex)
{
    if (!AvailableSkins.IsValidIndex(SkinIndex))
    {
        return;
    }

    const FWeaponSkin& Skin = AvailableSkins[SkinIndex];

    // 从缓存移除
    LoadedMeshes.Remove(Skin.SkinId);
    LoadedMaterials.Remove(Skin.SkinId);

    // 如果是当前皮肤，清除引用
    if (CurrentSkinIndex == SkinIndex)
    {
        CurrentSkinIndex = -1;
        CurrentLoadState = ESkinLoadState::NotLoaded;
    }

    // 资产可被GC回收
}

float AWeaponSkinSystem::GetLoadProgress() const
{
    if (CurrentLoadHandle.IsValid() && !CurrentLoadHandle->HasLoadCompleted())
    {
        return CurrentLoadHandle->GetProgress();
    }
    return CurrentLoadState == ESkinLoadState::Loaded ? 1.0f : 0.0f;
}

const FWeaponSkin& AWeaponSkinSystem::GetCurrentSkin() const
{
    static FWeaponSkin EmptySkin;
    if (AvailableSkins.IsValidIndex(CurrentSkinIndex))
    {
        return AvailableSkins[CurrentSkinIndex];
    }
    return EmptySkin;
}

void AWeaponSkinSystem::OnSkinAssetsLoaded(FName SkinId)
{
    // 检查是否是我们期待的皮肤
    if (PendingSkinIndex < 0 || PendingSkinIndex >= AvailableSkins.Num())
    {
        CurrentLoadState = ESkinLoadState::Failed;
        OnSkinLoadFailed.Broadcast(SkinId, TEXT("Invalid pending skin"));
        return;
    }

    FWeaponSkin& Skin = AvailableSkins[PendingSkinIndex];
    if (Skin.SkinId != SkinId)
    {
        // 可能是之前的加载请求
        return;
    }

    // 获取加载的资产
    UStaticMesh* Mesh = Skin.SkinMesh.Get();
    UMaterialInterface* Material = Skin.SkinMaterial.Get();

    if (!Mesh)
    {
        CurrentLoadState = ESkinLoadState::Failed;
        OnSkinLoadFailed.Broadcast(SkinId, TEXT("Failed to load mesh"));
        CurrentLoadHandle.Reset();
        return;
    }

    // 缓存资产
    LoadedMeshes.Add(SkinId, Mesh);
    if (Material)
    {
        LoadedMaterials.Add(SkinId, Material);
    }

    // 更新状态
    CurrentSkinIndex = PendingSkinIndex;
    CurrentLoadState = ESkinLoadState::Loaded;
    PendingSkinIndex = -1;

    // 应用皮肤
    ApplySkin(Skin);

    // 广播完成事件
    OnSkinLoaded.Broadcast(SkinId, Mesh, Material);

    // 清理句柄（如果不需要保持引用）
    // CurrentLoadHandle.Reset();
}

void AWeaponSkinSystem::ApplySkin(const FWeaponSkin& Skin)
{
    // 这里实现实际的皮肤应用逻辑
    // 例如：设置网格组件的静态网格和材质

    UE_LOG(LogTemp, Log, TEXT("Applied skin: %s"), *Skin.SkinId.ToString());
}

FStreamableManager& AWeaponSkinSystem::GetStreamableManager()
{
    return UAssetManager::Get().GetStreamableManager();
}
```

---

## 3.4 按需加载策略

### 策略一：懒加载（Lazy Loading）

```cpp
// 第一次访问时加载
UCLASS()
class ULazyAssetCache : public UObject
{
    GENERATED_BODY()

public:
    template<typename T>
    T* GetOrLoadAsset(const FString& Path)
    {
        // 检查缓存
        if (T** Cached = AssetCache.Find(Path))
        {
            return *Cached;
        }

        // 加载资产
        T* Asset = LoadObject<T>(nullptr, *Path);
        if (Asset)
        {
            AssetCache.Add(Path, Asset);
        }

        return Asset;
    }

private:
    UPROPERTY()
    TMap<FString, UObject*> AssetCache;
};
```

### 策略二：预加载（Preloading）

```cpp
// 在空闲时预加载可能需要的资产
UCLASS()
class AAssetPreloader : public AActor
{
    GENERATED_BODY()

public:
    // 预加载队列
    TArray<FSoftObjectPath> PreloadQueue;

    // 预加载间隔
    UPROPERTY(EditDefaultsOnly)
    float PreloadInterval = 0.5f;

    virtual void BeginPlay() override
    {
        Super::BeginPlay();

        // 启动预加载定时器
        GetWorld()->GetTimerManager().SetTimer(
            PreloadTimer,
            this,
            &AAssetPreloader::ProcessPreloadQueue,
            PreloadInterval,
            true
        );
    }

    void AddToPreloadQueue(const FSoftObjectPath& Path)
    {
        if (!PreloadQueue.Contains(Path))
        {
            PreloadQueue.Add(Path);
        }
    }

private:
    FTimerHandle PreloadTimer;

    void ProcessPreloadQueue()
    {
        if (PreloadQueue.Num() == 0)
        {
            return;
        }

        // 取出第一个
        FSoftObjectPath PathToPreload = PreloadQueue[0];
        PreloadQueue.RemoveAt(0);

        // 后台加载
        UAssetManager::Get().GetStreamableManager().RequestAsyncLoad(
            PathToPreload,
            FStreamableDelegate::CreateLambda([]()
            {
                // 预加载完成，不需要处理
            }),
            -100  // 最低优先级
        );
    }
};
```

### 策略三：分批加载（Batch Loading）

```cpp
// 分批加载大量资产
UCLASS()
class UBatchAssetLoader : public UObject
{
    GENERATED_BODY()

public:
    struct FBatchLoadRequest
    {
        TArray<FSoftObjectPath> AllAssets;
        int32 CurrentIndex = 0;
        int32 BatchSize = 5;
        TFunction<void(int32, int32)> OnProgress;
        TFunction<void()> OnComplete;
    };

    void LoadAssetsInBatches(
        const TArray<FSoftObjectPath>& AssetPaths,
        int32 InBatchSize,
        TFunction<void(int32, int32)> OnProgress,
        TFunction<void()> OnComplete)
    {
        Request = MakeShared<FBatchLoadRequest>();
        Request->AllAssets = AssetPaths;
        Request->BatchSize = InBatchSize;
        Request->OnProgress = OnProgress;
        Request->OnComplete = OnComplete;

        ProcessNextBatch();
    }

private:
    TSharedPtr<FBatchLoadRequest> Request;
    TSharedPtr<FStreamableHandle> CurrentHandle;

    void ProcessNextBatch()
    {
        if (!Request.IsValid() || Request->CurrentIndex >= Request->AllAssets.Num())
        {
            // 全部完成
            if (Request.IsValid() && Request->OnComplete)
            {
                Request->OnComplete();
            }
            Request.Reset();
            return;
        }

        // 获取当前批次
        TArray<FSoftObjectPath> CurrentBatch;
        int32 EndIndex = FMath::Min(
            Request->CurrentIndex + Request->BatchSize,
            Request->AllAssets.Num()
        );

        for (int32 i = Request->CurrentIndex; i < EndIndex; i++)
        {
            CurrentBatch.Add(Request->AllAssets[i]);
        }

        // 加载当前批次
        CurrentHandle = UAssetManager::Get().GetStreamableManager().RequestAsyncLoad(
            CurrentBatch,
            FStreamableDelegate::CreateUObject(this, &UBatchAssetLoader::OnBatchComplete)
        );
    }

    void OnBatchComplete()
    {
        // 更新进度
        if (Request.IsValid() && Request->OnProgress)
        {
            Request->OnProgress(
                Request->CurrentIndex,
                Request->AllAssets.Num()
            );
        }

        // 移动到下一批次
        if (Request.IsValid())
        {
            Request->CurrentIndex += Request->BatchSize;
        }

        // 处理下一批次
        ProcessNextBatch();
    }
};
```

---

## 3.5 内存管理

### 软引用与内存释放

```cpp
// 正确的内存管理示例
UCLASS()
class AAssetManagerExample : public AActor
{
    GENERATED_BODY()

public:
    // 当前使用的资产（保持引用）
    UPROPERTY()
    UStaticMesh* CurrentMesh;

    // 软引用列表（不阻止GC）
    UPROPERTY(EditDefaultsOnly)
    TArray<TSoftObjectPtr<UStaticMesh>> AvailableMeshes;

    // 切换资产
    void SwitchToMesh(int32 Index)
    {
        // 释放当前资产
        CurrentMesh = nullptr;

        // 可选：强制GC
        // GEngine->ForceGarbageCollection(false);

        // 加载新资产
        if (AvailableMeshes.IsValidIndex(Index))
        {
            CurrentMesh = AvailableMeshes[Index].LoadSynchronous();
        }
    }

    // 清理所有缓存
    void ClearCache()
    {
        CurrentMesh = nullptr;

        // 重置所有软引用
        for (TSoftObjectPtr<UStaticMesh>& SoftPtr : AvailableMeshes)
        {
            SoftPtr.Reset();
        }

        // 请求GC
        GEngine->ForceGarbageCollection(true);
    }
};
```

### 内存监控

```cpp
// 监控软引用资产的内存占用
void MonitorSoftReferenceMemory()
{
    TArray<FSoftObjectPath> LoadedPaths;

    // 收集所有已加载的软引用
    for (TObjectIterator<UObject> It; It; ++It)
    {
        // 这里可以根据需要收集信息
    }

    // 输出内存统计
    const FPlatformMemoryStats& MemStats = FPlatformMemory::GetStats();
    UE_LOG(LogTemp, Log, TEXT("Used Memory: %.2f MB"),
        MemStats.UsedPhysical / 1024.0 / 1024.0);
}
```

---

## 3.6 蓝图集成

### 蓝图友好的软引用

```cpp
// 暴露给蓝图的软引用函数
UCLASS(BlueprintType)
class USoftReferenceLibrary : public UBlueprintFunctionLibrary
{
    GENERATED_BODY()

public:
    // 检查软引用状态
    UFUNCTION(BlueprintPure, Category = "Soft Reference")
    static bool IsSoftReferencePending(const TSoftObjectPtr<UObject>& SoftRef)
    {
        return SoftRef.IsPending();
    }

    // 检查是否已加载
    UFUNCTION(BlueprintPure, Category = "Soft Reference")
    static bool IsSoftReferenceLoaded(const TSoftObjectPtr<UObject>& SoftRef)
    {
        return SoftRef.Get() != nullptr;
    }

    // 获取路径字符串
    UFUNCTION(BlueprintPure, Category = "Soft Reference")
    static FString GetSoftReferencePath(const TSoftObjectPtr<UObject>& SoftRef)
    {
        return SoftRef.ToString();
    }

    // 同步加载
    UFUNCTION(BlueprintCallable, Category = "Soft Reference")
    static UObject* LoadSoftReferenceSync(UPARAM(ref) TSoftObjectPtr<UObject>& SoftRef)
    {
        return SoftRef.LoadSynchronous();
    }

    // 重置软引用
    UFUNCTION(BlueprintCallable, Category = "Soft Reference")
    static void ResetSoftReference(UPARAM(ref) TSoftObjectPtr<UObject>& SoftRef)
    {
        SoftRef.Reset();
    }
};
```

---

## 3.7 实践练习

### 练习1：实现资产缓存系统

```cpp
// 创建一个带有过期机制的资产缓存
UCLASS()
class UAssetCache : public UObject
{
    GENERATED_BODY()

public:
    struct FCachedAsset
    {
        UObject* Asset;
        double LastAccessTime;
        int32 AccessCount;
    };

    // 获取或加载资产
    template<typename T>
    T* GetOrLoad(const FString& Path, float MaxAge = 60.0f)
    {
        // 实现缓存逻辑
    }

    // 清理过期资产
    void CleanupExpiredAssets()
    {
        // 实现清理逻辑
    }

    // 清理最少使用
    void CleanupLRU(int32 MaxCount)
    {
        // 实现LRU清理
    }

private:
    TMap<FString, FCachedAsset> Cache;
};
```

### 练习2：实现预加载管理器

```cpp
// 基于玩家位置预加载附近区域的资产
UCLASS()
class ALocationBasedPreloader : public AActor
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly)
    float PreloadRadius = 10000.0f;

    virtual void Tick(float DeltaTime) override;

private:
    void UpdatePreloadBasedOnLocation(FVector PlayerLocation);
};
```

---

## 3.8 小结

### 核心概念

| 概念 | 说明 |
|------|------|
| **FSoftObjectPath** | 资产路径的封装，是软引用的基础 |
| **TSoftObjectPtr** | 类型安全的软引用，支持延迟加载 |
| **TSoftClassPtr** | 类的软引用，用于蓝图类 |
| **延迟加载** | 按需加载资产，减少初始内存占用 |

### 最佳实践

1. **大型资产用软引用**：避免初始加载压力
2. **按需加载**：只在需要时才加载资产
3. **预加载策略**：预测玩家行为，提前加载
4. **及时释放**：使用完毕后释放引用
5. **内存监控**：定期检查内存使用情况

### 下节课预告

第4课将学习资产注册表和资产管理器，掌握大型项目的资产管理技巧。
