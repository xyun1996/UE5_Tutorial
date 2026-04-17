# 第4课：资产注册与资产管理器

本课深入讲解资产注册表和资产管理器，这是UE5中管理大型项目资产的核心系统。

---

## 4.1 资产注册表

### 概述

资产注册表（Asset Registry）是一个全局数据库，用于追踪项目中所有的资产信息。

```
┌─────────────────────────────────────────────────────────────────┐
│                    资产注册表架构                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                     IAssetRegistry                              │
│                          │                                      │
│           ┌──────────────┼──────────────┐                      │
│           │              │              │                       │
│           ▼              ▼              ▼                        │
│   ┌───────────────┐ ┌─────────────┐ ┌──────────────────┐       │
│   │ 扫描资产文件  │ │ 建立索引   │ │ 提供查询API      │       │
│   │ (.uasset)     │ │             │ │                  │       │
│   │               │ │             │ │                  │       │
│   │ - 包扫描     │ │ - 路径索引 │ │ - 按路径查找     │       │
│   │ - 依赖分析   │ │ - 类索引   │ │ - 按类查找       │       │
│   │ - 标签提取   │ │ - 标签索引 │ │ - 按标签查找     │       │
│   └───────────────┘ └─────────────┘ └──────────────────┘       │
│                                                                 │
│   核心数据结构：FAssetData                                      │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  - ObjectPath: 资产完整路径                              │  │
│   │  - PackagePath: 包路径                                   │  │
│   │  - AssetName: 资产名称                                   │  │
│   │  - AssetClassPath: 资产类                                │  │
│   │  - TagsAndValues: 标签-值映射                            │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### FAssetData详解

```cpp
// FAssetData 包含资产的元数据
struct FAssetData
{
    // 资产完整路径
    // 格式：/Game/Path/Asset.AssetName
    FTopLevelAssetPath ObjectPath;

    // 包路径（不含资产名）
    // 格式：/Game/Path
    FName PackagePath;

    // 资产名称
    FName AssetName;

    // 资产类路径
    FTopLevelAssetPath AssetClassPath;

    // 标签-值映射（资产的元数据）
    TMap<FName, FString> TagsAndValues;

    // 包标志
    uint32 PackageFlags;

    // ========== 核心方法 ==========

    // 获取完整名称
    FString GetFullName() const;

    // 转换为软对象路径
    FSoftObjectPath ToSoftObjectPath() const;

    // 获取标签值
    bool GetTagValue(FName Tag, FString& OutValue) const;
    FString GetTagValueRef(FName Tag) const;

    // 获取资产（会触发加载）
    UObject* GetAsset() const;

    // 检查是否是UClass
    bool IsUAsset() const;

    // 检查是否已加载
    bool IsAssetLoaded() const;
};
```

### 获取资产注册表

```cpp
#include "AssetRegistry/AssetRegistryModule.h"

