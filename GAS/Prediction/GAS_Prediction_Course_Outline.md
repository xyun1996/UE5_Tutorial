# GAS 预测系统课程大纲

> 10节课深入掌握 UE5.7 Gameplay Ability System 预测原理
> 每节课约 45-60 分钟

---

## 课程总览

| 课次 | 主题 | 核心内容 |
|-----|------|---------|
| 第1课 | GAS网络基础与预测概述 | 网络复制原理、预测的必要性 |
| 第2课 | FPredictionKey 核心结构 | 预测键的生成、序列化、生命周期 |
| 第3课 | Ability 激活预测流程 | TryActivateAbility 完整流程分析 |
| 第4课 | GameplayEffect 预测 | GE应用、属性修改、Tag修改的预测 |
| 第5课 | 属性预测与回滚机制 | 增量预测、RepNotify、回滚实现 |
| 第6课 | GameplayCue 预测 | GC事件预测、避免重复执行 |
| 第7课 | 预测窗口与 AbilityTask | FScopedPredictionWindow 深入 |
| 第8课 | 依赖链与链式激活 | Base Key、依赖管理、链式回滚 |
| 第9课 | 高级话题与限制 | 弱预测、Meta属性、百分比效果 |
| 第10课 | 实战案例与调试技巧 | Lyra分析、调试方法、最佳实践 |

---

## 第1课：GAS网络基础与预测概述

### 学习目标
- 理解客户端预测的概念和必要性
- 掌握 GAS 的基本网络架构
- 了解预测系统要解决的6个核心问题

### 课程内容

#### 1.1 为什么需要客户端预测？（15分钟）

```
无预测的情况：
┌────────────────────────────────────────────────────┐
│ 客户端按下技能键                                    │
│     ↓                                              │
│ 发送 RPC 到服务器（RTT ~50-100ms）                  │
│     ↓                                              │
│ 服务器验证并执行                                    │
│     ↓                                              │
│ 复制结果回客户端（~50-100ms）                       │
│     ↓                                              │
│ 客户端看到技能生效                                  │
│                                                    │
│ 总延迟：100-200ms，玩家感觉"卡顿"                   │
└────────────────────────────────────────────────────┘

有预测的情况：
┌────────────────────────────────────────────────────┐
│ 客户端按下技能键                                    │
│     ↓                                              │
│ 本地立即执行（0ms）                                 │
│     ↓                                              │
│ 同时发送 RPC 到服务器                               │
│     ↓                                              │
│ 服务器验证，结果与预测一致 → 无需额外处理            │
│ 服务器验证失败 → 回滚预测                           │
│                                                    │
│ 玩家体验：即时响应                                  │
└────────────────────────────────────────────────────┘
```

#### 1.2 GAS 网络架构概览（15分钟）

```
                    ┌─────────────────┐
                    │    服务器        │
                    │                 │
                    │  ASC (Authority)│
                    │       ↓         │
                    │  GameplayEffects│
                    │  Attributes     │
                    │  GameplayTags   │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              ↓              ↓              ↓
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │ 客户端 A  │   │ 客户端 B  │   │ 客户端 C  │
        │          │   │          │   │          │
        │ ASC      │   │ ASC      │   │ ASC      │
        │ (自主)   │   │ (代理)   │   │ (代理)   │
        │ ↓        │   │          │   │          │
        │ 预测执行  │   │ 仅接收   │   │ 仅接收   │
        └──────────┘   └──────────┘   └──────────┘
```

**关键概念**：
- **Authority（权威）**：服务器拥有状态的所有权
- **Autonomous Proxy（自主代理）**：玩家控制的客户端，可以预测
- **Simulated Proxy（模拟代理）**：其他客户端的复制，不能预测

#### 1.3 预测系统的6个核心问题（15分钟）

阅读 `GameplayPrediction.h:52-59`：

```cpp
// 问题1: "Can I do this?" - 基本预测协议
// 问题2: "Undo" - 如何回滚副作用
// 问题3: "Redo" - 如何避免重复执行
// 问题4: "Completeness" - 确保预测完整性
// 问题5: "Dependencies" - 管理依赖链
// 问题6: "Override" - 覆盖服务器状态
```

### 课后练习
1. 阅读 `GameplayPrediction.h` 第22-64行的概述注释
2. 使用 `net.PK.Draw` 控制台命令观察预测键

---

## 第2课：FPredictionKey 核心结构

### 学习目标
- 掌握 FPredictionKey 的数据结构
- 理解预测键的生成和序列化机制
- 了解预测键的生命周期

### 课程内容

#### 2.1 FPredictionKey 结构分析（20分钟）

**源码位置**：`GameplayPrediction.h:296-414`

