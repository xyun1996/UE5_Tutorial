# 第6课：GameplayCue 预测

> 学习时间：45-60分钟
> 难度：进阶
> 前置知识：第1-5课内容、GameplayCue基础

---

## 学习目标

完成本课后，你将能够：
1. 掌握 GameplayCue 的类型和触发时机
2. 理解 GC 预测的完整流程
3. 学会解决 GC 的 "Redo" 问题
4. 掌握独立 GC 和 GE 内 GC 的预测差异

---

## 6.1 GameplayCue 概述

### 6.1.1 GC 的作用

```mermaid
mindmap
  root((GameplayCue))
    作用
      视觉效果
      音效
      粒子系统
      相机抖动
    特点
      非游戏逻辑
      可预测
      可本地执行
    触发方式
      GE 自动触发
      手动执行
      Tag 触发
```

### 6.1.2 GC 类型与触发时机

```mermaid
flowchart TB
    subgraph GC类型["GameplayCue 类型"]
        Actor["GameplayCueNotify_Actor<br/>持久化效果<br/>光环、持续特效"]
        Static["GameplayCueNotify_Static<br/>一次性效果<br/>打击、爆炸"]
    end

    subgraph 触发时机["触发时机"]
        OnActive["OnActive<br/>GE 应用时"]
        OnRemove["OnRemove<br/>GE 移除时"]
        Execute["Execute<br/>即时执行"]
    end

    subgraph 关联方式["关联方式"]
        GE内["GE 内的 GameplayCues"]
        独立["独立执行的 GC"]
    end

    GC类型 --> 触发时机 --> 关联方式
```

### 6.1.3 GC 与游戏逻辑分离

```mermaid
flowchart LR
    subgraph 游戏逻辑["游戏逻辑 (服务器权威)"]
        L1["属性修改"]
        L2["Tag 修改"]
        L3["状态变更"]
    end

    subgraph 表现层["表现层 (可预测)"]
        R1["特效"]
        R2["音效"]
        R3["动画"]
    end

    游戏逻辑 -->|"触发"| 表现层

    Note["GC 属于表现层<br/>可以安全预测<br/>不会影响游戏逻辑"]
```

---

## 6.2 GE 内的 GameplayCue 预测

### 6.2.1 GE 触发 GC 流程

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant GE as ActiveGameplayEffect
    participant GC as GameplayCue
    participant Server as 服务器

    Note over Client,Server: 预测应用带 GC 的 GE

    Client->>GE: ApplyGameplayEffect(BuffGE)
    Note over GE: BuffGE 包含:<br/>- 属性修改 +10<br/>- GC: ShieldEffect

    GE->>GE: 创建 ActiveGE (Key=10)

    alt 预测场景
        GE->>GC: 触发 OnActive GC
        GC->>GC: 播放护盾特效
        Note over GC: 本地预测播放
    end

    Client->>Server: ServerTryActivateAbility(Key=10)

    Server->>GE: ApplyGameplayEffect(BuffGE)
    Server->>GC: 触发 OnActive GC
    Server->>Server: 多播到所有客户端

    Note over Client: 收到 GE 复制

    Client->>GE: PostReplicatedAdd()
    GE->>GE: 检查 PredictionKey

    alt Key=10 是自己的
        Note over GE: 跳过 GC 触发<br/>避免重复播放
    else Key 不是自己的
        GE->>GC: 触发 OnActive GC
    end
```

### 6.2.2 GC 在 ActiveGE 中的存储

```mermaid
classDiagram
    class FActiveGameplayEffect {
        +FPredictionKey PredictionKey
        +TArray~FGameplayEffectCue~ GameplayCues
        +PostReplicatedAdd()
        +PreReplicatedRemove()
    }

    class FGameplayEffectCue {
        +FGameplayTag GameplayCueTag
        +FGameplayCueProxy Proxy
    }

    class FGameplayCueProxy {
        +AActor* TargetActor
        +FVector Location
        +FRotator Rotation
    }

    FActiveGameplayEffect --> FGameplayEffectCue : 包含
    FGameplayEffectCue --> FGameplayCueProxy : 使用
