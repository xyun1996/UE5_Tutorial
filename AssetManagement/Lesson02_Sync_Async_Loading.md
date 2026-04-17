# 第2课：同步与异步加载

本课深入讲解UE5中的资产加载机制，包括同步加载和异步加载的使用方法与最佳实践。

---

## 2.1 资产加载概述

### 加载方式对比

```
┌─────────────────────────────────────────────────────────────────┐
│                    加载方式对比                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   同步加载（Synchronous）                                       │
│   ────────────────────────────                                  │
│   • 阻塞主线程直到加载完成                                       │
│   • 适合小型资产或必须立即使用的情况                             │
│   • 可能导致卡顿                                                 │
│                                                                 │
│   时间线：                                                       │
│   ├─────────────────────────────────────────────────────────────┤
│   │ 游戏逻辑 │====加载资产(阻塞)====│ 继续执行 │                │
│   └─────────────────────────────────────────────────────────────┘
│                                                                 │
│   异步加载（Asynchronous）                                       │
│   ────────────────────────────                                  │
│   • 不阻塞主线程，后台加载                                       │
│   • 适合大型资产或可延迟使用的情况                               │
│   • 需要回调机制                                                 │
│                                                                 │
│   时间线：                                                       │
│   ├─────────────────────────────────────────────────────────────┤
│   │ 游戏逻辑 │ 继续执行 │ 继续执行 │ 回调触发 │                │
│   │          │====后台加载====│           │                    │
│   └─────────────────────────────────────────────────────────────┘
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.2 同步加载

### LoadObject

`LoadObject` 是最常用的同步加载函数。

```cpp
// 函数签名
template<class T>
T* LoadObject(UObject* Outer, const TCHAR* Name, ...)
{
    return Cast<T>(StaticLoadObject(T::StaticClass(), Outer, Name, ...));
}

// 基本用法
UStaticMesh* Mesh = LoadObject<UStaticMesh>(
    nullptr,                                    // Outer，通常为nullptr
    TEXT("/Game/Meshes/SM_Cube.SM_Cube")        // 资产路径
);

if (Mesh)
{
    // 加载成功，可以使用
    MeshComponent->SetStaticMesh(Mesh);
}
else
{
    // 加载失败
    UE_LOG(LogTemp, Warning, TEXT("Failed to load mesh"));
}
```

### LoadObject完整参数

```cpp
// 完整参数列表
UStaticMesh* Mesh = LoadObject<UStaticMesh>(
    nullptr,                                    // Outer：父对象（通常为nullptr）
    TEXT("/Game/Meshes/SM_Cube.SM_Cube"),       // Name：资产路径
    nullptr,                                    // Filename：指定文件名（通常为nullptr）
    0,                                          // LoadFlags：加载标志
    nullptr                                     // Sandbox：沙盒路径（通常为nullptr）
);

// 加载标志
enum ELoadFlags
{
    LOAD_None                   = 0x00000000,   // 默认
    LOAD_Async                  = 0x00000002,   // 异步加载
    LOAD_NoWarn                 = 0x00000004,   // 不显示警告
    LOAD_Verify                 = 0x00000010,   // 验证
    LOAD_NoRedirects            = 0x00100000,   // 不跟随重定向
    LOAD_Quiet                  = 0x02000000,   // 静默模式
};
```

### LoadClass

`LoadClass` 用于加载蓝图类，注意路径后缀 `_C`。

```cpp
// 加载蓝图类
UClass* ActorClass = LoadClass<AActor>(
    nullptr,
    TEXT("/Game/Blueprints/BP_MyActor.BP_MyActor_C")  // 注意_C后缀！
);

// 使用加载的类
if (ActorClass)
{
    AActor* NewActor = GetWorld()->SpawnActor<AActor>(ActorClass);
}

