# 第9课：高级话题与限制

> 学习时间：45-60分钟
> 难度：高级
> 前置知识：第1-8课内容

---

## 学习目标

完成本课后，你将能够：
1. 理解当前预测系统的限制
2. 了解弱预测的概念和应用场景
3. 掌握 Meta 属性和百分比效果的预测问题
4. 了解未来可能的改进方向

---

## 9.1 当前预测系统限制总览

### 9.1.1 不可预测内容

```mermaid
mindmap
  root((不可预测))
    GameplayEffect 移除
      需要追踪历史
      回滚复杂
    周期效果 DOT
      时间同步困难
      丢包处理复杂
    Execution 计算
      自定义逻辑
      难以预测
    Meta 属性
      只支持即时效果
      无状态存储
```

### 9.1.2 已知问题

```mermaid
flowchart TB
    subgraph 问题1["问题1: 链式激活回滚"]
        P1["触发事件的链式回滚不完整"]
        P2["需要使用 Tag 系统辅助"]
    end

    subgraph 问题2["问题2: 百分比效果"]
        P3["客户端缺少完整修改器链"]
        P4["计算结果可能不准确"]
    end

    subgraph 问题3["问题3: 弱预测"]
        P5["某些场景无法建立预测键"]
        P6["弱预测模式尚未实现"]
    end

    问题1 --> 问题2 --> 问题3
```

---

## 9.2 GameplayEffect 移除

### 9.2.1 为什么不可预测

```mermaid
flowchart TB
    subgraph 移除操作["GE 移除操作"]
        A["RemoveActiveGameplayEffect()"]
        B["需要指定要移除的 GE"]
        C["需要知道 GE 的来源"]
    end

    subgraph 预测困难["预测困难"]
        D["客户端不知道服务器有哪些 GE"]
        E["GE 可能已被服务器修改"]
        F["移除后无法恢复"]
    end

    subgraph 结果["结果"]
        G["GE 移除不可预测"]
        H["必须等待服务器确认"]
    end

    移除操作 --> 预测困难 --> 结果
```

### 9.2.2 可能的解决方案

```mermaid
flowchart TB
    subgraph 方案1["方案1: 记录 GE 历史"]
        A1["存储所有 GE 操作"]
        A2["支持回滚移除"]
        A3["问题: 内存开销大"]
    end

    subgraph 方案2["方案2: 标记移除"]
        B1["预测时标记为移除"]
        B2["服务器确认后真正移除"]
        B3["问题: GE 仍在列表中"]
    end

    subgraph 方案3["方案3: 使用 Duration"]
        C1["用 Duration GE 替代"]
        C2["设置很短的持续时间"]
        C3["自动过期移除"]
    end

    方案1 --> 方案2 --> 方案3
```

### 9.2.3 源码注释

```cpp
// GameplayPrediction.h 第47-49行

/**
 * Some things we don't predict (most of these we potentially could, but currently dont):
 * - GameplayEffect removal
 * - GameplayEffect periodic effects (dots ticking)
 */
```

---

## 9.3 周期效果（DOT）

### 9.3.1 周期效果的工作方式

```mermaid
sequenceDiagram
    participant GE as Duration GE
    participant Timer as 周期计时器
    participant Attr as 属性
    participant Server as 服务器

    Note over GE,Server: 周期效果执行

    GE->>Timer: 设置周期 (如每秒)
    Timer->>Attr: 第1次触发: -10 HP
    Timer->>Attr: 第2次触发: -10 HP
    Timer->>Attr: 第3次触发: -10 HP

    Note over Server: 服务器时间线

    Server->>Server: T=0: 应用 GE
    Server->>Server: T=1: 触发周期
    Server->>Server: T=2: 触发周期
    Server->>Server: T=3: 触发周期
```

### 9.3.2 预测困难

```mermaid
flowchart TB
    subgraph 时间同步["时间同步问题"]
        T1["客户端和服务器时间不同步"]
        T2["周期触发时机不一致"]
        T3["丢包导致触发丢失"]
    end

    subgraph 状态追踪["状态追踪问题"]
        S1["需要追踪每个周期触发"]
        S2["回滚需要撤销特定触发"]
        S3["状态复杂度高"]
    end

    subgraph 结果["结果"]
        R1["周期效果不可预测"]
        R2["客户端等待服务器触发"]
    end

    时间同步 --> 状态追踪 --> 结果
```