```

### 6.2.3 PostReplicatedAdd 中的 GC 处理

```cpp
// GameplayEffect.cpp (简化版)

void FActiveGameplayEffect::PostReplicatedAdd(const FActiveGameplayEffectsContainer& InArray)
{
    // 检查是否是自己预测的 GE
    if (PredictionKey.IsLocalClientKey())
    {
        // 是自己预测的，跳过所有 OnApplied 逻辑
        // 包括 GameplayCue 的触发
        // 这解决了 GC 的 Redo 问题
        return;
    }

    // 不是自己预测的，正常处理

    // 1. 应用属性修改器
    InArray.Owner->UpdateAggregatorsForEffect(this);

    // 2. 触发 GameplayCue
    for (const FGameplayEffectCue& Cue : GameplayCues)
    {
        InArray.Owner->InvokeGameplayCueAdded(
            Cue.GameplayCueTag,
            PredictionKey,
            Cue.Proxy);
    }
}
```

### 6.2.4 PreReplicatedRemove 中的 GC 处理

```mermaid
sequenceDiagram
    participant Server as 服务器
    participant Net as 网络
    participant Client as 客户端
    participant GE as ActiveGE
    participant GC as GameplayCue

    Note over Server,GC: GE 被移除

    Server->>GE: 移除 GE
    Server->>Net: 复制移除

    Net->>Client: 收到移除通知
    Client->>GE: PreReplicatedRemove()

    GE->>GE: 检查 PredictionKey

    alt Key 是自己的预测键
        Note over GE: 跳过 GC OnRemove<br/>因为从未触发过 OnActive
    else Key 不是自己的
        GE->>GC: 触发 OnRemove GC
        GC->>GC: 停止特效
    end
```

---

## 6.3 独立 GameplayCue 预测

### 6.3.1 独立 GC 执行流程

```mermaid
sequenceDiagram
    participant Ability as Ability
    participant ASC as ASC
    participant GC as GameplayCue
    participant Server as 服务器

    Note over Ability,Server: 在 Ability 中独立执行 GC

    Ability->>ASC: ExecuteGameplayCue(HitCue)

    ASC->>ASC: 获取 ScopedPredictionKey

    alt 有 PredictionKey (预测窗口内)
        ASC->>GC: 本地预测执行 GC
        GC->>GC: 播放打击特效

        ASC->>Server: 不发送 RPC<br/>(GC 不需要单独同步)
    else 无 PredictionKey
        ASC->>Server: NetMulticast ExecuteGameplayCue
        Server->>GC: 多播到所有客户端
    end
```

### 6.3.2 ExecuteGameplayCue 实现

```cpp
// AbilitySystemComponent.cpp (简化版)

void UAbilitySystemComponent::ExecuteGameplayCue(
    const FGameplayTag& GameplayCueTag,
    FPredictionKey PredictionKey,
    const FGameplayCueParameters& Parameters)
{
    if (IsNetAuthority())
    {
        // 服务器：多播到所有客户端
        NetMulticast_InvokeGameplayCueExecuted(
            GameplayCueTag,
            PredictionKey,
            Parameters);
    }
    else if (PredictionKey.IsValidKey())
    {
        // 客户端预测：本地执行
        // 不发送 RPC，等待服务器通过其他方式同步
        InvokeGameplayCueExecuted(GameplayCueTag, Parameters);
    }
    // 无 PredictionKey 时，客户端不执行
}
```

### 6.3.3 服务器多播处理

```mermaid
sequenceDiagram
    participant Server as 服务器
    participant Net as 网络
    participant ClientA as 客户端A<br/>(预测者)
    participant ClientB as 客户端B
    participant GC as GameplayCue

    Note over Server,GC: 服务器多播 GC

    Server->>Net: NetMulticast_InvokeGameplayCueExecuted<br/>(Tag, Key=10, Params)

    par 发送到客户端A
        Net->>ClientA: 收到多播
        ClientA->>ClientA: Key=10 是自己的
        Note over ClientA: 跳过执行<br/>避免重复
    and 发送到客户端B
        Net->>ClientB: 收到多播
        ClientB->>ClientB: Key=0 (无效)
        ClientB->>GC: 执行 GC
        Note over ClientB: 正常播放特效
    end
