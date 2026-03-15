# GAS 预测系统学习指南

> 基于 UE5.7 源码分析
> 核心文件：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayPrediction.h`

---

## 目录

1. [核心文件阅读顺序](#核心文件阅读顺序)
2. [核心概念理解](#核心概念理解)
3. [关键数据结构](#关键数据结构)
4. [学习路径建议](#学习路径建议)
5. [当前限制](#当前限制)

---

## 核心文件阅读顺序

| 顺序 | 文件 | 重点内容 |
|-----|------|---------|
| **1** | `GameplayPrediction.h` (第22-264行) | **必读！** 完整的预测系统设计文档 |
| **2** | `GameplayPrediction.cpp` | 预测键的实现细节 |
| **3** | `AbilitySystemComponent.h` | `ScopedPredictionKey` 成员和预测窗口管理 |
| **4** | `AbilitySystemComponent_Abilities.cpp` | TryActivateAbility 的预测流程实现 |
| **5** | `GameplayEffect.cpp` | GE应用时的预测处理 |

### 文件路径

```
Engine/Plugins/Runtime/GameplayAbilities/
├── Source/GameplayAbilities/
│   ├── Public/
│   │   ├── GameplayPrediction.h          # 核心预测系统
│   │   ├── AbilitySystemComponent.h      # ASC主类
│   │   ├── GameplayEffect.h              # GE定义
│   │   └── Abilities/
│   │       └── GameplayAbility.h         # Ability基类
│   └── Private/
│       ├── GameplayPrediction.cpp        # 预测实现
│       ├── AbilitySystemComponent.cpp
│       ├── AbilitySystemComponent_Abilities.cpp
│       └── Tests/
│           └── PredictionKeyTests.cpp    # 单元测试
```

---

## 核心概念理解

根据源码注释，预测系统解决 **6个核心问题**：

### 1. "Can I do this?" - 基本预测协议

客户端能否执行某个操作？通过预测键与服务器通信确认。

### 2. "Undo" - 回滚机制

预测失败时如何回滚副作用？
- 通过 `FPredictionKeyDelegates` 注册回滚逻辑
- 当收到 `ClientActivateAbilityFailed` 时立即执行回滚

### 3. "Redo" - 避免重复执行

如何避免重放本地已预测但服务器也复制的副作用？
- 预测键只复制给发起预测的客户端
- 收到带预测键的复制数据时，跳过"OnApplied"逻辑

### 4. "Completeness" - 完整性

确保真正预测了所有副作用。
- 预测窗口内的所有副作用关联同一个预测键
- 窗口结束后预测键失效

### 5. "Dependencies" - 依赖管理

管理依赖预测和预测事件链。
- 使用 `Base` 键建立依赖关系
- 父键被拒绝时，子键自动无效

### 6. "Override" - 状态覆盖

如何预测性地覆盖服务器拥有的状态？
- 属性使用增量预测而非绝对值预测
- 使用 `REPNOTIFY_Always` 确保每次复制都触发回调

---

## 关键数据结构

### FPredictionKey (预测键)

```cpp
USTRUCT()
struct FPredictionKey
{
    GENERATED_USTRUCT_BODY()

    typedef int16 KeyType;

    // 唯一ID
    UPROPERTY()
    int16 Current = 0;

    // 依赖链的基础键（非复制）
    UPROPERTY(NotReplicated)
    int16 Base = 0;

    // 是否服务器发起
    UPROPERTY()
    bool bIsServerInitiated = false;

    // 关键方法
    static FPredictionKey CreateNewPredictionKey(const UAbilitySystemComponent*);
    static FPredictionKey CreateNewServerInitiatedKey(const UAbilitySystemComponent*);
    void GenerateDependentPredictionKey();
    bool IsValidKey() const;
    bool IsLocalClientKey() const;
};
```

**重要特性**：
- 预测键**只复制给发起预测的客户端**（通过 `NetSerialize` 实现）
- 其他客户端收到的是无效键（Current = 0）

### FScopedPredictionWindow (预测窗口)

```cpp
struct FScopedPredictionWindow
{
    // 服务器端：接收客户端传来的预测键
    FScopedPredictionWindow(UAbilitySystemComponent* ASC, FPredictionKey InKey, bool InSetReplicatedPredictionKey = true);

    // 客户端：生成新的预测键
    FScopedPredictionWindow(UAbilitySystemComponent* ASC, bool CanGenerateNewKey = true);

    ~FScopedPredictionWindow(); // 窗口结束时确认预测键
};
```

**用途**：
- 控制预测键的生命周期
- 在作用域内有效，超出作用域预测键失效
- 用于 AbilityTask 中创建新的预测窗口

### FPredictionKeyDelegates (回调委托)

```cpp
struct FPredictionKeyDelegates
{
    struct FDelegates
    {
        // 预测被拒绝时的回滚逻辑
        TArray<FPredictionKeyEvent> RejectedDelegates;

