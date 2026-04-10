# Lesson 6: 调试、排障与工程实践

## 这节课解决什么问题

最后一节课的目标不是继续堆新概念, 而是把前面五节真正转化成工程能力。  
也就是说, 当你在项目里遇到这些问题时:

- GC hitch
- 对象误回收
- 对象误保活
- 内存持续增长
- 某类对象总是在错误时机消失

你应该知道怎么切问题、怎么定位、怎么修设计, 而不是只会试配置。

---

## 一、先建立调试总框架

遇到 GC 问题时, 先把问题归到下面三类之一:

### 1.1 正确性问题

- 对象不该死却死了
- 对象该死却没死

### 1.2 性能问题

- 某次 GC 单帧特别卡
- 某个阶段总是过重

### 1.3 设计问题

- 对象数量失控
- 引用图过于复杂
- native 与 UObject 边界不清晰

如果这一步不做, 你后面很容易乱查。

---

## 二、对象“本该活却死了”怎么查

### 2.1 第一优先：查引用图可见性

最常见根因:

- 裸指针没有进入 GC 图
- native 容器里的对象没桥接
- 关键引用没有进入 schema / ARO 路径

### 2.2 第二优先：查引用语义是否写错

典型错误:

- 把 `TWeakObjectPtr` 当强引用
- 把 `TSoftObjectPtr` 当强引用

### 2.3 第三优先：查 native 持有者

如果对象是被某个 native manager 持有:

- 看它是否正确用了 `FGCObject`
- 或者是否根本没接入 GC

### 2.4 结论

对象误死, 往往不是 GC 太凶, 而是:

> 你没有把应有的强引用关系表达给 GC

---

## 三、对象“本该死却没死”怎么查

### 3.1 第一优先：查根链路

最常见原因就是:

- 还有某条 root 到对象的路径没断

### 3.2 第二优先：查 `KeepFlags`

特别是在编辑器环境下:

- `GARBAGE_COLLECTION_KEEPFLAGS` 可能影响保留行为

### 3.3 第三优先：查 `FGCObject` / 缓存层

有些 native 管理器会长期保活对象而你自己没意识到。

### 3.4 第四优先：查 cluster

对象可能是跟着 cluster root 一起被保活。

### 3.5 结论

对象不死通常不是“GC 漏了”, 而是:

> 有一条保活路径还在

---

## 四、GC Hitch 的排查方法

### 4.1 先问：卡在哪一段

优先分成:

- traversal
- gather / unhash
- begin destroy / purge

### 4.2 再问：对象图规模是否合理

典型症状:

- UObject 数量暴涨
- 某类短命对象过多
- 缓存层把大量对象长时间挂住
- 纯数据对象化过度

### 4.3 再问：是不是该上优化器

- parallel
- cluster
- incremental reachability
- incremental gather / begin destroy

### 4.4 最后才调时间预算

预算参数是精修手段。  
它不应该是你对 GC hitch 的第一反应。

---

## 五、为什么 `AddToRoot` 是危险的“快解法”

### 5.1 它为什么诱人

对象一消失, 很多人的第一反应是:

- “先 `AddToRoot` 看看”

这通常确实能让 bug 暂时消失。

### 5.2 它为什么危险

因为它可能把真正的问题从:

- “引用图设计错误”

掩盖成:

- “对象一直被强制保活”

然后产生新问题:

- 长期泄漏
- 生命周期失去解释性
- 后续维护者根本不知道对象为什么一直活

### 5.3 正确使用姿势

`AddToRoot` 更适合:

- 调试验证
- 极少数生命周期特殊对象
- 临时过渡期

而不是常规设计方案。

---

## 六、工程上最重要的三条设计原则

### 6.1 拥有与观察要分开

- 拥有关系: 强引用, 进入 GC 图
- 观察关系: 弱引用, 不保活

### 6.2 UObject 与 native 边界要清楚

一旦 native 世界持有 `UObject`, 就必须明确:

- 这是短期临时使用
- 还是长期持有
- 长期持有要不要用 `FGCObject`

### 6.3 不要把纯数据无脑对象化

如果只是纯数据:

- 优先考虑 struct

因为每多一个 UObject, 就多一份:

- 全局对象表负担
- 引用图负担
- 生命周期管理负担

---

## 七、一个实用排障清单

以后遇到 GC 问题, 你可以按下面顺序检查:

1. 这是正确性问题还是性能问题?
2. 如果是正确性问题, 是误死还是误活?
3. 误死先查引用图可见性, 误活先查根链路
4. 如果是性能问题, 先分 traversal / gather / purge
5. 再评估对象数量、布局复杂度、销毁成本
6. 最后再决定是否要调 parallel / cluster / incremental

如果你能坚持这个顺序, 基本就不会陷入“乱试参数”。

---

## 八、第六轮源码导读

这一节建议回头交叉阅读前面所有关键文件, 但重点关注:

1. `GarbageCollection.cpp`
2. `ReachabilityAnalysisState.cpp`
3. `ObjectPtr.h`
4. `GCObject.h`
5. `GarbageCollectionSchema.cpp`

### 带着这些问题去看

1. 哪些状态和日志最能帮助区分 traversal 与 purge 问题?
2. 哪些路径最容易造成对象图可见性缺失?
3. 哪些设计会导致 cluster / incremental 收益很差?

---

## 九、常见面试题

### 面试题 1

GC hitch 时你第一步做什么?

### 回答要点

- 先分段定位
- 看 traversal、gather 还是 purge
- 不盲调时间预算

### 面试题 2

对象本该活却死了, 你先查什么?

### 回答要点

- 先查引用图可见性
- 看是否误用裸指针、弱引用、native 容器

### 面试题 3

对象本该死却没死, 你先查什么?

### 回答要点

- 先查 root 链路
- 再查 keep flags、FGCObject、cluster

### 面试题 4

为什么对象建模通常优先于 GC 参数调优?

### 回答要点

- 因为 GC 成本源头是对象图
- 参数只能调度成本, 不能消灭成本

---

## 十、本节自测

1. 对象误死时最常见的根因是什么?
2. 对象误活时为什么优先查 root 链路?
3. 为什么 `AddToRoot` 常常不是好答案?
4. 为什么“懂原理”不等于“会排障”?
5. 如果一个系统长期持有大量 UObject, 你会优先从哪些设计层面考虑优化?

---

## 十一、整套六节课的总总结

如果你把这六节课串起来, 应该得到下面这条总线:

1. Lesson 1:
   UObject GC 的边界与基本对象模型
2. Lesson 2:
   引用图如何建立, schema / ARO / FGCObject 如何协作
3. Lesson 3:
   一轮 GC 主流程如何推进
4. Lesson 4:
   销毁语义、弱引用与生命周期陷阱
5. Lesson 5:
   性能架构: schema、parallel、cluster、incremental
6. Lesson 6:
   如何把前面的原理真正用来调试和设计系统

如果你真的学透了, 你应该能做到:

- 不再把 GC 当黑盒
- 能从对象图角度解释大多数现象
- 能把正确性问题和性能问题分开
- 能说清楚 UE5 GC 的现代特征在哪里

---

## 十二、本节的一个结论句

> 深入理解 UE5 GC 的最终目标不是“会背 API 和 CVar”, 而是能把对象图、生命周期、主流程和优化器统一到一个工程视角下。

---

## 源码锚点

- `Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollection.cpp`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/ReachabilityAnalysisState.cpp`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/ObjectPtr.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/GCObject.h`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollectionSchema.cpp`