void UseAssetRegistry()
{
    // 方式1：通过模块获取
    FAssetRegistryModule& AssetRegistryModule =
        FModuleManager::LoadModuleChecked<FAssetRegistryModule>(
            FName("AssetRegistry")
        );
    IAssetRegistry& AssetRegistry = AssetRegistryModule.Get();

    // 方式2：直接获取（UE5推荐）
    IAssetRegistry& AssetRegistry = UAssetManager::Get().GetAssetRegistry();
}
```

### 基本查询操作

```cpp
void AssetRegistryQueries()
{
    IAssetRegistry& AssetRegistry = UAssetManager::Get().GetAssetRegistry();
    TArray<FAssetData> AssetDataList;

    // ========== 按路径查询 ==========

    // 查询路径下的所有资产
    AssetRegistry.GetAssetsByPath(FName("/Game/Meshes"), AssetDataList);

    // 递归查询子目录
    AssetRegistry.GetAssetsByPath(
        FName("/Game"),
        AssetDataList,
        true  // bRecursive
    );

    // ========== 按类查询 ==========

    // 查询所有静态网格
    AssetRegistry.GetAssetsByClass(
        UStaticMesh::StaticClass()->GetClassPathName(),
        AssetDataList
    );

    // 查询多个类
    TArray<FTopLevelAssetPath> ClassPaths;
    ClassPaths.Add(UStaticMesh::StaticClass()->GetClassPathName());
    ClassPaths.Add(USkeletalMesh::StaticClass()->GetClassPathName());
    AssetRegistry.GetAssetsByClasses(ClassPaths, AssetDataList);

    // ========== 按名称查询 ==========

    // 按资产名称搜索
    AssetRegistry.GetAssetsByName(FName("SM_Weapon"), AssetDataList);

    // 按完整路径获取单个资产
    FAssetData AssetData = AssetRegistry.GetAssetByObjectPath(
        FSoftObjectPath("/Game/Meshes/SM_Weapon.SM_Weapon")
    );

    // ========== 按标签查询 ==========

    // 查询带特定标签的资产
    TMap<FName, FString> TagValues;
    TagValues.Add(FName("Category"), FString("Weapon"));
    AssetRegistry.GetAssetsByTagValues(TagValues, AssetDataList);
}
```

### 高级查询

```cpp
void AdvancedQueries()
{
    IAssetRegistry& AssetRegistry = UAssetManager::Get().GetAssetRegistry();

    // ========== 使用过滤条件 ==========

    FARFilter Filter;
    Filter.bRecursivePaths = true;
    Filter.PackagePaths.Add(FName("/Game"));
    Filter.ClassPaths.Add(UStaticMesh::StaticClass()->GetClassPathName());

    // 添加标签过滤
    Filter.TagsAndValues.Add(FName("Category"), FString("Weapon"));

    // 排除某些路径
    Filter.RecursivePathsExclusion.Add(FName("/Game/Developers"));

    // 只包含已加载的资产
    Filter.bIncludeOnlyOnDiskAssets = false;

    TArray<FAssetData> AssetDataList;
    AssetRegistry.GetAssets(Filter, AssetDataList);

    // ========== 依赖查询 ==========

    // 查询资产的依赖
    TArray<FName> Dependencies;
    AssetRegistry.GetDependencies(
        FName("/Game/Meshes/SM_Weapon.SM_Weapon"),
        Dependencies,
        UE::AssetRegistry::EDependencyQuery::Hard
    );

    // 查询谁依赖了这个资产
    TArray<FName> Referencers;
    AssetRegistry.GetReferencers(
        FName("/Game/Meshes/SM_Weapon.SM_Weapon"),
        Referencers
    );
}
```

### 资产标签

```cpp
// 资产标签是存储在资产中的元数据
// 可以在编辑器中设置，也可以在代码中读取

// 读取资产标签
void ReadAssetTags(const FAssetData& AssetData)
{
    // 获取单个标签
    FString CategoryValue;
    if (AssetData.GetTagValue(FName("Category"), CategoryValue))
    {
        UE_LOG(LogTemp, Log, TEXT("Category: %s"), *CategoryValue);
    }

    // 遍历所有标签
    for (const auto& Pair : AssetData.TagsAndValues)
    {
        UE_LOG(LogTemp, Log, TEXT("Tag: %s = %s"),
            *Pair.Key.ToString(), *Pair.Value);
    }

    // 常见的内置标签
    // - ParentClass: 蓝图的父类
    // - GenerateType: 蓝图生成类型
    // - NativeClassName: C++类名
}