// 如果不确定类名，可以用蓝图名称
UClass* BlueprintClass = LoadObject<UBlueprint>(
    nullptr,
    TEXT("/Game/Blueprints/BP_MyActor.BP_MyActor")
)->GeneratedClass;
```

### FindObject

`FindObject` 只查找已加载的对象，不触发加载。

```cpp
// 查找已加载的对象
UStaticMesh* Mesh = FindObject<UStaticMesh>(
    nullptr,                                    // Outer
    TEXT("/Game/Meshes/SM_Cube.SM_Cube")        // 路径
);

if (Mesh)
{
    // 对象已加载
}
else
{
    // 对象未加载，需要用LoadObject
}

// FindObject不触发加载，性能更好
// 适合检查对象是否已加载
```

### ConstructorHelpers

在构造函数中加载资产的推荐方式。

```cpp
// 构造函数中加载
UCLASS()
class AMyActor : public AActor
{
    GENERATED_BODY()

    UPROPERTY(VisibleAnywhere)
    UStaticMeshComponent* MeshComponent;

    AMyActor()
    {
        // 创建组件
        MeshComponent = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("Mesh"));

        // 加载资产
        static ConstructorHelpers::FObjectFinder<UStaticMesh> MeshFinder(
            TEXT("/Game/Meshes/SM_Default.SM_Default")
        );

        if (MeshFinder.Succeeded())
        {
            MeshComponent->SetStaticMesh(MeshFinder.Object);
        }

        // 加载蓝图类
        static ConstructorHelpers::FClassFinder<AActor> ActorFinder(
            TEXT("/Game/Blueprints/BP_Enemy.BP_Enemy_C")
        );

        if (ActorFinder.Succeeded())
        {
            DefaultActorClass = ActorFinder.Class;
        }
    }

    UPROPERTY()
    TSubclassOf<AActor> DefaultActorClass;
};
```

### 同步加载的问题

```cpp
// 问题1：加载大型资产会卡顿
void LoadBigAsset()
{
    // 这可能阻塞主线程数百毫秒
    UTexture2D* HugeTexture = LoadObject<UTexture2D>(
        nullptr,
        TEXT("/Game/Textures/T_4KTexture.T_4KTexture")  // 4K纹理，可能很大
    );
}

// 问题2：批量加载会累积延迟
void LoadManyAssets()
{
    for (int32 i = 0; i < 100; i++)
    {
        // 每次加载都会阻塞
        FString Path = FString::Printf(TEXT("/Game/Assets/Asset_%d.Asset_%d"), i, i);
        UObject* Asset = LoadObject<UObject>(nullptr, *Path);
    }
    // 总延迟 = 所有加载时间之和
}

// 问题3：主线程阻塞影响帧率
void Tick(float DeltaTime)
{
    // 在Tick中同步加载会严重影响帧率！
    if (bNeedLoadAsset)
    {
        LoadObject<UObject>(...);  // ❌ 不要这样做
    }
}
```

---

## 2.3 异步加载

### FStreamableManager概述

```cpp
// 获取StreamableManager
#include "AssetManager.h"

UAssetManager& AssetManager = UAssetManager::Get();
FStreamableManager& StreamableManager = AssetManager.GetStreamableManager();
```

### 基本异步加载

```cpp
// 单个资产异步加载
void LoadAssetAsync()
{
    FSoftObjectPath AssetPath("/Game/Meshes/SM_BigMesh.SM_BigMesh");

    FStreamableDelegate Callback = FStreamableDelegate::CreateLambda([]()
    {
        UE_LOG(LogTemp, Log, TEXT("Asset loaded!"));
    });

    StreamableManager.RequestAsyncLoad(
        AssetPath,       // 要加载的路径
        Callback         // 完成回调
    );
}
```

### 使用加载句柄

```cpp
// TSharedPtr<FStreamableHandle> 用于管理加载过程
TSharedPtr<FStreamableHandle> LoadHandle;

