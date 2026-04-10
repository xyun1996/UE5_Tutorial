# Appendix 1: UE5 GC 术语精讲

## 这份附录怎么用

这不是正文课程的替代品, 而是配套术语表。  
建议用法:

- 第一次学习时, 遇到名词不稳就来查
- 第二次复习时, 直接从这份表反向回忆整套 GC 框架
- 做面试准备时, 用它检查自己是否真的把术语讲清了

这份术语表尽量做到三件事:

1. 给出一句话定义
2. 指出它在 UE5 GC 里的真实位置
3. 说明最容易和什么概念混淆

---

## 一、UObject

### 一句话定义

UE 反射系统中的基础对象类型, 也是 GC 主要管理的核心对象单元。

### 在 GC 里的位置

- GC 的判活对象主要就是 `UObject` 及其派生类型
- 可达性分析最终判断的也是“某个 UObject 是否从根可达”

### 最常见误区

- 把“所有 UE 里的对象”都当成 `UObject`
- 以为“只要是 UObject, 就什么都不用管, GC 会自动处理一切”

正确理解:

- UObject 进入了 GC 世界
- 但你仍然必须正确表达引用关系

---

## 二、Root / RootSet

### 一句话定义

GC 引用图的起点集合。

### 在 GC 里的位置

- 可达性分析从根出发
- 只要从根能一路走到某对象, 该对象就会被保活

### 最常见误区

- 把 RootSet 和 `KeepFlags` 混为一谈
- 把 root 理解成“永不销毁对象”

正确理解:

- root 是“本轮图遍历的起点”
- 不是“一种永远不死的对象分类”

---

## 三、Reachability / 可达性

### 一句话定义

某对象是否能从根集合沿着 GC 可见引用关系一路到达。

### 在 GC 里的位置

- 这是 UE5 GC 判断对象是否存活的核心标准

### 最常见误区

- 把“我手里还有一个指针”当成“对象可达”

正确理解:

- 可达性看的是 GC 图
- 不是你的主观逻辑印象

---

## 四、Strong Reference / GC 强引用

### 一句话定义

GC 在遍历引用图时能够沿着它继续走到目标对象的引用。

### 在 GC 里的位置

- 强引用构成 UObject 可达性图的边

### 最常见误区

- 把“有个地址”当成“强引用”

正确理解:

- 强引用不是 C++ 类型层面的称呼
- 是 GC 视角下“能否被遍历到”的称呼

---

## 五、Weak Reference / 弱引用

### 一句话定义

可以安全观测对象是否还活着, 但不会阻止对象被 GC 的引用形式。

### 在 UE 里的典型形式

- `TWeakObjectPtr`

### 最常见误区

- 把弱引用当成“比较安全的强引用”

正确理解:

- 它的目标是减少悬空指针风险
- 不是延长对象生命周期

---

## 六、Soft Reference / 软引用

### 一句话定义

更偏向“资源定位与延迟加载”的引用形式, 默认不提供强保活语义。

### 在 UE 里的典型形式

- `TSoftObjectPtr`

### 最常见误区

- 以为软引用也会阻止对象回收

正确理解:

- 软引用更像“路径 + 需要时加载”
- 不是常规强保活手段

---

## 七、KeepFlags

### 一句话定义

GC 入口传入的对象保留策略, 表示带某些 `EObjectFlags` 的对象本轮默认要保留。

### 在 GC 里的位置

- 影响一轮 GC 中哪些对象先被视为要保留

### 最常见误区

- 把它和 RootSet 等同
- 以为它是对象的永久属性

正确理解:

- 它是“本轮 GC 的策略输入”
- RootSet 是对象状态
- 两者不是一回事

---

## 八、`GARBAGE_COLLECTION_KEEPFLAGS`

### 一句话定义

UE 提供的默认 `KeepFlags` 宏。

### 当前版本定义

```cpp
#define GARBAGE_COLLECTION_KEEPFLAGS (GIsEditor ? RF_Standalone : RF_NoFlags)
```

### 最常见误区

- 套用旧资料里的过时组合

正确理解:

- 它和运行环境有关
- 编辑器态与游戏态可能表现不同

---

## 九、`GUObjectArray`

### 一句话定义

全局 UObject 数组, 是 GC 和 UObject 运行时的重要对象总表。

### 在 GC 里的位置

- 遍历所有对象
- 索引对象
- 管理 `FUObjectItem`

### 最常见误区

- 把它想成“只是一个普通数组”

正确理解:

