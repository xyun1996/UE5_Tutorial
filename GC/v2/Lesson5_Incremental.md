# Lesson 5: 增量 GC 的真实开关与运行方式

## 本节定位

incremental 是 GC 课程里最容易被“名词理解了, 机制没理解”的部分。  
如果只记一句:

> 增量 GC = 分帧执行

那你后面会解释不了这些问题:

- 为什么增量期间要有 barrier
- 为什么 incremental reachability 和 incremental purge 不是一回事
- 为什么有时开了增量还是会卡
- 为什么 pending GC 可能被强制收尾

这一节就是专门解决这些问题的。

---

## 学习目标

学完本节你应该能掌握:

- “增量”在 UE5 GC 里至少分哪几层
- `FReachabilityAnalysisState` 在管理什么
- `gc.AllowIncrementalReachability`、`gc.AllowIncrementalGather`、`gc.IncrementalBeginDestroyEnabled` 分别控制哪段流程
- UObject 侧 reachable barrier 的基本意义
- 为什么 incremental 是摊平尖峰, 而不是消灭所有 GC 代价

---

## 一、先把“增量”拆开, 不要混

在 UE5 GC 里, “incremental” 至少有三层含义:

### 1.1 增量 Reachability Analysis

也就是可达性分析分多轮、多帧推进。

### 1.2 增量 Gather / Unhash

不可达对象的收集、解绑等后段工作分批推进。

### 1.3 增量 BeginDestroy / Purge

已经判死对象的销毁与清理分批推进。

### 1.4 为什么一定要拆开学

因为它们分别解决不同问题:

- traversal 尖峰
- gather 尖峰
- destroy / purge 尖峰

如果你把它们混成“增量 GC 总开关”, 几乎一定会调错方向。

---

## 二、增量 Reachability 的正式 API

### 2.1 源码位置

- `Engine/Source/Runtime/CoreUObject/Public/UObject/ReachabilityAnalysis.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectGlobals.h`

### 2.2 公开接口

```cpp
void SetIncrementalReachabilityAnalysisEnabled(bool bEnabled);
bool GetIncrementalReachabilityAnalysisEnabled();
void SetReachabilityAnalysisTimeLimit(float TimeLimitSeconds);
float GetReachabilityAnalysisTimeLimit();

bool IsIncrementalReachabilityAnalysisPending();
void PerformIncrementalReachabilityAnalysis(double TimeLimit);
void FinalizeIncrementalReachabilityAnalysis();
```

### 2.3 从这些 API 你应该读出什么

这说明增量 reachability:

- 是正式对外能力
- 有启用状态
- 有时间预算
- 有 pending 概念
- 能继续推进
- 能显式 finalize

所以它不是“GC 内部偷偷做的优化”, 而是一个明确可控的机制。

---

## 三、`FReachabilityAnalysisState` 是增量的状态核心

### 3.1 源码位置