void StartAsyncLoad()
{
    FSoftObjectPath AssetPath("/Game/Meshes/SM_BigMesh.SM_BigMesh");

    LoadHandle = StreamableManager.RequestAsyncLoad(
        AssetPath,
        FStreamableDelegate::CreateLambda([AssetPath]()
        {
            UObject* LoadedAsset = AssetPath.ResolveObject();
            if (LoadedAsset)
            {
                UE_LOG(LogTemp, Log, TEXT("Loaded: %s"), *LoadedAsset->GetName());
            }
        })
    );
}

// 检查加载状态
void CheckLoadStatus()
{
    if (LoadHandle.IsValid())
    {
        if (LoadHandle->HasLoadCompleted())
        {
            UE_LOG(LogTemp, Log, TEXT("Load completed"));
        }
        else if (LoadHandle->IsLoadingInProgress())
        {
            float Progress = LoadHandle->GetProgress();
            UE_LOG(LogTemp, Log, TEXT("Loading: %.1f%%"), Progress * 100.0f);
        }
    }
}

// 取消加载
void CancelLoad()
{
    if (LoadHandle.IsValid())
    {
        LoadHandle->CancelHandle();
        LoadHandle.Reset();
    }
}
```

### 批量异步加载

```cpp
// 加载多个资产
void LoadMultipleAssets()
{
    TArray<FSoftObjectPath> AssetPaths;
    AssetPaths.Add(FSoftObjectPath("/Game/Meshes/Mesh1.Mesh1"));
    AssetPaths.Add(FSoftObjectPath("/Game/Meshes/Mesh2.Mesh2"));
    AssetPaths.Add(FSoftObjectPath("/Game/Meshes/Mesh3.Mesh3"));

    TSharedPtr<FStreamableHandle> Handle = StreamableManager.RequestAsyncLoad(
        AssetPaths,
        FStreamableDelegate::CreateLambda([AssetPaths]()
        {
            UE_LOG(LogTemp, Log, TEXT("All assets loaded"));

            for (const FSoftObjectPath& Path : AssetPaths)
            {
                if (UObject* Asset = Path.ResolveObject())
                {
                    UE_LOG(LogTemp, Log, TEXT("  - %s"), *Asset->GetName());
                }
            }
        })
    );
}
```

### 带优先级的加载

```cpp
// 设置加载优先级
void LoadWithPriority()
{
    // 优先级常量
    const int32 LowPriority = -100;
    const int32 DefaultPriority = 0;
    const int32 HighPriority = 100;
    const int32 CriticalPriority = 1000;

    // 高优先级加载（UI相关、玩家立即可见）
    StreamableManager.RequestAsyncLoad(
        PlayerAssetPath,
        FStreamableDelegate::CreateLambda([]() {}),
        HighPriority
    );

    // 低优先级加载（预缓存、背景内容）
    StreamableManager.RequestAsyncLoad(
        CacheAssetPath,
        FStreamableDelegate::CreateLambda([]() {}),
        LowPriority
    );
}
```

### 加载回调方式

```cpp
// 方式1：Lambda
StreamableManager.RequestAsyncLoad(
    AssetPath,
    FStreamableDelegate::CreateLambda([]()
    {
        // 处理加载完成
    })
);

// 方式2：UObject成员函数
StreamableManager.RequestAsyncLoad(
    AssetPath,
    FStreamableDelegate::CreateUObject(this, &AMyActor::OnAssetLoaded)
);

void AMyActor::OnAssetLoaded()
{
    // 处理加载完成
}

// 方式3：原始函数指针
StreamableManager.RequestAsyncLoad(
    AssetPath,
    FStreamableDelegate::CreateStatic(&StaticCallbackFunction)
);

void StaticCallbackFunction()
{
    // 静态回调
}

// 方式4：带参数的回调
StreamableManager.RequestAsyncLoad(
    AssetPath,
    FStreamableDelegate::CreateUObject(this, &AMyActor::OnAssetLoadedWithId, AssetId)
);

