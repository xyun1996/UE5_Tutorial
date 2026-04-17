# UE5 资源管理课程大纲

本课程系统讲解UE5的资源（Asset）管理机制，涵盖资产的加载、引用、释放、流式加载等核心内容。

---

## 课程概览

| 课时 | 主题 | 难度 | 核心内容 |
|------|------|------|---------|
| 第1课 | 资产基础与引用机制 | ★☆☆ | 资产类型、硬引用、软引用、引用计数 |
| 第2课 | 同步与异步加载 | ★★☆ | LoadClass、LoadObject、AsyncLoad、加载策略 |
| 第3课 | 软引用与延迟加载 | ★★☆ | TSoftObjectPtr、FSoftObjectPath、SoftObjectReference |
| 第4课 | 资产注册与资产管理器 | ★★★ | AssetRegistry、AssetManager、Primary Assets |
| 第5课 | 垃圾回收与资产释放 | ★★★ | GC机制、引用链、强制释放、内存优化 |
| 第6课 | 流式加载与高级主题 | ★★★★ | Level Streaming、World Partition、资产热更新 |

---

## 第1课：资产基础与引用机制

### 学习目标

- 理解UE5中资产的概念与分类
- 掌握硬引用与软引用的区别
- 了解引用计数与资产生命周期

### 课程内容

#### 1.1 资产概述

```
┌─────────────────────────────────────────────────────────────────┐
│                      UE5 资产体系                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   核心资产类型                                                   │
│   ─────────────                                                 │
│   ├── uasset     所有资产的基础容器                              │
│   ├── umap       关卡资产                                       │
│   ├── ublueprint 蓝图类资产                                     │
│   ├── uwidget    UMG界面资产                                    │
│   └── ...                                                       │
│                                                                 │
│   常见资产类型                                                   │
│   ─────────────                                                 │
│   ├── 纹理类      UTexture2D, UTextureCube, URenderTarget      │
│   ├── 静态网格    UStaticMesh                                   │
│   ├── 骨骼网格    USkeletalMesh                                 │
│   ├── 材质类      UMaterial, UMaterialInstance                 │
│   ├── 动画类      UAnimSequence, UAnimMontage                  │
│   ├── 音频类      USoundWave, USoundCue                        │
│   ├── 粒子类      UNiagaraSystem, UParticleSystem              │
│   └── 数据类      UDataTable, UCurveTable                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 1.2 资产在内存中的表示

```cpp
// UObject是所有资产的基类
UCLASS()
class UAsset : public UObject
{
    // 资产名称
    FName AssetName;

    // 所在包（.uasset文件）
    UPackage* Outer;

    // 资产路径
    // 格式：/Game/Path/AssetName.AssetName
    FSoftObjectPath AssetPath;
};
```

#### 1.3 硬引用（Hard Reference）

**定义**：直接持有对象指针，被引用对象会被自动加载且不会被GC回收。

```cpp
// 方式1：UPROPERTY直接引用
UPROPERTY(EditDefaultsOnly, Category = "Assets")
UStaticMesh* HardMeshReference;  // 硬引用

// 方式2：代码中直接引用
UStaticMesh* Mesh = LoadObject<UStaticMesh>(...);
```

**硬引用特点**：
- 资产会随持有者一起加载
- 增加内存占用
- 可能导致循环依赖
- 适合高频使用的核心资产

```
资产加载链：
┌─────────────┐     硬引用      ┌─────────────┐
│  Actor BP   │ ──────────────► │ StaticMesh  │
└─────────────┘                 └─────────────┘
       │                               │
       │ 加载                          │ 自动加载
       ▼                               ▼
   内存中                         内存中
```

#### 1.4 软引用（Soft Reference）

**定义**：持有资产路径而非对象指针，需要时才加载，不影响资产生命周期。

```cpp
// 方式1：TSoftObjectPtr
UPROPERTY(EditDefaultsOnly, Category = "Assets")
TSoftObjectPtr<UStaticMesh> SoftMeshReference;

// 方式2：TSoftClassPtr（用于类引用）
UPROPERTY(EditDefaultsOnly, Category = "Classes")
TSoftClassPtr<AActor> SoftActorClass;

// 方式3：FSoftObjectPath
UPROPERTY(EditDefaultsOnly, Category = "Assets")
FSoftObjectPath SoftObjectPath;

// 方式4：FSoftClassPath
UPROPERTY(EditDefaultsOnly, Category = "Classes")
FSoftClassPath SoftClassPath;
```

**软引用特点**：
- 只存储路径字符串，不加载资产
- 减少初始内存占用
- 避免循环依赖
- 适合可选或延迟加载的资产

#### 1.5 引用关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                      引用关系示意图                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   关卡资产 (Level.umap)                                         │
│   │                                                             │
│   ├── [硬引用] Actor A ──────────► StaticMesh (始终在内存)      │
│   │         │                                                    │
│   │         └── [软引用] Particle ──► 仅路径字符串              │
│   │                                   需要时才加载               │
│   │                                                             │
│   ├── [硬引用] Actor B ──────────► Material (始终在内存)        │
│   │         │                                                    │
│   │         └── [硬引用] Texture ─► 随Material加载              │
│   │                                                             │
│   └── [软引用] Optional Asset ───► 仅路径                       │
│                                                                 │
│   加载顺序：                                                     │
│   1. Level.umap 加载                                            │
│   2. 硬引用的 Actor A/B 加载                                    │
│   3. 硬引用的 Mesh/Material 加载                                │
│   4. 软引用资产不加载（直到手动请求）                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 1.6 引用计数与生命周期

```cpp
// 查看对象的引用计数
void DebugObjectRefCount(UObject* Obj)
{
    if (Obj)
    {
        FUObjectItem* Item = Obj->GetGUObjectItem();
        int32 RefCount = Item->GetRefCount();
        UE_LOG(LogTemp, Log, TEXT("%s RefCount: %d"),
            *Obj->GetName(), RefCount);
    }
}

// 资产被持有的方式
// 1. UPROPERTY引用 → 阻止GC回收
// 2. UPGC::ReferencedObjects → GC根集合
// 3. AddReferencedObjects → 自定义引用
// 4. TStrongObjectPtr → 引用计数方式
```

### 实践练习

1. 创建一个Actor，分别使用硬引用和软引用引用不同的资产
2. 使用 `obj list` 命令观察资产加载情况
3. 使用 `stat memory` 分析内存占用

### 扩展阅读

- `Object.h` - UObject基类定义
- `SoftObjectPath.h` - 软引用实现
- `UObjectGlobals.h` - 对象加载函数

---

## 第2课：同步与异步加载

### 学习目标

- 掌握同步加载函数的使用
- 理解异步加载的机制与应用场景
- 学会选择合适的加载策略

### 课程内容

#### 2.1 同步加载

**特点**：阻塞主线程，加载完成后继续执行。适合小型资产或必须立即使用的情况。

```cpp
// 方式1：LoadObject - 最常用的同步加载
UStaticMesh* Mesh = LoadObject<UStaticMesh>(
    nullptr,                                    // Outer（通常为nullptr）
    TEXT("/Game/Meshes/MyMesh.MyMesh")          // 路径
);

