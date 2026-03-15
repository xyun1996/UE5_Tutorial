# 第10课：实战案例与调试技巧

> 学习时间：45-60分钟
> 难度：高级
> 前置知识：第1-9课内容

---

## 学习目标

完成本课后，你将能够：
1. 分析 Lyra 项目中的 GAS 预测实现
2. 掌握完整的调试技巧和工具
3. 了解预测系统的最佳实践
4. 对整个课程进行总结回顾

---

## 10.1 Lyra 项目分析

### 10.1.1 Lyra GAS 架构

```mermaid
flowchart TB
    subgraph LyraGAS["Lyra GAS 架构"]
        ASC["ULyraAbilitySystemComponent"]
        Ability["ULyraGameplayAbility"]
        AttributeSet["ULyraAttributeSet"]
        GE["UGameplayEffect"]
    end

    subgraph 特点["Lyra 特点"]
        T1["输入标签驱动激活"]
        T2["自定义 ASC 逻辑"]
        T3["Hero 和 NPC 共用系统"]
    end

    LyraGAS --> 特点
```

### 10.1.2 LyraAbilitySystemComponent

```mermaid
classDiagram
    class UAbilitySystemComponent {
        +TryActivateAbility()
        +ServerTryActivateAbility()
        +ClientActivateAbilityFailed()
    }

    class ULyraAbilitySystemComponent {
        +TMap~FGameplayTag, FGameplayAbilitySpecHandle~ InputTagSpecHandles
        +Input_AbilityInputTagPressed()
        +Input_AbilityInputTagReleased()
        +ProcessAbilityInput()
    }

    UAbilitySystemComponent <|-- ULyraAbilitySystemComponent

    note for ULyraAbilitySystemComponent "扩展了输入处理<br/>支持标签驱动的激活"
```

### 10.1.3 Lyra 输入处理流程

```mermaid
sequenceDiagram
    participant Player as 玩家
    participant Input as 输入系统
    participant ASC as LyraASC
    participant Ability as Ability
    participant Server as 服务器

    Note over Player,Server: Lyra 输入驱动激活

    Player->>Input: 按下技能键
    Input->>ASC: Input_AbilityInputTagPressed(Tag)

    ASC->>ASC: 查找匹配 Tag 的 AbilitySpec

    loop 每个 AbilitySpec
        ASC->>Ability: TryActivateAbility()
        Ability->>Ability: 生成 PredictionKey
        Ability->>Server: ServerTryActivateAbility(Key)
        Ability->>Ability: 本地预测执行
    end

    Note over Server: 服务器验证

    Server->>ASC: 验证结果
    ASC->>Ability: 成功/失败响应
```

### 10.1.4 LyraGameplayAbility

```mermaid
classDiagram
    class UGameplayAbility {
        +ActivateAbility()
        +EndAbility()
        +CanActivateAbility()
    }

    class ULyraGameplayAbility {
        +FGameplayTagContainer StartupInputTags
        +FGameplayTagContainer ActivationRequiredTags
        +FGameplayTagContainer ActivationBlockedTags
        +GetAbilityLevel()
        +OnGiveAbility()
    }

    UGameplayAbility <|-- ULyraGameplayAbility

    note for ULyraGameplayAbility "扩展了标签系统<br/>支持输入标签配置"
```

### 10.1.5 Lyra 示例 Ability 分析

```cpp
// LyraGame/AbilitySystem/Abilities/LyraGameplayAbility.h (简化版)

UCLASS()
class ULyraGameplayAbility : public UGameplayAbility
{
    GENERATED_BODY()

public:
    // 输入标签 - 决定哪个输入激活此 Ability
    UPROPERTY(EditDefaultsOnly, Category = "Lyra|Input")
    FGameplayTagContainer StartupInputTags;

    // 激活所需的标签
    UPROPERTY(EditDefaultsOnly, Category = "Lyra|Activation")
    FGameplayTagContainer ActivationRequiredTags;

    // 阻止激活的标签
    UPROPERTY(EditDefaultsOnly, Category = "Lyra|Activation")
    FGameplayTagContainer ActivationBlockedTags;

    // Cost GE
    UPROPERTY(EditDefaultsOnly, Category = "Lyra|Cost")
    TSubclassOf<UGameplayEffect> CostGameplayEffectClass;

    // Cooldown GE
    UPROPERTY(EditDefaultsOnly, Category = "Lyra|Cooldown")
    TSubclassOf<UGameplayEffect> CooldownGameplayEffectClass;

protected:
    virtual bool CheckCost(...) const override;
    virtual void ApplyCost(...) override;
    virtual bool CheckCooldown(...) const override;
    virtual void ApplyCooldown(...) override;
};
```