- 它是 UObject 世界的核心运行时基础设施之一

---

## 十、`FUObjectItem`

### 一句话定义

每个 UObject 在全局对象数组中的元数据槽位。

### 在 GC 里的位置

- 记录对象指针
- 内部标记
- 序列信息
- cluster 归属

### 最常见误区

- 把它讲成“引用计数对象”

正确理解:

- UE5 GC 的核心不是引用计数
- `FUObjectItem` 是对象的 GC 侧档案, 不是计数式生命周期管理器

---

## 十一、Internal Flags / 内部对象标记

### 一句话定义

UObject 在 GC / UObject 运行时内部维护的一组状态位。

### 在 GC 里的位置

这些标记参与表达:

- 对象是否不可达
- 是否处于 cluster root
- 是否 root
- 是否垃圾

### 最常见误区

- 把 `EObjectFlags` 和内部标记混成一套东西

正确理解:

- `EObjectFlags` 偏公开对象标志
- internal flags 偏引擎内部运行时状态

---

## 十二、Reference Graph / 引用图

### 一句话定义

以对象为节点、以 GC 可见强引用为边构成的有向图。

### 在 GC 里的位置

- 可达性分析的对象就是这张图

### 最常见误区

- 把引用图想成“所有 C++ 指针组成的图”

正确理解:

- 只有 GC 看得见的引用才是图中的边

---

## 十三、ReferenceSchema / `FSchemaView`

### 一句话定义

类或结构体的 GC 引用布局描述, 指导 GC 如何高效遍历成员引用。

### 在 GC 里的位置

- UE5 新版 traversal 的核心基础

### 最常见误区

- 以为 GC 仍然主要靠“现场慢反射”

正确理解:

- schema 是“预组织的遍历计划”
- 这是 UE5 GC 的关键现代化特征之一

---

## 十四、`AddReferencedObjects` / ARO

### 一句话定义

由对象自己手动向 GC 报告额外强引用的机制。

### 在 GC 里的位置

- 处理反射系统不方便直接描述的引用关系

### 最常见误区

- 以为 schema 出现后 ARO 就过时了
- 或者反过来, 什么都丢给 ARO

正确理解:

- 常规引用优先走 schema / 正常字段
- ARO 用来补复杂引用

---

## 十五、`FGCObject`

### 一句话定义

让非 `UObject` 类型把自己持有的 `UObject` 引用桥接进 GC 的机制。

### 在 GC 里的位置

- native 世界进入 UObject GC 图的桥

### 最常见误区

- 把它当成偷懒替代 `UPROPERTY` 的万能方案

正确理解:

- 它是跨边界补桥工具
- 不是常规字段声明替代品

---

## 十六、Mark / MarkObjectsAsUnreachable

### 一句话定义

GC 前半段对对象状态做“可能不可达”铺底与保留初始化的过程。

### 在 GC 里的位置

- 它先把对象放入候选状态
- 再等待后续 reachability 来翻正可达对象

### 最常见误区

- 把 `MarkObjectsAsUnreachable` 理解成“已经确认垃圾”

正确理解:

- 这是前置铺底阶段
- 真正判活核心仍是 reachability analysis

---

## 十七、Reachability Analysis

### 一句话定义

从根集合出发遍历引用图, 找出真正可达对象的过程。

### 在 GC 里的位置

- 这是 GC 正确性的核心阶段

### 最常见误区

- 把 GC 看成“只做一次标记”

正确理解:

- UE5 GC 的主判定逻辑就在这一步

---

## 十八、Unreachable

### 一句话定义

当前 GC 轮次下, 无法从根集合沿 GC 可见引用到达的对象状态。

### 最常见误区

- 以为 “unreachable = 内存已经释放”

正确理解:

- unreachable 是判定结果
- 不是释放动作本身

---

## 十九、Garbage

### 一句话定义

在 GC 流程中已进入垃圾处理语义的对象状态。

### 最常见误区

- 把 garbage、unreachable、destroyed 混成一个时刻

正确理解:

- 它们是相关但不同阶段的概念

---

## 二十、`ConditionalBeginDestroy` / `BeginDestroy` / `FinishDestroy`

### 一句话定义

UObject 销毁收尾流程中的不同阶段。

### 在 GC 里的位置

- 对象被判死后, 不会立刻简单 `delete`
- 会进入 begin / finish / purge 等收尾步骤

### 最常见误区

- 以为对象一变成垃圾就立即物理释放

正确理解:

- 逻辑死亡和物理释放通常分离

---