// 方式2：LoadClass - 加载蓝图类
UClass* ActorClass = LoadClass<AActor>(
    nullptr,
    TEXT("/Game/Blueprints/BP_MyActor.BP_MyActor_C")  // 注意_C后缀
);

// 方式3：StaticLoadObject - 旧式API（不推荐）
UObject* Obj = StaticLoadObject(
    UStaticMesh::StaticClass(),
    nullptr,
    TEXT("/Game/Meshes/MyMesh.MyMesh")
);

// 方式4：FindObject - 仅查找已加载的对象（不触发加载）
UStaticMesh* LoadedMesh = FindObject<UStaticMesh>(
    nullptr,
    TEXT("/Game/Meshes/MyMesh.MyMesh")
);
```

**同步加载的问题**：
```
主线程时间线：
├────────────────────────────────────────────────────────────────┤
│  游戏逻辑  │  同步加载(阻塞)  │  游戏逻辑继续  │               │
│            ║██████████████████║                │               │
│            ║   可能卡顿       ║                │               │
└────────────────────────────────────────────────────────────────┘
```

#### 2.2 异步加载

**特点**：不阻塞主线程，加载完成后通过回调通知。适合大型资产或非即时需要的资产。

##### 2.2.1 FStreamableManager

```cpp
// 获取StreamableManager
UAssetManager& AssetManager = UAssetManager::Get();
FStreamableManager& StreamableManager = AssetManager.GetStreamableManager();

// 定义要加载的资产列表
TArray<FSoftObjectPath> AssetsToLoad;
AssetsToLoad.Add(FSoftObjectPath("/Game/Meshes/BigMesh.BigMesh"));
AssetsToLoad.Add(FSoftObjectPath("/Game/Textures/BigTexture.BigTexture"));

// 异步加载
FStreamableDelegate Callback = FStreamableDelegate::CreateLambda([&]()
{
    UE_LOG(LogTemp, Log, TEXT("Assets loaded!"));

    // 加载完成，可以安全使用
    UStaticMesh* Mesh = Cast<UStaticMesh>(
        AssetToLoad.ResolveObject()
    );
});

StreamableManager.RequestAsyncLoad(
    AssetsToLoad,           // 要加载的资产列表
    Callback,               // 完成回调
    FStreamableManager::DefaultAsyncLoadPriority,  // 优先级
    false,                  // bManageActiveHandle
    false,                  // bStartStalled
    TEXT("MyLoadingTag")    // 调试标签
);
```

##### 2.2.2 TSoftObjectPtr异步加载

```cpp
// 使用TSoftObjectPtr进行异步加载
UPROPERTY(EditDefaultsOnly)
TSoftObjectPtr<UStaticMesh> SoftMesh;

void LoadMeshAsync()
{
    if (SoftMesh.IsPending())
    {
        UAssetManager& AssetManager = UAssetManager::Get();
        FStreamableManager& Streamable = AssetManager.GetStreamableManager();

        Streamable.RequestAsyncLoad(
            SoftMesh.ToSoftObjectPath(),
            FStreamableDelegate::CreateUObject(this, &AClass::OnMeshLoaded)
        );
    }
}

void OnMeshLoaded()
{
    UStaticMesh* Mesh = SoftMesh.Get();
    if (Mesh)
    {
        // 使用加载完成的网格
        MeshComponent->SetStaticMesh(Mesh);
    }
}
```

##### 2.2.3 异步加载句柄

```cpp
// 获取加载句柄用于管理
TSharedPtr<FStreamableHandle> LoadHandle;

void StartAsyncLoad()
{
    FSoftObjectPath AssetPath("/Game/BigAsset.BigAsset");

    LoadHandle = StreamableManager.RequestAsyncLoad(
        AssetPath,
        FStreamableDelegate::CreateLambda([]()
        {
            UE_LOG(LogTemp, Log, TEXT("Load complete"));
        })
    );
}

void CancelLoad()
{
    if (LoadHandle.IsValid())
    {
        LoadHandle->CancelHandle();  // 取消加载
        LoadHandle.Reset();
    }
}

bool IsLoadComplete()
{
    return LoadHandle.IsValid() && LoadHandle->HasLoadCompleted();
}

float GetLoadProgress()
{
    if (LoadHandle.IsValid())
    {
        return LoadHandle->GetProgress();  // 0.0 - 1.0
    }
    return 0.0f;
}
```

#### 2.3 加载策略对比

```
┌─────────────────────────────────────────────────────────────────┐
│                      加载策略决策树                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   需要加载资产？                                                │
│       │                                                         │
│       ├── 否 → 不加载                                           │
│       │                                                         │
│       └── 是 → 资产是否已加载？                                  │
│                  │                                              │
│                  ├── 是 → FindObject直接使用                    │
│                  │                                              │
│                  └── 否 → 资产大小？                             │
│                              │                                  │
│                              ├── 小型（<1MB）                   │
│                              │       │                          │
│                              │       └── 必须立即使用？          │
│                              │              │                   │
│                              │              ├── 是 → 同步加载   │
│                              │              │                   │
│                              │              └── 否 → 异步加载   │
│                              │                                  │
│                              └── 大型（>1MB）                   │
│                                      │                          │
│                                      └── 异步加载              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 2.4 批量加载

```cpp
// 批量异步加载
void LoadMultipleAssets(const TArray<FSoftObjectPath>& AssetPaths)
{
    UAssetManager& AssetManager = UAssetManager::Get();
    FStreamableManager& Streamable = AssetManager.GetStreamableManager();

    TSharedPtr<FStreamableHandle> Handle = Streamable.RequestAsyncLoad(
        AssetPaths,
        FStreamableDelegate::CreateLambda([AssetPaths]()
        {
            for (const FSoftObjectPath& Path : AssetPaths)
            {
                if (UObject* LoadedAsset = Path.ResolveObject())
                {
                    UE_LOG(LogTemp, Log, TEXT("Loaded: %s"), *Path.ToString());
                }
            }
        }),
        FStreamableManager::DefaultAsyncLoadPriority,
        true  // 管理句柄，保持资产引用
    );
}
```

#### 2.5 加载优先级

```cpp
namespace FStreamableManager
{
    // 优先级常量
    const int32 DefaultAsyncLoadPriority = 0;
    const int32 HighPriorityAsyncLoadPriority = 100;
    const int32 HighestPriority = 1000;
    const int32 LowPriority = -100;
}

// 设置加载优先级
StreamableManager.RequestAsyncLoad(
    AssetPath,
    Callback,
    FStreamableManager::HighPriorityAsyncLoadPriority  // 高优先级
);
```

### 实践练习

1. 创建一个Actor，在BeginPlay时同步加载一个小型资产
2. 实现异步加载一个大型材质，加载完成后应用到Mesh上
3. 实现加载进度显示UI

### 扩展阅读

- `AssetManager.h` - 资产管理器
- `StreamableManager.h` - 流式加载管理器
- `AsyncLoad.h` - 异步加载实现

---

## 第3课：软引用与延迟加载

### 学习目标

- 深入理解软引用的实现原理
- 掌握延迟加载的最佳实践
- 学会使用资产引用属性

### 课程内容

#### 3.1 FSoftObjectPath 深入解析

