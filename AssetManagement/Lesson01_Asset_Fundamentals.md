# 第1课：资产基础与引用机制

本课介绍UE5资产系统的基础概念，重点讲解硬引用与软引用的区别及其应用场景。

---

## 1.1 资产概述

### 什么是资产（Asset）

在UE5中，资产是存储在磁盘上的资源文件，在运行时加载到内存中使用。

```
┌─────────────────────────────────────────────────────────────────┐
│                      UE5 资产体系                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   资产文件格式                                                   │
│   ─────────────                                                 │
│   ├── .uasset    所有资产的基础容器（二进制格式）                │
│   ├── .umap      关卡资产（特殊的uasset）                       │
│   └── .umeta     元数据文件（仅编辑器使用）                      │
│                                                                 │
│   资产路径格式                                                   │
│   ─────────────                                                 │
│   /Game/Path/AssetName.AssetName                                │
│   │    │    │          │                                        │
│   │    │    │          └── 资产名称（含子对象时为 AssetName:SubObj）│
│   │    │    └── 资产文件名                                      │
│   │    └── 相对路径                                              │
│   └── 挂载点（Game=项目Content目录）                             │
│                                                                 │
│   常见挂载点                                                     │
│   ─────────────                                                 │
│   /Game/        项目的Content目录                                │
│   /Engine/      引擎的Content目录                                │
│   /PluginName/  插件的Content目录                                │
│   /Script/      C++类模块                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 常见资产类型

```cpp
// 纹理类
UTexture2D          // 2D纹理
UTextureCube        // 立方体贴图（天空盒）
URenderTarget       // 渲染目标

// 网格类
UStaticMesh         // 静态网格
USkeletalMesh       // 骨骼网格

// 材质类
UMaterial           // 材质
UMaterialInstance   // 材质实例
UMaterialInterface  // 材质接口

// 动画类
UAnimSequence       // 动画序列
UAnimMontage        // 动画蒙太奇
UAnimBlueprint      // 动画蓝图（生成类）

// 音频类
USoundWave          // 音频文件
USoundCue           // 音频节点图

// 粒子类
UNiagaraSystem      // Niagara粒子系统
UParticleSystem     // Cascade粒子系统（旧）

// 数据类
UDataTable          // 数据表格
UCurveTable         // 曲线表格
UBlueprint          // 蓝图（生成类）
```

### 资产在内存中的表示

```cpp
// 所有资产都继承自UObject
UCLASS()
class UObject : public UObjectBase
{
    // 资产名称
    FName Name;

    // 外部对象（通常是UPackage）
    UObject* Outer;

    // 类对象
    UClass* Class;
};

// UPackage代表一个.uasset文件
UCLASS()
class UPackage : public UObject
{
    // 包名称（即资产路径）
    FName PackageName;

    // 包中的主对象
    UObject* FindObjectInPackage();
};
```

---

## 1.2 硬引用（Hard Reference）

### 定义

硬引用是直接持有对象指针的引用方式。被引用的对象会被自动加载，且在引用有效期间不会被GC回收。

### 声明方式

```cpp
// 方式1：UPROPERTY直接引用（最常见）
UCLASS()
class AWeaponActor : public AActor
{
    GENERATED_BODY()

    // EditDefaultsOnly: 只能在蓝图类默认值中设置
    UPROPERTY(EditDefaultsOnly, Category = "Weapon")
    UStaticMesh* WeaponMesh;

    // EditAnywhere: 可以在实例中设置
    UPROPERTY(EditAnywhere, Category = "Weapon")
    UMaterialInterface* WeaponMaterial;

