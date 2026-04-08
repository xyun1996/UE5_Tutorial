# 第九课：子对象复制

> **学习目标**: 掌握子对象(Subobject)的网络复制方法
> **时长**: 约60分钟
> **前置知识**: 第八课内容

---

## 一、子对象复制概述

### 1.1 什么是子对象?

子对象是**Actor拥有的、需要独立复制的UObject**，包括：

- UActorComponent（组件）
- 自定义UObject子类
- 动态创建的对象

### 1.2 为什么需要子对象复制?

```
┌─────────────────────────────────────────────────────────────────────┐
│                    子对象复制场景                                    │
└─────────────────────────────────────────────────────────────────────┘

Actor (服务器)
    │
    ├── WeaponComponent (组件)
    │       └── CurrentWeapon (对象)
    │               └── Ammo (属性)
    │
    └── InventoryComponent (组件)
            └── Items[] (数组)

需要复制:
    - Actor本身
    - 每个组件的状态
    - 组件内的子对象
```

---

## 二、子对象复制机制

### 2.1 ReplicateSubobjects函数

```cpp
/**
 * 在ActorChannel中复制子对象
 *
 * @param Channel   当前ActorChannel
 * @param Bunch     输出数据束
 * @param RepFlags  复制标志
 * @return 是否写入了数据
 */
virtual bool ReplicateSubobjects(
    UActorChannel* Channel,
    FOutBunch* Bunch,
    FReplicationFlags* RepFlags
);
```

### 2.2 ActorChannel中的子对象管理

```cpp
class UActorChannel : public UChannel
{
public:
    // Actor的复制器
    TSharedPtr<FObjectReplicator> ActorReplicator;

    // 子对象复制器映射
    TMap<UObject*, TSharedRef<FObjectReplicator>> ReplicationMap;

    // 创建的子对象列表
    TArray<TObjectPtr<UObject>> CreateSubObjects;
};
```

### 2.3 子对象复制流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                   子对象复制流程                                      │
└─────────────────────────────────────────────────────────────────────┘

UActorChannel::ReplicateActor()
        │
        ├── 复制Actor属性
        │
        ▼
DoSubObjectReplication()
        │
        ├── 遍历注册的子对象
        │   │
        │   ├── ReplicateSubobject()
        │   │       │
        │   │       ├── 检查是否需要复制
        │   │       │
        │   │       ├── 获取/创建ObjectReplicator
        │   │       │
        │   │       └── ReplicateProperties()
        │   │
        │   └── 写入内容块到Bunch
        │
        └── 返回是否写入数据
```

---

## 三、实现子对象复制

### 3.1 基本实现

```cpp
// MyActor.h
UCLASS()
class AMyActor : public AActor
{
    GENERATED_BODY()

public:
    virtual bool ReplicateSubobjects(
        UActorChannel* Channel,
        FOutBunch* Bunch,
        FReplicationFlags* RepFlags) override;

protected:
    // 子对象列表
    UPROPERTY()
    TArray<UObject*> ReplicatedSubobjects;

    // 添加子对象
    void AddReplicatedSubobject(UObject* Subobject);
};
```

```cpp
// MyActor.cpp
bool AMyActor::ReplicateSubobjects(
    UActorChannel* Channel,
    FOutBunch* Bunch,
    FReplicationFlags* RepFlags)
{
    bool bWroteSomething = Super::ReplicateSubobjects(Channel, Bunch, RepFlags);

    // 复制每个子对象
    for (UObject* Subobject : ReplicatedSubobjects)
    {
        if (Subobject && !Subobject->IsPendingKillPending())
        {
            bWroteSomething |= Channel->ReplicateSubobject(Subobject, *Bunch, *RepFlags);
        }
    }

    return bWroteSomething;
}

