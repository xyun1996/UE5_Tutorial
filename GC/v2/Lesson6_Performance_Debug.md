# Lesson 6: 性能调优与 GC 问题定位

## 本节定位

前五节讲的是:

- 机制
- 结构
- 流程
- 优化器

这一节讲的是:

- 当你在真实项目里遇到 GC 问题时, 到底怎么下手

如果你学完前五节, 但一碰到:

- GC hitch
- 对象误回收
- 对象误保活
- 内存长期不下降

还是只会盲调参数, 那说明知识还没有真正落地。  
本节的目标就是把它落地。

---

## 学习目标

学完本节, 你应该能建立一套可执行框架:

- 遇到 GC hitch 先分段定位
- 遇到对象“没死”先查根链路与保活来源
- 遇到对象“误死”先查引用图可见性
- 明白为什么对象建模永远比调 CVar 更优先
- 明白 `AddToRoot` 的作用边界

---

## 一、先建立一条总原则

GC 优化的第一优先级通常不是:

- 多开几个开关
- 多试几个时间预算
- 一上来就搞 cluster

而是:

- 减少不必要的 `UObject`
- 缩短强引用链
- 把纯数据从 `UObject` 拆回普通 struct
- 让对象生命周期边界更清晰

### 1.1 为什么这条原则如此重要

因为 GC 的根本成本来自:

- 要扫描多少对象
- 要遍历多少条引用
- 每个对象的结构复杂不复杂
- 死对象清理成本高不高

如果对象图本身就失控, 参数只能止痛, 不能治本。

---

## 二、先判断“慢在哪一段”, 不要先改参数

GC 是多阶段流程, 所以性能问题也必须分段看。

### 2.1 Reachability traversal 慢

典型原因:

- 对象太多
- 强引用太多
- schema 布局复杂
- 某些类大量落到较慢的 ARO 路径

### 2.2 Gather / Unhash 慢

典型原因:

- 本轮不可达对象太多
- 后段整理工作量大

### 2.3 BeginDestroy / Purge 慢

典型原因:

- 死对象资源清理成本高
- 大量对象在同一帧集中进入销毁

### 2.4 为什么分段是第一步

因为你只有先回答:

> GC 时间主要耗在 traversal、gather 还是 purge?

才知道后面该调哪一类工具。

---

## 三、当前版本最值得关注的开关

### 3.1 遍历阶段

- `gc.AllowParallelGC`
- `gc.AllowIncrementalReachability`
- `gc.IncrementalReachabilityTimeLimit`

### 3.2 后段处理

- `gc.AllowIncrementalGather`
- `gc.IncrementalGatherTimeLimit`
- `gc.IncrementalBeginDestroyEnabled`
- `gc.IncrementalBeginDestroyGranularity`

### 3.3 触发与行为

- `gc.FlushStreamingOnGC`
- `gc.NumRetriesBeforeForcingGC`

### 3.4 调试辅助

- `gc.GarbageReferenceTrackingEnabled`
- `gc.VerifyNoUnreachableObjects`
- `gc.DumpSchemaStats`

### 3.5 应该怎么记它们

不要孤立记变量名。  
要按“对应哪一段流程”来记。

---

## 四、为什么 `gc.DumpSchemaStats` 值得放进课件

过去很多 GC 课件只会提:

- `stat gc`
- `obj refs`

这些命令当然有价值。  
但对 UE5 来说, 只停留在这个层面是不够的。

### 4.1 它代表的思路是什么

`gc.DumpSchemaStats` 提醒你:

- GC 成本不只是“对象数量”
- 还和类 / 结构体的引用布局复杂度有关

### 4.2 为什么这很重要

因为新版 UE5 的 traversal 很大程度上是 schema 驱动的。  
如果你完全不关注 schema 复杂度, 就会遗漏很重要的一半性能来源。

---

## 五、GC hitch 的推荐排查顺序

### 5.1 第一步: 看趋势

先判断:

- 是偶发尖峰还是持续偏高
- 是加载切换时发生还是运行中持续发生
- 是 traversal 主导还是 purge 主导

### 5.2 第二步: 看对象图规模

优先怀疑:

- UObject 总数暴涨
- 某些 manager 长期保活大对象树
- 数据本应是 struct 却建成大量 UObject
- 缓存层把很多对象一直挂住

### 5.3 第三步: 看对象图是否适合优化器

再判断:

- 是否值得 parallel
- 是否适合 incremental
- 是否适合 cluster

### 5.4 第四步: 最后才调预算

预算参数适合做精修, 不适合做第一反应。

---

## 六、对象“该死没死”怎么查

这类问题非常常见。  
建议固定按下面顺序排查, 不要靠直觉乱猜。

### 6.1 先查根链路

最常见原因仍然是:

- 对象还能从某个根一路走到

### 6.2 再查 `KeepFlags`

特别是编辑器环境下:

- `GARBAGE_COLLECTION_KEEPFLAGS` 会影响默认保留行为