    // BlueprintReadOnly: 蓝图可读不可写
    UPROPERTY(BlueprintReadOnly, Category = "Weapon")
    USoundBase* FireSound;
};
```

### 硬引用特点

```
┌─────────────────────────────────────────────────────────────────┐
│                      硬引用特点                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ✓ 自动加载：持有者加载时，被引用资产一起加载                   │
│   ✓ 阻止GC：只要引用存在，资产不会被垃圾回收                     │
│   ✓ 即时可用：访问时资产已在内存中                               │
│                                                                 │
│   ✗ 增加内存：所有引用资产常驻内存                               │
│   ✗ 增加载入时间：启动时需要加载更多资产                         │
│   ✗ 可能循环依赖：A引用B，B引用A                                │
│                                                                 │
│   适用场景                                                       │
│   ─────────────                                                 │
│   • 高频使用的核心资产（主角Mesh、UI材质）                       │
│   • 必须立即可用的资产（武器网格、关键音效）                     │
│   • 内存占用小的资产                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 硬引用加载链

```
资产加载链示例：

┌─────────────────────┐
│   BP_Character      │  ← 玩家加载关卡
└──────────┬──────────┘
           │ 硬引用
           ▼
┌─────────────────────┐
│   SK_BodyMesh       │  ← 自动加载
└──────────┬──────────┘
           │ 硬引用
           ▼
┌─────────────────────┐
│   M_BodyMaterial    │  ← 自动加载
└──────────┬──────────┘
           │ 硬引用
           ▼
┌─────────────────────┐
│   T_BodyTexture     │  ← 自动加载
└─────────────────────┘

结果：加载角色时，所有关联资产全部加载进内存
```

### 代码中的硬引用

```cpp
// 硬引用会在对象构造时加载
UCLASS()
class AMyActor : public AActor
{
    GENERATED_BODY()

    // 构造函数中设置默认值
    AMyActor()
    {
        // 使用ConstructorHelpers加载资产
        static ConstructorHelpers::FObjectFinder<UStaticMesh> MeshFinder(
            TEXT("/Game/Meshes/SM_Default.SM_Default")
        );

        if (MeshFinder.Succeeded())
        {
            DefaultMesh = MeshFinder.Object;
        }
    }

    UPROPERTY(VisibleAnywhere)
    UStaticMeshComponent* MeshComponent;

    UPROPERTY()
    UStaticMesh* DefaultMesh;
};

// 运行时动态创建硬引用
void SpawnWeapon()
{
    // LoadObject创建硬引用
    UStaticMesh* WeaponMesh = LoadObject<UStaticMesh>(
        nullptr,
        TEXT("/Game/Weapons/SM_Rifle.SM_Rifle")
    );

    // 只要WeaponMesh变量有效，资产就不会被GC
    MeshComponent->SetStaticMesh(WeaponMesh);
}
```

---

## 1.3 软引用（Soft Reference）

### 定义

软引用只存储资产路径，不直接持有对象指针。资产不会自动加载，需要时手动请求加载。

### 声明方式

```cpp
// 方式1：TSoftObjectPtr（最常用）
UCLASS()
class AWeaponActor : public AActor
{
    GENERATED_BODY()

    // TSoftObjectPtr<T> - 对象软引用
    UPROPERTY(EditDefaultsOnly, Category = "Weapon")
    TSoftObjectPtr<UStaticMesh> WeaponMesh;

    // TSoftClassPtr<T> - 类软引用（蓝图类）
    UPROPERTY(EditDefaultsOnly, Category = "Weapon")
    TSoftClassPtr<AActor> ProjectileClass;
};

// 方式2：FSoftObjectPath（路径封装）
UCLASS()
class ADataManager : public AActor
{
    GENERATED_BODY()

    UPROPERTY(EditDefaultsOnly, Category = "Data")
    FSoftObjectPath DataAssetPath;
};

// 方式3：FSoftClassPath（类路径）
UCLASS()
class AActorSpawner : public AActor
{
    GENERATED_BODY()

    UPROPERTY(EditDefaultsOnly, Category = "Spawner")
    FSoftClassPath ActorClassPath;
};
```

### 软引用特点