### 9.3.3 替代方案

```mermaid
flowchart LR
    subgraph 替代方案["替代方案"]
        A["使用多个 Instant GE"]
        B["AbilityTask 定时触发"]
        C["服务器驱动周期"]
    end

    subgraph 优点["优点"]
        D["每个触发独立预测"]
        E["时间由逻辑控制"]
        F["避免同步问题"]
    end

    替代方案 --> 优点
```

---

## 9.4 Meta 属性限制

### 9.4.1 Meta 属性特点

```mermaid
flowchart TB
    subgraph Meta属性["Meta 属性特点"]
        A["不复制到客户端"]
        B["只在即时 GE 中使用"]
        C["用于转换计算"]
        D["如: Damage → Health"]
    end

    subgraph 示例["示例"]
        E["Damage = 30 (Meta)"]
        F["Health -= Damage"]
        G["Damage = 0 (重置)"]
    end

    Meta属性 --> 示例
```

### 9.4.2 预测问题

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant GE as Instant GE
    participant Meta as Meta属性
    participant Health as Health
    participant Server as 服务器

    Note over Client,Server: Meta 属性预测问题

    Client->>GE: 预测应用 Damage GE
    Note over GE: Duration = INSTANT

    GE->>GE: 转换为无限持续
    Note over GE: 但 Meta 属性转换<br/>只在即时 GE 中触发

    Note over Meta: PostGameplayEffectExecute<br/>不会被调用!

    Note over Client: Damage → Health<br/>转换未执行

    Note over Server: 服务器执行

    Server->>GE: 正常即时 GE
    Server->>Meta: Damage = 30
    Server->>Health: Health -= 30

    Note over Client: 客户端预测失败
```

### 9.4.3 源码注释

```cpp
// GameplayPrediction.h 第229-234行

/**
 * We are unable to apply meta attributes predictively. Meta attributes only work on instant effects,
 * in the back end of GameplayEffect (Pre/Post Modify Attribute on the UAttributeSet).
 * These events are not called when applying duration-based gameplay effects.
 *
 * In order to support this, we would probably add some limited support for duration based meta attributes,
 * and move the transform of the instant gameplay effect from the front end
 * (UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf)
 * to the backend (UAttributeSet::PostModifyAttribute).
 */
```

### 9.4.4 解决方案

```mermaid
flowchart TB
    subgraph 方案1["方案1: 不预测 Meta 属性"]
        A1["等待服务器确认"]
        A2["接受延迟"]
        A3["简单可靠"]
    end

    subgraph 方案2["方案2: 直接修改目标属性"]
        B1["使用普通属性修改"]
        B2["不使用 Meta 属性"]
        B3["可以预测"]
    end

    subgraph 方案3["方案3: 客户端模拟"]
        C1["客户端本地计算"]
        C2["仅用于显示"]
        C3["服务器确认后覆盖"]
    end

    方案1 --> 方案2 --> 方案3
```

---

## 9.5 百分比效果问题

### 9.5.1 问题描述

```mermaid
flowchart TB
    subgraph 场景["场景"]
        A["基础移动速度 = 500"]
        B["已有 +10% buff → 550"]
        C["预测另一个 +10% buff"]
    end

    subgraph 问题["问题"]
        D["客户端计算: 550 * 1.1 = 605"]
        E["正确计算: 500 * 1.2 = 600"]
        F["结果不一致!"]
    end

    场景 --> 问题
```

### 9.5.2 根本原因

```mermaid
flowchart TB
    subgraph 服务器状态["服务器完整状态"]
        S1["基础值 = 500"]
        S2["修改器1: +10%"]
        S3["修改器2: +10%"]
        S4["最终值 = 500 * 1.2 = 600"]
    end

    subgraph 客户端状态["客户端不完整状态"]
        C1["只知道最终值 = 550"]
        C2["不知道修改器链"]
        C3["预测新修改器时<br/>基于错误的基准"]
    end

    服务器状态 -->|"缺少信息"| 客户端状态
```

### 9.5.3 源码注释

```cpp
// GameplayPrediction.h 第237-247行