// 在蓝图中设置自定义标签
// 1. 打开蓝图编辑器
// 2. 在类设置中找到 "Asset Tags"
// 3. 添加键值对
```

---

## 4.2 资产管理器

### 概述

资产管理器（Asset Manager）是UE5中管理资产加载和生命周期的核心系统。

```
┌─────────────────────────────────────────────────────────────────┐
│                    资产管理器架构                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                       UAssetManager                             │
│                            │                                    │
│         ┌──────────────────┼──────────────────┐                │
│         │                  │                  │                 │
│         ▼                  ▼                  ▼                 │
│   ┌───────────────┐ ┌───────────────┐ ┌───────────────┐       │
│   │Streamable     │ │ Primary Asset │ │ Bundle        │       │
│   │Manager        │ │ Management    │ │ Management    │       │
│   │               │ │               │ │               │       │
│   │ - 异步加载   │ │ - 主资产管理 │ │ - 分组加载   │       │
│   │ - 引用计数   │ │ - 加载状态   │ │ - 条件加载   │       │
│   │ - 加载队列   │ │ - 缓存管理   │ │ - 依赖管理   │       │
│   └───────────────┘ └───────────────┘ └───────────────┘       │
│                                                                 │
│   Primary Assets（主资产）                                      │
│   ─────────────────────────                                     │
│   • 由游戏逻辑直接管理                                          │
│   • 有明确的加载/卸载时机                                       │
│   • 例如：关卡、角色、武器配置                                  │
│                                                                 │
│   Secondary Assets（次级资产）                                  │
│   ─────────────────────────                                     │
│   • 被主资产间接引用                                            │
│   • 随主资产自动加载/卸载                                       │
│   • 例如：材质、纹理、音效                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Primary Asset

```cpp
// Primary Asset 是由游戏逻辑管理的核心资产
// 继承自 UPrimaryDataAsset

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

    // 武器描述
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Weapon")
    FText Description;

    // 武器网格（软引用）
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Weapon")
    TSoftObjectPtr<UStaticMesh> WeaponMesh;

    // 武器图标
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Weapon")
    TSoftObjectPtr<UTexture2D> Icon;

    // 基础伤害
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Weapon")
    float BaseDamage = 10.0f;

    // 攻击速度
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Weapon")
    float AttackSpeed = 1.0f;

    // 关键：定义主资产ID
    virtual FPrimaryAssetId GetPrimaryAssetId() const override
    {
        return FPrimaryAssetId("Weapon", GetFName());
    }
};
```

### FPrimaryAssetId

```cpp
// FPrimaryAssetId 用于唯一标识一个主资产
struct FPrimaryAssetId
{
    // 主资产类型
    FName PrimaryAssetType;

    // 主资产名称
    FName PrimaryAssetName;

    // 构造
    FPrimaryAssetId() {}
    FPrimaryAssetId(FName InType, FName InName)
        : PrimaryAssetType(InType), PrimaryAssetName(InName) {}

    // 检查有效性
    bool IsValid() const
    {
        return PrimaryAssetType.IsValid() && PrimaryAssetName.IsValid();
    }

    // 转换为字符串
    FString ToString() const;

    // 从字符串解析
    static FPrimaryAssetId FromString(const FString& String);
};
```

### 配置资产管理器

```ini
; DefaultGame.ini

[/Script/Engine.AssetManagerSettings]
; Primary Asset Types to Scan
; 定义要扫描的主资产类型

+PrimaryAssetTypesToScan=(PrimaryAssetType="Map",AssetBaseClass="/Script/Engine.World",bHasBlueprintClasses=False,bIsEditorOnly=False,Directories=(Path="/Game/Maps"),SpecificAssets=,bSerializeAssetRegistry=True)

+PrimaryAssetTypesToScan=(PrimaryAssetType="Weapon",AssetBaseClass="/Script/MyGame.WeaponData",bHasBlueprintClasses=False,bIsEditorOnly=False,Directories=(Path="/Game/Data/Weapons"))

+PrimaryAssetTypesToScan=(PrimaryAssetType="Character",AssetBaseClass="/Script/MyGame.CharacterData",bHasBlueprintClasses=False,bIsEditorOnly=False,Directories=(Path="/Game/Data/Characters"))

; Directories to Exclude
; 排除的目录
+DirectoriesToExclude=(Path="/Game/Developers")

; Primary Assets to Load at Startup
; 启动时加载的主资产
+PrimaryAssetsToLoadAtStartup=PrimaryAssetId("Map","EntryMap")

; Only Cook Production Assets
bOnlyCookProductionAssets=True
```

### 使用资产管理器

