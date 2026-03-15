# 第5课：属性预测与回滚机制

> 学习时间：45-60分钟
> 难度：进阶
> 前置知识：第1-4课内容、AttributeSet基础

---

## 学习目标

完成本课后，你将能够：
1. 深入理解属性增量预测的原理
2. 掌握 RepNotify 在属性预测中的作用
3. 学会实现自定义属性的预测支持
4. 掌握属性回滚的完整机制

---

## 5.1 属性预测的挑战

### 5.1.1 核心问题

```mermaid
flowchart TB
    subgraph 问题1["问题1: 属性是服务器权威复制的"]
        A1["服务器拥有 BaseValue 的所有权"]
        A2["客户端不能直接修改 BaseValue"]
        A3["客户端修改会被服务器覆盖"]
    end

    subgraph 问题2["问题2: RepNotify 不触发"]
        B1["客户端预测: Health=80"]
        B2["服务器复制: Health=80"]
        B3["值相同, RepNotify 不触发!"]
        B4["客户端不知道服务器已确认"]
    end

    subgraph 问题3["问题3: 即时修改无状态"]
        C1["Instant GE 执行后消失"]
        C2["无法追踪修改历史"]
        C3["无法回滚"]
    end

    问题1 --> 问题2 --> 问题3
```

### 5.1.2 传统方式的失败

```mermaid
sequenceDiagram
    participant Server as 服务器
    participant Net as 网络
    participant Client as 客户端
    participant Attr as 属性

    Note over Server,Attr: 传统方式 (错误)

    Server->>Attr: Health BaseValue = 100
    Server->>Net: 复制 Health = 100
    Net->>Client: Health = 100

    Note over Client: 客户端预测施法

    Client->>Attr: Health = 80 (直接修改)

    Note over Server: 服务器确认

    Server->>Attr: Health BaseValue = 80
    Server->>Net: 复制 Health = 80
    Net->>Client: Health = 80

    Note over Client: 值相同! RepNotify 不触发<br/>不知道服务器已确认<br/>预测状态无法清理
```

---

## 5.2 增量预测方案

### 5.2.1 核心思想

```mermaid
flowchart LR
    subgraph 传统方式["传统方式 (错误)"]
        T1["预测绝对值"]
        T2["Health = 80"]
        T3["无法区分预测和确认"]
    end

    subgraph 增量预测["增量预测 (正确)"]
        I1["预测增量"]
        I2["HealthMod = -20"]
        I3["最终值 = Base + Mod"]
        I4["可以追踪和回滚"]
    end

    传统方式 -->|"失败"| 增量预测
```

### 5.2.2 增量预测流程

```mermaid
sequenceDiagram
    participant Server as 服务器
    participant Net as 网络
    participant Client as 客户端
    participant Base as BaseValue
    participant Mod as 修改器
    participant Final as CurrentValue

    Note over Server,Final: 增量预测方式

    Server->>Base: Health BaseValue = 100
    Server->>Net: 复制 Health = 100
    Net->>Base: Health BaseValue = 100
    Base->>Final: CurrentValue = 100

    Note over Client: 客户端预测施法

    Client->>Mod: 创建预测修改器<br/>HealthMod = -20
    Mod->>Final: CurrentValue = 100 + (-20) = 80

    Note over Client: 显示 HP = 80<br/>但 BaseValue 仍是 100

    Note over Server: 服务器确认

    Server->>Base: Health BaseValue = 80
    Server->>Net: 复制 Health = 80
    Net->>Base: Health BaseValue = 80

    Note over Base: RepNotify 触发!<br/>(使用 REPNOTIFY_Always)

    Client->>Mod: 移除预测修改器
    Mod->>Final: CurrentValue = 80 + 0 = 80

    Note over Final: 平滑过渡<br/>无闪烁
```

### 5.2.3 值的计算关系