        // 服务器状态追上时的清理逻辑
        TArray<FPredictionKeyEvent> CaughtUpDelegates;
    };

    TMap<FPredictionKey::KeyType, FDelegates> DelegateMap;

    // 注册回调
    static FPredictionKeyEvent& NewRejectedDelegate(FPredictionKey::KeyType Key);
    static FPredictionKeyEvent& NewCaughtUpDelegate(FPredictionKey::KeyType Key);

    // 触发回调
    static void BroadcastRejectedDelegate(FPredictionKey::KeyType Key);
    static void BroadcastCaughtUpDelegate(FPredictionKey::KeyType Key);

    // 建立依赖
    static void AddDependency(FPredictionKey::KeyType ThisKey, FPredictionKey::KeyType DependsOn);
};
```

### FReplicatedPredictionKeyMap (复制映射)

```cpp
USTRUCT()
struct FReplicatedPredictionKeyMap : public FFastArraySerializer
{
    UPROPERTY()
    TArray<FReplicatedPredictionKeyItem> PredictionKeys;

    void ReplicatePredictionKey(FPredictionKey Key);
};
```

**为什么用 FastArray 而非单个整数？**
- 避免丢包导致的状态不一致
- 每个预测键单独确认，而非"最高编号键"

---

## 学习路径建议

### 第一阶段：理解 Ability Activation 预测流程

**源码位置**：`GameplayPrediction.h:73-92`

```
客户端流程：
┌─────────────────────────────────────────────────────────────┐
│ 1. TryActivateAbility                                       │
│    └─→ 生成新的 FPredictionKey                              │
│                                                             │
│ 2. ServerTryActivateAbility (RPC)                           │
│    └─→ 发送预测键到服务器                                    │
│                                                             │
│ 3. 本地继续执行 ActivateAbility                              │
│    └─→ 副作用关联预测键                                      │
│                                                             │
│ 4. 等待服务器响应...                                         │
└─────────────────────────────────────────────────────────────┘

服务器响应：
┌─────────────────────────────────────────────────────────────┐
│ ClientActivateAbilityFailed                                 │
│ └─→ 立即回滚副作用                                          │
│     └─→ FPredictionKeyDelegates::BroadcastRejectedDelegate  │
│                                                             │
│ ClientActivateAbilitySucceed                                │
│ └─→ 等待属性同步后清理预测状态                               │
│     └─→ FReplicatedPredictionKeyItem::OnRep                 │
└─────────────────────────────────────────────────────────────┘
```

**关键代码**：`AbilitySystemComponent_Abilities.cpp`

```cpp
void UAbilitySystemComponent::TryActivateAbility(...)
{
    // 生成预测键
    FPredictionKey PredictionKey = FPredictionKey::CreateNewPredictionKey(this);

    // 发送到服务器
    ServerTryActivateAbility(AbilityToActivate, PredictionKey, ...);

    // 本地预测执行
    ActivateAbility(AbilityToActivate, PredictionKey, ...);
}
```

### 第二阶段：深入属性预测

**源码位置**：`GameplayPrediction.h:114-144`

**核心思想**：属性预测是"增量预测"而非"绝对值预测"

```
传统方式（错误）：
  客户端预测：Health = 90
  服务器复制：Health = 90
  问题：无法区分是预测还是服务器确认

增量预测方式（正确）：
  客户端预测：HealthDelta = -10（相对于服务器基础值）
  服务器基础值：Health = 100
  最终显示：100 + (-10) = 90

  当服务器确认后：
  服务器复制：Health = 90
  移除预测增量：HealthDelta = 0
  最终显示：90 + 0 = 90
```

**实现要点**：

```cpp
// 1. 属性必须使用 REPNOTIFY_Always
void UMyHealthSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 关键：使用 REPNOTIFY_Always，确保每次复制都触发回调
    // 即使本地值与复制值相同也要触发（因为客户端可能已预测修改）
    DOREPLIFETIME_CONDITION_NOTIFY(UMyHealthSet, Health, COND_None, REPNOTIFY_Always);
}

// 2. RepNotify 中更新最终值
void UMyHealthSet::OnRep_Health()
{
    // 这个宏会重新计算最终值（基础值 + 修改器）
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyHealthSet, Health);
}

// 3. 即时效果被当作无限持续效果
// 在 UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf 中
// 即时GE预测时会被转换为无限持续GE
```

### 第三阶段：理解依赖链

**源码位置**：`GameplayPrediction.h:170-189`

**场景**：Ability X → 触发 → Ability Y → 触发 → Ability Z

```
问题：
  如果 Y 被服务器拒绝，Z 也应该无效
  但服务器不知道 Z 的存在（Z 是客户端预测激活的）