---

## 10.2 调试技巧总结

### 10.2.1 关键断点位置

```mermaid
flowchart TB
    subgraph 预测键["预测键断点"]
        PK1["FPredictionKey::CreateNewPredictionKey<br/>键生成"]
        PK2["FPredictionKey::NetSerialize<br/>序列化"]
        PK3["GenerateDependentPredictionKey<br/>依赖键生成"]
    end

    subgraph 预测窗口["预测窗口断点"]
        PW1["FScopedPredictionWindow 构造<br/>窗口开始"]
        PW2["FScopedPredictionWindow 析构<br/>窗口结束"]
    end

    subgraph 激活流程["激活流程断点"]
        AF1["TryActivateAbility<br/>客户端入口"]
        AF2["ServerTryActivateAbility<br/>服务器入口"]
        AF3["ClientActivateAbilityFailed<br/>失败响应"]
    end

    subgraph GE["GE 断点"]
        GE1["ApplyGameplayEffectSpecToSelf<br/>应用 GE"]
        GE2["PostReplicatedAdd<br/>GE 复制"]
        GE3["PreReplicatedRemove<br/>GE 移除"]
    end

    subgraph 委托["委托断点"]
        DL1["BroadcastRejectedDelegate<br/>拒绝广播"]
        DL2["BroadcastCaughtUpDelegate<br/>追上广播"]
        DL3["AddDependency<br/>依赖注册"]
    end
```

### 10.2.2 日志配置

```cpp
// 推荐的日志配置

// 在 GameplayPrediction.cpp 开头添加
DEFINE_LOG_CATEGORY_STATIC(LogGameplayPrediction, Log, All);

// 预测键生成日志
UE_LOG(LogGameplayPrediction, Log,
       TEXT("CreateNewPredictionKey: Key=%d, ASC=%s"),
       NewKey.Current, *ASC->GetName());

// 预测窗口日志
UE_LOG(LogGameplayPrediction, Log,
       TEXT("PredictionWindow: %s, Key=%s"),
       bIsStart ? TEXT("START") : TEXT("END"),
       *PredictionKey.ToString());

// 委托触发日志
UE_LOG(LogGameplayPrediction, Warning,
       TEXT("BroadcastRejectedDelegate: Key=%d, Delegates=%d"),
       Key, KeyDelegates->RejectedDelegates.Num());

// GE 应用日志
UE_LOG(LogGameplayPrediction, Log,
       TEXT("ApplyGE: %s, Key=%s, Duration=%f"),
       *GEClass->GetName(), *PredictionKey.ToString(), Duration);
```

### 10.2.3 控制台命令

| 命令 | 用途 |
|-----|------|
| `stat net` | 显示网络统计 |
| `net.PK.Draw 1` | 显示预测键调试信息 |
| `AbilitySystem.Debug 1` | 显示 GAS 调试信息 |
| `net.Replication.Debug 1` | 显示复制信息 |
| `log LogGameplayAbilities verbose` | 详细 GAS 日志 |

### 10.2.4 调试流程

```mermaid
flowchart TB
    subgraph 步骤1["步骤1: 复现问题"]
        A1["确定问题场景"]
        A2["记录复现步骤"]
        A3["确定预期行为"]
    end

    subgraph 步骤2["步骤2: 设置断点"]
        B1["根据问题类型选择断点"]
        B2["添加必要的日志"]
        B3["准备网络环境"]
    end

    subgraph 步骤3["步骤3: 分析"]
        C1["观察预测键流程"]
        C2["检查委托触发"]
        C3["对比客户端/服务器状态"]
    end

    subgraph 步骤4["步骤4: 定位问题"]
        D1["确定问题环节"]
        D2["分析根本原因"]
        D3["设计解决方案"]
    end

    步骤1 --> 步骤2 --> 步骤3 --> 步骤4
```

---

## 10.3 最佳实践