```

### 6.3.4 NetMulticast 实现

```cpp
// AbilitySystemComponent.cpp (简化版)

void UAbilitySystemComponent::NetMulticast_InvokeGameplayCueExecuted_Implementation(
    const FGameplayTag& GameplayCueTag,
    FPredictionKey PredictionKey,
    const FGameplayCueParameters& Parameters)
{
    // 关键检查：是否是自己预测的 GC
    if (PredictionKey.IsLocalClientKey())
    {
        // 这是自己预测的 GC，跳过执行
        // 解决 Redo 问题
        return;
    }

    // 不是自己预测的，正常执行
    InvokeGameplayCueExecuted(GameplayCueTag, Parameters);
}
```

---

## 6.4 GameplayCue 预测总结

### 6.4.1 预测场景对比

```mermaid
flowchart TB
    subgraph GE内GC["GE 内的 GC"]
        A1["随 GE 预测"]
        A2["PostReplicatedAdd 检查"]
        A3["PredictionKey.IsLocalClientKey()"]
    end

    subgraph 独立GC["独立执行的 GC"]
        B1["需要 PredictionKey"]
        B2["本地预测执行"]
        B3["服务器多播时检查"]
    end

    subgraph 共同点["共同点"]
        C1["都检查 PredictionKey"]
        C2["跳过自己的预测"]
        C3["避免 Redo 问题"]
    end

    GE内GC --> 共同点
    独立GC --> 共同点
```

### 6.4.2 Redo 问题解决流程

```mermaid
flowchart TB
    subgraph 问题["Redo 问题"]
        P1["客户端预测播放特效"]
        P2["服务器多播同一特效"]
        P3["特效重复播放 ❌"]
    end

    subgraph 解决方案["解决方案"]
        S1["PredictionKey 识别"]
        S2["IsLocalClientKey 检查"]
        S3["跳过自己的预测 ✓"]
    end

    问题 --> 解决方案
```

### 6.4.3 完整流程图

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant ASC as ASC
    participant GC as GameplayCue
    participant Server as 服务器
    participant Others as 其他客户端

    Note over Client,Others: 完整 GC 预测流程

    Client->>ASC: ExecuteGameplayCue(Tag, Key=10)
    ASC->>GC: 本地预测执行
    GC->>GC: 播放特效 (第1次)

    Client->>Server: 相关 RPC (携带 Key=10)

    Server->>Server: 处理请求
    Server->>ASC: NetMulticast_InvokeGameplayCueExecuted

    par 多播到所有客户端
        Server->>Client: Key=10
        Server->>Others: Key=0
    end

    Client->>Client: Key=10 是自己的
    Note over Client: 跳过执行 ✓

    Others->>GC: Key=0 不是预测
    Others->>GC: 执行 GC
    Note over Others: 正常播放特效 ✓
```

---

## 6.5 GameplayCue 参数传递

### 6.5.1 参数结构

```mermaid
classDiagram
    class FGameplayCueParameters {
        +FGameplayTag GameplayCueTag
        +FGameplayEffectContextHandle EffectContext
        +float Magnitude
        +FVector Location
        +FRotator Rotation
        +FVector Normal
        +AActor* Instigator
        +AActor* EffectCauser
        +TArray~AActor~ TargetActors
        +FGameplayAbilityTargetDataHandle TargetData
    }

    class FGameplayEffectContext {
        +AActor* Instigator
        +AActor* EffectCauser
        +FVector Origin
        +TArray~AActor~ Actors
    }

    FGameplayCueParameters --> FGameplayEffectContext : 包含
```