```cpp
// 获取资产管理器
UAssetManager& AssetManager = UAssetManager::Get();

// ========== 加载主资产 ==========

// 同步加载
UWeaponData* WeaponData = Cast<UWeaponData>(
    AssetManager.LoadPrimaryAssetSync(FPrimaryAssetId("Weapon", FName("Rifle")))
);

// 异步加载
void LoadWeaponAsync(FName WeaponName)
{
    TArray<FName> Bundles;  // 加载的Bundle
    Bundles.Add(FName("Game"));

    AssetManager.LoadPrimaryAsset(
        FPrimaryAssetId("Weapon", WeaponName),
        Bundles,
        FStreamableDelegate::CreateLambda([WeaponName]()
        {
            UAssetManager& AM = UAssetManager::Get();
            UWeaponData* Data = Cast<UWeaponData>(
                AM.GetPrimaryAssetObject(FPrimaryAssetId("Weapon", WeaponName))
            );

            if (Data)
            {
                UE_LOG(LogTemp, Log, TEXT("Loaded weapon: %s"),
                    *Data->DisplayName.ToString());
            }
        })
    );
}

// ========== 卸载主资产 ==========

void UnloadWeapon(FName WeaponName)
{
    AssetManager.UnloadPrimaryAsset(
        FPrimaryAssetId("Weapon", WeaponName)
    );
}

// ========== 查询主资产 ==========

void ListAllWeapons()
{
    TArray<FPrimaryAssetId> WeaponIds;
    AssetManager.GetPrimaryAssetIdList(FName("Weapon"), WeaponIds);

    for (const FPrimaryAssetId& Id : WeaponIds)
    {
        UE_LOG(LogTemp, Log, TEXT("Weapon: %s"), *Id.ToString());
    }
}

// 获取主资产数据
void GetWeaponData(FName WeaponName)
{
    FPrimaryAssetId AssetId("Weapon", WeaponName);

    // 获取资产数据（不加载）
    FAssetData AssetData;
    if (AssetManager.GetPrimaryAssetData(AssetId, AssetData))
    {
        FString Category;
        if (AssetData.GetTagValue(FName("Category"), Category))
        {
            UE_LOG(LogTemp, Log, TEXT("Category: %s"), *Category);
        }
    }
}
```

---

## 4.3 Bundle系统

### 概述

Bundle用于分组加载资产的依赖项，实现条件性加载。

```
┌─────────────────────────────────────────────────────────────────┐
│                    Bundle系统原理                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   CharacterData（主资产）                                        │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                                                         │  │
│   │   Game Bundle:                                          │  │
│   │   ├── GameMesh (游戏中使用的网格)                        │  │
│   │   ├── GameMaterial (游戏中使用的材质)                    │  │
│   │   └── Icon (图标)                                        │  │
│   │                                                         │  │
│   │   Menu Bundle:                                          │  │
│   │   ├── MenuMesh (菜单中使用的高精度网格)                  │  │
│   │   ├── MenuMaterial (菜单中使用的材质)                    │  │
│   │   └── Icon (图标 - 与Game共享)                          │  │
│   │                                                         │  │
│   │   Preview Bundle:                                       │  │
│   │   └── PreviewMesh (编辑器预览网格)                       │  │
│   │                                                         │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   加载策略：                                                    │
│   ─────────────                                                │
│   • 进入游戏：加载 Game Bundle                                 │
│   • 打开菜单：加载 Menu Bundle                                 │
│   • 编辑器预览：加载 Preview Bundle                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 定义Bundle

```cpp
// 在主资产中使用AssetBundles元数据定义Bundle

UCLASS(BlueprintType)
class UCharacterData : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // 游戏中使用的网格
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Character",
        meta = (AssetBundles = "Game"))
    TSoftObjectPtr<USkeletalMesh> GameMesh;

    // 菜单中使用的高精度网格
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Character",
        meta = (AssetBundles = "Menu"))
    TSoftObjectPtr<USkeletalMesh> MenuMesh;

    // 编辑器预览网格
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Character",
        meta = (AssetBundles = "Preview"))
    TSoftObjectPtr<USkeletalMesh> PreviewMesh;

    // UI图标 - 多个Bundle都需要
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Character",
        meta = (AssetBundles = "Game,Menu,Preview"))
    TSoftObjectPtr<UTexture2D> CharacterIcon;

    // 背景故事 - 只有详情页需要
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Character",
        meta = (AssetBundles = "Detail"))
    FString Backstory;

    // 主资产ID
    virtual FPrimaryAssetId GetPrimaryAssetId() const override
    {
        return FPrimaryAssetId("Character", GetFName());
    }
};
```

### 加载指定Bundle

```cpp
// 根据场景加载不同的Bundle