```cpp
USTRUCT()
struct FPredictionKey
{
    GENERATED_USTRUCT_BODY()

    typedef int16 KeyType;

    // 核心成员
    UPROPERTY()
    int16 Current = 0;              // 唯一标识符

    UPROPERTY(NotReplicated)
    int16 Base = 0;                 // 依赖链的基础键

    UPROPERTY()
    bool bIsServerInitiated = false; // 是否服务器发起

    // 私有成员
    FObjectKey PredictiveConnectionObjectKey; // 网络连接标识
};
```

**关键设计决策**：

| 设计 | 原因 |
|-----|------|
| `Current` 使用 int16 | 节省带宽，65535个键足够单局游戏使用 |
| `Base` 不复制 | 依赖关系仅客户端维护，减少网络开销 |
| `PredictiveConnectionObjectKey` | 服务器识别预测键来源 |

#### 2.2 预测键的生成（15分钟）

**源码位置**：`GameplayPrediction.cpp`

```cpp
FPredictionKey FPredictionKey::CreateNewPredictionKey(const UAbilitySystemComponent* ASC)
{
    FPredictionKey NewKey;

    // 从 ASC 获取下一个键值
    NewKey.Current = ASC->GetNextPredictionKeyID();

    // 标记为客户端生成
    NewKey.bIsServerInitiated = false;

    return NewKey;
}

void FPredictionKey::GenerateDependentPredictionKey()
{
    // 保留当前键作为 Base
    Base = Current;

    // 生成新的 Current
    // 需要从 ASC 获取新的 ID
}
```

#### 2.3 预测键的序列化（20分钟）

**核心特性**：预测键只序列化给发起预测的客户端

```cpp
bool FPredictionKey::NetSerialize(FArchive& Ar, UPackageMap* Map, bool& bOutSuccess)
{
    if (Ar.IsSaving())
    {
        // 发送端
        Ar << Current;
        Ar << bIsServerInitiated;

        // 服务器记录此键来自哪个连接
        if (!bIsServerInitiated)
        {
            // 存储连接标识
        }
    }
    else
    {
        // 接收端
        Ar << Current;
        Ar << bIsServerInitiated;

        // 关键：如果不是发给自己的预测键，设为无效
        if (!bIsServerInitiated && !IsPredictingClient())
        {
            Current = 0; // 无效化
        }
    }

    return true;
}
```

**效果演示**：

```
场景：客户端A预测激活Ability

服务器复制GE时：
┌─────────────────────────────────────────────────┐
│ 客户端A：收到 PredictionKey = 10（有效）         │
│ 客户端B：收到 PredictionKey = 0（无效）          │
│ 客户端C：收到 PredictionKey = 0（无效）          │
└─────────────────────────────────────────────────┘

客户端A知道这是自己预测的，跳过重复执行
客户端B/C看到的是服务器权威状态，正常执行
```

### 课后练习
1. 在 `FPredictionKey::CreateNewPredictionKey` 设置断点，观察键的生成
2. 阅读 `FPredictionKey::NetSerialize` 完整实现

---

## 第3课：Ability 激活预测流程

### 学习目标
- 掌握 TryActivateAbility 的完整流程
- 理解客户端和服务器的交互过程
- 学会分析预测成功和失败的处理

### 课程内容

#### 3.1 客户端激活流程（20分钟）

**源码位置**：`AbilitySystemComponent_Abilities.cpp`

```
客户端 TryActivateAbility 流程：

Step 1: 检查是否可以激活
┌────────────────────────────────────────────────────┐
│ bool UAbilitySystemComponent::TryActivateAbility() │
│ {                                                  │
│     // 检查 AbilitySpec 是否有效                   │
│     // 检查冷却、Tag、Cost 等                       │
│     if (!CanActivateAbility()) return false;       │
│ }                                                  │
└────────────────────────────────────────────────────┘

Step 2: 生成预测键
┌────────────────────────────────────────────────────┐
│ // 创建新的预测键                                   │
│ FPredictionKey PredictionKey =                     │
│     FPredictionKey::CreateNewPredictionKey(this);  │
│                                                    │
│ // 开始预测窗口                                     │
│ FScopedPredictionWindow ScopedPred(this);          │
└────────────────────────────────────────────────────┘

Step 3: 发送 RPC 到服务器
┌────────────────────────────────────────────────────┐
│ ServerTryActivateAbility(AbilitySpecHandle,        │
│                          PredictionKey,            │
│                          ...);                     │
└────────────────────────────────────────────────────┘

Step 4: 本地预测执行
┌────────────────────────────────────────────────────┐
│ // 不等待服务器响应，立即本地执行                    │
│ ActivateAbility(AbilitySpecHandle,                 │
│                 PredictionKey,                     │
│                 ...);                              │
│                                                    │
│ // 所有副作用关联到 PredictionKey                   │
└────────────────────────────────────────────────────┘
```

#### 3.2 服务器处理流程（15分钟）