```mermaid
flowchart TB
    subgraph 属性结构["属性数据结构"]
        Base["BaseValue<br/>基础值 (服务器权威)"]
        Mods["Modifiers<br/>修改器列表"]
        Current["CurrentValue<br/>最终值"]
    end

    subgraph 计算公式["计算公式"]
        Formula["CurrentValue = BaseValue + Σ(Modifiers)"]
    end

    subgraph 示例["示例"]
        Ex1["BaseValue = 100 (服务器)"]
        Ex2["预测 Modifier = -20"]
        Ex3["CurrentValue = 100 + (-20) = 80"]
    end

    Base --> Formula
    Mods --> Formula
    Formula --> Current

    属性结构 --> 计算公式 --> 示例
```

---

## 5.3 RepNotify 实现

### 5.3.1 REPNOTIFY_Always 的必要性

```mermaid
flowchart TB
    subgraph 默认行为["默认 RepNotify 行为"]
        D1["值变化时触发"]
        D2["值相同时不触发"]
        D3["预测场景下失败 ❌"]
    end

    subgraph Always行为["REPNOTIFY_Always 行为"]
        A1["每次复制都触发"]
        A2["即使值相同也触发"]
        A3["预测场景下成功 ✓"]
    end

    默认行为 --> Always行为
```

### 5.3.2 属性复制配置

```cpp
// AttributeSet.h

void UMyAttributeSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 关键: 使用 REPNOTIFY_Always
    // 确保每次复制都触发回调，即使本地值与复制值相同
    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, Health, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, MaxHealth, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, Mana, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, Stamina, COND_None, REPNOTIFY_Always);
}
```

### 5.3.3 RepNotify 实现

```mermaid
sequenceDiagram
    participant Server as 服务器
    participant Net as 网络
    participant Client as 客户端
    participant RepNotify as OnRep_Health
    participant Macro as GAMEPLAYATTRIBUTE_REPNOTIFY
    participant ASC as ASC

    Server->>Net: 复制 Health = 80
    Net->>Client: 收到 Health = 80

    Note over Client: 本地 Health 可能已经是 80<br/>(预测修改后的值)

    Client->>RepNotify: OnRep_Health() 触发<br/>(因为 REPNOTIFY_Always)

    RepNotify->>Macro: GAMEPLAYATTRIBUTE_REPNOTIFY(UMyAttributeSet, Health)

    Macro->>ASC: GameEffectAttributePropertyReplicatedReceived()

    ASC->>ASC: 更新 BaseValue = 80
    ASC->>ASC: 重新计算 CurrentValue

    Note over ASC: 移除预测修改器后<br/>CurrentValue = 80 + 0 = 80
```

### 5.3.4 GAMEPLAYATTRIBUTE_REPNOTIFY 宏

```cpp
// AttributeSet.h

#define GAMEPLAYATTRIBUTE_REPNOTIFY(ClassName, PropertyName) \
{ \
    GameEffectAttributePropertyReplicatedReceived( \
        Get##PropertyName##Attribute(), \
        PropertyName); \
}

// 展开后的实际调用
void UMyAttributeSet::OnRep_Health()
{
    GameEffectAttributePropertyReplicatedReceived(
        GetHealthAttribute(),  // FGameplayAttribute
        Health);               // float&
}

// AttributeSet.cpp
void UAttributeSet::GameEffectAttributePropertyReplicatedReceived(
    FGameplayAttribute Attribute,
    float& NewValue)
{
    // 1. 更新 BaseValue 为服务器值
    SetBaseAttributeValueFromReplication(Attribute, NewValue);

    // 2. 重新计算最终值 (BaseValue + 所有修改器)
    // 这会自动处理预测修改器的清理
}
```

---

## 5.4 属性回滚机制

### 5.4.1 回滚流程

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant GE as ActiveGE
    participant Mod as 修改器
    participant Attr as 属性
    participant Delegate as 委托

    Note over Client,Delegate: 预测失败, 需要回滚

    Client->>Delegate: BroadcastRejectedDelegate(Key=10)

    Delegate->>GE: 执行 Rejected 委托

    GE->>Mod: 移除修改器

    Mod->>Attr: 重新计算 CurrentValue

    Note over Attr: BaseValue 未改变<br/>修改器已移除<br/>CurrentValue = BaseValue

    Note over Attr: 属性恢复到预测前状态
