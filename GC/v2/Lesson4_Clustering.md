# Lesson 4: GC Clustering 的真实用途与限制

## 本节定位

cluster 是 UE5 GC 里最容易被讲得“像魔法”的部分。  
很多资料只给一句话:

> cluster 就是把一堆对象绑起来一起处理。

这句话方向没错, 但远远不够。  
如果你不理解它的边界和代价, 很容易把 cluster 学成“GC 变快开关”。

本节的目标是把 cluster 放回它真正的位置:

- 它是什么
- 它解决什么问题
- 它依赖什么前提
- 它为什么会失效并 dissolve

---

## 学习目标

学完本节, 你应该能说清:

- cluster 是 GC 性能优化器, 不是另一套 GC
- `FUObjectCluster` 大致包含哪些信息
- `CanBeClusterRoot` / `CanBeInCluster` 这类接口为什么存在
- cluster 为什么偏爱稳定、cooked、对象图清晰的场景
- cluster 为什么会 dissolve
- 什么情况下值得考虑 cluster, 什么情况下不该先上它

---

## 一、先给 cluster 定位

### 1.1 源码位置

- `Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectArray.h`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/UObjectClusters.cpp`

### 1.2 关键结构

```cpp
struct FUObjectCluster
class FUObjectClusterContainer
extern COREUOBJECT_API FUObjectClusterContainer GUObjectClusters;
```

### 1.3 一句话定义

你可以把 cluster 理解成:

> 对稳定对象群做的 GC 图压缩与批处理优化。

注意这里有三个关键词:

- 稳定对象群
- 图压缩
- 批处理优化

缺一个都会理解偏。

---

## 二、为什么会需要 cluster

### 2.1 先想一个典型资源对象图

例如一个 cooked 资源:

- 一个 `UStaticMesh`
- 挂着多个内部数据对象
- 再指向一批稳定引用资源
- 这些对象的拓扑在运行时几乎不变

如果每次 GC 都对这批对象做最细粒度遍历:

- 要反复读大量稳定关系
- 每轮 traversal 都重复付出类似成本
- 对性能不划算

### 2.2 cluster 想优化的不是“对象会不会死”

而是:

- 能不能更便宜地确认一整批对象的可达性

这点非常关键。  
cluster 的目标不是改变语义, 而是降低遍历成本。

---

## 三、从 `FUObjectCluster` 结构看 cluster 不是“简单数组”

学习 cluster 时不要只记“有个 root 和一堆对象”。  
源码里的 cluster 至少维护这些概念:

- `RootIndex`
- `Objects`
- `ReferencedClusters`
- `ReferencedByClusters`
- `MutableObjects`

### 3.1 `RootIndex`

cluster 的主对象入口。  
理解成“代表这个对象群的门牌号”比较贴切。

### 3.2 `Objects`

真正归属到该 cluster 的对象。

### 3.3 `ReferencedClusters`

说明这个 cluster 依赖哪些其他 cluster。

### 3.4 `ReferencedByClusters`

说明哪些 cluster 依赖当前 cluster。

### 3.5 `MutableObjects`

这是最容易被忽略、但最体现工程味的部分。  
它告诉你:

- 不是所有相关对象都适合被稳定纳入 cluster
- 有些对象会被单独记录成“需要保守对待的外部或不稳定关系”

这也是 cluster 不会退化成“无脑 DFS 全塞进去”的关键原因。

---

## 四、为什么有 `CanBeClusterRoot` / `CanBeInCluster`

### 4.1 相关接口

在 `UObjectBaseUtility` 里你能看到:

```cpp
virtual bool CanBeClusterRoot() const;
virtual bool CanBeInCluster() const;
virtual void CreateCluster();
void AddToCluster(UObjectBaseUtility* ClusterRootOrObjectFromCluster, bool bAddAsMutableObject = false);
```

### 4.2 这几个接口的存在本身就在说明一件事

不是所有对象都适合:

- 当 cluster root
- 被纳入 cluster

原因很简单:

- 有的对象拓扑稳定
- 有的对象运行时状态频繁变化
- 有的对象跨 package / 跨生命周期边界很复杂
- 有的对象本身就是“容易失稳”的节点

### 4.3 从教学角度该怎么理解

cluster 依赖一个前提:

> 这批对象之间的关系足够稳定, 值得提前打包。

如果这个前提不成立, cluster 不但帮不上忙, 还可能增加维护成本。

---

## 五、cluster 构建不是“递归把所有引用对象都收进去”

这是学习 cluster 时最容易形成的错误印象。

### 5.1 实际构建时的几类对象

在 cluster 构建过程中, 引擎会大致区分:

- 真正可以并入当前 cluster 的对象
- 应记录为其他 cluster 引用的对象
- 状态不稳定、需要保守处理的 mutable objects
- 跨 package 或跨边界对象

### 5.2 为什么不能无脑全收

如果把所有“能碰到的对象”都塞进 cluster:

- cluster 边界会迅速失控
- 跨系统依赖会变得难以维护
- 一旦有对象状态变化, 整个 cluster 很容易失效

所以 UE5 的 cluster 设计非常强调“有边界的压缩”。

### 5.3 一个更好的学习比喻

不要把 cluster 想成“把图里所有节点打包”。  
更像是:

> 从一个稳定 root 出发, 在同域、同生命周期、状态稳定的条件下, 打一个可维护的对象包。

---

## 六、cluster 为什么偏爱 cooked 和稳定资源图

你在源码里会看到 cluster 和加载流程关系很深。  
很多 cluster 创建时机就发生在:

- 包加载
- 异步加载
- cooked 资源进入运行时

### 6.1 为什么偏爱这类对象

因为它们通常具备这些特征:

- 结构相对固定
- 子对象批量存在
- 生命周期相近
- 运行时变化少

### 6.2 为什么不那么偏爱高频动态对象图

例如:

- 运行时频繁增删组件
- 大量动态挂接关系
- 对象在不同 owner 之间频繁转移

这类图更容易让 cluster 失稳, 甚至频繁 dissolve。

---

## 七、cluster 是怎么创建的

### 7.1 自动创建路径

在加载相关路径中, 引擎会收集适合成为 cluster root 的对象, 再调用 `CreateCluster()`。

这说明:

- cluster 和加载期对象图天然契合
- 它首先服务的是稳定批量对象, 而不是任意动态对象

### 7.2 类型自定义创建路径

例如:

- `ULevel`
- `ULevelActorContainer`

这类对象会根据自己的对象拓扑实现更专门的 cluster 构建逻辑。

这说明 cluster 不是一个“完全机械的通用打包器”, 而是带领域知识的优化结构。

---

## 八、cluster 和 GC 主流程怎么协作

可以用一个高层视角去看:

1. GC 开始时会考虑 cluster root 的状态
2. reachability 阶段可按 cluster 粒度快速保活对象群
3. 如果 cluster 结构可靠, traversal 成本下降
4. 如果 cluster 结构不再可靠, 会先 dissolve
5. dissolve 后相关对象退回普通对象处理路径

### 8.1 这说明什么

cluster 不是旁路机制。  
它是嵌入到 GC 主流程中的加速层。

### 8.2 为什么这个理解重要

因为很多人误以为:

- cluster 像某种“独立缓存”
- 或者“绕开可达性分析”

其实都不是。  
它只是在可达性分析里帮你减少细粒度工作量。

---

## 九、为什么 cluster 会 dissolve

源码里存在这些接口:

- `DissolveCluster`
- `DissolveClusterAndMarkObjectsAsUnreachable`
- `DissolveClusters`

这说明 cluster 从设计上就是可失效结构。

### 9.1 典型 dissolve 原因

- cluster 内对象状态变化
- 某些引用关系变得不稳定
- 对象进入 pending kill / 其他不适合稳定归组的状态
- 显式关闭 `gc.CreateGCClusters`
- 某些加载或清理路径需要重建对象边界

### 9.2 学习上的结论

不要把 cluster 想成:

- “创建完永远有效”

更准确的理解是:

- 它是运行时性能结构
- 只要收益不再可靠, 引擎就会退回普通路径

---

## 十、当前版本里和 cluster 直接相关的 CVar

来自 `UObjectClusters.cpp`:

- `gc.CreateGCClusters`
- `gc.AssetClustreringEnabled`
- `gc.MinGCClusterSize`

### 10.1 这几个变量应该怎么记

`gc.CreateGCClusters`

- cluster 总开关

`gc.AssetClustreringEnabled`

- 更偏资源类 cluster 的启用

`gc.MinGCClusterSize`

- 过小 cluster 是否值得保留

### 10.2 一个小但实用的注意点

当前源码里就是 `AssetClustrering` 这套拼写。  
课件里不要自作主张改成 `Clustering`, 不然学生回头搜源码和控制台命令会对不上。

---

## 十一、什么时候值得考虑 cluster

### 11.1 值得考虑的场景

- cooked 资源图稳定
- 同包对象很多
- 子对象数量大
- 生命周期相近
- GC 时间明显耗在这批稳定对象群上

### 11.2 不值得优先考虑的场景

- 动态关系频繁变化
- 大量对象运行时经常换 owner
- 对象跨 package 边界很复杂
- 你甚至还没先控制对象数量

### 11.3 一个非常常见的错误顺序

错误顺序:

1. 先上 cluster
2. 希望 GC 自动变快

更合理的顺序通常是:

1. 先减少对象数
2. 再治理引用图
3. 再看 parallel / incremental
4. 最后评估 cluster 是否有收益

---

## 十二、一个帮助记忆的比喻

把 cluster 理解成“预打包的稳定对象群”是有帮助的。  
但一定补上两个限制:

- 不是所有对象群都值得预打包
- 一旦稳定性失效, 这个包会被拆开

如果缺这两个限制, 你就会把 cluster 学成“GC 万能加速器”。

---

## 十三、如何配合源码学习这一课

建议阅读顺序:

1. `UObjectArray.h`
   先认识 `FUObjectCluster`
2. `UObjectBaseUtility.h`
   看 `CanBeClusterRoot`、`CanBeInCluster`、`CreateCluster`
3. `UObjectClusters.cpp`
   看构建和 dissolve 主逻辑
4. `Level.cpp`、`LevelActorContainer.cpp`
   看具体系统如何定制 cluster

### 13.1 阅读时建议带着这些问题

- 哪些对象真的被并入 cluster?
- 哪些对象只是被记为 mutable / external relationship?
- cluster 失效时引擎怎么退回普通对象处理?

---

## 十四、本节后的自检题

1. cluster 优化的是 GC 的哪一类成本?
2. 为什么 cluster 不是“把所有引用对象都递归收进去”?
3. `MutableObjects` 的存在说明了什么?
4. 为什么 cluster 会 dissolve?
5. 哪些项目场景适合 cluster, 哪些不适合?
6. 为什么 cluster 要放在“对象建模和引用治理之后”再考虑?

---

## 十五、进入第五课前你应该达到的状态

如果你已经能比较自然地说出下面这句话, 就可以继续学 incremental:

> cluster 不改变 UObject 的生死语义, 它只是把稳定对象图打包成更适合 GC 快速处理的结构, 一旦对象关系不再稳定, 引擎会 dissolve 它并退回普通处理路径。

---

## 源码锚点

- `Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectBaseUtility.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectArray.h`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/UObjectClusters.cpp`
- `Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncLoading.cpp`
- `Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncLoading2.cpp`
- `Engine/Source/Runtime/Engine/Private/Level.cpp`
- `Engine/Source/Runtime/Engine/Private/LevelActorContainer.cpp`