```cpp
// FSoftObjectPath 结构
struct FSoftObjectPath
{
private:
    // 资产路径，格式：/Game/Path/Asset.AssetName
    FName AssetPath;

    // 子对象路径（可选），格式：AssetName:SubObject
    FString SubPathString;

public:
    // 构造方式
    FSoftObjectPath();
    FSoftObjectPath(const FString& PathString);
    FSoftObjectPath(const FName& InAssetPath, const FString& InSubPath);

    // 核心方法
    FString ToString() const;           // 获取完整路径字符串
    FName GetAssetPath() const;         // 获取资产路径
    FString GetAssetName() const;       // 获取资产名称
    UObject* ResolveObject() const;     // 解析为已加载的对象
    bool IsValid() const;               // 检查路径是否有效
    bool IsPending() const;             // 检查是否等待加载
};
```

#### 3.2 TSoftObjectPtr 详解

```cpp
// TSoftObjectPtr 是 FSoftObjectPath 的类型安全包装
template<class T>
class TSoftObjectPtr
{
private:
    FSoftObjectPath SoftObjectPath;
    mutable TWeakObjectPtr<T> CachedPtr;  // 缓存的弱指针

public:
    // 构造
    TSoftObjectPtr() {}
    TSoftObjectPtr(T* Object);
    TSoftObjectPtr(const FSoftObjectPath& InPath);

    // 核心方法
    bool IsPending() const;              // 是否需要加载
    bool IsValid() const;                // 路径是否有效
    bool IsNull() const;                 // 是否为空
    T* Get() const;                       // 获取对象（如果已加载）
    T* LoadSynchronous();                // 同步加载并返回

    FSoftObjectPath ToSoftObjectPath() const;  // 获取路径
    FString ToString() const;

    // 重置
    void Reset();
};
```

**使用示例**：
```cpp
// 声明
UPROPERTY(EditDefaultsOnly, Category = "Assets")
TSoftObjectPtr<UStaticMesh> WeaponMesh;

// 检查状态
if (WeaponMesh.IsPending())
{
    UE_LOG(LogTemp, Log, TEXT("WeaponMesh not loaded yet"));
}

// 同步加载
UStaticMesh* Mesh = WeaponMesh.LoadSynchronous();
if (Mesh)
{
    MeshComponent->SetStaticMesh(Mesh);
}

// 异步加载
void LoadWeaponMeshAsync()
{
    UAssetManager::Get().GetStreamableManager().RequestAsyncLoad(
        WeaponMesh.ToSoftObjectPath(),
        FStreamableDelegate::CreateLambda([this]()
        {
            if (UStaticMesh* Mesh = WeaponMesh.Get())
            {
                OnWeaponMeshLoaded(Mesh);
            }
        })
    );
}
```

#### 3.3 TSoftClassPtr

```cpp
// TSoftClassPtr 用于引用蓝图类
template<class T>
class TSoftClassPtr
{
    // 类似TSoftObjectPtr，但专门用于 UClass
};

// 使用示例
UPROPERTY(EditDefaultsOnly, Category = "Classes")
TSoftClassPtr<AActor> SpawnableActorClass;

void SpawnActorFromSoftClass()
{
    // 同步加载类
    UClass* ActorClass = SpawnableActorClass.LoadSynchronous();
    if (ActorClass)
    {
        FActorSpawnParameters SpawnParams;
        AActor* NewActor = GetWorld()->SpawnActor<AActor>(
            ActorClass,
            SpawnLocation,
            SpawnRotation,
            SpawnParams
        );
    }
}
```

#### 3.4 延迟加载模式

```
┌─────────────────────────────────────────────────────────────────┐
│                      延迟加载流程图                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   初始化阶段                                                     │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  TSoftObjectPtr<UMaterial> MaterialRef;                 │  │
│   │  MaterialRef = "/Game/Materials/M_Optional.M_Optional"  │  │
│   │                                                         │  │
│   │  此时：内存中只有路径字符串，资产未加载                   │  │
│   └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│   使用阶段                                                       │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  void ApplyMaterial()                                   │  │
│   │  {                                                      │  │
│   │      if (MaterialRef.IsPending())                       │  │
│   │      {                                                  │  │
│   │          // 按需加载                                    │  │
│   │          AsyncLoad(MaterialRef, [this]() {              │  │
│   │              ApplyToMesh(MaterialRef.Get());             │  │
│   │          });                                            │  │
│   │      }                                                  │  │
│   │  }                                                      │  │
│   └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│   释放阶段                                                       │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  void ReleaseMaterial()                                 │  │
│   │  {                                                      │  │
│   │      MaterialRef.Reset();  // 释放引用                  │  │
│   │      // 资产可被GC回收                                  │  │
│   │  }                                                      │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 3.5 软引用最佳实践

```cpp
// 推荐实践：组件中的软引用
UCLASS()
class UWeaponComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    // 配置为EditDefaultsOnly，在蓝图类中设置
    UPROPERTY(EditDefaultsOnly, Category = "Weapon")
    TArray<TSoftObjectPtr<UStaticMesh>> WeaponMeshes;

    // 当前激活的武器索引
    UPROPERTY(BlueprintReadWrite, Category = "Weapon")
    int32 CurrentWeaponIndex = 0;

    // 加载句柄，用于管理加载过程
    TSharedPtr<FStreamableHandle> CurrentLoadHandle;

    // 切换武器
    void SwitchWeapon(int32 Index)
    {
        if (WeaponMeshes.IsValidIndex(Index))
        {
            // 取消之前的加载
            if (CurrentLoadHandle.IsValid())
            {
                CurrentLoadHandle->CancelHandle();
            }

            CurrentWeaponIndex = Index;
            TSoftObjectPtr<UStaticMesh>& WeaponMesh = WeaponMeshes[Index];

            if (WeaponMesh.IsPending())
            {
                LoadWeaponAsync(WeaponMesh);
            }
            else
            {
                ApplyWeaponMesh(WeaponMesh.Get());
            }
        }
    }

private:
    void LoadWeaponAsync(TSoftObjectPtr<UStaticMesh>& SoftMesh)
    {
        CurrentLoadHandle = UAssetManager::Get()
            .GetStreamableManager()
            .RequestAsyncLoad(
                SoftMesh.ToSoftObjectPath(),
                FStreamableDelegate::CreateUObject(
                    this,
                    &UWeaponComponent::OnWeaponLoaded,
                    CurrentWeaponIndex
                )
            );
    }

    void OnWeaponLoaded(int32 LoadedIndex)
    {
        if (LoadedIndex == CurrentWeaponIndex)
        {
            ApplyWeaponMesh(WeaponMeshes[LoadedIndex].Get());
        }
        CurrentLoadHandle.Reset();
    }

    void ApplyWeaponMesh(UStaticMesh* Mesh);
};
```

#### 3.6 蓝图中的软引用

```cpp
// C++中暴露给蓝图的软引用
UCLASS(BlueprintType)
class AWeaponActor : public AActor
{
    GENERATED_BODY()

    // 暴露给蓝图编辑的软引用
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Weapon")
    TSoftObjectPtr<UStaticMesh> BaseMesh;

    // 蓝图可调用的加载函数
    UFUNCTION(BlueprintCallable, Category = "Weapon")
    void LoadBaseMesh();