### 10.3.1 Ability 设计原则

```mermaid
flowchart TB
    subgraph 原则1["原则1: 保持预测窗口短"]
        A1["避免跨帧操作"]
        A2["使用 AbilityTask 处理异步"]
        A3["每次异步创建新窗口"]
    end

    subgraph 原则2["原则2: 原子性操作"]
        B1["预测窗口内操作应逻辑原子"]
        B2["要么全部成功要么全部回滚"]
        B3["避免部分成功场景"]
    end

    subgraph 原则3["原则3: 使用 Tag 系统"]
        C1["用 Tag 表示状态"]
        C2["用 Tag 确保依赖关系"]
        C3["服务器验证 Tag"]
    end

    原则1 --> 原则2 --> 原则3
```

### 10.3.2 AttributeSet 设计

```mermaid
flowchart TB
    subgraph 必须实现["必须实现"]
        M1["REPNOTIFY_Always"]
        M2["GAMEPLAYATTRIBUTE_REPNOTIFY"]
        M3["GetLifetimeReplicatedProps"]
    end

    subgraph 推荐实现["推荐实现"]
        R1["PreAttributeChange 限制范围"]
        R2["PostGameplayEffectExecute 处理 Meta"]
        R3["属性变化事件"]
    end

    subgraph 避免["避免"]
        A1["直接修改 BaseValue"]
        A2["在预测窗口外修改属性"]
        A3["不使用 RepNotify"]
    end

    必须实现 --> 推荐实现 --> 避免
```

### 10.3.3 GameplayEffect 设计

```mermaid
flowchart TB
    subgraph DurationGE["Duration GE"]
        D1["可以预测"]
        D2["存储 PredictionKey"]
        D3["适合 Buff/Debuff"]
    end

    subgraph InstantGE["Instant GE"]
        I1["转换为无限持续预测"]
        I2["服务器确认后移除"]
        I3["适合一次性效果"]
    end

    subgraph 注意事项["注意事项"]
        N1["避免预测 GE 移除"]
        N2["避免预测周期效果"]
        N3["Execution 不可预测"]
    end

    DurationGE --> 注意事项
    InstantGE --> 注意事项
```

### 10.3.4 GameplayCue 设计

```mermaid
flowchart TB
    subgraph 最佳实践["GC 最佳实践"]
        G1["GC 属于表现层"]
        G2["不影响游戏逻辑"]
        G3["预测失败影响较小"]
    end

    subgraph 实现["实现建议"]
        I1["使用 PredictionKey 识别"]
        I2["检查 IsLocalClientKey"]
        I3["避免重复播放"]
    end

    subgraph 性能["性能考虑"]
        P1["控制 GC 数量"]
        P2["使用对象池"]
        P3["异步加载资源"]
    end

    最佳实践 --> 实现 --> 性能
```

---

## 10.4 常见问题解决

### 10.4.1 预测不生效

```mermaid
flowchart TB
    subgraph 问题["问题: 预测不生效"]
        P["客户端操作没有立即执行"]
    end

    subgraph 检查["检查步骤"]
        C1["是否有有效 PredictionKey?"]
        C2["是否在预测窗口内?"]
        C3["是否是 Autonomous Proxy?"]
    end

    subgraph 解决["解决方案"]
        S1["确保在预测窗口内操作"]
        S2["检查网络角色"]
        S3["检查 Ability 激活流程"]
    end

    问题 --> 检查 --> 解决
```

### 10.4.2 重复执行

```mermaid
flowchart TB
    subgraph 问题["问题: 效果重复执行"]
        P["特效播放两次或更多"]
    end

    subgraph 检查["检查步骤"]
        C1["PredictionKey 是否正确传递?"]
        C2["IsLocalClientKey 检查是否存在?"]
        C3["NetSerialize 是否正确实现?"]
    end

    subgraph 解决["解决方案"]
        S1["确保 PredictionKey 关联到 GE/GC"]
        S2["添加 IsLocalClientKey 检查"]
        S3["检查网络序列化"]
    end

    问题 --> 检查 --> 解决
```

### 10.4.3 回滚不完整