class UCharacterManager : public UObject
{
public:
    // 进入游戏时
    void LoadCharacterForGame(FName CharacterName)
    {
        TArray<FName> Bundles;
        Bundles.Add(FName("Game"));

        UAssetManager::Get().LoadPrimaryAsset(
            FPrimaryAssetId("Character", CharacterName),
            Bundles,
            FStreamableDelegate::CreateLambda([CharacterName]()
            {
                OnCharacterLoaded(CharacterName);
            })
        );
    }

    // 打开菜单时
    void LoadCharacterForMenu(FName CharacterName)
    {
        TArray<FName> Bundles;
        Bundles.Add(FName("Menu"));
        // 不需要加载Game Bundle

        UAssetManager::Get().LoadPrimaryAsset(
            FPrimaryAssetId("Character", CharacterName),
            Bundles,
            FStreamableDelegate::CreateLambda([CharacterName]()
            {
                OnCharacterLoadedForMenu(CharacterName);
            })
        );
    }

    // 加载多个Bundle
    void LoadCharacterWithDetails(FName CharacterName)
    {
        TArray<FName> Bundles;
        Bundles.Add(FName("Game"));
        Bundles.Add(FName("Detail"));

        UAssetManager::Get().LoadPrimaryAsset(
            FPrimaryAssetId("Character", CharacterName),
            Bundles,
            FStreamableDelegate::CreateLambda([]()
            {
                // 包含Game和Detail的内容
            })
        );
    }
};
```

---

## 4.4 完整示例：装备系统

```cpp
// EquipmentSystem.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/DataAsset.h"
#include "EquipmentSystem.generated.h"

// 装备类型
UENUM(BlueprintType)
enum class EEquipmentType : uint8
{
    Weapon,
    Armor,
    Accessory,
    Consumable
};

// 稀有度
UENUM(BlueprintType)
enum class ERarity : uint8
{
    Common,
    Uncommon,
    Rare,
    Epic,
    Legendary
};

// 装备数据
UCLASS(BlueprintType)
class UEquipmentData : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // 装备ID
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Equipment")
    FName EquipmentId;

    // 显示名称
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Equipment")
    FText DisplayName;

    // 描述
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Equipment",
        meta = (AssetBundles = "Detail"))
    FText Description;

    // 装备类型
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Equipment")
    EEquipmentType Type;

    // 稀有度
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Equipment")
    ERarity Rarity;

    // 图标
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Equipment",
        meta = (AssetBundles = "Game,Menu"))
    TSoftObjectPtr<UTexture2D> Icon;

    // 模型 - 游戏中使用
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Equipment",
        meta = (AssetBundles = "Game"))
    TSoftObjectPtr<UStaticMesh> GameMesh;

    // 模型 - 菜单预览
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Equipment",
        meta = (AssetBundles = "Menu"))
    TSoftObjectPtr<UStaticMesh> PreviewMesh;

    // 材质
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Equipment",
        meta = (AssetBundles = "Game,Menu"))
    TSoftObjectPtr<UMaterialInterface> Material;

    // 基础属性
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Stats")
    int32 Attack = 0;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Stats")
    int32 Defense = 0;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Stats")
    int32 Speed = 0;

    // 主资产ID
    virtual FPrimaryAssetId GetPrimaryAssetId() const override
    {
        return FPrimaryAssetId("Equipment", GetFName());
    }
};

// 装备管理器
UCLASS(BlueprintType)
class UEquipmentManager : public UObject
{
    GENERATED_BODY()

public:
    // 初始化
    UFUNCTION(BlueprintCallable, Category = "Equipment")
    void Initialize();

    // 获取所有装备ID
    UFUNCTION(BlueprintCallable, Category = "Equipment")
    void GetAllEquipmentIds(TArray<FName>& OutIds);