    // 蓝图可绑定的事件
    UPROPERTY(BlueprintAssignable, Category = "Weapon")
    FOnMeshLoaded OnBaseMeshLoaded;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnMeshLoaded);
```

### 实践练习

1. 创建一个包含多个软引用的武器系统，实现按需加载
2. 实现软引用的异步加载进度UI
3. 比较硬引用和软引用的内存占用差异

### 扩展阅读

- `SoftObjectPath.h` - 软引用路径实现
- `SoftObjectPtr.h` - 软引用指针实现
- `AssetManager.h` - 资产管理器API

---

## 第4课：资产注册与资产管理器

### 学习目标

- 理解资产注册表的作用与机制
- 掌握资产管理器的使用
- 学会配置Primary Assets

### 课程内容

#### 4.1 资产注册表

```cpp
// 资产注册表用于追踪和搜索项目中的所有资产
// 位于 AssetRegistryModule.h

void SearchAssets()
{
    FAssetRegistryModule& AssetRegistryModule =
        FModuleManager::LoadModuleChecked<FAssetRegistryModule>(
            FName("AssetRegistry")
        );

    IAssetRegistry& AssetRegistry = AssetRegistryModule.Get();

    // 方法1：按路径搜索
    TArray<FAssetData> AssetDataList;
    AssetRegistry.GetAssetsByPath(FName("/Game/Meshes"), AssetDataList);

    // 方法2：按类搜索
    AssetRegistry.GetAssetsByClass(
        UStaticMesh::StaticClass()->GetClassPathName(),
        AssetDataList
    );

    // 方法3：按标签搜索
    TArray<FName> Tags;
    Tags.Add(FName("Weapon"));
    AssetRegistry.GetAssetsByTagValues(Tags, TMap<FName, FString>(), AssetDataList);

    // 遍历结果
    for (const FAssetData& AssetData : AssetDataList)
    {
        UE_LOG(LogTemp, Log, TEXT("Found asset: %s"),
            *AssetData.GetFullName());

        // 获取资产路径
        FSoftObjectPath AssetPath = AssetData.ToSoftObjectPath();

        // 获取标签值
        FString TagValue;
        if (AssetData.GetTagValue(FName("WeaponType"), TagValue))
        {
            UE_LOG(LogTemp, Log, TEXT("  WeaponType: %s"), *TagValue);
        }
    }
}
```

#### 4.2 FAssetData 结构

```cpp
// FAssetData 包含资产的元数据，无需加载资产即可获取
struct FAssetData
{
    // 资产路径
    FName ObjectPath;

    // 资产名称
    FName AssetName;

    // 所在包路径
    FName PackagePath;

    // 资产类
    FName AssetClassPath;

    // 标签-值映射
    TMap<FName, FString> TagsAndValues;

    // 核心方法
    FString GetFullName() const;
    FSoftObjectPath ToSoftObjectPath() const;
    bool GetTagValue(FName Tag, FString& OutValue) const;
    UObject* GetAsset() const;  // 会触发加载
};
```

#### 4.3 资产管理器架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    资产管理器架构                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    UAssetManager                                │
│                         │                                       │
│           ┌─────────────┼─────────────┐                        │
│           │             │             │                         │
│           ▼             ▼             ▼                          │
│   ┌───────────────┐ ┌─────────────┐ ┌──────────────────┐       │
│   │Streamable     │ │Asset        │ │Primary Asset     │       │
│   │Manager        │ │Registry     │ │Manager           │       │
│   │               │ │             │ │                  │       │
│   │- 异步加载     │ │- 资产搜索   │ │- 主资产管理     │       │
│   │- 加载队列     │ │- 元数据查询 │ │- Bundle管理     │       │
│   │- 引用追踪     │ │- 依赖关系   │ │- 加载状态       │       │
│   └───────────────┘ └─────────────┘ └──────────────────┘       │
│                                                                 │
│   Primary Assets（主资产）                                      │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  - 由游戏逻辑直接管理                                    │  │
│   │  - 有明确的加载/卸载时机                                 │  │
│   │  - 例如：关卡、角色、武器配置                            │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   Secondary Assets（次级资产）                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  - 被主资产间接引用                                      │  │
│   │  - 随主资产自动加载/卸载                                 │  │
│   │  - 例如：材质、纹理、音效                                │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 4.4 配置Primary Assets

```cpp
// DefaultGame.ini
[/Script/Engine.AssetManagerSettings]
; Primary Asset Types to Scan
+PrimaryAssetTypesToScan=(PrimaryAssetType="Map",AssetBaseClass="/Script/Engine.World",bHasBlueprintClasses=False,bIsEditorOnly=False,Directories=(Path="/Game/Maps"))
+PrimaryAssetTypesToScan=(PrimaryAssetType="Weapon",AssetBaseClass="/Script/MyGame.WeaponData",bHasBlueprintClasses=False,bIsEditorOnly=False,Directories=(Path="/Game/Data/Weapons"))

; Primary Asset to Load at Startup
+PrimaryAssetsToLoadAtStartup=PrimaryAssetId("Map","EntryMap")

; Directories to Exclude
+DirectoriesToExclude=(Path="/Game/Developers")
```

#### 4.5 自定义Primary Asset

```cpp
// 自定义主资产类
UCLASS(BlueprintType)
class UWeaponData : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // 武器ID
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Weapon")
    FName WeaponId;

    // 武器名称
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Weapon")
    FText DisplayName;

    // 武器网格（软引用）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Weapon")
    TSoftObjectPtr<UStaticMesh> WeaponMesh;

    // 武器图标
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Weapon")
    TSoftObjectPtr<UTexture2D> Icon;

    // 伤害值
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Weapon")
    float Damage = 10.0f;

    // 获取主资产ID
    virtual FPrimaryAssetId GetPrimaryAssetId() const override
    {
        return FPrimaryAssetId("Weapon", GetFName());
    }
};

// 使用主资产
void LoadWeaponData(FName WeaponId)
{
    UAssetManager& AssetManager = UAssetManager::Get();

    FPrimaryAssetId WeaponAssetId("Weapon", WeaponId);

    // 加载主资产
    TArray<FName> BundlesToLoad;
    BundlesToLoad.Add(FName("Game"));

    AssetManager.LoadPrimaryAsset(
        WeaponAssetId,
        BundlesToLoad,
        FStreamableDelegate::CreateLambda([WeaponAssetId]()
        {
            UAssetManager& AM = UAssetManager::Get();
            if (UWeaponData* WeaponData = Cast<UWeaponData>(
                AM.GetPrimaryAssetObject(WeaponAssetId)))
            {
                UE_LOG(LogTemp, Log, TEXT("Loaded weapon: %s"),
                    *WeaponData->DisplayName.ToString());
            }
        })
    );
}
```

#### 4.6 Bundle系统

```cpp
// Bundle用于分组加载资产的依赖项
// 常见Bundle类型：
// - "Game"    游戏运行时需要
// - "Menu"    主菜单需要
// - "Preview" 编辑器预览需要