```cpp
// 服务器端 RPC 处理
void UAbilitySystemComponent::ServerTryActivateAbility_Implementation(
    FGameplayAbilitySpecHandle AbilityToActivate,
    FPredictionKey PredictionKey,
    ...)
{
    // 1. 设置预测窗口（使用客户端传来的键）
    FScopedPredictionWindow ScopedPred(this, PredictionKey);

    // 2. 验证是否可以激活
    if (!CanActivateAbility(AbilityToActivate))
    {
        // 拒绝：通知客户端回滚
        ClientActivateAbilityFailed(AbilityToActivate, PredictionKey);
        return;
    }

    // 3. 执行 Ability
    ActivateAbility(AbilityToActivate, PredictionKey, ...);

    // 4. 成功：设置复制预测键
    // ScopedPred 析构时会自动设置 ReplicatedPredictionKey
}
```

#### 3.3 客户端接收响应（15分钟）

**成功情况**：

```cpp
// 服务器复制状态追上时
void FReplicatedPredictionKeyItem::OnRep()
{
    // 触发 CatchUp 委托
    FPredictionKeyDelegates::BroadcastCaughtUpDelegate(PredictionKey.Current);

    // 清理预测的副作用
    // - 移除预测的 GE
    // - 保留服务器复制的 GE
}
```

**失败情况**：

```cpp
void UAbilitySystemComponent::ClientActivateAbilityFailed_Implementation(
    FGameplayAbilitySpecHandle AbilityToActivate,
    FPredictionKey PredictionKey)
{
    // 立即触发拒绝委托
    FPredictionKeyDelegates::BroadcastRejectedDelegate(PredictionKey.Current);

    // 回滚所有关联的副作用
    // - 移除预测的 GE
    // - 恢复属性值
    // - 移除预测的 Tag
    // - 停止预测的 Montage
}
```

### 课后练习
1. 在 `TryActivateAbility` 和 `ServerTryActivateAbility_Implementation` 设置断点
2. 构造一个会被服务器拒绝的 Ability，观察回滚过程

---

## 第4课：GameplayEffect 预测

### 学习目标
- 掌握 GE 应用的预测机制
- 理解 Duration GE 和 Instant GE 的预测差异
- 学会处理 GE 的 "Undo" 和 "Redo" 问题

### 课程内容

#### 4.1 GE 预测的基本原则（15分钟）

**源码位置**：`GameplayPrediction.h:95-111`

```cpp
// GE 是 Ability 激活的副作用，不单独确认/拒绝
// GE 的命运与激活它的 Ability 绑定

// 可预测的 GE 效果：
// - 属性修改（Attribute Modifiers）
// - GameplayTag 修改
// - GameplayCue 事件

// 不可预测的 GE 效果：
// - Execution 计算（Custom Execution）
// - 周期效果（Periodic Effects）
```

#### 4.2 Duration GE 的预测（20分钟）

```
预测 Duration GE 的流程：

客户端：
┌────────────────────────────────────────────────────┐
│ 1. ApplyGameplayEffectToSelf()                     │
│    └─→ 检查是否有有效的 PredictionKey               │
│                                                    │
│ 2. 创建 FActiveGameplayEffect                      │
│    └─→ 存储 PredictionKey                          │
│                                                    │
│ 3. 应用效果                                        │
│    └─→ 修改属性、添加 Tag                          │
│    └─→ 触发 GameplayCue                            │
└────────────────────────────────────────────────────┘

服务器：
┌────────────────────────────────────────────────────┐
│ 1. ApplyGameplayEffectToSelf()                     │
│    └─→ 使用相同的 PredictionKey                    │
│                                                    │
│ 2. 创建 FActiveGameplayEffect                      │
│    └─→ 存储 PredictionKey（会复制回客户端）         │
└────────────────────────────────────────────────────┘

客户端接收复制：
┌────────────────────────────────────────────────────┐
│ FActiveGameplayEffect::PostReplicatedAdd()         │
│ {                                                  │
│     if (PredictionKey.IsValid())                   │
│     {                                              │
│         // 这是自己预测的，跳过 OnApplied 逻辑      │
│         // 解决 "Redo" 问题                        │
│         return;                                    │
│     }                                              │
│     // 正常处理服务器 GE                            │
│ }                                                  │
└────────────────────────────────────────────────────┘
```

**关键代码**：`GameplayEffect.cpp`

```cpp
void FActiveGameplayEffect::PostReplicatedAdd(const FActiveGameplayEffectsContainer& InArray)
{
    // 检查是否是自己预测的
    if (PredictionKey.IsLocalClientKey())
    {
        // 跳过 GameplayCue 的 OnAdded 事件
        // 避免重复播放特效
        return;
    }

    // 正常处理服务器 GE
    InArray.Owner->ExecuteGameplayCue(OnAddedCue);
}
```

#### 4.3 Instant GE 的预测（15分钟）

**特殊处理**：Instant GE 没有持续时间，无法直接存储 PredictionKey

**解决方案**：将预测的 Instant GE 转换为无限持续 GE