```

### 5.4.2 修改器的追踪

```mermaid
classDiagram
    class FActiveGameplayEffect {
        +FPredictionKey PredictionKey
        +TArray~FGameplayModifierInfo~ Modifiers
        +FGameplayEffectHandle Handle
    }

    class FGameplayModifierInfo {
        +FGameplayAttribute Attribute
        +float Magnitude
        +EGameplayModOp::Type ModifierOp
    }

    class FAggregatorModChannel {
        +TArray~FAggregatorMod~ Mods
        +float Evaluate()
    }

    class FAggregatorMod {
        +float EvaluatedMagnitude
        +FActiveGameplayEffectHandle SourceHandle
        +FPredictionKey PredictionKey
    }

    FActiveGameplayEffect --> FGameplayModifierInfo : 包含
    FGameplayModifierInfo --> FAggregatorModChannel : 应用到
    FAggregatorModChannel --> FAggregatorMod : 包含
    FActiveGameplayEffect --> FPredictionKey : 存储
    FAggregatorMod --> FPredictionKey : 可追踪
```

### 5.4.3 回滚时的属性恢复

```mermaid
flowchart TB
    subgraph 预测状态["预测状态"]
        P1["BaseValue = 100 (未变)"]
        P2["预测修改器 = -20"]
        P3["CurrentValue = 80"]
    end

    subgraph 回滚操作["回滚操作"]
        R1["移除预测修改器"]
        R2["重新计算 CurrentValue"]
    end

    subgraph 回滚后["回滚后状态"]
        A1["BaseValue = 100"]
        A2["修改器 = 0"]
        A3["CurrentValue = 100"]
    end

    预测状态 --> 回滚操作 --> 回滚后
```

### 5.4.4 源码解析

```cpp
// GameplayEffect.cpp (简化版)

void FActiveGameplayEffectsContainer::RemoveActiveGameplayEffect_NoReturn(
    FActiveGameplayEffectHandle Handle,
    bool bInvokeRemoveCallbacks)
{
    FActiveGameplayEffect* ActiveGE = GetActiveGameplayEffect(Handle);
    if (!ActiveGE)
    {
        return;
    }

    // 检查是否需要跳过回调 (预测清理时)
    if (ActiveGE->PredictionKey.IsLocalClientKey() && !bInvokeRemoveCallbacks)
    {
        // 这是自己预测的 GE，跳过 OnRemove 逻辑
        // 避免 GameplayCue 重复移除
        ActiveEffects.Remove(ActiveGE);
        return;
    }

    // 正常移除流程
    // 1. 移除修改器
    for (const FGameplayModifierInfo& Mod : ActiveGE->Modifiers)
    {
        RemoveModifierFromAggregator(Mod, ActiveGE->Handle);
    }

    // 2. 触发 OnRemove 回调
    if (bInvokeRemoveCallbacks)
    {
        OnRemovedDelegate.Broadcast(ActiveGE);
    }

    // 3. 移除 GE
    ActiveEffects.Remove(ActiveGE);
}
```

---

## 5.5 自定义属性集实现

### 5.5.1 完整示例

```cpp
// MyAttributeSet.h

#pragma once

#include "CoreMinimal.h"
#include "AttributeSet.h"
#include "AbilitySystemComponent.h"
#include "MyAttributeSet.generated.h"