// 配置Bundle规则
// DefaultGame.ini
[/Script/Engine.AssetManagerSettings]
+PrimaryAssetTypesToScan=(PrimaryAssetType="Character",AssetBaseClass="/Script/MyGame.CharacterData",bHasBlueprintClasses=False,bIsEditorOnly=False,Directories=(Path="/Game/Data/Characters"))

// 在资产中配置Bundle依赖
UCLASS(BlueprintType)
class UCharacterData : public UPrimaryDataAsset
{
    GENERATED_BODY()

    // 这个网格只在游戏中加载
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Character", meta=(AssetBundles="Game"))
    TSoftObjectPtr<USkeletalMesh> GameMesh;

    // 这个网格只在菜单中加载（可能是高精度版本）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Character", meta=(AssetBundles="Menu"))
    TSoftObjectPtr<USkeletalMesh> MenuMesh;

    // UI图标在两个场景都需要
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Character", meta=(AssetBundles="Game,Menu"))
    TSoftObjectPtr<UTexture2D> CharacterIcon;
};
```

#### 4.7 资产状态查询

```cpp
// 查询资产加载状态
void CheckAssetState()
{
    UAssetManager& AssetManager = UAssetManager::Get();

    FPrimaryAssetId AssetId("Weapon", FName("Sword"));

    // 检查是否已加载
    bool bIsLoaded = AssetManager.GetPrimaryAssetObject(AssetId) != nullptr;

    // 检查是否正在加载
    bool bIsLoading = AssetManager.GetPrimaryAssetLoadState(AssetId) ==
        EPrimaryAssetLoadState::Loading;

    // 获取资产引用
    TArray<FName> ActiveBundles;
    AssetManager.GetPrimaryAssetLoadState(AssetId, &ActiveBundles);

    UE_LOG(LogTemp, Log, TEXT("Asset %s - Loaded: %d, Active Bundles: %d"),
        *AssetId.ToString(), bIsLoaded, ActiveBundles.Num());
}
```

### 实践练习

1. 创建自定义Primary Asset类（如武器数据、角色数据）
2. 配置Asset Manager Settings
3. 实现基于Bundle的条件加载

### 扩展阅读

- `AssetManager.h` - 资产管理器API
- `PrimaryDataAsset.h` - 主资产基类
- `AssetRegistry.h` - 资产注册表API

---

## 第5课：垃圾回收与资产释放

### 学习目标

- 深入理解GC与资产释放的关系
- 掌握资产引用管理技巧
- 学会检测和处理内存问题

### 课程内容

#### 5.1 资产与GC的关系

```
┌─────────────────────────────────────────────────────────────────┐
│                    资产生命周期                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   1. 创建/加载                                                   │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  NewObject() / LoadObject() / AsyncLoad()              │  │
│   │      │                                                  │  │
│   │      ▼                                                  │  │
│   │  对象分配内存，加入全局对象池                            │  │
│   └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│   2. 使用中                                                      │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  被引用 = 可达 → GC不会回收                             │  │
│   │                                                         │  │
│   │  引用方式：                                              │  │
│   │  - UPROPERTY 硬引用                                     │  │
│   │  - AddReferencedObjects                                 │  │
│   │  - Root Set                                             │  │
│   │  - TStrongObjectPtr                                     │  │
│   └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│   3. 解除引用                                                    │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  置空指针 / 重置软引用 / 离开作用域                      │  │
│   │      │                                                  │  │
│   │      ▼                                                  │  │
│   │  无引用 = 不可达 → 等待GC回收                           │  │
│   └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│   4. GC回收                                                      │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  GC标记阶段 → 发现不可达对象                            │  │
│   │      │                                                  │  │
│   │      ▼                                                  │  │
│   │  GC清理阶段 → 调用析构，释放内存                        │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 5.2 资产引用检查

```cpp
// 检查资产被什么引用
void DebugAssetReferences(UObject* Asset)
{
    if (!Asset) return;

    // 使用 obj refs 命令
    FString Command = FString::Printf(TEXT("obj refs name=%s"), *Asset->GetName());
    GEngine->Exec(NULL, *Command);

    // 代码方式获取引用
    TArray<UObject*> Referencers;
    FReferenceFinder Finder(nullptr, nullptr, false, true, false, false);
    Finder.FindReferences(Referencers, Asset);

    UE_LOG(LogTemp, Log, TEXT("=== Referencers of %s ==="), *Asset->GetName());
    for (UObject* Ref : Referencers)
    {
        UE_LOG(LogTemp, Log, TEXT("  - %s"), *Ref->GetFullName());
    }
}
```

#### 5.3 正确释放资产

```cpp
// 示例：武器系统的资产释放
UCLASS()
class UWeaponSystem : public UObject
{
    GENERATED_BODY()

    // 当前武器（硬引用）
    UPROPERTY()
    UWeaponData* CurrentWeapon;

    // 已加载的武器缓存
    UPROPERTY()
    TMap<FName, UWeaponData*> LoadedWeapons;

    // 异步加载句柄
    TSharedPtr<FStreamableHandle> LoadHandle;

public:
    // 切换武器
    void SwitchWeapon(FName WeaponId)
    {
        // 1. 释放当前武器
        if (CurrentWeapon)
        {
            UnloadWeapon(CurrentWeapon);
        }

        // 2. 加载新武器
        LoadWeapon(WeaponId);
    }

private:
    void UnloadWeapon(UWeaponData* Weapon)
    {
        if (!Weapon) return;

        // 取消正在进行的加载
        if (LoadHandle.IsValid())
        {
            LoadHandle->CancelHandle();
            LoadHandle.Reset();
        }

        // 从缓存移除
        FName WeaponId = Weapon->GetWeaponId();
        LoadedWeapons.Remove(WeaponId);

        // 清除当前引用
        if (CurrentWeapon == Weapon)
        {
            CurrentWeapon = nullptr;
        }

        // 资产现在可以被GC回收了
    }

    void LoadWeapon(FName WeaponId)
    {
        // 检查是否已加载
        if (UWeaponData** Found = LoadedWeapons.Find(WeaponId))
        {
            CurrentWeapon = *Found;
            return;
        }

        // 异步加载
        FPrimaryAssetId AssetId("Weapon", WeaponId);

        LoadHandle = UAssetManager::Get().LoadPrimaryAsset(
            AssetId,
            TArray<FName>(),
            FStreamableDelegate::CreateUObject(this, &UWeaponSystem::OnWeaponLoaded, WeaponId)
        );
    }

    void OnWeaponLoaded(FName WeaponId)
    {
        FPrimaryAssetId AssetId("Weapon", WeaponId);
        CurrentWeapon = Cast<UWeaponData>(
            UAssetManager::Get().GetPrimaryAssetObject(AssetId)
        );

        if (CurrentWeapon)
        {
            LoadedWeapons.Add(WeaponId, CurrentWeapon);
        }

        LoadHandle.Reset();
    }
};
```

#### 5.4 强制GC与内存清理