```cpp
// 源码位置：AbilitySystemComponent.cpp

void UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf(...)
{
    if (PredictionKey.IsValid())
    {
        if (EffectSpec.GetDuration() == UGameplayEffect::INSTANT_APPLICATION)
        {
            // 即时效果预测时转换为无限持续
            // 这样才能在预测失败时回滚
            EffectSpec.ConvertToInfiniteDuration();
        }
    }

    // 正常应用 GE...
}
```

### 课后练习
1. 在 `FActiveGameplayEffect::PostReplicatedAdd` 设置断点
2. 创建一个 Duration GE，观察预测和复制的完整流程

---

## 第5课：属性预测与回滚机制

### 学习目标
- 掌握属性增量预测的原理
- 理解 RepNotify 在属性预测中的作用
- 学会实现自定义属性的预测支持

### 课程内容

#### 5.1 属性预测的挑战（15分钟）

**源码位置**：`GameplayPrediction.h:114-124`

```
问题1：属性是标准 UProperty 复制
┌────────────────────────────────────────────────────┐
│ 客户端预测：Health = 90                            │
│ 服务器复制：Health = 90                            │
│                                                    │
│ 问题：RepNotify 不会触发（值相同）                  │
│ 客户端不知道服务器已确认                           │
└────────────────────────────────────────────────────┘

问题2：即时修改无状态
┌────────────────────────────────────────────────────┐
│ 客户端预测：Health -= 10                           │
│ 服务器拒绝：...                                    │
│                                                    │
│ 问题：如何恢复 -10 的修改？                        │
│ 没有记录修改历史                                   │
└────────────────────────────────────────────────────┘
```

#### 5.2 增量预测方案（20分钟）

**核心思想**：不预测绝对值，预测增量

```
传统方式（错误）：
┌────────────────────────────────────────────────────┐
│ 服务器基础值：Health = 100                         │
│ 客户端预测：Health = 90                            │
│                                                    │
│ 服务器复制 Health = 90 时：                        │
│   客户端本地已经是 90，不触发 RepNotify             │
│   无法区分是预测还是确认                           │
└────────────────────────────────────────────────────┘

增量预测方式（正确）：
┌────────────────────────────────────────────────────┐
│ 服务器基础值：Health = 100                         │
│                                                    │
│ 客户端预测修改：-10                                │
│   └─→ 创建一个"无限持续"GE：HealthMod = -10        │
│   └─→ 最终值 = 100 + (-10) = 90                    │
│                                                    │
│ 服务器确认后复制：Health = 90                       │
│   └─→ RepNotify 触发（使用 REPNOTIFY_Always）      │
│   └─→ 移除预测 GE                                  │
│   └─→ 最终值 = 90 + 0 = 90                         │
└────────────────────────────────────────────────────┘
```

#### 5.3 实现属性预测支持（15分钟）

**Step 1：配置属性复制**

```cpp
// AttributeSet.h

void UMyHealthSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 关键：使用 REPNOTIFY_Always
    // 确保每次复制都触发回调，即使本地值相同
    DOREPLIFETIME_CONDITION_NOTIFY(UMyHealthSet, Health, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyHealthSet, MaxHealth, COND_None, REPNOTIFY_Always);
}
```

**Step 2：实现 RepNotify**

```cpp
void UMyHealthSet::OnRep_Health()
{
    // 这个宏会：
    // 1. 更新基础值（服务器复制的值）
    // 2. 重新计算最终值（基础值 + 修改器）
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyHealthSet, Health);
}

void UMyHealthSet::OnRep_MaxHealth()
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyHealthSet, MaxHealth);
}
```

**Step 3：理解 GAMEPLAYATTRIBUTE_REPNOTIFY 宏**

```cpp
// 展开后的逻辑
#define GAMEPLAYATTRIBUTE_REPNOTIFY(ClassName, PropertyName) \
{ \
    GameEffectAttributePropertyReplicatedReceived( \
        Get##PropertyName##Attribute(), \
        PropertyName); \
}

// 内部实现
void UAttributeSet::GameEffectAttributePropertyReplicatedReceived(
    FGameplayAttribute Attribute,
    float& NewValue)
{
    // 1. 设置新的基础值
    SetBaseAttributeValueFromReplication(Attribute, NewValue);

    // 2. 重新计算最终值
    // 最终值 = 基础值 + 所有修改器
    float FinalValue = CalculateFinalValue(Attribute);

    // 3. 更新当前值
    Attribute.SetNumericValueChecked(FinalValue, this);
}
```

### 课后练习
1. 创建自定义 AttributeSet，实现属性预测支持
2. 使用网络分析工具观察属性复制过程

---

## 第6课：GameplayCue 预测

### 学习目标
- 掌握 GameplayCue 的预测机制
- 理解 GC 的 "Redo" 问题解决方案
- 学会正确使用 ExecuteGameplayCue

### 课程内容

#### 6.1 GameplayCue 概述（10分钟）