```mermaid
flowchart TB
    subgraph 问题["问题: 回滚不完整"]
        P["预测失败后状态未恢复"]
    end

    subgraph 检查["检查步骤"]
        C1["委托是否正确注册?"]
        C2["回滚逻辑是否正确?"]
        C3["是否有遗漏的副作用?"]
    end

    subgraph 解决["解决方案"]
        S1["检查 FPredictionKeyDelegates 注册"]
        S2["确保所有副作用注册委托"]
        S3["检查委托执行顺序"]
    end

    问题 --> 检查 --> 解决
```

### 10.4.4 状态不同步

```mermaid
flowchart TB
    subgraph 问题["问题: 客户端服务器状态不同步"]
        P["客户端和服务器状态不一致"]
    end

    subgraph 检查["检查步骤"]
        C1["属性 RepNotify 是否正确?"]
        C2["GE 是否正确复制?"]
        C3["网络是否丢包?"]
    end

    subgraph 解决["解决方案"]
        S1["检查 REPNOTIFY_Always"]
        S2["检查 GE 复制条件"]
        S3["检查网络连接"]
    end

    问题 --> 检查 --> 解决
```

---

## 10.5 课程总结

### 10.5.1 知识体系回顾

```mermaid
mindmap
  root((GAS 预测系统))
    基础概念
      网络延迟问题
      客户端预测原理
      6个核心问题
    核心机制
      FPredictionKey
      FScopedPredictionWindow
      FPredictionKeyDelegates
    预测内容
      Ability 激活
      GameplayEffect
      属性修改
      GameplayCue
    高级话题
      依赖链
      AbilityTask
      系统限制
    实践技能
      调试技巧
      最佳实践
      问题解决
```

### 10.5.2 核心概念总结

```mermaid
flowchart TB
    subgraph 预测键["FPredictionKey"]
        K1["唯一标识预测操作"]
        K2["定向序列化"]
        K3["支持依赖链"]
    end

    subgraph 预测窗口["FScopedPredictionWindow"]
        W1["控制键的生命周期"]
        W2["窗口内操作自动关联"]
        W3["窗口结束确认键"]
    end

    subgraph 委托系统["FPredictionKeyDelegates"]
        D1["Rejected 回滚"]
        D2["CaughtUp 清理"]
        D3["Dependency 依赖"]
    end

    预测键 --> 预测窗口 --> 委托系统
```

### 10.5.3 关键流程总结

```mermaid
sequenceDiagram
    participant C as 客户端
    participant K as PredictionKey
    participant W as 预测窗口
    participant S as 服务器
    participant D as 委托

    Note over C,D: 完整预测流程

    C->>K: 1. 生成预测键
    C->>W: 2. 开始预测窗口
    C->>S: 3. 发送 RPC (携带 Key)
    C->>C: 4. 本地预测执行
    C->>D: 5. 注册回滚委托
    C->>W: 6. 结束预测窗口

    S->>S: 7. 验证并执行

    alt 成功
        S->>C: 8a. 属性复制
        C->>D: 9a. CatchUp 委托
        C->>C: 10a. 清理预测状态
    else 失败
        S->>C: 8b. ClientActivateAbilityFailed
        C->>D: 9b. Rejected 委托
        C->>C: 10b. 回滚预测状态
    end
```

### 10.5.4 学习路径回顾

```mermaid
flowchart LR
    L1["第1课<br/>网络基础"] --> L2["第2课<br/>预测键结构"]
    L2 --> L3["第3课<br/>激活流程"]
    L3 --> L4["第4课<br/>GE预测"]
    L4 --> L5["第5课<br/>属性预测"]
    L5 --> L6["第6课<br/>GC预测"]
    L6 --> L7["第7课<br/>预测窗口"]
    L7 --> L8["第8课<br/>依赖链"]
    L8 --> L9["第9课<br/>高级话题"]
    L9 --> L10["第10课<br/>实战总结"]
```

---

## 10.6 进一步学习资源

### 10.6.1 官方资源

| 资源 | 链接 |
|-----|------|
| 官方文档 | https://dev.epicgames.com/documentation/en-us/unreal-engine/gameplay-ability-system-in-unreal-engine |
| Lyra 示例项目 | Engine/Samples/Games/Lyra |
| 源码 | Engine/Plugins/Runtime/GameplayAbilities |

### 10.6.2 社区资源