### 6.5.2 参数传递流程

```mermaid
sequenceDiagram
    participant Ability as Ability
    participant Spec as GE Spec
    participant Params as GC Parameters
    participant GC as GameplayCue

    Ability->>Spec: 创建 GE Spec
    Spec->>Spec: 设置 Context<br/>(Instigator, Causer)

    Note over Spec: GE 包含 GC

    Spec->>Params: 创建 GC Parameters
    Params->>Params: 从 Context 复制数据
    Params->>Params: 设置 Magnitude

    Spec->>GC: 执行 GC with Parameters
    GC->>GC: 使用参数生成特效
```

---

## 6.6 实践：自定义 GameplayCue

### 6.6.1 创建 GameplayCueNotify

```mermaid
flowchart TB
    subgraph 步骤1["步骤1: 创建类"]
        A1["继承 GameplayCueNotify_Static"]
        A2["或 GameplayCueNotify_Actor"]
    end

    subgraph 步骤2["步骤2: 实现逻辑"]
        B1["重写 OnExecute"]
        B2["或 OnActive/OnRemove"]
    end

    subgraph 步骤3["步骤3: 配置 Tag"]
        C1["在编辑器中设置"]
        C2["GameplayCueTag"]
    end

    步骤1 --> 步骤2 --> 步骤3
```

### 6.6.2 示例：打击特效

```cpp
// MyHitGameplayCue.h

#pragma once

#include "CoreMinimal.h"
#include "GameplayCueNotify_Static.h"
#include "MyHitGameplayCue.generated.h"

UCLASS()
class MYGAME_API UMyHitGameplayCue : public UGameplayCueNotify_Static
{
    GENERATED_BODY()

public:
    // 打击特效
    UPROPERTY(EditDefaultsOnly, Category = "Effects")
    UNiagaraSystem* HitEffect;

    // 打击音效
    UPROPERTY(EditDefaultsOnly, Category = "Effects")
    USoundBase* HitSound;

    virtual bool OnExecute_Implementation(
        AActor* Target,
        const FGameplayCueParameters& Parameters) override;
};
```

```cpp
// MyHitGameplayCue.cpp

#include "MyHitGameplayCue.h"
#include "NiagaraFunctionLibrary.h"
#include "Kismet/GameplayStatics.h"

bool UMyHitGameplayCue::OnExecute_Implementation(
    AActor* Target,
    const FGameplayCueParameters& Parameters)
{
    // 获取位置
    FVector Location = Parameters.Location;
    if (Location.IsNearlyZero())
    {
        Location = Target->GetActorLocation();
    }

    // 播放特效
    if (HitEffect)
    {
        UNiagaraFunctionLibrary::SpawnSystemAtLocation(
            GetWorld(),
            HitEffect,
            Location);
    }

    // 播放音效
    if (HitSound)
    {
        UGameplayStatics::PlaySoundAtLocation(
            GetWorld(),
            HitSound,
            Location);
    }

    return true;
}
```

### 6.6.3 在 Ability 中使用

```cpp
// MyAttackAbility.cpp

void UMyAttackAbility::OnHitConfirmed(FGameplayAbilityTargetDataHandle TargetData)
{
    // 创建新的预测窗口
    FScopedPredictionWindow ScopedPred(GetAbilitySystemComponent());

    // 执行打击 GC (预测)
    FGameplayCueParameters Params;
    Params.Location = TargetData.Get(0)->GetHitResult()->Location;

    GetAbilitySystemComponent()->ExecuteGameplayCue(
        FGameplayTag::RequestGameplayTag(FName("GameplayCue.Hit")),
        ScopedPred.ScopedPredictionKey,
        Params);
}
```

---

## 6.7 调试 GameplayCue

### 6.7.1 断点位置