```
GameplayCue 类型：
┌────────────────────────────────────────────────────┐
│ GameplayCueNotify_Actor                            │
│   └─→ 持久化效果（光环、持续特效）                  │
│                                                    │
│ GameplayCueNotify_Static                           │
│   └─→ 一次性效果（打击特效、声音）                  │
└────────────────────────────────────────────────────┘

触发时机：
- OnActive：GE 应用时
- OnRemove：GE 移除时
- Execute：即时执行（不关联 GE）
```

#### 6.2 GC 预测流程（20分钟）

**源码位置**：`GameplayPrediction.h:146-153`

```cpp
// 从 GE 内部触发的 GC
void UAbilitySystemComponent::ExecuteGameplayCueFromGameplayEffect(...)
{
    if (IsNetAuthority())
    {
        // 服务器：多播到所有客户端
        NetMulticast_InvokeGameplayCueExecuted(..., PredictionKey);
    }
    else if (PredictionKey.IsValid())
    {
        // 客户端预测：本地执行
        InvokeGameplayCueExecuted(...);
    }
}

// 独立执行的 GC
void UAbilitySystemComponent::ExecuteGameplayCue(FGameplayTag CueTag, ...)
{
    if (IsNetAuthority())
    {
        // 服务器：多播
        NetMulticast_InvokeGameplayCueExecuted(...);
    }
    else if (ScopedPredictionKey.IsValid())
    {
        // 客户端预测
        InvokeGameplayCueExecuted(...);
    }
}
```

#### 6.3 解决 "Redo" 问题（15分钟）

```cpp
// 服务器多播
void UAbilitySystemComponent::NetMulticast_InvokeGameplayCueExecuted_Implementation(
    FGameplayTag CueTag,
    FPredictionKey PredictionKey,
    ...)
{
    // 关键检查
    if (PredictionKey.IsLocalClientKey())
    {
        // 这是自己预测的 GC，跳过
        // 避免重复播放
        return;
    }

    // 正常执行 GC
    InvokeGameplayCueExecuted(CueTag, ...);
}
```

**流程图**：

```
场景：客户端预测播放打击特效

┌─────────────────────────────────────────────────────┐
│ 客户端：                                            │
│   ExecuteGameplayCue(HitCue, PredictionKey=10)      │
│   └─→ 本地播放特效 ✓                                │
│                                                     │
│ 服务器：                                            │
│   NetMulticast_InvokeGameplayCueExecuted(           │
│       HitCue, PredictionKey=10)                     │
│                                                     │
│ 客户端接收多播：                                     │
│   PredictionKey=10 是自己的                         │
│   └─→ 跳过，不重复播放 ✓                            │
│                                                     │
│ 其他客户端接收多播：                                  │
│   PredictionKey=0（无效）                           │
│   └─→ 正常播放特效 ✓                                │
└─────────────────────────────────────────────────────┘
```

### 课后练习
1. 创建自定义 GameplayCue，观察预测和复制流程
2. 在 `NetMulticast_InvokeGameplayCueExecuted` 设置断点

---

## 第7课：预测窗口与 AbilityTask

### 学习目标
- 掌握 FScopedPredictionWindow 的使用
- 理解 AbilityTask 中的预测机制
- 学会在 Ability 中创建新的预测窗口

### 课程内容

#### 7.1 预测窗口的概念（15分钟）

**问题**：预测键只在单次函数调用内有效

```cpp
void UGameplayAbility::ActivateAbility(...)
{
    // 预测键在此有效

    // 问题：如果 Ability 等待外部事件
    // 如 WaitInputRelease、WaitTargetData 等
    // 此时预测窗口已结束

    // 等待后执行的代码无法使用原预测键
}
```

**解决**：FScopedPredictionWindow

```cpp
struct FScopedPredictionWindow
{
    // 客户端：生成新的预测键
    FScopedPredictionWindow(UAbilitySystemComponent* ASC, bool CanGenerateNewKey = true);

    // 服务器：使用客户端传来的预测键
    FScopedPredictionWindow(UAbilitySystemComponent* ASC, FPredictionKey Key, bool SetReplicatedKey = true);

    ~FScopedPredictionWindow(); // 窗口结束时确认预测键
};
```

#### 7.2 AbilityTask 中的预测窗口（20分钟）

**案例**：`UAbilityTask_WaitInputRelease`

```cpp
// 客户端
void UAbilityTask_WaitInputRelease::OnReleaseCallback()
{
    // 1. 创建新的预测窗口
    FScopedPredictionWindow ScopedPred(AbilitySystemComponent);

    // 2. 发送 RPC，携带新的预测键
    AbilitySystemComponent->ServerInputRelease(ScopedPred.ScopedPredictionKey);

    // 3. 此作用域内的副作用使用新预测键
    // ... 执行逻辑 ...

    // 4. 窗口结束，预测键被确认
}

// 服务器
void UAbilitySystemComponent::ServerInputRelease_Implementation(FPredictionKey PredictionKey)
{
    // 1. 使用客户端传来的预测键
    FScopedPredictionWindow ScopedPred(this, PredictionKey);

    // 2. 在此作用域内执行逻辑
    // 所有副作用关联到客户端的预测键

    // 3. 窗口结束，设置 ReplicatedPredictionKey
}
```

