# 第八课：对象引用与NetGUID系统

> **学习目标**: 理解网络对象引用和NetGUID的工作原理
> **时长**: 约60分钟
> **前置知识**: 第七课内容

---

## 一、NetGUID概述

### 1.1 什么是NetGUID?

NetGUID (Network Global Unique Identifier) 是UE网络系统中用于**标识网络对象**的唯一ID。

```
┌─────────────────────────────────────────────────────────────────────┐
│                      NetGUID的作用                                   │
└─────────────────────────────────────────────────────────────────────┘

服务器                              客户端
  │                                   │
  │  Actor A (NetGUID: 100)           │
  │    ↓                              │
  │  序列化: "NetGUID: 100"            │
  │    ↓                              │
  │  ─────── 网络传输 ──────────────→  │
  │                                   │  解析: NetGUID: 100
  │                                   │    ↓
  │                                   │  查找本地对象
  │                                   │    ↓
  │                                   │  获取 Actor A 的引用
```

### 1.2 为什么需要NetGUID?

- **对象引用同步**: 服务器和客户端的对象指针不同
- **动态对象**: 运行时创建的对象需要统一标识
- **异步加载**: 支持对象加载完成后再绑定

---

## 二、NetGUID结构

### 2.1 FNetworkGUID

```cpp
// NetworkGuid.h
struct FNetworkGUID
{
    uint32 Value;

    // 是否有效
    bool IsValid() const { return Value != 0; }

    // 是否是静态对象
    bool IsStatic() const { return (Value & 1) != 0; }

    // 是否是动态对象
    bool IsDynamic() const { return (Value & 1) == 0; }

    // 获取对象ID部分
    uint32 GetObjectID() const { return Value >> 1; }

    bool operator==(const FNetworkGUID& Other) const { return Value == Other.Value; }
};
```

### 2.2 GUID分配规则

```
┌─────────────────────────────────────────────────────────────────────┐
│                    NetGUID分配规则                                   │
└─────────────────────────────────────────────────────────────────────┘

静态对象 (从地图加载):
    - 最低位为1
    - 例如: 0x00000003 = 静态对象ID 1
    - 在地图中稳定存在

动态对象 (运行时创建):
    - 最低位为0
    - 例如: 0x00000002 = 动态对象ID 1
    - 由服务器分配

特殊值:
    - 0x00000000 = 无效GUID
    - 0x00000001 = 静态对象ID 0 (保留)
```

---

## 三、PackageMap系统

### 3.1 UPackageMap

**位置**: `Engine/Classes/Engine/PackageMapClient.h`

```cpp
/**
 * 管理NetGUID和对象的映射
 */
UCLASS(Transient, Abstract)
class UPackageMap : public UObject
{
    GENERATED_BODY()

public:
    // 序列化对象引用
    virtual bool SerializeObject(
        FArchive& Ar,
        UClass* InClass,
        UObject*& Obj,
        FNetworkGUID* OutNetGUID = nullptr
    );

    // 序列化对象 (新版本)
    virtual bool WriteObject(
        FArchive& Ar,
        UObject* InOuter,
        FNetworkGUID NetGUID,
        FString ObjName
    );
};
```

### 3.2 FNetGUIDCache

```cpp
/**
 * 缓存NetGUID到对象的映射
 */
class FNetGUIDCache
{
public:
    // 通过GUID获取对象
    UObject* GetObjectFromNetGUID(FNetworkGUID NetGUID);

    // 通过对象获取GUID
    FNetworkGUID GetNetGUID(UObject* Object);

    // 注册对象
    void RegisterNetGUID(FNetworkGUID NetGUID, UObject* Object);

    // 注销对象
    void UnregisterNetGUID(FNetworkGUID NetGUID);

private:
    // GUID -> 对象映射
    TMap<FNetworkGUID, TWeakObjectPtr<UObject>> GuidToObjectMap;

    // 对象 -> GUID映射
    TMap<TWeakObjectPtr<UObject>, FNetworkGUID> ObjectToGuidMap;
};
```

---

## 四、对象引用序列化

### 4.1 序列化流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                   对象引用序列化流程                                  │
└─────────────────────────────────────────────────────────────────────┘

发送端 (服务器):
┌──────────────────────────────────────────────────────────────────┐
│ UObject* Obj                                                     │
│      ↓                                                           │
│ 查找/分配 NetGUID                                                 │
│      ↓                                                           │
│ 如果是新对象，导出对象信息:                                        │
│   - Class                                                        │
│   - Outer                                                        │
│   - Name                                                         │
│      ↓                                                           │
│ 序列化 NetGUID 到 Bunch                                           │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                         网络传输
                              │
                              ▼