/**
 * There are also limitations when predicting % based gameplay effects.
 * Since the server replicates down the 'final value' of an attribute,
 * but not the entire aggregator chain of what is modifying it,
 * we may run into cases where the client cannot accurately predict new gameplay effects.
 *
 * For example:
 * - Client has a perm +10% movement speed buff with base movement speed of 500 -> 550
 * - Client has an ability which grants an additional 10% movement speed buff.
 *   It is expected to *sum* the % based multipliers for a final 20% bonus to 500 -> 600.
 * - However on the client, we just apply a 10% buff to 550 -> 605.
 *
 * This will need to be fixed by replicating down the aggregator chain for attributes.
 */
```

### 9.5.4 可能的解决方案

```mermaid
flowchart TB
    subgraph 方案1["方案1: 复制聚合器链"]
        A1["服务器复制完整修改器链"]
        A2["客户端知道所有修改器"]
        A3["问题: 带宽开销大"]
    end

    subgraph 方案2["方案2: 使用加法而非乘法"]
        B1["+10% + +10% = +20%"]
        B2["避免乘法累积"]
        B3["需要设计时考虑"]
    end

    subgraph 方案3["方案3: 接受误差"]
        C1["小误差可接受"]
        C2["服务器确认后修正"]
        C3["适用于非关键属性"]
    end

    方案1 --> 方案2 --> 方案3
```

---

## 9.6 弱预测概念

### 9.6.1 弱预测场景

```mermaid
flowchart TB
    subgraph 场景["无法建立预测键的场景"]
        S1["碰撞触发的 GE"]
        S2["大量触发的效果"]
        S3["服务器无法处理大量 RPC"]
    end

    subgraph 问题["问题"]
        P1["无法发送每次碰撞的 RPC"]
        P2["无法建立预测键关联"]
        P3["无法准确同步"]
    end

    场景 --> 问题
```

### 9.6.2 弱预测定义

```mermaid
flowchart TB
    subgraph 强预测["强预测 (当前实现)"]
        A1["新鲜预测键"]
        A2["精确关联副作用"]
        A3["解决 Undo/Redo/Completeness"]
    end

    subgraph 弱预测["弱预测 (未实现)"]
        B1["无新鲜预测键"]
        B2["假设客户端预测所有副作用"]
        B3["只解决 Redo 问题"]
        B4["不解决 Completeness 问题"]
    end

    强预测 -->|"无法建立键时"| 弱预测
```

### 9.6.3 源码注释

```cpp
// GameplayPrediction.h 第250-263行

/**
 * We will probably still have cases that do not fit well into this system.
 * Some situations will exist where a prediction key exchange is not feasible.
 * For example, an ability where any one that player collides with/touches
 * receives a GameplayEffect that slows them and their material blue.
 * Since we can't send Server RPCs every time this happens
 * (and the server couldn't necessarily handle the message at its point in the simulation),
 * there is no way to correlate the gameplay effect side effects between client and server.
 *
 * One approach here may be to think about a weaker form of prediction.
 * One where there is not a fresh prediction key used and instead the server assumes
 * the client will predict all side effects from an entire ability.
 * This would at least solve the "redo" problem but would not solve the "completeness" problem.
 */
```

### 9.6.4 弱预测实现思路

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant Ability as Ability
    participant WeakPred as 弱预测模式
    participant Server as 服务器

    Note over Client,Server: 弱预测流程 (概念)

    Client->>Ability: 激活碰撞 Ability
    Ability->>WeakPred: 进入弱预测模式

    Note over WeakPred: 标记整个 Ability 为弱预测

    loop 每次碰撞
        Client->>Client: 本地预测 GE
        Note over Client: 不发送 RPC
    end

    Client->>Server: Ability 完成 RPC

    Server->>Server: 执行 Ability
    Server->>Server: 应用所有 GE

    Note over Server: 多播所有 GE

    Client->>Client: 收到服务器 GE
    Note over Client: 弱预测模式: 跳过已预测的 GC<br/>只执行未预测的部分
```

---

## 9.7 其他限制与注意事项

### 9.7.1 网络条件影响

```mermaid
flowchart TB
    subgraph 高延迟["高延迟场景"]
        H1["预测窗口可能已过期"]
        H2["服务器响应延迟"]
        H3["需要更大的预测窗口"]
    end

    subgraph 丢包["丢包场景"]
        L1["RPC 可能丢失"]
        L2["属性复制延迟"]
        L3["状态不一致风险"]
    end

    subgraph 建议["设计建议"]
        S1["关键操作使用可靠 RPC"]
        S2["非关键操作可接受丢失"]
        S3["设计时考虑网络条件"]
    end

    高延迟 --> 建议
    丢包 --> 建议
```