#### 7.3 更多 AbilityTask 示例（15分钟）

**WaitTargetData**：

```cpp
void UAbilityTask_WaitTargetData::OnTargetDataReadyCallback(const FGameplayAbilityTargetDataHandle& Data)
{
    // 目标数据确认时创建新预测窗口
    FScopedPredictionWindow ScopedPred(AbilitySystemComponent);

    // 发送目标数据到服务器
    AbilitySystemComponent->ServerSetTargetData(Data, ScopedPred.ScopedPredictionKey);

    // 继续执行 Ability 逻辑
}
```

**WaitNetSync**：

```cpp
void UAbilityTask_WaitNetSync::OnSignalCallback()
{
    // 网络同步点
    FScopedPredictionWindow ScopedPred(AbilitySystemComponent);

    // 确保客户端和服务器状态同步
}
```

### 课后练习
1. 阅读 `UAbilityTask_WaitInputRelease` 完整实现
2. 创建自定义 AbilityTask，使用预测窗口

---

## 第8课：依赖链与链式激活

### 学习目标
- 掌握预测键依赖链的原理
- 理解链式激活的回滚机制
- 学会设计可靠的链式 Ability

### 课程内容

#### 8.1 依赖链问题（15分钟）

**源码位置**：`GameplayPrediction.h:170-189`

```
场景：Ability X → 触发 → Ability Y → 触发 → Ability Z

┌─────────────────────────────────────────────────────┐
│ 问题：                                              │
│   如果 Y 被服务器拒绝，Z 也应该无效                  │
│   但服务器不知道 Z 的存在（Z 是客户端预测激活的）    │
│                                                     │
│ 服务器只收到：                                       │
│   - ServerTryActivateAbility(X, Key=10)            │
│   - ServerTryActivateAbility(Y, Key=10)            │
│   - ServerTryActivateAbility(Z, Key=10)            │
│                                                     │
│ 服务器不知道 Y 和 Z 之间的依赖关系                   │
└─────────────────────────────────────────────────────┘
```

#### 8.2 Base PredictionKey 机制（20分钟）

```cpp
// 客户端生成依赖键

// Ability X
FPredictionKey KeyX = FPredictionKey::CreateNewPredictionKey(ASC);
// KeyX = { Current: 10, Base: 0 }

// Ability Y（在 X 内触发）
FPredictionKey KeyY = KeyX;
KeyY.GenerateDependentPredictionKey();
// KeyY = { Current: 11, Base: 10 }

// Ability Z（在 Y 内触发）
FPredictionKey KeyZ = KeyY;
KeyZ.GenerateDependentPredictionKey();
// KeyZ = { Current: 12, Base: 10 }
```

**依赖关系注册**：

```cpp
void FPredictionKeyDelegates::AddDependency(FPredictionKey::KeyType ThisKey, FPredictionKey::KeyType DependsOn)
{
    // 注册依赖关系
    // 当 DependsOn 被拒绝时，ThisKey 也会被拒绝

    FDelegates& ParentDelegates = DelegateMap.FindOrAdd(DependsOn);
    FDelegates& ChildDelegates = DelegateMap.FindOrAdd(ThisKey);

    // 当父键被拒绝时，也拒绝子键
    ParentDelegates.RejectedDelegates.Add([ThisKey]() {
        BroadcastRejectedDelegate(ThisKey);
    });
}
```

#### 8.3 设计可靠的链式 Ability（15分钟）

**使用 GameplayTag 确保依赖**：

```cpp
// GA_Combo1
UCLASS()
class UGA_Combo1 : public UGameplayAbility
{
    // 成功时给予 Tag
    UPROPERTY(EditDefaultsOnly)
    FGameplayTag Combo1SuccessTag;
};

// GA_Combo2
UCLASS()
class UGA_Combo2 : public UGameplayAbility
{
    // 需要 GA_Combo1 的 Tag 才能激活
    UPROPERTY(EditDefaultsOnly)
    FGameplayTagContainer ActivationRequiredTags;
};
```

**服务器端验证**：

```cpp
bool UGA_Combo2::CanActivateAbility(...)
{
    // 检查是否有前置 Tag
    if (!OwnerASC->HasAllMatchingGameplayTags(ActivationRequiredTags))
    {
        return false;
    }

    return Super::CanActivateAbility(...);
}
```

### 课后练习
1. 创建一个简单的连击系统，测试依赖链
2. 在 `FPredictionKeyDelegates::AddDependency` 设置断点

---

## 第9课：高级话题与限制

### 学习目标
- 了解当前预测系统的限制
- 理解弱预测的概念
- 学会处理边界情况

### 课程内容