```mermaid
flowchart LR
    subgraph 执行断点["执行断点"]
        A1["ExecuteGameplayCue<br/>入口"]
        A2["InvokeGameplayCueExecuted<br/>实际执行"]
        A3["NetMulticast_InvokeGameplayCueExecuted<br/>多播接收"]
    end

    subgraph GE断点["GE 相关断点"]
        B1["PostReplicatedAdd<br/>GE 添加"]
        B2["PreReplicatedRemove<br/>GE 移除"]
    end
```

### 6.7.2 添加日志

```cpp
// AbilitySystemComponent.cpp

void UAbilitySystemComponent::ExecuteGameplayCue(
    const FGameplayTag& GameplayCueTag,
    FPredictionKey PredictionKey,
    const FGameplayCueParameters& Parameters)
{
    UE_LOG(LogGameplayAbilities, Log,
           TEXT("ExecuteGameplayCue: Tag=%s, Key=%s, IsAuthority=%d"),
           *GameplayCueTag.ToString(),
           *PredictionKey.ToString(),
           IsNetAuthority());

    // ...
}

void UAbilitySystemComponent::NetMulticast_InvokeGameplayCueExecuted_Implementation(...)
{
    UE_LOG(LogGameplayAbilities, Log,
           TEXT("NetMulticast GC: Tag=%s, Key=%s, IsLocalClientKey=%d"),
           *GameplayCueTag.ToString(),
           *PredictionKey.ToString(),
           PredictionKey.IsLocalClientKey());

    // ...
}
```

---

## 6.8 总结

### 6.8.1 核心概念图

```mermaid
mindmap
  root((GameplayCue 预测))
    GC类型
      Notify_Static
      Notify_Actor
    触发时机
      OnActive
      OnRemove
      Execute
    预测方式
      GE内GC随GE预测
      独立GC需要Key
      都检查IsLocalClientKey
    Redo解决
      PredictionKey识别
      跳过自己的预测
      其他客户端正常执行
```

### 6.8.2 关键要点

```mermaid
flowchart TB
    subgraph 要点1["要点1: GC 属于表现层"]
        A1["不影响游戏逻辑"]
        A2["可以安全预测"]
        A3["预测失败影响较小"]
    end

    subgraph 要点2["要点2: PredictionKey 识别"]
        B1["IsLocalClientKey 检查"]
        B2["跳过自己的预测"]
        B3["解决 Redo 问题"]
    end

    subgraph 要点3["要点3: 两种执行方式"]
        C1["GE 内 GC: 随 GE 预测"]
        C2["独立 GC: 需要 PredictionKey"]
        C3["都通过多播同步"]
    end

    要点1 --> 要点2 --> 要点3
```

---

## 课后练习

### 练习1：创建自定义 GC

1. 创建继承 GameplayCueNotify_Static 的类
2. 实现打击特效和音效
3. 在 Ability 中调用 ExecuteGameplayCue

### 练习2：观察 GC 预测

1. 在 NetMulticast_InvokeGameplayCueExecuted 设置断点
2. 观察 PredictionKey 的状态
3. 验证 Redo 问题的解决

### 练习3：思考题

1. 为什么 GC 可以安全预测而属性修改需要增量预测？
2. 如果 GC 执行失败（预测成功但服务器失败），会有什么影响？
3. 如何处理 GC 的网络带宽优化？

---

## 下节课预告

```mermaid
flowchart LR
    L6["第6课: GameplayCue 预测"] --> L7["第7课: 预测窗口与 AbilityTask"]

    subgraph 第7课内容["第7课内容"]
        A["FScopedPredictionWindow 详解"]
        B["AbilityTask 中的预测"]
        C["跨帧预测处理"]
        D["常见 AbilityTask 示例"]
    end

    L7 --> 第7课内容
```

---

## 参考资料

- **源码**：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueInterface.h`
- **源码**：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCue_Types.h`
- **官方文档**：[Gameplay Cues](https://dev.epicgames.com/documentation/en-us/unreal-engine/gameplay-cues-in-unreal-engine)