void AMyActor::OnAssetLoadedWithId(FName Id)
{
    // 带参数的回调
}
```

---

## 2.4 TSoftObjectPtr异步加载

### 完整示例

```cpp
UCLASS()
class AWeaponSystem : public AActor
{
    GENERATED_BODY()

public:
    // 软引用配置
    UPROPERTY(EditDefaultsOnly, Category = "Weapons")
    TArray<TSoftObjectPtr<UStaticMesh>> WeaponMeshes;

    // 当前武器索引
    UPROPERTY(BlueprintReadOnly, Category = "Weapons")
    int32 CurrentWeaponIndex = -1;

    // 加载句柄
    TSharedPtr<FStreamableHandle> CurrentLoadHandle;

    // 同步加载
    UFUNCTION(BlueprintCallable, Category = "Weapons")
    void LoadWeaponSync(int32 Index)
    {
        if (!WeaponMeshes.IsValidIndex(Index)) return;

        TSoftObjectPtr<UStaticMesh>& SoftMesh = WeaponMeshes[Index];

        // 同步加载
        UStaticMesh* Mesh = SoftMesh.LoadSynchronous();
        if (Mesh)
        {
            CurrentWeaponIndex = Index;
            ApplyWeaponMesh(Mesh);
        }
    }

    // 异步加载
    UFUNCTION(BlueprintCallable, Category = "Weapons")
    void LoadWeaponAsync(int32 Index)
    {
        if (!WeaponMeshes.IsValidIndex(Index)) return;

        // 取消之前的加载
        if (CurrentLoadHandle.IsValid())
        {
            CurrentLoadHandle->CancelHandle();
        }

        TSoftObjectPtr<UStaticMesh>& SoftMesh = WeaponMeshes[Index];

        if (SoftMesh.IsPending())
        {
            // 需要加载
            CurrentLoadHandle = UAssetManager::Get()
                .GetStreamableManager()
                .RequestAsyncLoad(
                    SoftMesh.ToSoftObjectPath(),
                    FStreamableDelegate::CreateUObject(
                        this,
                        &AWeaponSystem::OnWeaponLoaded,
                        Index
                    )
                );
        }
        else
        {
            // 已加载
            CurrentWeaponIndex = Index;
            ApplyWeaponMesh(SoftMesh.Get());
        }
    }

    // 取消加载
    UFUNCTION(BlueprintCallable, Category = "Weapons")
    void CancelLoading()
    {
        if (CurrentLoadHandle.IsValid())
        {
            CurrentLoadHandle->CancelHandle();
            CurrentLoadHandle.Reset();
        }
    }

    // 获取加载进度
    UFUNCTION(BlueprintPure, Category = "Weapons")
    float GetLoadProgress() const
    {
        if (CurrentLoadHandle.IsValid() && !CurrentLoadHandle->HasLoadCompleted())
        {
            return CurrentLoadHandle->GetProgress();
        }
        return 1.0f;
    }

private:
    void OnWeaponLoaded(int32 Index)
    {
        if (WeaponMeshes.IsValidIndex(Index))
        {
            CurrentWeaponIndex = Index;
            CurrentLoadHandle.Reset();

            UStaticMesh* Mesh = WeaponMeshes[Index].Get();
            if (Mesh)
            {
                ApplyWeaponMesh(Mesh);
                OnWeaponChanged.Broadcast(Index);
            }
        }
    }

    void ApplyWeaponMesh(UStaticMesh* Mesh);

    UPROPERTY(BlueprintAssignable, Category = "Weapons")
    FOnWeaponChanged OnWeaponChanged;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnWeaponChanged, int32, WeaponIndex);