#### 9.1 当前不可预测的内容（15分钟）

**源码位置**：`GameplayPrediction.h:47-49, 218-247`

```cpp
// 不可预测：
// 1. GameplayEffect 移除
// 2. GameplayEffect 周期效果（DOT）
// 3. Meta 属性（Damage/Healing）
// 4. 百分比乘法效果
```

**原因分析**：

| 限制 | 原因 |
|-----|------|
| GE 移除 | 需要追踪 GE 的完整历史 |
| 周期效果 | 时间同步复杂，丢包难以处理 |
| Meta 属性 | 只支持即时效果，无状态存储 |
| 百分比效果 | 缺少聚合器链的完整复制 |

#### 9.2 Meta 属性问题（15分钟）

**源码位置**：`GameplayPrediction.h:229-234`

```
Meta 属性特点：
- 只在即时 GE 中使用
- 在 AttributeSet::Pre/PostModifyAttribute 中处理
- 用于转换（如 Damage → Health）

问题：
- 预测时将即时 GE 转为无限持续
- 但 Meta 属性的转换逻辑只在即时 GE 中触发
- 预测的无限持续 GE 不会触发 Meta 属性转换
```

**解决方案**（源码建议）：

```cpp
// 可能需要添加对持续 Meta 属性的有限支持
// 将转换逻辑从前端移到后端（AttributeSet::PostModifyAttribute）
```

#### 9.3 百分比效果预测问题（15分钟）

**源码位置**：`GameplayPrediction.h:237-247`

```
示例：
┌─────────────────────────────────────────────────────┐
│ 客户端状态：                                        │
│   基础移动速度 = 500                                │
│   已有 +10% buff = 550 最终速度                     │
│                                                     │
│ 预测另一个 +10% buff：                              │
│   错误计算：550 * 1.1 = 605                         │
│   正确计算：500 * 1.2 = 600                         │
│                                                     │
│ 原因：客户端不知道完整的修改器链                     │
└─────────────────────────────────────────────────────┘
```

**未来改进**：复制完整的聚合器链

#### 9.4 弱预测概念（10分钟）

**源码位置**：`GameplayPrediction.h:250-263`

```
场景：碰撞触发的 GameplayEffect
- 无法发送 RPC 每次碰撞
- 服务器无法处理大量消息
- 无法建立预测键关联

弱预测方案：
- 不使用新鲜预测键
- 服务器假设客户端会预测所有副作用
- 只解决 "Redo" 问题
- 不解决 "Completeness" 问题

实现方式：
- 只预测 GameplayCue Execute 事件
- 不预测 OnAdded/OnRemove 事件
- 最小化客户端预测内容
```

### 课后练习
1. 思考如何在项目中规避这些限制
2. 阅读 `GameplayPrediction.h` 的 "Unsupported" 部分

---

## 第10课：实战案例与调试技巧

### 学习目标
- 分析 Lyra 项目中的 GAS 预测实现
- 掌握预测系统的调试方法
- 学习最佳实践

### 课程内容

#### 10.1 Lyra 项目分析（20分钟）

**文件位置**：`Samples/Games/Lyra/Source/LyraGame/AbilitySystem/`

```cpp
// LyraAbilitySystemComponent.h
class ULyraAbilitySystemComponent : public UAbilitySystemComponent
{
    // 自定义能力激活逻辑
    virtual void TryActivateAbility(...) override;

    // 输入处理
    void Input_AbilityInputTagPressed(FGameplayTag InputTag);
    void Input_AbilityInputTagReleased(FGameplayTag InputTag);
};

// LyraGameplayAbility.h
class ULyraGameplayAbility : public UGameplayAbility
{
    // Lyra 特有的能力配置
    UPROPERTY(EditDefaultsOnly, Category = "Lyra")
    FGameplayTagContainer StartupInputTags;
};
```

**关键实现**：

```cpp
// Lyra 的输入处理
void ULyraAbilitySystemComponent::Input_AbilityInputTagPressed(FGameplayTag InputTag)
{
    // 查找匹配输入标签的 AbilitySpec
    for (FGameplayAbilitySpec& Spec : ActivatableAbilities.Items)
    {
        if (Spec.DynamicAbilityTags.HasTagExact(InputTag))
        {
            if (Spec.Ability)
            {
                // 尝试激活，会自动创建预测键
                TryActivateAbility(Spec.Handle);
            }
        }
    }
}
```

#### 10.2 调试技巧（20分钟）

**断点位置**：

| 函数 | 用途 |
|-----|------|
| `FPredictionKey::CreateNewPredictionKey` | 预测键生成 |
| `FPredictionKeyDelegates::BroadcastRejectedDelegate` | 预测拒绝 |
| `FPredictionKeyDelegates::BroadcastCaughtUpDelegate` | 服务器确认 |
| `FScopedPredictionWindow::~FScopedPredictionWindow` | 窗口结束 |
| `FActiveGameplayEffect::PostReplicatedAdd` | GE 复制 |
| `UAttributeSet::GameEffectAttributePropertyReplicatedReceived` | 属性复制 |