UCLASS()
class MYGAME_API UMyAttributeSet : public UAttributeSet
{
    GENERATED_BODY()

public:
    // 属性定义
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Attributes")
    FGameplayAttributeData Health;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, Health)

    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Attributes")
    FGameplayAttributeData MaxHealth;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, MaxHealth)

    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Attributes")
    FGameplayAttributeData Mana;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, Mana)

    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Attributes")
    FGameplayAttributeData MaxMana;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, MaxMana)

    // Meta 属性 (不复制，用于即时效果)
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Attributes")
    FGameplayAttributeData Damage;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, Damage)

    // 复制配置
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    // RepNotify 函数
    UFUNCTION()
    virtual void OnRep_Health(const FGameplayAttributeData& OldValue);

    UFUNCTION()
    virtual void OnRep_MaxHealth(const FGameplayAttributeData& OldValue);

    UFUNCTION()
    virtual void OnRep_Mana(const FGameplayAttributeData& OldValue);

    UFUNCTION()
    virtual void OnRep_MaxMana(const FGameplayAttributeData& OldValue);

    // 属性修改前后回调
    virtual void PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue) override;
    virtual void PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data) override;
};
```

### 5.5.2 实现文件

```cpp
// MyAttributeSet.cpp

#include "MyAttributeSet.h"
#include "Net/UnrealNetwork.h"

void UMyAttributeSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 关键: 使用 REPNOTIFY_Always 支持预测
    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, Health, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, MaxHealth, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, Mana, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, MaxMana, COND_None, REPNOTIFY_Always);

    // Meta 属性不复制
    // Damage 不需要复制
}

void UMyAttributeSet::OnRep_Health(const FGameplayAttributeData& OldValue)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyAttributeSet, Health, OldValue);
}

void UMyAttributeSet::OnRep_MaxHealth(const FGameplayAttributeData& OldValue)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyAttributeSet, MaxHealth, OldValue);
}

void UMyAttributeSet::OnRep_Mana(const FGameplayAttributeData& OldValue)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyAttributeSet, Mana, OldValue);
}

void UMyAttributeSet::OnRep_MaxMana(const FGameplayAttributeData& OldValue)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyAttributeSet, MaxMana, OldValue);
}

void UMyAttributeSet::PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)
{
    Super::PreAttributeChange(Attribute, NewValue);

    // 限制属性范围
    if (Attribute == GetHealthAttribute())
    {
        NewValue = FMath::Clamp(NewValue, 0.0f, GetMaxHealth());
    }
    else if (Attribute == GetManaAttribute())
    {
        NewValue = FMath::Clamp(NewValue, 0.0f, GetMaxMana());
    }
}

void UMyAttributeSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
    Super::PostGameplayEffectExecute(Data);

    // 处理 Meta 属性
    if (Data.EvaluatedData.Attribute == GetDamageAttribute())
    {
        // Damage -> Health
        const float DamageDone = GetDamage();
        SetDamage(0.0f); // 重置

        if (DamageDone > 0.0f)
        {
            SetHealth(FMath::Clamp(GetHealth() - DamageDone, 0.0f, GetMaxHealth()));
        }
    }
}
```

### 5.5.3 属性集配置流程

```mermaid
flowchart TB
    subgraph 步骤1["步骤1: 定义属性"]
        A1["使用 FGameplayAttributeData"]
        A2["添加 ATTRIBUTE_ACCESSORS 宏"]
    end

    subgraph 步骤2["步骤2: 配置复制"]
        B1["GetLifetimeReplicatedProps"]
        B2["DOREPLIFETIME_CONDITION_NOTIFY"]
        B3["REPNOTIFY_Always"]
    end

    subgraph 步骤3["步骤3: 实现 RepNotify"]
        C1["OnRep_AttributeName"]
        C2["GAMEPLAYATTRIBUTE_REPNOTIFY 宏"]
    end

    subgraph 步骤4["步骤4: 可选回调"]
        D1["PreAttributeChange"]
        D2["PostGameplayEffectExecute"]
    end

    步骤1 --> 步骤2 --> 步骤3 --> 步骤4
```

---

## 5.6 Meta 属性的限制

### 5.6.1 Meta 属性特点

```mermaid
flowchart TB
    subgraph Meta属性["Meta 属性特点"]
        A["不复制"]
        B["只在即时 GE 中使用"]
        C["用于转换计算"]
        D["如: Damage → Health"]
    end

    subgraph 预测限制["预测限制"]
        E["无法预测 Meta 属性"]
        F["Duration GE 不触发"]
        G["预测转换逻辑缺失"]
    end

    Meta属性 --> 预测限制