| 资源 | 说明 |
|-----|------|
| Unreal Forums | 官方论坛 GAS 板块 |
| Reddit r/unrealengine | 社区讨论 |
| GitHub Issues | 源码问题追踪 |

### 10.6.3 推荐实践项目

```mermaid
flowchart TB
    subgraph 入门["入门项目"]
        E1["简单技能系统"]
        E2["基础 Buff/Debuff"]
        E3["属性修改"]
    end

    subgraph 进阶["进阶项目"]
        A1["连击系统"]
        A2["复杂 AbilityTask"]
        A3["自定义 AttributeSet"]
    end

    subgraph 高级["高级项目"]
        H1["完整战斗系统"]
        H2["多人竞技游戏"]
        H3["网络优化"]
    end

    入门 --> 进阶 --> 高级
```

---

## 10.7 结语

### 10.7.1 学习建议

```mermaid
flowchart TB
    subgraph 建议["学习建议"]
        B1["从简单开始，逐步深入"]
        B2["多阅读源码注释"]
        B3["动手实践，调试观察"]
        B4["理解原理，举一反三"]
    end

    subgraph 技巧["学习技巧"]
        T1["画流程图帮助理解"]
        T2["添加日志观察执行"]
        T3["断点调试跟踪状态"]
        T4["对比正常/异常情况"]
    end

    建议 --> 技巧
```

### 10.7.2 课程完成

恭喜你完成了 GAS 预测系统的完整学习！

```mermaid
flowchart LR
    Start["开始学习"] --> Complete["课程完成!"]

    subgraph 收获["你的收获"]
        G1["理解预测原理"]
        G2["掌握核心机制"]
        G3["学会调试技巧"]
        G4["了解最佳实践"]
    end

    Complete --> 收获
```

---

## 附录

### A. 源码文件索引

| 文件 | 主要内容 |
|-----|---------|
| `GameplayPrediction.h` | 预测系统核心定义和注释 |
| `GameplayPrediction.cpp` | 预测系统实现 |
| `AbilitySystemComponent.h` | ASC 定义 |
| `AbilitySystemComponent_Abilities.cpp` | Ability 激活实现 |
| `GameplayEffect.h` | GE 定义 |
| `GameplayEffect.cpp` | GE 实现 |
| `AttributeSet.h` | AttributeSet 定义 |
| `AttributeSet.cpp` | AttributeSet 实现 |

### B. 调试命令汇总

```
// 网络调试
stat net
net.PK.Draw 1
net.Replication.Debug 1

// GAS 调试
AbilitySystem.Debug 1

// 日志级别
log LogGameplayAbilities verbose
log LogGameplayPrediction verbose
```

### C. 常用断点位置

| 断点位置 | 用途 |
|---------|------|
| `FPredictionKey::CreateNewPredictionKey` | 观察键生成 |
| `FScopedPredictionWindow::构造/析构` | 观察窗口生命周期 |
| `TryActivateAbility` | 观察激活入口 |
| `ServerTryActivateAbility_Implementation` | 观察服务器处理 |
| `ClientActivateAbilityFailed_Implementation` | 观察失败响应 |
| `BroadcastRejectedDelegate` | 观察拒绝广播 |
| `BroadcastCaughtUpDelegate` | 观察追上广播 |
| `PostReplicatedAdd` | 观察 GE 复制 |
| `OnRep_Attribute` | 观察属性复制 |

---

## 课程完成！

你已经完成了 GAS 预测系统的 10 节课程学习。

### 教程文件列表

```
Tutorial/
├── GAS_Prediction_System_Guide.md           # 总体学习指南
├── GAS_Prediction_Course_Outline.md         # 课程大纲
├── Lesson01_GAS_Network_Basics_and_Prediction_Overview.md
├── Lesson02_FPredictionKey_Core_Structure.md
├── Lesson03_Ability_Activation_Prediction_Flow.md
├── Lesson04_GameplayEffect_Prediction.md
├── Lesson05_Attribute_Prediction_and_Rollback.md
├── Lesson06_GameplayCue_Prediction.md
├── Lesson07_Prediction_Window_and_AbilityTask.md
├── Lesson08_Dependency_Chain_and_Chained_Activation.md
├── Lesson09_Advanced_Topics_and_Limitations.md
└── Lesson10_Practical_Cases_and_Debugging.md
```

祝你在 UE5 开发中取得成功！