**日志输出**：

```cpp
// 在 GameplayPrediction.cpp 添加
DEFINE_LOG_CATEGORY_STATIC(LogGameplayPrediction, Log, All);

FPredictionKey FPredictionKey::CreateNewPredictionKey(...)
{
    UE_LOG(LogGameplayPrediction, Log, TEXT("Creating PredictionKey: %d"), NewKey.Current);
    return NewKey;
}

void FPredictionKeyDelegates::BroadcastRejectedDelegate(FPredictionKey::KeyType Key)
{
    UE_LOG(LogGameplayPrediction, Warning, TEXT("PredictionKey Rejected: %d"), Key);
    // ...
}
```

**控制台命令**：

```
// 网络统计
stat net

// 预测键调试
net.PK.Draw 1

// 属性复制调试
net.Replication.Debug 1

// GAS 调试
AbilitySystem.Debug 1
```

#### 10.3 最佳实践（15分钟）

**1. 属性配置**

```cpp
// 始终使用 REPNOTIFY_Always
DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, Health, COND_None, REPNOTIFY_Always);

// 始终使用 GAMEPLAYATTRIBUTE_REPNOTIFY
void OnRep_Health() { GAMEPLAYATTRIBUTE_REPNOTIFY(UMyAttributeSet, Health); }
```

**2. Ability 设计**

```cpp
// 保持 ActivateAbility 同步执行
// 避免在预测窗口内使用 Timer 或 Latent Action

void UMyAbility::ActivateAbility(...)
{
    // ✓ 好的做法：同步执行
    ApplyGameplayEffectToSelf(DamageGE);
    PlayMontage(AttackMontage);

    // ✗ 避免：异步操作
    // SetTimer(...);  // 预测窗口已结束
}

// 需要异步时使用 AbilityTask + 新预测窗口
void UMyAbility::ActivateAbility(...)
{
    // 初始预测窗口
    ApplyGameplayEffectToSelf(InitialGE);

    // 等待输入释放
    UAbilityTask_WaitInputRelease* Task = UAbilityTask_WaitInputRelease::WaitInputRelease(this);
    Task->OnRelease.AddDynamic(this, &UMyAbility::OnInputReleased);
    Task->ReadyForActivation();
}

void UMyAbility::OnInputReleased()
{
    // 新的预测窗口在 Task 内部创建
    ApplyGameplayEffectToSelf(FinalGE);
}
```

**3. 依赖链设计**

```cpp
// 使用 Tag 确保依赖关系
UCLASS()
class UGA_ComboAttack : public UGameplayAbility
{
    // 前置条件
    UPROPERTY(EditDefaultsOnly, Category = "Combo")
    FGameplayTagContainer RequiredComboTags;

    // 成功时给予的 Tag
    UPROPERTY(EditDefaultsOnly, Category = "Combo")
    FGameplayTag GrantedComboTag;
};
```

**4. 错误处理**

```cpp
// 监听预测失败
void UMyAbility::ActivateAbility(...)
{
    // 注册预测失败回调
    if (GetCurrentPredictionKey().IsValid())
    {
        FPredictionKeyDelegates::NewRejectedDelegate(GetCurrentPredictionKey().Current).AddLambda([this]() {
            // 预测失败时的清理
            CleanupPredictedState();
        });
    }
}
```

### 课后练习
1. 使用 Lyra 项目运行多人游戏，观察预测系统工作
2. 添加日志输出，绘制完整的预测流程图

---

## 附录

### A. 关键源码文件索引

| 文件 | 行号 | 内容 |
|-----|------|------|
| `GameplayPrediction.h` | 22-264 | 完整设计文档 |
| `GameplayPrediction.h` | 296-414 | FPredictionKey 定义 |
| `GameplayPrediction.h` | 435-467 | FPredictionKeyDelegates |
| `GameplayPrediction.h` | 479-504 | FScopedPredictionWindow |
| `AbilitySystemComponent.h` | 108+ | UAbilitySystemComponent |
| `GameplayEffect.h` | - | FActiveGameplayEffect |

### B. 推荐阅读顺序

1. `GameplayPrediction.h` 第22-264行（设计文档）
2. `GameplayPrediction.cpp`（实现细节）
3. `AbilitySystemComponent_Abilities.cpp`（激活流程）
4. `GameplayEffect.cpp`（GE 预测）
5. Lyra 示例项目

### C. 常见问题

**Q: 为什么我的属性预测不生效？**
A: 检查是否使用了 `REPNOTIFY_Always` 和 `GAMEPLAYATTRIBUTE_REPNOTIFY`

**Q: 为什么 GameplayCue 重复播放？**
A: 检查是否正确传递了 PredictionKey

**Q: 如何调试预测失败？**
A: 在 `BroadcastRejectedDelegate` 设置断点，查看调用栈