    // 获取某类型的装备
    UFUNCTION(BlueprintCallable, Category = "Equipment")
    void GetEquipmentByType(EEquipmentType Type, TArray<FName>& OutIds);

    // 加载装备数据
    UFUNCTION(BlueprintCallable, Category = "Equipment")
    void LoadEquipment(FName EquipmentId, bool bIncludeDetails, FOnEquipmentLoaded OnLoaded);

    // 同步获取已加载的装备
    UFUNCTION(BlueprintPure, Category = "Equipment")
    UEquipmentData* GetLoadedEquipment(FName EquipmentId);

    // 卸载装备
    UFUNCTION(BlueprintCallable, Category = "Equipment")
    void UnloadEquipment(FName EquipmentId);

    // 获取装备数据（不加载）
    UFUNCTION(BlueprintPure, Category = "Equipment")
    void GetEquipmentAssetData(FName EquipmentId, FAssetData& OutData);

    // 按稀有度过滤
    UFUNCTION(BlueprintCallable, Category = "Equipment")
    void FilterByRarity(ERarity MinRarity, TArray<FName>& OutIds);

private:
    // 已加载的装备缓存
    UPROPERTY()
    TMap<FName, UEquipmentData*> LoadedEquipment;

    // 正在加载的装备
    TSet<FName> LoadingEquipment;

    // 加载句柄
    TMap<FName, TSharedPtr<FStreamableHandle>> LoadHandles;
};

DECLARE_DYNAMIC_DELEGATE_OneParam(FOnEquipmentLoaded, FName, EquipmentId);
```

```cpp
// EquipmentSystem.cpp
#include "EquipmentSystem.h"
#include "AssetManager.h"

void UEquipmentManager::Initialize()
{
    LoadedEquipment.Empty();
    LoadingEquipment.Empty();
    LoadHandles.Empty();
}

void UEquipmentManager::GetAllEquipmentIds(TArray<FName>& OutIds)
{
    UAssetManager& AssetManager = UAssetManager::Get();
    TArray<FPrimaryAssetId> AssetIds;

    AssetManager.GetPrimaryAssetIdList(FName("Equipment"), AssetIds);

    for (const FPrimaryAssetId& Id : AssetIds)
    {
        OutIds.Add(Id.PrimaryAssetName);
    }
}

void UEquipmentManager::GetEquipmentByType(EEquipmentType Type, TArray<FName>& OutIds)
{
    IAssetRegistry& AssetRegistry = UAssetManager::Get().GetAssetRegistry();
    TArray<FPrimaryAssetId> AssetIds;

    UAssetManager::Get().GetPrimaryAssetIdList(FName("Equipment"), AssetIds);

    for (const FPrimaryAssetId& Id : AssetIds)
    {
        FAssetData AssetData;
        if (UAssetManager::Get().GetPrimaryAssetData(Id, AssetData))
        {
            // 检查类型标签
            FString TypeString;
            if (AssetData.GetTagValue(FName("Type"), TypeString))
            {
                EEquipmentType AssetType = StaticEnum<EEquipmentType>()->GetValueByNameString(TypeString);
                if (AssetType == Type)
                {
                    OutIds.Add(Id.PrimaryAssetName);
                }
            }
        }
    }
}

void UEquipmentManager::LoadEquipment(FName EquipmentId, bool bIncludeDetails, FOnEquipmentLoaded OnLoaded)
{
    // 检查是否已加载
    if (UEquipmentData** Found = LoadedEquipment.Find(EquipmentId))
    {
        OnLoaded.ExecuteIfBound(EquipmentId);
        return;
    }

    // 检查是否正在加载
    if (LoadingEquipment.Contains(EquipmentId))
    {
        return;
    }

    LoadingEquipment.Add(EquipmentId);

    // 设置要加载的Bundle
    TArray<FName> Bundles;
    Bundles.Add(FName("Game"));

    if (bIncludeDetails)
    {
        Bundles.Add(FName("Detail"));
    }

    FPrimaryAssetId AssetId("Equipment", EquipmentId);

    TSharedPtr<FStreamableHandle> Handle = UAssetManager::Get().LoadPrimaryAsset(
        AssetId,
        Bundles,
        FStreamableDelegate::CreateUObject(this, &UEquipmentManager::OnEquipmentLoadedInternal, EquipmentId, OnLoaded)
    );

    LoadHandles.Add(EquipmentId, Handle);
}