接收端 (客户端):
┌──────────────────────────────────────────────────────────────────┐
│ 解析 NetGUID                                                     │
│      ↓                                                           │
│ 查找本地缓存                                                      │
│      ↓                                                           │
│  ┌─────────────┬─────────────────────────────────────┐           │
│  │ 找到        │ 直接返回对象引用                      │           │
│  ├─────────────┼─────────────────────────────────────┤           │
│  │ 未找到      │ 处理导出信息:                         │           │
│  │             │   - 如果有导出数据，创建/加载对象      │           │
│  │             │   - 如果对象未加载，加入待处理队列     │           │
│  └─────────────┴─────────────────────────────────────┘           │
└──────────────────────────────────────────────────────────────────┘
```

### 4.2 对象导出 (Export)

```cpp
// 当发送新对象引用时，需要导出对象信息
struct FNetObjectExport
{
    // 对象的NetGUID
    FNetworkGUID NetGUID;

    // 对象类
    UClass* ObjectClass;

    // 外部对象
    UObject* Outer;

    // 对象名称
    FString ObjectName;

    // 是否已加载
    bool bIsLoaded;
};
```

### 4.3 对象导入 (Import)

```cpp
// 客户端接收对象引用时的处理
void FNetGUIDCache::ImportObject(
    FNetworkGUID NetGUID,
    UClass* ObjectClass,
    UObject* Outer,
    FString ObjectName)
{
    // 查找已存在的对象
    UObject* ExistingObject = FindObject(ObjectClass, Outer, *ObjectName);

    if (ExistingObject)
    {
        // 注册映射
        RegisterNetGUID(NetGUID, ExistingObject);
    }
    else
    {
        // 创建占位符或异步加载
        // ...
    }
}
```

---

## 五、静态与动态对象

### 5.1 静态对象

```cpp
// 静态对象 - 在编辑器中放置的对象
// 特点:
// - NetGUID低位为1
// - 在所有客户端上具有相同的路径
// - 不需要显式导出

// 示例：地图中放置的Actor
// 服务器和客户端都有相同的对象，只是指针不同
// 通过对象路径匹配
```

### 5.2 动态对象

```cpp
// 动态对象 - 运行时创建的对象
// 特点:
// - NetGUID低位为0
// - 由服务器分配GUID
// - 需要导出对象信息

// 示例：Spawn的Actor
AActor* SpawnedActor = GetWorld()->SpawnActor<AMyActor>(SpawnLocation);

// 服务器自动分配NetGUID
// 首次引用时会导出类信息
// 客户端根据导出信息创建对象
```

### 5.3 对象生命周期

```
┌─────────────────────────────────────────────────────────────────────┐
│                   动态对象生命周期                                    │
└─────────────────────────────────────────────────────────────────────┘

服务器:
    SpawnActor()
        │
        ├── 分配NetGUID
        │
        ├── 首次复制时导出
        │
        ├── 持续复制状态
        │
        └── Destroy() → 注销GUID

客户端:
    接收导出信息
        │
        ├── 创建本地对象
        │
        ├── 注册GUID映射
        │
        ├── 接收状态更新
        │
        └── 接收销毁消息 → 删除对象
```

---

## 六、异步对象加载

### 6.1 待处理对象队列

```cpp
// 当引用的对象还未加载时，需要等待
class FNetGUIDCache
{
    // 待处理的GUID
    TSet<FNetworkGUID> PendingGUIDs;

    // 等待GUID的Bunch队列
    TMap<FNetworkGUID, TArray<FInBunch*>> QueuedBunches;
};
```

### 6.2 处理流程

```
客户端收到Bunch:
    │
    ├── 解析对象引用
    │
    ├── GUID在缓存中？
    │   ├── 是 → 直接使用
    │   │
    │   └── 否 → 加入Pending队列
    │              │
    │              ├── 保存Bunch
    │              │
    │              └── 等待对象加载
    │
    对象加载完成:
    │
    ├── 注册GUID映射
    │
    └── 处理待处理的Bunch
```

### 6.3 代码示例

```cpp
// ActorChannel处理异步加载
void UActorChannel::ProcessBunch(FInBunch& Bunch)
{
    // 检查是否有未解析的GUID
    if (PendingGuidResolves.Num() > 0)
    {
        // 暂存Bunch
        QueuedBunches.Add(new FInBunch(Bunch, true));
        return;
    }

    // 处理Bunch
    ProcessBunchInternal(Bunch);
}