- `Engine/Source/Runtime/CoreUObject/Private/UObject/ReachabilityAnalysisState.h`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/ReachabilityAnalysisState.cpp`

### 3.2 这个类大致记住哪些状态

当前版本中, 它至少维护:

- worker contexts
- `ObjectKeepFlags`
- 本轮是否 full purge
- 当前是否 suspended
- iteration 时间预算
- 累积 traversal 时间
- iteration 次数

### 3.3 这告诉我们什么

增量 reachability 不是:

- 每一帧都“重新从头扫描一遍”

而是:

- 一轮 traversal 被暂停
- 保留上下文
- 下一轮继续推进

### 3.4 这是本节最重要的认识之一

你必须把增量 GC 理解成:

> 带状态机的分阶段推进

而不是:

> 简单地把一个大循环切成几段

---

## 四、增量 Reachability 的 CVar

在 `GarbageCollection.cpp` 中你会看到:

- `gc.AllowIncrementalReachability`
- `gc.IncrementalReachabilityTimeLimit`

### 4.1 `gc.AllowIncrementalReachability`

控制可达性分析是否允许分帧进行。

### 4.2 `gc.IncrementalReachabilityTimeLimit`

控制单次迭代的时间预算, 单位是秒。

### 4.3 它不是“整轮 GC 总预算”

这一点必须记牢。  
它更接近:

- “这一小轮 reachability 最多跑多久”

而不是:

- “这一轮 GC 总共就这么多时间”

---

## 五、为什么 full purge 和 incremental reachability 不能混成一个概念

如果你调用:

```cpp
CollectGarbage(..., true);
```

这意味着:

- 本轮是 full purge 路径

而 full purge 和增量 reachability 的目标并不一致。  
所以即使你开了:

- `gc.AllowIncrementalReachability`

也不代表每一轮 GC 都会慢慢分帧做 traversal。

### 5.1 正确理解

- full purge: 更强调本轮尽量把后续流程完整收掉
- incremental reachability: 更强调前半段遍历分摊到多帧

这是两个正交维度, 不是简单对立项。

---

## 六、为什么增量 GC 的难点不在“切片”, 而在“图还在变化”

### 6.1 看一个简单时序

```text
Frame 1:
  GC 开始做 Reachability

Frame 2:
  游戏逻辑又创建了一个新引用

Frame 3:
  GC 继续做 Reachability
```

问题来了:

- Frame 2 新增的那条引用怎么办?

如果 GC 完全不知道, 它就可能错杀对象。

### 6.2 这说明了什么

incremental 的真正难点不是:

- 把工作切成几块

而是:

- 在工作切开的同时, 保证对象图变化不会破坏正确性

这就是 barrier 机制存在的原因。

---

## 七、UObject 侧的 reachable barrier

### 7.1 源码位置

- `Engine/Source/Runtime/CoreUObject/Public/UObject/ObjectPtr.h`

### 7.2 关键条件

当:

```cpp
UE::GC::GIsIncrementalReachabilityPending
```

为真时, `FObjectPtr` / `TObjectPtr` 的某些路径会触发:

```cpp
UE::GC::MarkAsReachable(...)
```

### 7.3 教学理解

这相当于说:

- “GC 现在还在途中”
- “如果你刚刚建立了重要引用, 我需要把这个对象重新纳入 reachable 修正”

### 7.4 为什么这件事极其关键

因为它解释了:

- 为什么 `TObjectPtr` 在现代 UE5 语境下更重要
- 为什么增量 GC 不是只靠时间切片就能成立

---

## 八、Verse 的 `TWriteBarrier` 不是 UObject barrier 的同义词

当前源码中, Verse VM 也有自己的 `TWriteBarrier`。  
但课程里一定要避免偷懒讲法:

> UE5 的增量 GC 就是靠 `TWriteBarrier`

更准确的表述是:

- UObject 世界有自己的 reachable 修正机制
- Verse 世界有自己的 barrier 类型
- 在桥接 / Franken GC 场景里它们会协同
- 但概念上不是同一个东西

这点对后面读 Verse 相关代码时很重要。

---

## 九、增量 Gather 在解决什么问题

在 `GarbageCollection.cpp` 中你还能看到:

- `gc.AllowIncrementalGather`
- `gc.IncrementalGatherTimeLimit`

### 9.1 它对应的是哪段流程

是 reachability 之后、destroy 之前的一些后段处理:

- gather unreachable objects
- unhash / 解绑
- 某些后续准备工作

### 9.2 它为什么也需要增量

因为哪怕 traversal 已经不慢:

- 如果本轮不可达对象很多
- gather / unhash 仍然可能形成尖峰

所以后段也要切片。

---

## 十、增量 BeginDestroy / Purge 在解决什么问题

相关 CVar:

- `gc.IncrementalBeginDestroyEnabled`
- `gc.IncrementalBeginDestroyGranularity`

### 10.1 它针对的是“处理死对象”的成本

对象已经不可达以后, 问题就不再是:

- 它活不活

而是:

- 怎么把它平滑地清理掉

### 10.2 `Granularity` 要怎么理解

它控制的是:

- 每处理多少个对象, 检查一次时间预算

如果太大:

- 时间片不够细

如果太小:

- 调度开销上升

这就是很典型的工程折中参数。

---

## 十一、一个完整的增量 GC 心智模型

推荐用下面这张抽象图理解:

```text
Frame N:
  开始 GC
  MarkObjectsAsUnreachable
  Reachability iteration #1
  超时 -> suspend