解决：
  ┌─────────────────────────────────────────────┐
  │ Ability X: PredictionKey = 10               │
  │   └─→ Base = 0, Current = 10                │
  │                                             │
  │ Ability Y: PredictionKey = 11               │
  │   └─→ Base = 10, Current = 11               │
  │   └─→ AddDependency(11, 10)                 │
  │                                             │
  │ Ability Z: PredictionKey = 12               │
  │   └─→ Base = 10, Current = 12               │
  │   └─→ AddDependency(12, 11)                 │
  └─────────────────────────────────────────────┘

  当 Y (Key=11) 被拒绝时：
  └─→ Z (Key=12) 也被标记为无效（因为依赖 11）
```

**设计建议**：使用 GameplayTag 确保依赖关系

```cpp
// GA_Combo2 只有在 GA_Combo1 成功时才能激活
// GA_Combo1 成功时给予特定 Tag
// GA_Combo2 需要 Tag 才能激活

UPROPERTY(EditDefaultsOnly, Category = "Combo")
FGameplayTagContainer RequiredComboTags;
```

### 第四阶段：扩展预测窗口

**源码位置**：`GameplayPrediction.h:192-213`

**问题**：预测键只在单次 ActivateAbility 调用内有效，但 Ability 可能跨帧执行（等待输入、计时器等）

**解决**：使用 `FScopedPredictionWindow` 创建新的预测窗口

```cpp
// 示例：UAbilityTask_WaitInputRelease::OnReleaseCallback

void UAbilityTask_WaitInputRelease::OnReleaseCallback()
{
    // 1. 创建新的预测窗口
    FScopedPredictionWindow ScopedPred(AbilitySystemComponent);

    // 2. 发送新的预测键到服务器
    AbilitySystemComponent->ServerInputRelease(ScopedPred.ScopedPredictionKey);

    // 3. 此作用域内的副作用使用新的预测键
    // ... 执行逻辑 ...

    // 4. 窗口结束时，预测键被确认
}
```

**服务器端处理**：

```cpp
void UAbilitySystemComponent::ServerInputRelease_Implementation(FPredictionKey PredictionKey)
{
    // 接收客户端传来的预测键
    FScopedPredictionWindow ScopedPred(this, PredictionKey);

    // 在此作用域内，使用客户端传来的预测键
    // ... 执行逻辑 ...
}
```

---

## 当前限制

### 不可预测的内容

| 内容 | 原因 |
|-----|------|
| GameplayEffect 移除 | 需要额外的状态追踪 |
| GameplayEffect 周期效果（DOT） | 时间同步复杂 |
| Meta 属性（Damage/Healing） | 只支持即时效果 |
| 百分比乘法效果 | 需要聚合器链复制 |

### 已知问题

1. **触发事件的链式激活回滚不完整**
   - 场景：GA_Mispredict → GA_Predict1
   - 问题：GA_Mispredict 被拒绝时，GA_Predict1 可能仍在执行
   - 解决：使用 Tag 系统确保依赖关系

2. **百分比效果预测不准确**
   ```
   示例：
   - 客户端有 +10% 移动速度 buff，基础 500 → 最终 550
   - 客户端预测另一个 +10% buff
   - 错误结果：550 * 1.1 = 605
   - 正确结果：500 * 1.2 = 600
   ```
   - 原因：客户端不知道完整的修改器链

3. **弱预测模式尚未实现**
   - 某些场景无法建立预测键交换
   - 例如：碰撞触发的 GameplayEffect

---

## 调试技巧

### 断点位置

| 函数 | 用途 |
|-----|------|
| `FPredictionKey::CreateNewPredictionKey` | 预测键生成 |
| `FPredictionKeyDelegates::BroadcastRejectedDelegate` | 预测拒绝 |
| `FReplicatedPredictionKeyItem::OnRep` | 服务器确认 |
| `FScopedPredictionWindow::~FScopedPredictionWindow` | 预测窗口结束 |

### 日志输出

```cpp
// 在 GameplayPrediction.cpp 中添加
UE_LOG(LogGameplayAbilities, Log, TEXT("PredictionKey Created: %s"), *Key.ToString());
UE_LOG(LogGameplayAbilities, Log, TEXT("PredictionKey Rejected: %s"), *Key.ToString());
UE_LOG(LogGameplayAbilities, Log, TEXT("PredictionKey CaughtUp: %s"), *Key.ToString());
```

### 控制台命令

```
// 显示 ASC 调试信息
Net.PK.Draw

// 显示属性复制信息
Net.Replication.Debug
```

---

## 参考资源

1. **源码**：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/`
2. **单元测试**：`Private/Tests/PredictionKeyTests.cpp`
3. **示例项目**：`Samples/Games/Lyra/Source/LyraGame/AbilitySystem/`
4. **官方文档**：https://dev.epicgames.com/documentation/en-us/unreal-engine/gameplay-ability-system-in-unreal-engine