```

### 5.6.2 Meta 属性工作流程

```mermaid
sequenceDiagram
    participant GE as GameplayEffect
    participant Meta as Meta属性(Damage)
    participant Callback as PostGameplayEffectExecute
    participant Real as 真实属性(Health)

    Note over GE,Real: Meta 属性工作流程

    GE->>Meta: 应用 Damage = 30
    Note over Meta: Meta 属性不存储<br/>只用于传递

    GE->>Callback: PostGameplayEffectExecute()
    Callback->>Meta: 读取 Damage = 30
    Callback->>Meta: 重置 Damage = 0
    Callback->>Real: Health -= 30

    Note over Real: Health 已修改
```

### 5.6.3 为什么 Meta 属性难以预测

```mermaid
flowchart TB
    subgraph 问题["问题"]
        P1["预测时 Instant GE 转为无限持续"]
        P2["无限持续 GE 不触发 PostGameplayEffectExecute"]
        P3["Damage → Health 转换不执行"]
        P4["预测失败"]
    end

    subgraph 当前状态["当前状态 (UE5.7)"]
        S1["Meta 属性不可预测"]
        S2["需要等待服务器确认"]
        S3["会有延迟感"]
    end

    问题 --> 当前状态
```

---

## 5.7 完整示例

### 5.7.1 法力消耗预测

```mermaid
sequenceDiagram
    participant Player as 玩家
    participant Client as 客户端
    participant Mana as Mana属性
    participant Server as 服务器

    Note over Player,Server: 施放法术消耗 30 法力

    Player->>Client: 激活法术 Ability
    Client->>Client: Key=20

    Note over Client: 预测应用 Cost GE

    Client->>Mana: BaseValue = 100
    Client->>Client: 创建预测修改器 -30
    Client->>Mana: CurrentValue = 70

    Client->>Server: ServerTryActivateAbility(Key=20)

    Note over Server: 服务器验证

    Server->>Server: 实际 Mana = 95
    Server->>Server: 消耗 30, Mana = 65
    Server->>Server: 复制 Mana = 65

    Client->>Client: 收到 Mana = 65
    Client->>Client: OnRep_Mana 触发
    Client->>Client: BaseValue = 65
    Client->>Client: CatchUp(Key=20)
    Client->>Client: 移除预测修改器 -30

    Note over Mana: CurrentValue = 65 + 0 = 65

    Note over Client: 法力值正确
```

### 5.7.2 预测失败回滚

```mermaid
sequenceDiagram
    participant Player as 玩家
    participant Client as 客户端
    participant Mana as Mana属性
    participant Server as 服务器

    Note over Player,Server: 法力不足但客户端显示足够

    Player->>Client: 激活法术 Ability
    Client->>Client: Key=25
    Client->>Mana: 本地显示 Mana = 50
    Client->>Client: 预测消耗 30
    Client->>Mana: CurrentValue = 20

    Client->>Server: ServerTryActivateAbility(Key=25)

    Note over Server: 服务器验证

    Server->>Server: 实际 Mana = 25 (不足!)
    Server->>Server: 验证失败!
    Server->>Client: ClientActivateAbilityFailed(Key=25)

    Client->>Client: Rejected(Key=25)
    Client->>Client: 移除预测修改器 -30

    Note over Mana: CurrentValue = 50 + 0 = 50<br/>(恢复到预测前)

    Server->>Client: 复制实际 Mana = 25
    Client->>Client: OnRep_Mana 触发
    Client->>Mana: BaseValue = 25
    Client->>Mana: CurrentValue = 25

    Note over Client: 最终显示正确值
```

---

## 5.8 实践：调试属性预测

### 5.8.1 断点位置

```mermaid
flowchart LR
    subgraph 属性修改["属性修改断点"]
        A1["PreAttributeChange<br/>修改前"]
        A2["PostGameplayEffectExecute<br/>修改后"]
    end

    subgraph 复制["复制断点"]
        B1["OnRep_Attribute<br/>收到复制"]
        B2["GameEffectAttributePropertyReplicatedReceived<br/>处理复制"]
    end

    subgraph 回滚["回滚断点"]
        C1["RemoveActiveGameplayEffect<br/>移除GE"]
        C2["修改器移除<br/>属性重算"]
    end