void AMyActor::AddReplicatedSubobject(UObject* Subobject)
{
    if (Subobject && !ReplicatedSubobjects.Contains(Subobject))
    {
        ReplicatedSubobjects.Add(Subobject);
    }
}
```

### 3.2 自定义可复制子对象

```cpp
// MySubobject.h
UCLASS()
class UMySubobject : public UObject
{
    GENERATED_BODY()

public:
    // 子对象也需要注册复制属性
    virtual void GetLifetimeReplicatedProps(
        TArray<FLifetimeProperty>& OutLifetimeProps) const override;

protected:
    UPROPERTY(Replicated)
    int32 SubobjectValue;

    UPROPERTY(ReplicatedUsing=OnRep_State)
    FString State;

    UFUNCTION()
    void OnRep_State();
};
```

```cpp
// MySubobject.cpp
void UMySubobject::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(UMySubobject, SubobjectValue);
    DOREPLIFETIME_CONDITION_NOTIFY(UMySubobject, State, COND_None, REPNOTIFY_OnChanged);
}
```

### 3.3 完整示例：武器系统

```cpp
// MyWeaponObject.h
UCLASS()
class UMyWeaponObject : public UObject
{
    GENERATED_BODY()

public:
    virtual void GetLifetimeReplicatedProps(
        TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    // 武器属性
    UPROPERTY(Replicated)
    FString WeaponName;

    UPROPERTY(Replicated)
    int32 CurrentAmmo;

    UPROPERTY(Replicated)
    int32 MaxAmmo;

    UPROPERTY(Replicated)
    float Damage;

    // 方法
    bool CanFire() const { return CurrentAmmo > 0; }
    void Fire() { if (CanFire()) CurrentAmmo--; }
    void Reload() { CurrentAmmo = MaxAmmo; }
};
```

```cpp
// MyCharacter.h
UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    AMyCharacter();

    virtual void GetLifetimeReplicatedProps(
        TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    virtual bool ReplicateSubobjects(
        UActorChannel* Channel,
        FOutBunch* Bunch,
        FReplicationFlags* RepFlags) override;

    // 武器操作
    UFUNCTION(Server, Reliable, WithValidation)
    void ServerFire();

    UFUNCTION(Server, Reliable, WithValidation)
    void ServerReload();

protected:
    // 当前武器
    UPROPERTY(ReplicatedUsing=OnRep_CurrentWeapon)
    UMyWeaponObject* CurrentWeapon;

    UFUNCTION()
    void OnRep_CurrentWeapon();

    // 武器库存
    UPROPERTY()
    TArray<UMyWeaponObject*> WeaponInventory;
};
```

```cpp
// MyCharacter.cpp
AMyCharacter::AMyCharacter()
{
    bReplicates = true;

    // 创建初始武器
    CurrentWeapon = NewObject<UMyWeaponObject>(this);
    CurrentWeapon->WeaponName = TEXT("Pistol");
    CurrentWeapon->CurrentAmmo = 12;
    CurrentWeapon->MaxAmmo = 12;
    CurrentWeapon->Damage = 10.0f;
}

void AMyCharacter::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME_CONDITION_NOTIFY(
        AMyCharacter, CurrentWeapon,
        COND_None, REPNOTIFY_OnChanged
    );
}

bool AMyCharacter::ReplicateSubobjects(
    UActorChannel* Channel,
    FOutBunch* Bunch,
    FReplicationFlags* RepFlags)
{
    bool bWroteSomething = Super::ReplicateSubobjects(Channel, Bunch, RepFlags);

    // 复制当前武器
    if (CurrentWeapon)
    {
        bWroteSomething |= Channel->ReplicateSubobject(CurrentWeapon, *Bunch, *RepFlags);
    }

    // 复制库存中的武器
    for (UMyWeaponObject* Weapon : WeaponInventory)
    {
        if (Weapon)
        {
            bWroteSomething |= Channel->ReplicateSubobject(Weapon, *Bunch, *RepFlags);
        }
    }

    return bWroteSomething;
}