Frame N+1:
  游戏逻辑继续运行
  新引用产生
  barrier 修正 reachable 集合
  Reachability iteration #2

Frame N+2:
  Reachability 完成
  Incremental Gather
  Incremental BeginDestroy
  Incremental Purge
```

### 11.1 这张图最重要的不是“跨了三帧”

而是:

- 图在运行中变化
- GC 还要持续保持正确性

---

## 十二、为什么 incremental 不是“永不卡”的保证

### 12.1 pending GC 可能被强制收尾

如果当前已经有 pending incremental reachability:

- 又来了新的 GC 请求

引擎可能会:

- 先把当前 pending 的流程收完
- 再开始下一轮

### 12.2 这意味着什么

incremental 的价值是:

- 摊平尖峰

不是:

- 彻底消灭所有 GC 成本

### 12.3 full purge 依然有存在价值

某些场景下 full purge 仍然合理:

- 切关
- 大资源卸载后
- 返回大厅
- 编辑器空闲阶段

不要把“现代 UE5”理解成“以后永远都不该做 full purge”。

---

## 十三、实践中的调优顺序

推荐顺序:

1. 先查对象数量和引用图规模
2. 再确认 traversal 是否是主瓶颈
3. 再开启 parallel / incremental reachability
4. 再看 gather / begin destroy / purge 是否也要切片
5. 最后再精调时间预算

### 13.1 为什么顺序不能反过来

如果你先调预算:

- 很可能只是把问题挪位置
- 而对象图本身没有变好

---

## 十四、如何配合源码阅读这一课

建议顺序:

1. `ReachabilityAnalysis.h`
   先看公开 API
2. `ReachabilityAnalysisState.h`
   看状态成员
3. `ReachabilityAnalysisState.cpp`
   看暂停 / 继续逻辑
4. `ObjectPtr.h`
   看 incremental 期间的 reachable barrier
5. `GarbageCollection.cpp`
   看 incremental CVar 与流程衔接

### 14.1 阅读时建议带着这几个问题

- 哪一段表示“本轮 traversal 还没结束”?
- 哪一段负责继续上一轮?
- 哪一段在 incremental 期间修正新 reachable 对象?
- gather 和 begin destroy 是怎么分别切片的?

---

## 十五、本节后的自检题

1. “增量 GC = 分帧执行”为什么不够准确?
2. `FReachabilityAnalysisState` 存在的核心意义是什么?
3. 为什么增量期间必须有 reachable barrier?
4. `gc.AllowIncrementalGather` 和 `gc.AllowIncrementalReachability` 分别作用在哪一段?
5. 为什么 full purge 在某些场景下仍然是必要策略?
6. 为什么 incremental 只能摊平尖峰, 不能替代对象图治理?

---

## 十六、进入第六课前你应该达到的状态

如果你现在已经能顺着讲清下面这句话, 就可以进入调试与优化课:

> UE5 的 incremental 不是单一开关, 而是一组分阶段能力; 其中最核心的是带状态机的增量 reachability, 它必须配合 barrier 才能在跨帧对象图变化时保持正确性。

---

## 源码锚点

- `Engine/Source/Runtime/CoreUObject/Public/UObject/ReachabilityAnalysis.h`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/ReachabilityAnalysisState.h`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/ReachabilityAnalysisState.cpp`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/ObjectPtr.h`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollection.cpp`