```cpp
// 强制执行垃圾回收
void ForceGarbageCollection()
{
    // 方式1：请求GC（下一帧执行）
    GEngine->ForceGarbageCollection(false);

    // 方式2：强制完整GC（立即执行）
    GEngine->ForceGarbageCollection(true);

    // 方式3：使用控制台命令
    GEngine->Exec(nullptr, TEXT("obj gc"));
}

// 强制卸载未使用的资产
void FlushUnusedAssets()
{
    UAssetManager& AssetManager = UAssetManager::Get();

    // 卸载所有非主资产
    TArray<FPrimaryAssetId> LoadedAssets;
    AssetManager.GetPrimaryAssetIdList(FName(), LoadedAssets);

    for (const FPrimaryAssetId& AssetId : LoadedAssets)
    {
        AssetManager.UnloadPrimaryAsset(AssetId);
    }

    // 执行GC
    GEngine->ForceGarbageCollection(true);
}

// 清理资产管理器缓存
void ClearAssetManagerCache()
{
    UAssetManager& AssetManager = UAssetManager::Get();

    // 获取引用计数为0的资产
    TArray<UObject*> UnreferencedAssets;
    for (TObjectIterator<UObject> It; It; ++It)
    {
        UObject* Obj = *It;
        if (!Obj->IsRooted() && Obj->GetOutermost() != GetTransientPackage())
        {
            UnreferencedAssets.Add(Obj);
        }
    }

    UE_LOG(LogTemp, Log, TEXT("Found %d unreferenced assets"),
        UnreferencedAssets.Num());

    // 清除StreamableManager的缓存
    AssetManager.GetStreamableManager().ReleaseAllResolves();
}
```

#### 5.5 内存监控与调试

```cpp
// 内存监控组件
UCLASS()
class UMemoryMonitorComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    virtual void TickComponent(float DeltaTime, ... ) override
    {
        if (bEnabled)
        {
            UpdateMemoryStats();
        }
    }

    UFUNCTION(BlueprintCallable, Category = "Memory")
    void DumpMemoryStats()
    {
        // 总内存使用
        FPlatformMemoryStats MemStats = FPlatformMemory::GetStats();
        UE_LOG(LogTemp, Log, TEXT("=== Memory Stats ==="));
        UE_LOG(LogTemp, Log, TEXT("  Total Physical: %.2f MB"),
            MemStats.TotalPhysical / 1024.0 / 1024.0);
        UE_LOG(LogTemp, Log, TEXT("  Used Physical: %.2f MB"),
            MemStats.UsedPhysical / 1024.0 / 1024.0);
        UE_LOG(LogTemp, Log, TEXT("  Available Physical: %.2f MB"),
            MemStats.AvailablePhysical / 1024.0 / 1024.0);

        // UObject统计
        int32 TotalObjects = GUObjectArray.GetObjectCount();
        UE_LOG(LogTemp, Log, TEXT("  Total UObjects: %d"), TotalObjects);

        // 资产管理器统计
        UAssetManager& AssetManager = UAssetManager::Get();
        TArray<FPrimaryAssetId> LoadedAssets;
        AssetManager.GetPrimaryAssetIdList(FName(), LoadedAssets);
        UE_LOG(LogTemp, Log, TEXT("  Loaded Primary Assets: %d"),
            LoadedAssets.Num());
    }

    UFUNCTION(BlueprintCallable, Category = "Memory")
    void DumpAssetMemoryUsage()
    {
        // 按类型统计资产内存
        TMap<UClass*, SIZE_T> ClassMemoryMap;

        for (TObjectIterator<UObject> It; It; ++It)
        {
            UObject* Obj = *It;
            UClass* Class = Obj->GetClass();
            SIZE_T ObjSize = Obj->GetResourceSizeBytes(EResourceSizeMode::Exclusive);

            SIZE_T& TotalSize = ClassMemoryMap.FindOrAdd(Class);
            TotalSize += ObjSize;
        }

        // 排序并输出
        ClassMemoryMap.ValueSort([](SIZE_T A, SIZE_T B) { return A > B; });

        UE_LOG(LogTemp, Log, TEXT("=== Top Memory Consumers ==="));
        int32 Count = 0;
        for (auto& Pair : ClassMemoryMap)
        {
            if (Count++ >= 10) break;
            UE_LOG(LogTemp, Log, TEXT("  %s: %.2f MB"),
                *Pair.Key->GetName(),
                Pair.Value / 1024.0 / 1024.0);
        }
    }

private:
    void UpdateMemoryStats()
    {
        FPlatformMemoryStats MemStats = FPlatformMemory::GetStats();
        CurrentMemoryMB = MemStats.UsedPhysical / 1024.0 / 1024.0;

        // 检查内存阈值
        if (CurrentMemoryMB > WarningThresholdMB)
        {
            OnMemoryWarning.Broadcast(CurrentMemoryMB);
        }
    }

    UPROPERTY(EditDefaultsOnly, Category = "Memory")
    float WarningThresholdMB = 1024.0f;

    UPROPERTY(BlueprintReadOnly, Category = "Memory")
    float CurrentMemoryMB = 0.0f;

    UPROPERTY(BlueprintAssignable, Category = "Memory")
    FOnMemoryWarning OnMemoryWarning;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnMemoryWarning, float, MemoryMB);
```

#### 5.6 常见内存问题与解决

```cpp
// 问题1：资产泄漏（忘记释放引用）
// 解决方案：使用智能指针或自动管理

// 问题代码
UWeaponData* Weapon;  // 裸指针，容易忘记置空

// 正确代码
UPROPERTY()
UWeaponData* Weapon;  // UPROPERTY自动管理引用

// 或者使用TStrongObjectPtr
TStrongObjectPtr<UWeaponData> Weapon;

// 问题2：循环引用
// 解决方案：使用软引用打破循环

// 问题代码
UPROPERTY()
UItemData* Item;
UPROPERTY()
UCharacterData* Owner;  // 循环引用

// 正确代码
UPROPERTY()
UItemData* Item;
UPROPERTY()
TSoftObjectPtr<UCharacterData> Owner;  // 软引用

// 问题3：资产重复加载
// 解决方案：使用资产管理器缓存

// 问题代码
UStaticMesh* Mesh = LoadObject<UStaticMesh>(...);  // 可能重复加载

// 正确代码
FPrimaryAssetId MeshId("StaticMesh", MeshName);
UStaticMesh* Mesh = Cast<UStaticMesh>(
    UAssetManager::Get().GetPrimaryAssetObject(MeshId)
);
```

### 实践练习

1. 实现一个内存监控HUD，显示实时内存使用
2. 使用 `obj list` 和 `obj refs` 分析资产引用
3. 实现资产按需加载和卸载系统

### 扩展阅读

- `GarbageCollection.cpp` - GC实现源码
- `ReferenceChainSearch.h` - 引用链搜索
- `MemoryProfiler.h` - 内存分析工具

---

## 第6课：流式加载与高级主题

### 学习目标

- 掌握关卡流式加载机制
- 理解World Partition系统
- 了解资产热更新和打包策略

### 课程内容

#### 6.1 关卡流式加载