```
┌─────────────────────────────────────────────────────────────────┐
│                      软引用特点                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ✓ 延迟加载：资产不会自动加载，按需请求                         │
│   ✓ 节省内存：未使用的资产不占用内存                             │
│   ✓ 避免循环依赖：只存路径，不会形成引用环                       │
│   ✓ 蓝图友好：可以在编辑器中选择资产                             │
│                                                                 │
│   ✗ 需要手动加载：使用前必须显式加载                             │
│   ✗ 可能为空：资产可能加载失败                                   │
│   ✗ 略微延迟：首次使用时需要加载时间                             │
│                                                                 │
│   适用场景                                                       │
│   ─────────────                                                 │
│   • 可选资产（多武器、多皮肤）                                   │
│   • 大型资产（过场动画、高清材质）                               │
│   • 远程资产（DLC内容）                                          │
│   • 可能不使用的资产                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### TSoftObjectPtr详解

```cpp
// TSoftObjectPtr 内部结构
template<class T>
class TSoftObjectPtr
{
private:
    // 资产路径
    FSoftObjectPath SoftObjectPath;

    // 缓存的弱指针（加载后可用）
    mutable TWeakObjectPtr<T> CachedPtr;

public:
    // 构造函数
    TSoftObjectPtr() {}
    TSoftObjectPtr(const FSoftObjectPath& InPath);
    TSoftObjectPtr(T* Object);

    // 核心方法
    bool IsPending() const;      // 是否需要加载
    bool IsValid() const;        // 路径是否有效
    bool IsNull() const;         // 是否为空
    T* Get() const;              // 获取对象（已加载时）
    T* LoadSynchronous();        // 同步加载

    FSoftObjectPath ToSoftObjectPath() const;  // 获取路径
    FString ToString() const;

    void Reset();                // 重置
};

// 使用示例
void UseSoftReference()
{
    TSoftObjectPtr<UStaticMesh> SoftMesh;

    // 在编辑器中设置路径
    SoftMesh = FSoftObjectPath("/Game/Meshes/SM_Weapon.SM_Weapon");

    // 检查状态
    if (SoftMesh.IsPending())
    {
        UE_LOG(LogTemp, Log, TEXT("Asset not loaded yet"));

        // 同步加载
        UStaticMesh* Mesh = SoftMesh.LoadSynchronous();
        if (Mesh)
        {
            // 使用资产
            MeshComponent->SetStaticMesh(Mesh);
        }
    }
    else
    {
        // 已加载，直接获取
        UStaticMesh* Mesh = SoftMesh.Get();
    }
}
```

### FSoftObjectPath详解

```cpp
// FSoftObjectPath 内部结构
struct FSoftObjectPath
{
private:
    // 资产路径，如 /Game/Path/Asset.AssetName
    FName AssetPath;

    // 子对象路径，如 SubObject（可选）
    FString SubPathString;

public:
    // 构造
    FSoftObjectPath() {}
    FSoftObjectPath(const FString& PathString);
    FSoftObjectPath(const FName& InAssetPath, const FString& InSubPath);

    // 路径操作
    FString ToString() const;
    FName GetAssetPath() const;
    FString GetAssetName() const;
    FString GetSubPathString() const;

    // 解析对象
    UObject* ResolveObject() const;  // 获取已加载对象

    // 状态检查
    bool IsValid() const;
    bool IsPending() const;

    // 比较操作
    bool operator==(const FSoftObjectPath& Other) const;
    bool operator!=(const FSoftObjectPath& Other) const;

    // 哈希
    friend uint32 GetTypeHash(const FSoftObjectPath& Path);
};