```

### 5.8.2 添加调试日志

```cpp
// MyAttributeSet.cpp

void UMyAttributeSet::OnRep_Health(const FGameplayAttributeData& OldValue)
{
    UE_LOG(LogGameplayAbilities, Log,
           TEXT("OnRep_Health: Old=%.2f, New=%.2f, Base=%.2f"),
           OldValue.GetCurrentValue(),
           GetHealth(),
           GetHealthAttribute().GetNumericValue(this));

    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyAttributeSet, Health, OldValue);
}

void UMyAttributeSet::PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)
{
    UE_LOG(LogGameplayAbilities, Log,
           TEXT("PreAttributeChange: %s, NewValue=%.2f"),
           *Attribute.GetName(),
           NewValue);

    Super::PreAttributeChange(Attribute, NewValue);
}
```

### 5.8.3 观察属性值变化

```mermaid
flowchart TB
    subgraph 观察内容["需要观察的内容"]
        O1["BaseValue 变化"]
        O2["CurrentValue 变化"]
        O3["修改器列表"]
        O4["PredictionKey 状态"]
    end

    subgraph 工具["调试工具"]
        T1["断点调试"]
        T2["日志输出"]
        T3["控制台命令<br/>AbilitySystem.Debug 1"]
    end

    观察内容 --> 工具
```

---

## 5.9 总结

### 5.9.1 核心概念图

```mermaid
mindmap
  root((属性预测))
    核心原理
      增量预测
      BaseValue不变
      修改器追踪
    关键实现
      REPNOTIFY_Always
      GAMEPLAYATTRIBUTE_REPNOTIFY
      修改器管理
    回滚机制
      移除修改器
      重新计算值
      委托触发
    Meta属性
      不可预测
      仅即时GE
      需等待服务器
```

### 5.9.2 关键要点

```mermaid
flowchart TB
    subgraph 要点1["要点1: 增量预测"]
        A1["不预测绝对值"]
        A2["预测增量修改器"]
        A3["BaseValue 保持服务器值"]
    end

    subgraph 要点2["要点2: RepNotify"]
        B1["必须使用 REPNOTIFY_Always"]
        B2["每次复制都触发回调"]
        B3["使用 GAMEPLAYATTRIBUTE_REPNOTIFY 宏"]
    end

    subgraph 要点3["要点3: 回滚"]
        C1["移除预测修改器"]
        C2["CurrentValue 自动重算"]
        C3["恢复到预测前状态"]
    end

    要点1 --> 要点2 --> 要点3
```

---

## 课后练习

### 练习1：实现自定义属性集

1. 创建自定义 AttributeSet 类
2. 添加 Health, Mana 属性
3. 配置 RepNotify 和复制

### 练习2：观察预测流程

1. 在 `OnRep_Health` 添加日志
2. 激活消耗法力的 Ability
3. 观察属性值的变化过程

### 练习3：思考题

1. 为什么不能直接修改 BaseValue？
2. 如果属性有多个修改器，回滚时如何处理？
3. 如何处理属性的上限限制（如 Health 不能超过 MaxHealth）？

---

## 下节课预告

```mermaid
flowchart LR
    L5["第5课: 属性预测与回滚机制"] --> L6["第6课: GameplayCue 预测"]

    subgraph 第6课内容["第6课内容"]
        A["GameplayCue 类型"]
        B["GC 预测流程"]
        C["避免重复执行"]
        D["GC 与 GE 的关系"]
    end

    L6 --> 第6课内容
```

---

## 参考资料

- **源码**：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h`
- **源码**：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AttributeSet.cpp`
- **官方文档**：[Gameplay Attributes](https://dev.epicgames.com/documentation/en-us/unreal-engine/gameplay-attributes-in-unreal-engine)