```

---

## 2.5 加载策略选择

### 决策流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                    加载策略决策                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   需要加载资产？                                                │
│       │                                                         │
│       └── 是 → 资产是否已加载？                                  │
│                  │                                              │
│                  ├── 是 → FindObject直接使用（最快）            │
│                  │                                              │
│                  └── 否 → 资产大小？                             │
│                              │                                  │
│                              ├── 小型（< 1MB）                  │
│                              │       │                          │
│                              │       ├── 必须立即使用？          │
│                              │       │       │                  │
│                              │       │       ├── 是 → 同步加载  │
│                              │       │       │                  │
│                              │       │       └── 否 → 异步加载  │
│                              │       │                          │
│                              │       └── 批量加载？              │
│                              │               │                  │
│                              │               ├── 是 → 异步批量  │
│                              │               │                  │
│                              │               └── 否 → 同步单个  │
│                              │                                  │
│                              └── 大型（> 1MB）                  │
│                                      │                          │
│                                      └── 异步加载（必须）        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 策略对比表

| 场景 | 推荐方式 | 原因 |
|------|---------|------|
| 构造函数初始化 | ConstructorHelpers | 只执行一次，编辑器时 |
| 小型配置资产 | 同步加载 | 延迟可忽略 |
| 大型纹理/网格 | 异步加载 | 避免卡顿 |
| 批量预加载 | 异步批量 | 后台执行 |
| UI必需资产 | 高优先级异步 | 快速响应 |
| 可选皮肤 | 按需异步 | 节省内存 |
| Tick中的加载 | 异步加载 | 避免帧率下降 |

### 最佳实践

```cpp
// ✅ 好的做法：检查是否已加载
UStaticMesh* GetOrLoadMesh(const FString& Path)
{
    // 先查找
    UStaticMesh* Mesh = FindObject<UStaticMesh>(nullptr, *Path);
    if (Mesh)
    {
        return Mesh;  // 已加载，直接返回
    }

    // 未加载，再加载
    return LoadObject<UStaticMesh>(nullptr, *Path);
}

// ✅ 好的做法：异步加载大型资产
void LoadBigAsset()
{
    StreamableManager.RequestAsyncLoad(
        AssetPath,
        FStreamableDelegate::CreateLambda([]()
        {
            // 加载完成后再使用
        })
    );
}

// ❌ 坏的做法：在Tick中同步加载
void Tick(float DeltaTime)
{
    if (NeedAsset)
    {
        LoadObject<UObject>(...);  // 会导致卡顿！
    }
}

// ✅ 好的做法：在Tick中启动异步加载
void Tick(float DeltaTime)
{
    if (NeedAsset && !bIsLoading)
    {
        bIsLoading = true;
        StreamableManager.RequestAsyncLoad(...);
    }
}
```

---

## 2.6 加载错误处理

### 错误类型

```cpp
// 1. 路径错误
UObject* Asset = LoadObject<UObject>(nullptr, TEXT("/Game/Wrong/Path"));
// 返回 nullptr

// 2. 类型不匹配
UStaticMesh* Mesh = LoadObject<UStaticMesh>(nullptr, TEXT("/Game/Textures/T_Some"));
// 返回 nullptr，类型转换失败

// 3. 资产不存在
UObject* Asset = LoadObject<UObject>(nullptr, TEXT("/Game/Deleted/Asset"));
// 返回 nullptr
```

### 错误处理模式

```cpp
// 完整的错误处理
UStaticMesh* SafeLoadMesh(const FString& Path)
{
    // 检查路径有效性
    if (Path.IsEmpty())
    {
        UE_LOG(LogTemp, Warning, TEXT("Empty asset path"));
        return nullptr;
    }

    // 尝试加载
    UStaticMesh* Mesh = LoadObject<UStaticMesh>(nullptr, *Path);

    if (!Mesh)
    {
        UE_LOG(LogTemp, Error, TEXT("Failed to load mesh: %s"), *Path);
        return nullptr;
    }

    // 验证资产有效性
    if (!Mesh->IsValidLowLevel())
    {
        UE_LOG(LogTemp, Error, TEXT("Loaded mesh is invalid: %s"), *Path);
        return nullptr;
    }

    return Mesh;
}