// 使用示例
void UseSoftObjectPath()
{
    // 创建路径
    FSoftObjectPath AssetPath("/Game/Materials/M_Water.M_Water");

    // 获取路径信息
    UE_LOG(LogTemp, Log, TEXT("Full Path: %s"), *AssetPath.ToString());
    UE_LOG(LogTemp, Log, TEXT("Asset Name: %s"), *AssetPath.GetAssetName().ToString());

    // 解析已加载对象
    UObject* LoadedAsset = AssetPath.ResolveObject();
    if (LoadedAsset)
    {
        UE_LOG(LogTemp, Log, TEXT("Asset is loaded: %s"), *LoadedAsset->GetName());
    }
    else
    {
        UE_LOG(LogTemp, Log, TEXT("Asset is not loaded"));
    }
}
```

---

## 1.4 硬引用与软引用对比

### 对比表

| 特性 | 硬引用 | 软引用 |
|------|--------|--------|
| **存储内容** | UObject指针 | 资产路径字符串 |
| **内存占用** | 指针大小 + 对象本身 | 仅路径字符串 |
| **自动加载** | 是 | 否 |
| **GC保护** | 是 | 否 |
| **访问速度** | 立即可用 | 需要加载 |
| **适合场景** | 核心资产 | 可选/大型资产 |

### 选择决策树

```
需要引用资产？
    │
    ├── 否 → 不引用
    │
    └── 是 → 资产是否必须随持有者一起加载？
              │
              ├── 是 → 资产大小是否可控（<1MB）？
              │         │
              │         ├── 是 → 使用硬引用
              │         │
              │         └── 否 → 考虑异步加载 + 硬引用
              │
              └── 否 → 是否可能不使用该资产？
                        │
                        ├── 是 → 使用软引用
                        │
                        └── 否 → 资产是否很大？
                                  │
                                  ├── 是 → 使用软引用 + 按需加载
                                  │
                                  └── 否 → 可用硬引用
```

### 混合使用示例

```cpp
// 实际项目中经常混合使用
UCLASS()
class AWeapon : public AActor
{
    GENERATED_BODY()

    // 硬引用：核心资产，必须立即可用
    UPROPERTY(EditDefaultsOnly, Category = "Weapon")
    UStaticMesh* BaseMesh;  // 武器基础网格

    // 硬引用：高频使用
    UPROPERTY(EditDefaultsOnly, Category = "Weapon")
    USoundBase* FireSound;  // 射击音效

    // 软引用：可选皮肤
    UPROPERTY(EditDefaultsOnly, Category = "Weapon")
    TArray<TSoftObjectPtr<UMaterialInterface>> SkinMaterials;

    // 软引用：大型资产
    UPROPERTY(EditDefaultsOnly, Category = "Weapon")
    TSoftObjectPtr<UAnimMontage> SpecialReloadAnim;  // 特殊装弹动画

    // 软引用：可能不需要
    UPROPERTY(EditDefaultsOnly, Category = "Weapon")
    TSoftObjectPtr<UNiagaraSystem> MuzzleFlash;  // 枪口火焰（可能用材质替代）
};
```

---

## 1.5 引用计数与生命周期

### 引用与GC的关系

```
┌─────────────────────────────────────────────────────────────────┐
│                    引用与GC关系图                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   GC Root Set（根集合）                                          │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  - 全局对象（GameInstance, GameMode等）                  │  │
│   │  - UPROPERTY引用                                         │  │
│   │  - AddReferencedObjects添加的引用                       │  │
│   │  - TStrongObjectPtr持有                                  │  │
│   │  - RootSet标记的对象                                     │  │
│   └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│   可达性分析                                                     │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  从Root Set开始遍历所有引用                             │  │
│   │  标记所有可达对象                                       │  │
│   └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│   清理不可达对象                                                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  未被标记的对象 → 被回收                                │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   硬引用：对象在引用链中 → 可达 → 不被回收                      │
│   软引用：只有路径字符串 → 不在引用链中 → 可被回收              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### UPROPERTY的引用管理