// GUID解析后处理队列
void UActorChannel::OnGuidResolved(FNetworkGUID ResolvedGuid)
{
    PendingGuidResolves.Remove(ResolvedGuid);

    if (PendingGuidResolves.Num() == 0)
    {
        // 处理所有队列中的Bunch
        ProcessQueuedBunches();
    }
}
```

---

## 七、NetGUID在RPC中的使用

### 7.1 对象参数序列化

```cpp
// RPC传递对象参数
UFUNCTION(Server, Reliable, WithValidation)
void ServerEquipWeapon(AWeapon* NewWeapon);

// 序列化过程:
// 1. 获取对象的NetGUID
// 2. 如果是新对象，导出对象信息
// 3. 序列化NetGUID
// 4. 接收端根据GUID获取对象引用
```

### 7.2 实际应用

```cpp
// 服务器RPC传递Actor
void AMyCharacter::ServerInteract_Implementation(AActor* Target)
{
    // Target是通过NetGUID序列化的对象引用
    if (Target)
    {
        Target->Interact(this);
    }
}

// 客户端调用
void AMyCharacter::InteractWithTarget(AActor* Target)
{
    // Target会自动序列化为NetGUID
    ServerInteract(Target);
}
```

---

## 八、调试NetGUID

### 8.1 控制台命令

```
; 显示NetGUID缓存信息
net.NetGUID.Debug 1

; 显示对象导出信息
net.Export.Debug 1

; 统计信息
stat net
```

### 8.2 日志输出

```cpp
// 追踪NetGUID分配
void FNetGUIDCache::RegisterNetGUID(FNetworkGUID NetGUID, UObject* Object)
{
    UE_LOG(LogNet, Verbose,
        TEXT("Registering NetGUID: %u -> %s"),
        NetGUID.Value,
        Object ? *Object->GetName() : TEXT("nullptr")
    );

    // ...
}
```

### 8.3 常见问题

| 问题         | 原因         | 解决方案             |
| ------------ | ------------ | -------------------- |
| 对象引用为空 | GUID未注册   | 检查对象是否正确复制 |
| 异步加载卡住 | 导出信息丢失 | 检查网络连接         |
| GUID冲突     | 手动分配冲突 | 让系统自动分配       |
| 内存泄漏     | 对象未注销   | 确保Destroy时清理    |

---

## 九、最佳实践

### 9.1 使用对象引用

```cpp
// 好：使用对象指针，自动处理NetGUID
UFUNCTION(Server, Reliable, WithValidation)
void ServerSetTarget(AActor* Target);

// 避免：使用原始GUID
// UFUNCTION(Server, Reliable, WithValidation)
// void ServerSetTargetGUID(uint32 TargetGUID);
```

### 9.2 处理空引用

```cpp
void AMyCharacter::ServerEquipWeapon_Implementation(AWeapon* NewWeapon)
{
    // 检查对象是否有效
    if (!NewWeapon)
    {
        UE_LOG(LogNet, Warning, TEXT("Invalid weapon reference"));
        return;
    }

    // 检查对象是否已销毁
    if (NewWeapon->IsPendingKillPending())
    {
        return;
    }

    // 执行逻辑
    EquipWeapon(NewWeapon);
}
```

### 9.3 异步处理

```cpp
// 等待对象加载完成
void AMyCharacter::OnWeaponLoaded()
{
    if (PendingWeaponGuid.IsValid())
    {
        UObject* LoadedWeapon = GetNetGUIDCache()->GetObjectFromNetGUID(PendingWeaponGuid);
        if (LoadedWeapon)
        {
            EquipWeapon(Cast<AWeapon>(LoadedWeapon));
            PendingWeaponGuid = FNetworkGUID();
        }
    }
}
```

---

## 十、总结

### 10.1 核心要点

1. **NetGUID**: 网络对象的唯一标识符
2. **静态对象**: 地图中放置，GUID低位为1
3. **动态对象**: 运行时创建，GUID低位为0
4. **PackageMap**: 管理GUID和对象的映射
5. **异步加载**: 支持对象加载完成后再绑定

### 10.2 下一课预告

**第九课：子对象复制**

将深入讲解：

- ReplicateSubobjects实现
- 子对象生命周期管理
- 组件复制

---

_上一课: [第七课：RPC远程过程调用](./Lesson07_RPC.md)_
_下一课: [第九课：子对象复制](./Lesson09_SubobjectReplication.md)_