### 9.7.2 性能考虑

```mermaid
flowchart TB
    subgraph 预测开销["预测开销"]
        O1["委托注册和管理"]
        O2["副作用追踪"]
        O3["回滚逻辑执行"]
    end

    subgraph 优化建议["优化建议"]
        R1["避免过多预测操作"]
        R2["及时清理完成的预测"]
        R3["使用对象池管理委托"]
    end

    预测开销 --> 优化建议
```

### 9.7.3 调试难度

```mermaid
flowchart TB
    subgraph 调试挑战["调试挑战"]
        D1["网络时序复杂"]
        D2["状态难以复现"]
        D3["多客户端同步"]
    end

    subgraph 调试工具["调试工具"]
        T1["详细日志输出"]
        T2["断点调试"]
        T3["网络分析工具"]
        T4["回放系统"]
    end

    调试挑战 --> 调试工具
```

---

## 9.8 未来改进方向

### 9.8.1 源码中的 TODO

```mermaid
flowchart TB
    subgraph 改进1["改进1: 触发事件复制"]
        I1["支持触发事件显式复制"]
        I2["跨玩家/AI 事件支持"]
    end

    subgraph 改进2["改进2: 聚合器链复制"]
        I3["复制完整修改器链"]
        I4["解决百分比效果问题"]
    end

    subgraph 改进3["改进3: 弱预测模式"]
        I5["实现弱预测框架"]
        I6["支持无预测键场景"]
    end

    subgraph 改进4["改进4: Meta 属性支持"]
        I7["支持持续 Meta 属性"]
        I8["改进转换逻辑位置"]
    end

    改进1 --> 改进2 --> 改进3 --> 改进4
```

### 9.8.2 社区贡献方向

```mermaid
mindmap
  root((未来改进))
    性能优化
      减少委托开销
      优化内存使用
    功能扩展
      弱预测模式
      GE 移除预测
    工具支持
      可视化调试
      状态检查工具
    文档完善
      最佳实践
      常见问题解决
```

---

## 9.9 总结

### 9.9.1 核心概念图

```mermaid
mindmap
  root((限制与高级话题))
    不可预测内容
      GE 移除
      周期效果
      Execution
      Meta 属性
    已知问题
      链式回滚不完整
      百分比效果误差
      弱预测未实现
    解决方案
      Tag 系统辅助
      设计规避
      接受限制
    未来方向
      聚合器链复制
      弱预测模式
      触发事件复制
```

### 9.9.2 设计建议

```mermaid
flowchart TB
    subgraph 建议1["建议1: 了解限制"]
        A1["知道什么不能预测"]
        A2["设计时规避限制"]
    end

    subgraph 建议2["建议2: 使用替代方案"]
        B1["用 Tag 确保依赖"]
        B2["用 AbilityTask 处理异步"]
        B3["用 Duration GE 替代移除"]
    end

    subgraph 建议3["建议3: 接受权衡"]
        C1["某些延迟可接受"]
        C2["某些误差可接受"]
        C3["优先保证正确性"]
    end

    建议1 --> 建议2 --> 建议3
```

---

## 课后练习

### 练习1：识别限制场景

1. 列出你项目中需要预测的功能
2. 检查是否有不可预测的内容
3. 设计替代方案

### 练习2：处理百分比效果

1. 创建一个百分比 buff 系统
2. 观察客户端和服务器计算差异
3. 思考如何减少误差

### 练习3：思考题

1. 为什么 GE 移除比 GE 应用更难预测？
2. 弱预测模式适用于什么场景？
3. 如何在项目中规避这些限制？

---

## 下节课预告

```mermaid
flowchart LR
    L9["第9课: 高级话题与限制"] --> L10["第10课: 实战案例与调试技巧"]

    subgraph 第10课内容["第10课内容"]
        A["Lyra 项目分析"]
        B["调试技巧总结"]
        C["最佳实践"]
        D["课程总结"]
    end

    L10 --> 第10课内容
```

---

## 参考资料

- **源码**：`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayPrediction.h` (第218-263行)
- **官方文档**：[Gameplay Ability System](https://dev.epicgames.com/documentation/en-us/unreal-engine/gameplay-ability-system-in-unreal-engine)
- **社区讨论**：[Unreal Engine Forums](https://forums.unrealengine.com/)