```cpp
// UPROPERTY自动管理引用
UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

    // 这些引用会阻止GC回收
    UPROPERTY(EditDefaultsOnly)
    UStaticMesh* WeaponMesh;  // ← 被GC视为可达

    UPROPERTY(BlueprintReadWrite)
    TArray<AActor*> InventoryItems;  // ← 数组中的所有对象可达

    UPROPERTY()
    TMap<FName, UObject*> AssetCache;  // ← Map中的所有值可达
};

// 非UPROPERTY的指针不会被GC追踪
class AMyCharacter : public ACharacter
{
    // 这个指针不会被GC追踪！
    UObject* UntrackedPointer;  // ← 危险：可能指向已销毁对象
};
```

### AddReferencedObjects

```cpp
// 为非UPROPERTY引用添加GC追踪
UCLASS()
class UMyObject : public UObject
{
    GENERATED_BODY()

    // 非UPROPERTY引用
    TArray<UObject*> ManualReferences;

    // 重写此方法告诉GC额外的引用
    static void AddReferencedObjects(
        UObject* InThis,
        FReferenceCollector& Collector
    )
    {
        UMyObject* This = CastChecked<UMyObject>(InThis);

        // 添加手动引用
        for (UObject* Ref : This->ManualReferences)
        {
            Collector.AddReferencedObject(Ref);
        }

        // 调用父类
        Super::AddReferencedObjects(InThis, Collector);
    }
};
```

### FGCObject

```cpp
// 对于非UObject类，使用FGCObject管理引用
class FMyManager : public FGCObject
{
public:
    // 需要保护的UObject引用
    TArray<UObject*> ManagedObjects;

    // 实现此方法
    virtual void AddReferencedObjects(FReferenceCollector& Collector) override
    {
        Collector.AddReferencedObjects(ManagedObjects);
    }

    // 实现此方法（用于调试）
    virtual FString GetReferencerName() const override
    {
        return TEXT("FMyManager");
    }
};
```

### 查看对象引用

```cpp
// 控制台命令
// obj refs name=ObjectName  - 查看谁引用了这个对象
// obj garbage               - 强制GC

// 代码方式
void DebugObjectReferences(UObject* Object)
{
    if (!Object) return;

    // 获取引用此对象的所有对象
    TArray<UObject*> Referencers;
    FArchiveFindCulprit FindCulprit(Object, Referencers);

    UE_LOG(LogTemp, Log, TEXT("=== References to %s ==="), *Object->GetName());
    for (UObject* Ref : Referencers)
    {
        UE_LOG(LogTemp, Log, TEXT("  Referenced by: %s"), *Ref->GetFullName());
    }
}
```

---

## 1.6 实践练习

### 练习1：创建引用测试Actor

```cpp
// 创建一个测试Actor，包含硬引用和软引用
UCLASS()
class AReferenceTestActor : public AActor
{
    GENERATED_BODY()

public:
    AReferenceTestActor();

    // 硬引用
    UPROPERTY(EditDefaultsOnly, Category = "Hard Reference")
    UStaticMesh* HardRefMesh;

    // 软引用
    UPROPERTY(EditDefaultsOnly, Category = "Soft Reference")
    TSoftObjectPtr<UStaticMesh> SoftRefMesh;

    // 打印状态
    UFUNCTION(CallInEditor, Category = "Debug")
    void PrintReferenceStatus();
};

void AReferenceTestActor::PrintReferenceStatus()
{
    UE_LOG(LogTemp, Log, TEXT("=== Reference Status ==="));

    // 硬引用状态
    if (HardRefMesh)
    {
        UE_LOG(LogTemp, Log, TEXT("HardRef: %s (Loaded)"), *HardRefMesh->GetName());
    }
    else
    {
        UE_LOG(LogTemp, Log, TEXT("HardRef: Not Set"));
    }

    // 软引用状态
    if (SoftRefMesh.IsPending())
    {
        UE_LOG(LogTemp, Log, TEXT("SoftRef: %s (Pending)"), *SoftRefMesh.ToString());
    }
    else if (UStaticMesh* Mesh = SoftRefMesh.Get())
    {
        UE_LOG(LogTemp, Log, TEXT("SoftRef: %s (Loaded)"), *Mesh->GetName());
    }
    else if (SoftRefMesh.IsValid())
    {
        UE_LOG(LogTemp, Log, TEXT("SoftRef: %s (Valid but not loaded)"), *SoftRefMesh.ToString());
    }
    else
    {
        UE_LOG(LogTemp, Log, TEXT("SoftRef: Not Set"));
    }
}
```