void UEquipmentManager::OnEquipmentLoadedInternal(FName EquipmentId, FOnEquipmentLoaded OnLoaded)
{
    LoadingEquipment.Remove(EquipmentId);
    LoadHandles.Remove(EquipmentId);

    FPrimaryAssetId AssetId("Equipment", EquipmentId);
    UEquipmentData* Data = Cast<UEquipmentData>(
        UAssetManager::Get().GetPrimaryAssetObject(AssetId)
    );

    if (Data)
    {
        LoadedEquipment.Add(EquipmentId, Data);
    }

    OnLoaded.ExecuteIfBound(EquipmentId);
}

UEquipmentData* UEquipmentManager::GetLoadedEquipment(FName EquipmentId)
{
    if (UEquipmentData** Found = LoadedEquipment.Find(EquipmentId))
    {
        return *Found;
    }
    return nullptr;
}

void UEquipmentManager::UnloadEquipment(FName EquipmentId)
{
    LoadedEquipment.Remove(EquipmentId);
    LoadingEquipment.Remove(EquipmentId);
    LoadHandles.Remove(EquipmentId);

    UAssetManager::Get().UnloadPrimaryAsset(
        FPrimaryAssetId("Equipment", EquipmentId)
    );
}

void UEquipmentManager::FilterByRarity(ERarity MinRarity, TArray<FName>& OutIds)
{
    IAssetRegistry& AssetRegistry = UAssetManager::Get().GetAssetRegistry();
    TArray<FPrimaryAssetId> AssetIds;

    UAssetManager::Get().GetPrimaryAssetIdList(FName("Equipment"), AssetIds);

    for (const FPrimaryAssetId& Id : AssetIds)
    {
        FAssetData AssetData;
        if (UAssetManager::Get().GetPrimaryAssetData(Id, AssetData))
        {
            FString RarityString;
            if (AssetData.GetTagValue(FName("Rarity"), RarityString))
            {
                ERarity Rarity = StaticEnum<ERarity>()->GetValueByNameString(RarityString);
                if (Rarity >= MinRarity)
                {
                    OutIds.Add(Id.PrimaryAssetName);
                }
            }
        }
    }
}
```

---

## 4.5 实践练习

### 练习1：创建自定义主资产类型

```cpp
// 创建一个技能数据主资产
UCLASS(BlueprintType)
class USkillData : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    FName SkillId;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    FText DisplayName;

    // 实现GetPrimaryAssetId
    virtual FPrimaryAssetId GetPrimaryAssetId() const override;
};
```

### 练习2：实现资产搜索系统

```cpp
// 实现一个基于标签的资产搜索系统
UCLASS()
class UAssetSearchSystem : public UObject
{
    GENERATED_BODY()

public:
    // 按标签搜索
    UFUNCTION(BlueprintCallable)
    void SearchByTags(const TMap<FName, FString>& Tags, TArray<FAssetData>& Results);

    // 按类和路径搜索
    UFUNCTION(BlueprintCallable)
    void SearchByClassAndPath(TSubclassOf<UObject> AssetClass, const FString& Path, TArray<FAssetData>& Results);
};
```

---

## 4.6 小结

### 核心概念

| 概念 | 说明 |
|------|------|
| **Asset Registry** | 全局资产数据库，存储资产元数据 |
| **FAssetData** | 资产元数据，无需加载即可访问 |
| **Asset Manager** | 资产生命周期管理 |
| **Primary Asset** | 由游戏逻辑直接管理的资产 |
| **Bundle** | 条件性加载的资产分组 |

### 最佳实践

1. **使用Primary Asset管理核心资产**
2. **利用Bundle实现条件加载**
3. **使用资产标签增强搜索能力**
4. **配置正确的扫描规则**

### 下节课预告

第5课将学习垃圾回收与资产释放，掌握内存管理技巧。