void AMyCharacter::OnRep_CurrentWeapon()
{
    // 更新武器UI
    UpdateWeaponUI();
}

void AMyCharacter::ServerFire_Implementation()
{
    if (CurrentWeapon && CurrentWeapon->CanFire())
    {
        CurrentWeapon->Fire();
        // 执行开火逻辑
    }
}

void AMyCharacter::ServerReload_Implementation()
{
    if (CurrentWeapon)
    {
        CurrentWeapon->Reload();
    }
}
```

---

## 四、组件复制

### 4.1 自动组件复制

ActorComponent默认支持网络复制：

```cpp
// MyComponent.h
UCLASS()
class UMyComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    UMyComponent();

    virtual void GetLifetimeReplicatedProps(
        TArray<FLifetimeProperty>& OutLifetimeProps) const override;

protected:
    UPROPERTY(Replicated)
    bool bIsActive;

    UPROPERTY(ReplicatedUsing=OnRep_Energy)
    float Energy;

    UFUNCTION()
    void OnRep_Energy();
};
```

```cpp
// MyComponent.cpp
UMyComponent::UMyComponent()
{
    // 启用组件复制
    SetIsReplicated(true);

    bIsActive = true;
    Energy = 100.0f;
}

void UMyComponent::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(UMyComponent, bIsActive);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyComponent, Energy, COND_None, REPNOTIFY_OnChanged);
}
```

### 4.2 组件复制选项

```cpp
// 在Actor中控制组件复制
AMyActor::AMyActor()
{
    // 创建组件
    MyComponent = CreateDefaultSubobject<UMyComponent>(TEXT("MyComponent"));

    // 启用组件复制
    MyComponent->SetIsReplicated(true);
}
```

---

## 五、注册列表方式（推荐）

### 5.1 新的子对象复制方式

UE5引入了注册列表方式，更高效：

```cpp
// MyActor.h
UCLASS()
class AMyActor : public AActor
{
    GENERATED_BODY()

public:
    AMyActor();

    // 不需要重写ReplicateSubobjects
    // 使用注册列表自动处理

protected:
    // 启用注册列表
    // 在构造函数中设置 bReplicateUsingRegisteredSubObjectList = true

    UPROPERTY(Replicated)
    UMySubobject* MySubobject;
};
```

```cpp
// MyActor.cpp
AMyActor::AMyActor()
{
    bReplicates = true;

    // 启用注册列表方式
    bReplicateUsingRegisteredSubObjectList = true;

    // 创建子对象
    MySubobject = NewObject<UMySubobject>(this);

    // 注册子对象
    AddReplicatedSubObject(MySubobject);
}

// 删除时注销
void AMyActor::RemoveSubobject()
{
    if (MySubobject)
    {
        RemoveReplicatedSubObject(MySubobject);
        MySubobject = nullptr;
    }
}
```

### 5.2 注册列表API

```cpp
// 注册子对象
void AddReplicatedSubObject(UObject* Subobject, ELifetimeCondition Condition = COND_None);

// 移除子对象
void RemoveReplicatedSubObject(UObject* Subobject);

// 检查是否已注册
bool IsSubObjectReplicated(UObject* Subobject) const;
```

### 5.3 条件控制

```cpp
// 根据条件注册子对象
AddReplicatedSubObject(SecretSubobject, COND_OwnerOnly);
AddReplicatedSubObject(PublicSubobject, COND_None);
AddReplicatedSubObject(ViewerSubobject, COND_SkipOwner);
```

---

## 六、子对象生命周期

### 6.1 创建流程

```
服务器端:
    NewObject<UObject>(OwnerActor)
        │
        ├── 设置Owner
        │
        ├── 注册复制
        │   └── AddReplicatedSubObject()
        │
        └── 下次复制时发送