### 练习2：内存占用对比

```cpp
// 对比硬引用和软引用的内存差异
void CompareMemoryUsage()
{
    // 记录初始内存
    FPlatformMemoryStats StartStats = FPlatformMemory::GetStats();

    // 加载100个硬引用资产
    TArray<UObject*> HardRefAssets;
    for (int32 i = 0; i < 100; i++)
    {
        FString Path = FString::Printf(TEXT("/Game/Test/Asset_%d.Asset_%d"), i, i);
        UObject* Asset = LoadObject<UObject>(nullptr, *Path);
        HardRefAssets.Add(Asset);
    }

    FPlatformMemoryStats AfterHardRef = FPlatformMemory::GetStats();

    // 清除硬引用，GC
    HardRefAssets.Empty();
    GEngine->ForceGarbageCollection(true);

    // 创建100个软引用
    TArray<FSoftObjectPath> SoftRefAssets;
    for (int32 i = 0; i < 100; i++)
    {
        FString Path = FString::Printf(TEXT("/Game/Test/Asset_%d.Asset_%d"), i, i);
        SoftRefAssets.Add(FSoftObjectPath(Path));
    }

    FPlatformMemoryStats AfterSoftRef = FPlatformMemory::GetStats();

    // 打印结果
    UE_LOG(LogTemp, Log, TEXT("Memory Comparison:"));
    UE_LOG(LogTemp, Log, TEXT("  Start: %.2f MB"), StartStats.UsedPhysical / 1024.0 / 1024.0);
    UE_LOG(LogTemp, Log, TEXT("  After HardRef: %.2f MB (+%.2f MB)"),
        AfterHardRef.UsedPhysical / 1024.0 / 1024.0,
        (AfterHardRef.UsedPhysical - StartStats.UsedPhysical) / 1024.0 / 1024.0);
    UE_LOG(LogTemp, Log, TEXT("  After SoftRef: %.2f MB (+%.2f MB)"),
        AfterSoftRef.UsedPhysical / 1024.0 / 1024.0,
        (AfterSoftRef.UsedPhysical - AfterHardRef.UsedPhysical) / 1024.0 / 1024.0);
}
```

### 练习3：使用控制台命令

```
常用调试命令：

1. 列出所有已加载的静态网格
   obj list class=StaticMesh

2. 查看特定对象的引用
   obj refs name=SM_Weapon

3. 强制垃圾回收
   obj gc

4. 查看内存统计
   stat memory

5. 查看对象数量
   stat objects
```

---

## 1.7 小结

### 核心概念回顾

| 概念 | 说明 |
|------|------|
| **资产** | 存储在磁盘上的.uasset文件 |
| **硬引用** | 直接持有对象指针，自动加载，阻止GC |
| **软引用** | 只存储路径，手动加载，不阻止GC |
| **UPROPERTY** | 自动管理引用，被GC追踪 |

### 最佳实践

1. **核心资产用硬引用**：必须立即可用的资产
2. **可选资产用软引用**：可能不使用的大型资产
3. **使用UPROPERTY**：确保引用被正确追踪
4. **避免循环引用**：用软引用打破引用环
5. **及时释放引用**：置空指针或重置软引用

### 下节课预告

第2课将深入讲解同步与异步加载机制，学习如何高效地加载资产。