```
┌─────────────────────────────────────────────────────────────────┐
│                    关卡流式加载架构                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   持久关卡 (Persistent Level)                                   │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                                                         │  │
│   │   始终加载，包含全局逻辑和资产                           │  │
│   │                                                         │  │
│   │   ┌─────────┐ ┌─────────┐ ┌─────────┐                  │  │
│   │   │Player   │ │GameMode │ │Global   │                  │  │
│   │   │Start    │ │         │ │Assets   │                  │  │
│   │   └─────────┘ └─────────┘ └─────────┘                  │  │
│   │                                                         │  │
│   └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│           ┌───────────────┼───────────────┐                    │
│           │               │               │                     │
│           ▼               ▼               ▼                     │
│   ┌───────────────┐ ┌───────────────┐ ┌───────────────┐       │
│   │ 子关卡A       │ │ 子关卡B       │ │ 子关卡C       │       │
│   │ (Streaming)   │ │ (Streaming)   │ │ (Streaming)   │       │
│   │               │ │               │ │               │       │
│   │ - 按需加载    │ │ - 按需加载    │ │ - 按需加载    │       │
│   │ - 可卸载     │ │ - 可卸载     │ │ - 可卸载     │       │
│   └───────────────┘ └───────────────┘ └───────────────┘       │
│           │               │               │                     │
│           ▼               ▼               ▼                     │
│      加载区域         未加载区域        加载区域                 │
│    (玩家在范围内)    (玩家远离)       (玩家在范围内)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 6.2 流式关卡代码实现

```cpp
// 流式关卡管理器
UCLASS()
class ALevelStreamingManager : public AActor
{
    GENERATED_BODY()

public:
    // 流式关卡配置
    USTRUCT(BlueprintType)
    struct FStreamingLevelConfig
    {
        GENERATED_BODY()

        UPROPERTY(EditAnywhere, BlueprintReadWrite)
        FName LevelName;

        UPROPERTY(EditAnywhere, BlueprintReadWrite)
        FVector LevelOffset;

        UPROPERTY(EditAnywhere, BlueprintReadWrite)
        float LoadRadius = 5000.0f;

        UPROPERTY(EditAnywhere, BlueprintReadWrite)
        bool bShouldBeLoaded = false;

        UPROPERTY(EditAnywhere, BlueprintReadWrite)
        bool bShouldBeVisible = true;
    };

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Streaming")
    TArray<FStreamingLevelConfig> StreamingLevels;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Streaming")
    bool bAutoLoadBasedOnDistance = true;

    virtual void Tick(float DeltaTime) override
    {
        if (bAutoLoadBasedOnDistance)
        {
            UpdateStreamingBasedOnPlayerLocation();
        }
    }

    UFUNCTION(BlueprintCallable, Category = "Streaming")
    void LoadStreamingLevel(FName LevelName, bool bMakeVisible = true)
    {
        UWorld* World = GetWorld();
        if (!World) return;

        // 查找或创建流式关卡
        ULevelStreaming* StreamingLevel = World->GetStreamingLevels().FindRef(LevelName);

        if (!StreamingLevel)
        {
            // 创建新的流式关卡
            StreamingLevel = UGameplayStatics::GetStreamingLevel(World, LevelName);
        }

        if (StreamingLevel)
        {
            // 设置加载状态
            StreamingLevel->SetShouldBeLoaded(true);
            StreamingLevel->SetShouldBeVisible(bMakeVisible);

            // 绑定完成回调
            StreamingLevel->OnLevelLoaded.AddDynamic(
                this, &ALevelStreamingManager::OnLevelLoaded);
        }
    }

    UFUNCTION(BlueprintCallable, Category = "Streaming")
    void UnloadStreamingLevel(FName LevelName)
    {
        UWorld* World = GetWorld();
        if (!World) return;

        if (ULevelStreaming* StreamingLevel =
            World->GetStreamingLevels().FindRef(LevelName))
        {
            StreamingLevel->SetShouldBeLoaded(false);
            StreamingLevel->SetShouldBeVisible(false);
        }
    }

private:
    void UpdateStreamingBasedOnPlayerLocation()
    {
        APlayerController* PC = GetWorld()->GetFirstPlayerController();
        if (!PC) return;

        FVector PlayerLocation = PC->GetPawn()->GetActorLocation();

        for (const FStreamingLevelConfig& Config : StreamingLevels)
        {
            float Distance = FVector::Dist2D(PlayerLocation, Config.LevelOffset);
            bool bShouldLoad = Distance <= Config.LoadRadius;

            if (bShouldLoad != Config.bShouldBeLoaded)
            {
                if (bShouldLoad)
                {
                    LoadStreamingLevel(Config.LevelName, Config.bShouldBeVisible);
                }
                else
                {
                    UnloadStreamingLevel(Config.LevelName);
                }
            }
        }
    }

    UFUNCTION()
    void OnLevelLoaded()
    {
        UE_LOG(LogTemp, Log, TEXT("Level loaded"));
        OnStreamingLevelLoaded.Broadcast();
    }

    UPROPERTY(BlueprintAssignable, Category = "Streaming")
    FOnLevelLoaded OnStreamingLevelLoaded;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnLevelLoaded);
```

#### 6.3 World Partition 系统

```cpp
// World Partition 是UE5的新一代流式加载系统
// 自动将大世界划分为可管理的区块

/*
配置方式（项目设置）:
- 启用 World Partition
- 设置 Cell Size（区块大小）
- 配置 Data Layers（数据层）

编辑器操作：
- World Partition > Enable World Partition
- World Partition > Generate Streaming
*/

// 运行时控制加载区域
UCLASS()
class AWorldPartitionLoader : public AActor
{
    GENERATED_BODY()

public:
    // 加载半径
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Loading")
    float LoadRadius = 10000.0f;

    // 卸载半径
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Loading")
    float UnloadRadius = 15000.0f;

    // 加载区域
    UFUNCTION(BlueprintCallable, Category = "Loading")
    void LoadAreaAroundPosition(FVector Position)
    {
        UWorld* World = GetWorld();
        if (!World) return;

        // World Partition API（需要启用World Partition）
        // 注意：这是概念性代码，实际API可能有所不同

        // 请求加载区域
        FBox LoadingBox(
            Position - FVector(LoadRadius),
            Position + FVector(LoadRadius)
        );

        // UE5 World Partition 会自动处理加载
        // 也可以手动调用加载器

        UE_LOG(LogTemp, Log, TEXT("Requesting load area: %s"),
            *LoadingBox.ToString());
    }

    // 获取已加载的区块信息
    UFUNCTION(BlueprintCallable, Category = "Loading")
    void GetLoadedCells(TArray<FVector>& LoadedCellPositions)
    {
        // 返回已加载区块的位置
        // 用于调试和可视化
    }
};
```

#### 6.4 Data Layers（数据层）

```cpp
// 数据层用于条件性加载内容
// 例如：根据游戏进度加载不同区域

UCLASS()
class ADataLayerManager : public AActor
{
    GENERATED_BODY()

public:
    // 激活数据层
    UFUNCTION(BlueprintCallable, Category = "DataLayers")
    void SetDataLayerState(FName DataLayerName, bool bActive)
    {
        UWorld* World = GetWorld();
        if (!World) return;

        // 获取数据层子系统
        UDataLayerSubsystem* DataLayerSubsystem =
            World->GetSubsystem<UDataLayerSubsystem>();

        if (DataLayerSubsystem)
        {
            // 设置数据层状态
            if (bActive)
            {
                DataLayerSubsystem->SetDataLayerState(
                    DataLayerName,
                    EDataLayerState::Loaded
                );
            }
            else
            {
                DataLayerSubsystem->SetDataLayerState(
                    DataLayerName,
                    EDataLayerState::Unloaded
                );
            }
        }
    }