// 异步加载错误处理
void LoadAsyncWithErrorHandling()
{
    TSharedPtr<FStreamableHandle> Handle = StreamableManager.RequestAsyncLoad(
        AssetPath,
        FStreamableDelegate::CreateLambda([AssetPath]()
        {
            UObject* Asset = AssetPath.ResolveObject();
            if (!Asset)
            {
                UE_LOG(LogTemp, Error, TEXT("Async load failed: %s"),
                    *AssetPath.ToString());
                return;
            }

            // 使用资产
        })
    );

    // 检查句柄是否有效
    if (!Handle.IsValid())
    {
        UE_LOG(LogTemp, Error, TEXT("Failed to start async load"));
    }
}
```

---

## 2.7 实践练习

### 练习1：创建资产加载器

```cpp
// 创建一个通用的资产加载器
UCLASS()
class UAssetLoader : public UObject
{
    GENERATED_BODY()

public:
    // 同步加载
    template<typename T>
    static T* LoadSync(const FString& Path)
    {
        return LoadObject<T>(nullptr, *Path);
    }

    // 异步加载
    template<typename T>
    static TSharedPtr<FStreamableHandle> LoadAsync(
        const FString& Path,
        TFunction<void(T*)> OnLoaded)
    {
        FSoftObjectPath SoftPath(Path);

        return UAssetManager::Get().GetStreamableManager().RequestAsyncLoad(
            SoftPath,
            FStreamableDelegate::CreateLambda([SoftPath, OnLoaded]()
            {
                if (T* Asset = Cast<T>(SoftPath.ResolveObject()))
                {
                    OnLoaded(Asset);
                }
            })
        );
    }

    // 批量异步加载
    static TSharedPtr<FStreamableHandle> LoadMultipleAsync(
        const TArray<FString>& Paths,
        TFunction<void()> OnAllLoaded)
    {
        TArray<FSoftObjectPath> SoftPaths;
        for (const FString& Path : Paths)
        {
            SoftPaths.Add(FSoftObjectPath(Path));
        }

        return UAssetManager::Get().GetStreamableManager().RequestAsyncLoad(
            SoftPaths,
            FStreamableDelegate::CreateLambda(OnAllLoaded)
        );
    }
};

// 使用示例
void Example()
{
    // 同步加载
    UStaticMesh* Mesh = UAssetLoader::LoadSync<UStaticMesh>(
        TEXT("/Game/Meshes/SM_Cube.SM_Cube")
    );

    // 异步加载
    UAssetLoader::LoadAsync<UStaticMesh>(
        TEXT("/Game/Meshes/SM_Big.SM_Big"),
        [](UStaticMesh* LoadedMesh)
        {
            if (LoadedMesh)
            {
                // 使用加载的网格
            }
        }
    );
}
```

### 练习2：加载进度UI

```cpp
UCLASS()
class UAssetLoadingWidget : public UUserWidget
{
    GENERATED_BODY()

public:
    UFUNCTION(BlueprintCallable, Category = "Loading")
    void StartLoading(const TArray<FString>& AssetPaths)
    {
        TArray<FSoftObjectPath> SoftPaths;
        for (const FString& Path : AssetPaths)
        {
            SoftPaths.Add(FSoftObjectPath(Path));
        }

        LoadHandle = UAssetManager::Get().GetStreamableManager().RequestAsyncLoad(
            SoftPaths,
            FStreamableDelegate::CreateUObject(this, &UAssetLoadingWidget::OnLoadComplete)
        );

        // 开始更新进度
        GetWorld()->GetTimerManager().SetTimer(
            ProgressTimer,
            this,
            &UAssetLoadingWidget::UpdateProgress,
            0.1f,
            true
        );
    }

protected:
    // 进度条（在蓝图中绑定）
    UPROPERTY(BlueprintReadOnly, meta = (BindWidget))
    class UProgressBar* ProgressBar;