## 二十一、Purge

### 一句话定义

对已经进入销毁流程的对象做最终清理和释放的后段步骤。

### 最常见误区

- 把 purge 和 reachability 混在一起

正确理解:

- reachability 是判活
- purge 是收尾

---

## 二十二、Cluster

### 一句话定义

对稳定对象群做的 GC 图压缩与批处理优化结构。

### 在 GC 里的位置

- 帮助降低 traversal 成本

### 最常见误区

- 把 cluster 当成另一套 GC
- 把 cluster 当成“任何项目都该开启到底”的万能优化

正确理解:

- cluster 是主流程上的优化器
- 依赖稳定对象图

---

## 二十三、Cluster Root

### 一句话定义

代表一个 cluster 的入口对象。

### 最常见误区

- 把 cluster root 当成普通 root set

正确理解:

- 它和 root set 相关, 但语义不同
- 它属于 cluster 结构的一部分

---

## 二十四、Mutable Objects

### 一句话定义

和 cluster 有关、但不适合稳定纳入 cluster 主对象集合的对象。

### 为什么重要

它说明 cluster 不是“无脑把所有对象打包”, 而是保守处理不稳定关系。

---

## 二十五、Dissolve Cluster

### 一句话定义

当 cluster 不再适合维持时, 引擎将其拆解并退回普通对象处理路径。

### 最常见误区

- 以为 cluster 一旦创建就永久存在

正确理解:

- cluster 是运行时优化结构
- 可以失效、可以回退

---

## 二十六、Parallel GC

### 一句话定义

利用多线程加速 GC 某些阶段, 主要是遍历与处理工作。

### 最常见误区

- 以为开并行就一定无脑更快

正确理解:

- 并行收益取决于对象规模、平台线程数、任务划分与同步成本

---

## 二十七、Incremental Reachability

### 一句话定义

把可达性分析拆成多次迭代, 分多帧推进的机制。

### 最常见误区

- 把它和“整个 GC 全都增量化”画等号

正确理解:

- 它只针对 reachability analysis 这段主判活流程

---

## 二十八、Incremental Gather

### 一句话定义

把不可达对象 gather / unhash 等后段工作分帧执行的机制。

### 最常见误区

- 以为它就是 incremental reachability 的一部分

正确理解:

- 它解决的是后段尖峰
- 和前段 traversal 是不同层次

---

## 二十九、Incremental BeginDestroy

### 一句话定义

把 begin destroy 相关销毁动作分批执行的机制。

### 最常见误区

- 以为只要 traversal 增量化就足够了

正确理解:

- 后段销毁同样可能是尖峰来源

---

## 三十、Barrier / Reachable Barrier

### 一句话定义

在增量 GC 期间, 当对象图发生变化时, 用于修正 reachable 集合的机制。

### 在 GC 里的位置

- 确保增量 traversal 跨帧时仍保持正确性

### 最常见误区

- 把 barrier 理解成“可有可无的性能细节”

正确理解:

- 对增量 GC 来说, barrier 是正确性条件之一

---

## 三十一、`TObjectPtr`

### 一句话定义

UE5 推荐使用的 UObject 指针封装形式, 在现代 GC 语境下比裸指针更有体系化支持。

### 最常见误区

- 只把它理解成“语法新写法”

正确理解:

- 它和现代 GC 路径, 特别是 incremental 期间的行为, 有更自然的协作关系

---

## 三十二、Full Purge

### 一句话定义

一轮 GC 中倾向于把后段清理尽量完整做完的策略。

### 最常见误区

- 把 full purge 和“非增量 GC”当成完全同义

正确理解:

- 它们高度相关, 但不应在概念上直接糊成一个词

---

## 三十三、Hitch

### 一句话定义

单帧耗时异常升高, 造成卡顿的现象。

### 在 GC 里的典型来源

- traversal 尖峰
- gather / unhash 尖峰
- begin destroy / purge 尖峰

### 最常见误区

- 一看到 hitch 就盲调 `IncrementalReachabilityTimeLimit`

正确理解:

- 先分段定位, 再决定调哪类开关

---

## 三十四、这一份术语表该怎么复习

推荐复习方式:

1. 先从 Root / Reachability / Strong Reference / Schema 这组核心词开始
2. 再记 Mark / Unreachable / BeginDestroy / Purge 这组流程词
3. 最后再记 Cluster / Parallel / Incremental / Barrier 这组优化器词

如果你能把这三组词分别说顺, 整套 GC 课程就会顺很多。