客户端:
    接收子对象创建信息
        │
        ├── 读取类信息
        │
        ├── 创建本地实例
        │   └── NewObject<UObject>()
        │
        └── 注册到ActorChannel
```

### 6.2 销毁流程

```
服务器端:
    移除注册
        │
        ├── RemoveReplicatedSubObject()
        │
        ├── ConditionalBeginDestroy()
        │
        └── 发送销毁消息

客户端:
    接收销毁消息
        │
        ├── 从ActorChannel移除
        │
        └── 销毁本地对象
```

### 6.3 正确销毁示例

```cpp
void AMyCharacter::DestroyWeapon(UMyWeaponObject* Weapon)
{
    if (!HasAuthority())
        return;

    // 1. 从注册列表移除
    RemoveReplicatedSubObject(Weapon);

    // 2. 从引用中移除
    if (CurrentWeapon == Weapon)
    {
        CurrentWeapon = nullptr;
    }
    WeaponInventory.Remove(Weapon);

    // 3. 销毁对象
    Weapon->ConditionalBeginDestroy();
}
```

---

## 七、子对象RPC

### 7.1 子对象中的RPC

```cpp
UCLASS()
class UMySubobject : public UObject
{
    GENERATED_BODY()

public:
    // 子对象也可以有RPC
    UFUNCTION(Server, Reliable, WithValidation)
    void ServerDoSomething();

    UFUNCTION(Client, Reliable)
    void ClientNotifyChange();

    UFUNCTION(NetMulticast, Unreliable)
    void MulticastEffect();
};
```

### 7.2 RPC执行

```cpp
void UMySubobject::ServerDoSomething_Implementation()
{
    // 在服务器上执行
    // 子对象的RPC通过拥有它的Actor的连接发送
    UE_LOG(LogTemp, Log, TEXT("Server RPC called on subobject"));
}

// 调用
void AMyActor::DoSomething()
{
    if (MySubobject)
    {
        MySubobject->ServerDoSomething();
    }
}
```

---

## 八、调试与最佳实践

### 8.1 调试命令

```
; 显示子对象复制统计
stat net

; 详细日志
LogNetSubObject Verbose
```

### 8.2 最佳实践

```
┌─────────────────────────────────────────────────────────────────────┐
│                   子对象复制最佳实践                                  │
└─────────────────────────────────────────────────────────────────────┘

1. 使用注册列表方式
   bReplicateUsingRegisteredSubObjectList = true;
   AddReplicatedSubObject(Subobject);

2. 正确设置Owner
   NewObject<UObject>(OwnerActor);

3. 及时清理
   RemoveReplicatedSubObject() + ConditionalBeginDestroy();

4. 检查有效性
   if (Subobject && !Subobject->IsPendingKillPending())

5. 避免循环引用
   注意子对象间的引用关系
```

### 8.3 常见问题

| 问题         | 原因           | 解决方案                   |
| ------------ | -------------- | -------------------------- |
| 子对象不同步 | 未注册或未复制 | 确保AddReplicatedSubObject |
| 客户端崩溃   | 空指针访问     | 检查子对象有效性           |
| 内存泄漏     | 未正确销毁     | 移除注册并销毁             |
| RPC不执行    | 无NetOwner     | 确保子对象有正确的Owner    |

---

## 九、总结

### 9.1 核心要点

1. **子对象复制**: 复制Actor拥有的独立UObject
2. **ReplicateSubobjects**: 手动实现子对象复制
3. **注册列表**: UE5推荐的自动方式
4. **生命周期管理**: 创建注册、销毁注销
5. **组件复制**: ActorComponent内置支持

### 9.2 下一课预告

**第十课：网络性能优化**

将深入讲解：

- Push Model系统
- NetDormancy休眠优化
- 带宽管理

---

_上一课: [第八课：对象引用与NetGUID系统](./Lesson08_NetGUID.md)_
_下一课: [第十课：网络性能优化](./Lesson10_NetworkOptimization.md)_