### 6.3 再查 `FGCObject`

某个 native manager 可能还在上报它。

### 6.4 再查 cluster

对象可能不是靠单独引用活着, 而是跟着 cluster root 一起活着。

### 6.5 最后再怀疑引擎 bug

绝大多数“没死”问题, 最后都能解释为对象图原因。

---

## 七、对象“该活却死了”怎么查

这类问题往往更危险, 因为它更容易造成:

- 崩溃
- 空指针
- 状态丢失

### 7.1 最常见根因

- 裸指针没进入 GC 图
- 弱引用被误当强引用
- native 持有者没有接入 `FGCObject`
- 临时对象没有真正被持有下来
- 自定义容器 / 缓存层绕开了标准引用上报

### 7.2 第一反应应该是什么

不是:

- 先 `AddToRoot`

而是:

- 先查这条引用为什么 GC 看不见

因为根因通常不是“GC 太激进”, 而是“引用图没搭对”。

---

## 八、为什么 `AddToRoot` 只能救急

`AddToRoot` 的确能快速保活对象。  
但它的问题同样明显:

- 会掩盖错误的引用设计
- 很容易造成长期根集合泄漏
- 会让对象生命周期越来越难解释

### 8.1 合理使用场景

- 生命周期极明确的系统对象
- 过渡期临时保活
- 调试时验证“是否真是 GC 杀掉了对象”

### 8.2 不合理使用场景

- 只要对象被回收就先 `AddToRoot`

正确心态应该是:

> `AddToRoot` 是止血钳, 不是日常保健品。

---

## 九、调优手段的正确顺序

### 9.1 第一层: 结构治理

- 减少短命 UObject
- 把纯数据改为 struct
- 清理无意义强引用
- 缩短对象链

这一层往往收益最大。

### 9.2 第二层: 执行优化

- parallel GC
- incremental reachability
- incremental gather
- incremental begin destroy

这一层是在“现有对象图不变”的前提下摊薄成本。

### 9.3 第三层: 高阶优化

- cluster
- 更细致的生命周期分层
- 加载窗口内的 full purge 策略

这是面向特定对象图和平台做精细打磨。

### 9.4 为什么顺序不能反

如果你先做第三层:

- 很容易把结构问题硬压成参数问题
- 后面维护难度会越来越高

---

## 十、一个实用的诊断框架

以后项目里遇到 GC 问题, 你可以固定问这 5 个问题:

1. 问题主要在 traversal、gather 还是 purge?
2. 当前 UObject 数量是否合理?
3. 当前引用图是否有明显设计问题?
4. 当前对象图是否适合 parallel / incremental / cluster?
5. 是否有 `FGCObject`、缓存层、`AddToRoot` 在意外保活对象?

这 5 个问题如果都能回答清楚, GC 问题基本不会再是黑箱。

---

## 十一、如何把前五课知识串起来

到这里, 你应该能把前五节课合成一个完整系统:

- Lesson 1: GC 管什么, 边界在哪里
- Lesson 2: 引用图怎么被看见
- Lesson 3: 主流程怎么走
- Lesson 4: cluster 如何压缩稳定对象图
- Lesson 5: incremental 如何分阶段摊平成本

### 11.1 为什么这一步很重要

因为只有当这五节课能串成体系:

- 调试时你才知道自己是在查哪一层问题

否则就会陷入“懂很多名词, 但不会定位”的状态。

---

## 十二、建议的复习方式

如果你真的拿这套课件做自学, 推荐按下面方式复习:

1. 回头重读 Lesson 1 和 Lesson 2
   重新确认边界和引用图
2. 再读 Lesson 3
   把整条时间线串起来
3. 再读 Lesson 4 和 Lesson 5
   把 cluster / incremental 视为主流程上的优化器
4. 最后用本节的诊断框架去复盘前面五节

这样比“按热词记忆”牢得多。

---

## 十三、本节后的自检题

1. 遇到 GC hitch 时为什么不该先盲调时间预算?
2. 对象“没死”时应该优先查什么?
3. 对象“误死”时为什么第一反应不该是 `AddToRoot`?
4. 为什么 `gc.DumpSchemaStats` 对 UE5 学习者有额外价值?
5. 结构治理、执行优化、高阶优化三层的顺序为什么不能颠倒?
6. 为什么“懂机制”和“会定位”是两回事?

---

## 十四、整套 GC 课程学完后的目标状态

如果你已经能比较自然地讲清下面这段话, 那这套 GC 课就算真正学进去了:

> UE5 GC 以 UObject 可达性图为核心, 通过 `ReferenceSchema`、`AddReferencedObjects`、`FGCObject` 等机制建立可见引用关系, 再用分阶段流程完成判活与销毁; cluster、parallel、incremental 都是在主流程之上的性能优化器, 真正的第一优先级始终是对象图设计本身。

---

## 源码锚点

- `Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollection.cpp`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/ReachabilityAnalysis.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/ObjectPtr.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/GCObject.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/GarbageCollectionSchema.h`