    // 根据游戏进度加载内容
    UFUNCTION(BlueprintCallable, Category = "DataLayers")
    void LoadContentForProgression(int32 ProgressionLevel)
    {
        switch (ProgressionLevel)
        {
            case 1:
                SetDataLayerState(FName("Chapter1"), true);
                SetDataLayerState(FName("Chapter2"), false);
                break;
            case 2:
                SetDataLayerState(FName("Chapter1"), false);
                SetDataLayerState(FName("Chapter2"), true);
                break;
            // ...
        }
    }
};
```

#### 6.5 资产打包与热更新

```cpp
// 资产打包配置
// DefaultGame.ini

/*
[/Script/UnrealEd.ProjectPackagingSettings]
; 打包目录
+DirectoriesToAlwaysCook=(Path="/Game/Maps")
+DirectoriesToAlwaysCook=(Path="/Game/Blueprints")

; 基于Primary Asset的打包
+PrimaryAssetGroups=(...AssetGroup)

;chunk分配（用于DLC）
bSplitBaseGamePak=True
+ChunkList=(ChunkId=1, AssetPath="/Game/DLC1")
*/

// 运行时检查资产是否已打包
UCLASS()
class UAssetPackagingHelper : public UObject
{
    GENERATED_BODY()

public:
    // 检查资产是否可用
    UFUNCTION(BlueprintCallable, Category = "Packaging")
    static bool IsAssetAvailable(const FSoftObjectPath& AssetPath)
    {
        // 检查资产是否在已安装的pak中
        FString PackageName = AssetPath.GetLongPackageName();

        // 使用资产注册表检查
        FAssetRegistryModule& AssetRegistryModule =
            FModuleManager::LoadModuleChecked<FAssetRegistryModule>(
                FName("AssetRegistry")
            );

        FAssetData AssetData;
        if (AssetRegistryModule.Get().GetAssetByObjectPath(
            AssetPath, AssetData))
        {
            return true;
        }

        return false;
    }

    // 获取资产所在的Chunk ID
    UFUNCTION(BlueprintCallable, Category = "Packaging")
    static int32 GetAssetChunkId(const FSoftObjectPath& AssetPath)
    {
        // 返回资产分配的Chunk ID
        // 用于DLC下载判断
        return 0;
    }
};
```

#### 6.6 性能优化总结

```
┌─────────────────────────────────────────────────────────────────┐
│                    资产管理优化清单                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   加载优化                                                       │
│   ──────────────────────────────────────────────────────────── │
│   ✓ 使用软引用减少初始加载量                                    │
│   ✓ 大型资产使用异步加载                                        │
│   ✓ 预加载即将需要的资产（后台加载）                            │
│   ✓ 设置合理的加载优先级                                        │
│                                                                 │
│   内存优化                                                       │
│   ──────────────────────────────────────────────────────────── │
│   ✓ 及时释放不再使用的资产                                      │
│   ✓ 使用对象池避免频繁创建销毁                                  │
│   ✓ 合理配置纹理/网格的LOD                                      │
│   ✓ 监控内存使用，设置警告阈值                                  │
│                                                                 │
│   流式加载优化                                                   │
│   ──────────────────────────────────────────────────────────── │
│   ✓ 设置合适的加载/卸载半径                                     │
│   ✓ 使用World Partition自动管理                                 │
│   ✓ 利用Data Layers分层加载                                    │
│   ✓ 预测玩家移动方向提前加载                                    │
│                                                                 │
│   打包优化                                                       │
│   ──────────────────────────────────────────────────────────── │
│   ✓ 合理划分Chunk用于DLC                                        │
│   ✓ 配置正确的打包规则                                          │
│   ✓ 使用压缩减少包体大小                                        │
│   ✓ 按需下载DLC内容                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 6.7 调试命令汇总

```cpp
// 常用控制台命令

// 资产相关
// -----------------------------------------------------------
// obj list                   列出所有对象
// obj list class=StaticMesh  列出特定类型的对象
// obj refs name=MeshName     查看对象引用
// obj gc                     强制GC
// obj dump name=AssetName    输出对象详细信息

// 内存相关
// -----------------------------------------------------------
// stat memory               显示内存统计
// stat memoryplatform       平台内存详情
// memreport                 生成内存报告
// dmp                       内存分析

// 流式加载相关
// -----------------------------------------------------------
// stat streaming            流式加载统计
// streaming pause           暂停流式加载
// streaming list            列出流式关卡

// 资产管理器相关
// -----------------------------------------------------------
// assetmanager dump         输出资产管理器状态
// assetmanager loaded       列出已加载的主资产
```

### 实践练习

1. 创建一个基于距离的关卡流式加载系统
2. 配置World Partition并测试大世界加载
3. 实现Data Layers的条件加载

### 扩展阅读

- `LevelStreaming.h` - 关卡流式加载
- `WorldPartition.h` - World Partition系统
- `DataLayerSubsystem.h` - 数据层子系统
- `PakFileUtilities.h` - Pak文件工具

---

## 课程总结

### 知识体系总览

```
┌─────────────────────────────────────────────────────────────────┐
│                    UE5 资源管理知识体系                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   第1课          第2课          第3课                           │
│   基础概念       加载机制       延迟加载                         │
│   ┌─────┐       ┌─────┐       ┌─────┐                          │
│   │ 引用 │──────►│同步 │──────►│软引用│                          │
│   │ 类型 │       │异步 │       │按需 │                          │
│   └─────┘       └─────┘       └─────┘                          │
│       │             │             │                              │
│       └─────────────┼─────────────┘                              │
│                     │                                            │
│                     ▼                                            │
│   第4课          第5课          第6课                           │
│   资产管理       内存管理       高级应用                         │
│   ┌─────┐       ┌─────┐       ┌─────┐                          │
│   │Asset│──────►│ GC  │──────►│流式 │                          │
│   │Mgr  │       │释放 │       │热更 │                          │
│   └─────┘       └─────┘       └─────┘                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 核心要点回顾

| 课时 | 核心概念 | 关键技能 |
|------|---------|---------|
| 第1课 | 硬引用/软引用 | 正确选择引用方式 |
| 第2课 | 同步/异步加载 | 根据场景选择加载策略 |
| 第3课 | 延迟加载 | TSoftObjectPtr使用 |
| 第4课 | Asset Manager | Primary Asset配置与使用 |
| 第5课 | GC与释放 | 引用管理与内存监控 |
| 第6课 | 流式加载 | Level Streaming、World Partition |

### 学习路径建议

```
入门 → 进阶 → 高级 → 专家
 │       │       │       │
 ▼       ▼       ▼       ▼
第1课   第2课   第4课   第6课
       第3课   第5课
```

### 推荐学习资源

1. **官方文档**
   - Asset Management
   - Garbage Collection
   - Level Streaming
   - World Partition

2. **源码阅读**
   - `AssetManager.cpp`
   - `GarbageCollection.cpp`
   - `LevelStreaming.cpp`

3. **实践项目**
   - 创建资产管理系统
   - 实现延迟加载武器系统
   - 构建流式加载大世界