    // 状态文本（在蓝图中绑定）
    UPROPERTY(BlueprintReadOnly, meta = (BindWidget))
    class UTextBlock* StatusText;

private:
    TSharedPtr<FStreamableHandle> LoadHandle;
    FTimerHandle ProgressTimer;

    void UpdateProgress()
    {
        if (LoadHandle.IsValid())
        {
            float Progress = LoadHandle->GetProgress();
            SetProgress(Progress);

            if (LoadHandle->HasLoadCompleted())
            {
                GetWorld()->GetTimerManager().ClearTimer(ProgressTimer);
            }
        }
    }

    void OnLoadComplete()
    {
        SetProgress(1.0f);
        OnLoadingComplete.Broadcast();
    }

    void SetProgress(float Progress)
    {
        if (ProgressBar)
        {
            ProgressBar->SetPercent(Progress);
        }
        if (StatusText)
        {
            StatusText->SetText(FText::FromString(
                FString::Printf(TEXT("Loading... %.0f%%"), Progress * 100.0f)
            ));
        }
    }

    UPROPERTY(BlueprintAssignable, Category = "Loading")
    FOnLoadingComplete OnLoadingComplete;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnLoadingComplete);
```

### 练习3：调试命令

```cpp
// 自定义控制台命令
UCLASS()
class AAssetDebugger : public AActor
{
    GENERATED_BODY()

public:
    UFUNCTION(Exec)
    void AssetDebug_LoadSync(const FString& Path)
    {
        double StartTime = FPlatformTime::Seconds();

        UObject* Asset = LoadObject<UObject>(nullptr, *Path);

        double LoadTime = FPlatformTime::Seconds() - StartTime;

        if (Asset)
        {
            UE_LOG(LogTemp, Log, TEXT("Loaded: %s in %.3f ms"),
                *Asset->GetName(), LoadTime * 1000.0);
        }
        else
        {
            UE_LOG(LogTemp, Warning, TEXT("Failed to load: %s"), *Path);
        }
    }

    UFUNCTION(Exec)
    void AssetDebug_LoadAsync(const FString& Path)
    {
        double StartTime = FPlatformTime::Seconds();

        FSoftObjectPath SoftPath(Path);
        StreamableManager.RequestAsyncLoad(
            SoftPath,
            FStreamableDelegate::CreateLambda([SoftPath, StartTime]()
            {
                double TotalTime = FPlatformTime::Seconds() - StartTime;

                if (UObject* Asset = SoftPath.ResolveObject())
                {
                    UE_LOG(LogTemp, Log, TEXT("Async loaded: %s in %.3f ms"),
                        *Asset->GetName(), TotalTime * 1000.0);
                }
            })
        );
    }
};
```

---

## 2.8 小结

### 核心API回顾

| API | 用途 | 特点 |
|-----|------|------|
| `LoadObject<T>()` | 同步加载对象 | 阻塞，返回指针 |
| `LoadClass<T>()` | 同步加载类 | 阻塞，注意_C后缀 |
| `FindObject<T>()` | 查找已加载对象 | 不阻塞，不触发加载 |
| `RequestAsyncLoad()` | 异步加载 | 不阻塞，回调通知 |
| `ConstructorHelpers` | 构造函数加载 | 编辑器时执行 |

### 最佳实践总结

1. **小型资产同步加载**：延迟可忽略
2. **大型资产异步加载**：避免卡顿
3. **先Find后Load**：避免重复加载
4. **使用句柄管理**：可取消、可查进度
5. **合理设置优先级**：重要资产优先
6. **错误处理**：检查返回值

### 下节课预告

第3课将深入学习软引用与延迟加载，掌握按需加载资产的高级技巧。
