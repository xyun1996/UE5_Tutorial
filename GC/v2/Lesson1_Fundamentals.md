# Lesson 1: UE5 GC 基础与边界

## 本节定位

这一节是整套 GC 课程的地基。  
如果这里的边界和词义没有建立准, 后面看到 `ReferenceSchema`、`MarkObjectsAsUnreachable`、`IncrementalReachability`、cluster 这些词时, 很容易把它们混成一团。

本节不追求“把所有实现细节看完”, 而是追求两件事:

1. 先建立正确的总心智模型
2. 给后续源码阅读建立目录感

---

## 学习目标

学完本节, 你应该能清楚回答:

- UE5 的 GC 负责管理什么, 不负责管理什么
- 为什么 UE 需要一套基于可达性的对象生命周期管理
- `CollectGarbage`、`GARBAGE_COLLECTION_KEEPFLAGS` 的真实语义是什么
- `GUObjectArray`、`FUObjectItem`、`ReferenceSchema`、`FGCObject`、`GUObjectClusters` 分别是什么
- 一次 GC 周期在宏观上经历了哪些阶段

---

## 建议前置知识

如果你已经熟悉这些内容, 学起来会更轻松:

- C++ 基础对象生命周期
- `UObject` / `AActor` / `UActorComponent` 的基本概念
- UE 反射系统基础, 特别是 `UPROPERTY`
- 一般性的 mark-sweep 算法概念

如果这些概念还不牢, 也可以继续学, 但遇到某些段落时建议慢一点。

---

## 一、为什么游戏引擎需要 GC

### 1.1 普通 C++ 项目中的生命周期问题

在纯 C++ 应用里, 对象通常由这些方式管理:

- 栈对象
- RAII
- `unique_ptr`
- `shared_ptr`
- 手动 `new/delete`

这些方式在“边界清晰、所有权明确”的系统里非常有效。  
但游戏引擎里的对象世界有几个天然复杂点:

- 生命周期跨多个系统
- 运行时动态生成和销毁频繁
- 脚本、蓝图、编辑器、异步加载都会介入
- 对象图不是简单树状结构, 而常常是稠密图

### 1.2 UE 的复杂性来自哪里

举几个常见场景:

- 关卡加载时一批对象整体进入世界
- 关卡卸载时又要整体退出
- 一个 Actor 生成后, 又动态挂上多个组件
- 蓝图里某段逻辑又运行时创建临时 UObject
- StreamableManager 或异步加载器在另一套时序里拉起对象
- 编辑器里对象还要经历构造、编辑、PIE、停止 PIE、重载等阶段

如果完全靠“谁创建谁销毁”的人工 discipline:

- 很难保证所有路径都清晰
- 很难防止交叉引用导致的泄漏
- 很难把蓝图和反射系统优雅接进来

### 1.3 为什么不是“全内存托管”

UE5 又不是托管运行时。它不能像 Java / C# 那样接管整个程序对象世界。  
它必须和 native C++ 共存:

- 引擎很多系统仍然是原生对象管理
- GPU / 渲染资源不可能交给 UObject GC 直接处理
- 高性能路径仍然大量依赖 C++ 显式控制

因此 UE 的做法不是:

> 让 GC 管一切

而是:

> 让 GC 专注于 `UObject` 生态中的对象图可达性与生命周期收尾。

这句话是本节最重要的起点。

---

## 二、GC 到底管理什么

### 2.1 直接归 GC 管的典型对象

典型例子:

- `UObject`
- `AActor`
- `UActorComponent`
- `UClass`
- 资源对象, 例如 `UTexture`、`UStaticMesh`、`UMaterial`

你可以粗略理解成:

- 所有挂在 UObject 体系上的核心反射对象

### 2.2 不归 UObject GC 直接管理的典型内容

- 普通 C++ `new/delete` 对象
- `TSharedPtr` / `TUniquePtr` 管理的 native 对象
- Slate 对象
- RHI / GPU 资源
- 线程对象、同步对象
- 任何没有接入 UObject 引用图的原生对象关系

### 2.3 这带来一个重要认识

UE5 里“内存管理”不是一个系统包打天下。  
至少要分两层看:

- UObject 生命周期与可达性: 主要由 GC 负责
- 非 UObject 资源与 native 内存: 主要由 C++ / 各子系统自己负责

这也是为什么你在项目里经常会同时用到:

- `UPROPERTY`
- `TObjectPtr`
- `TUniquePtr`
- `TSharedPtr`
- 手动销毁接口

它们不是互斥关系, 而是分工不同。

---

## 三、四个必须先背熟的判断句

### 3.1 判断句 1

> “我手里有个 `UObject*` 裸指针” 不等于 “GC 会保活它”。

### 3.2 判断句 2

> “对象当前还没被释放” 不等于 “它仍然从根可达”。

### 3.3 判断句 3

> “对象进入 GC 处理流程” 不等于 “它的内存已经立刻释放”。

### 3.4 判断句 4

> “GC 慢” 通常先怀疑对象数量和引用图设计, 再怀疑参数。

这四句话后面几乎每节课都会反复出现。

---

## 四、真实入口: `CollectGarbage`

### 4.1 源码位置

- `Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectGlobals.h`

### 4.2 真实签名

```cpp
COREUOBJECT_API void CollectGarbage(EObjectFlags KeepFlags, bool bPerformFullPurge = true);
COREUOBJECT_API bool TryCollectGarbage(EObjectFlags KeepFlags, bool bPerformFullPurge = true);
```

这两行声明至少告诉你五件事:

1. GC 入口是公开 API, 不是只能靠控制台命令触发
2. `KeepFlags` 会影响本轮保留策略
3. `bPerformFullPurge` 会影响本轮清理深度
4. GC 不是永远都要强制执行, 否则也不会有 `TryCollectGarbage`
5. full purge 和 incremental 不是同一个维度

### 4.3 `TryCollectGarbage` 为什么存在

这说明 GC 还受运行时同步环境约束:

- 可能有其他线程正在改 UObject 状态
- 当前时机可能不适合强行停下来做完整 GC

所以 GC 不只是算法, 还是调度与同步问题。

---

## 五、`GARBAGE_COLLECTION_KEEPFLAGS` 的真实语义

### 5.1 源码位置

- `Engine/Source/Runtime/CoreUObject/Public/UObject/GarbageCollection.h`

### 5.2 当前定义

```cpp
#define GARBAGE_COLLECTION_KEEPFLAGS (GIsEditor ? RF_Standalone : RF_NoFlags)
```

### 5.3 这比想象中更重要

很多老资料或经验帖会把它写成别的组合, 甚至说成固定的 `RF_Standalone | RF_Public`。  
当前版本不是这样。

这意味着:

- 编辑器环境下, 某些对象因为 `RF_Standalone` 更容易被保留
- 游戏环境下, 默认保留条件更收敛

### 5.4 学习上的意义

这会直接影响你理解调试现象:

- “为什么编辑器里对象没回收, 打包后却回收了?”
- “为什么 PIE 和 standalone 行为不完全一样?”

很多时候, 不只是引用图不同, 还有运行环境策略不同。

---

## 六、UE5 GC 的核心数据结构

这一节先只做“认识面孔”, 后面几节再详细展开。

### 6.1 `GUObjectArray`

源码位置:

- `Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectArray.h`

它是所有 UObject 的全局目录。  
你可以把它理解成:

- GC 的对象总表
- 弱引用校验、索引访问、对象遍历的重要基础设施

GC 不会在内存里盲扫“疑似 UObject 的东西”, 它有一张全局清单。

### 6.2 `FUObjectItem`

同样位于:

- `UObjectArray.h`

每个 UObject 在 `GUObjectArray` 中都有对应的 `FUObjectItem`。  
它记录的是对象在 GC / UObject 运行时视角下的关键元数据。

教学上可以把它理解成“对象在 GC 系统里的档案卡”。

这个“档案卡”里涉及的典型信息包括:

- 指向真实对象的指针
- 内部对象标记
- 序列号相关信息
- cluster 归属信息

### 6.3 `ReferenceSchema`

源码位置:

- `Engine/Source/Runtime/CoreUObject/Public/UObject/GarbageCollectionSchema.h`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollectionSchema.cpp`

这是 UE5 GC 学习里最值得重点投入的一部分。  
因为当前 UE5 已经不是“全靠临时反射慢扫字段”的模型了。

`ReferenceSchema` 的核心作用是:

- 描述类或结构体里的强引用布局
- 指导 GC 如何高效遍历这些引用

你可以把它理解成:

> “GC 的字段访问计划表”

### 6.4 `FGCObject`

源码位置:

- `Engine/Source/Runtime/CoreUObject/Public/UObject/GCObject.h`

它解决的问题不是“普通 UObject 成员怎么扫描”, 而是:

- 某个对象本身不是 `UObject`
- 但它又持有 `UObject*`
- 这些引用还确实要参与 GC 保活

这时就需要 `FGCObject` 这种桥接机制。

### 6.5 `GUObjectClusters`

同样来自:

- `UObjectArray.h`

它是 cluster 系统的全局容器。  
先记一句就够:

> cluster 是 GC 的性能优化器, 不是另一套 GC。

---

## 七、一次 GC 周期的宏观流程

在正式学习第三课之前, 先建立宏观图。

```text
CollectGarbage / TryCollectGarbage
    -> 预处理与锁管理
    -> MarkObjectsAsUnreachable
    -> 标记 root / keep / cluster root
    -> PerformReachabilityAnalysis
    -> 收集不可达对象
    -> 清理弱引用
    -> BeginDestroy / FinishDestroy / Purge
    -> 统计与回调
```

### 7.1 这张图最重要的不是顺序细节, 而是三个认识

认识 1:

- GC 不是一刀切动作, 是多阶段流水线

认识 2:

- reachability analysis 是核心, 但不是全部

认识 3:

- “对象被判死”和“对象内存立刻释放”中间可能隔着多个阶段

---

## 八、为什么 UE5 GC 不是引用计数

很多初学者一看到:

- `FUObjectItem`
- 对象标记
- 弱引用序列号

就容易误解成“UE 其实主要靠引用计数”。

这是错的。

更准确地说:

- UE5 采用的是基于可达性的判活思路
- 核心问题是“能否从根走到它”
- 而不是“当前有多少地方指向它”

### 8.1 为什么这个区别非常重要

因为引用计数思维会让你错误地依赖:

- 裸指针
- 临时局部变量
- 逻辑上看起来“还在用”

而可达性思维会逼你去问:

- 这条引用是否进入了 GC 图?
- 它是否从根可达?

这才是 UE5 GC 的正确学习方式。

---

## 九、四种典型错误理解

### 9.1 错误理解 1

“UObject 都会自动被安全管理, 我不需要关心引用形式。”

错。  
你必须关心 GC 能否看见这条引用。

### 9.2 错误理解 2

“既然是 GC, 那对象销毁时机就完全不可理解。”

也不对。  
虽然不是像栈对象那样确定, 但它依然有非常明确的阶段和规则。

### 9.3 错误理解 3

“只要对象没被释放, 就代表它还活着。”

错。  
对象可能已经不可达, 只是还在销毁收尾阶段。

### 9.4 错误理解 4

“GC 出问题就应该先调 CVar。”

通常也错。  
大多数问题的根源在对象图, 不是参数表。

---

## 十、如何配合源码学习这一课

建议阅读顺序:

1. `UObjectGlobals.h`
   先看公开入口
2. `GarbageCollection.h`
   看 keep flags、GC 基本声明
3. `UObjectArray.h`
   认识 `GUObjectArray`、`FUObjectItem`
4. `GCObject.h`
   认识 native 对象如何接入
5. `GarbageCollectionSchema.h`
   先建立 schema 概念
6. `GarbageCollection.cpp`
   最后再看实现

### 10.1 本节不建议一开始就做的事

- 不建议先直接读完整 `GarbageCollection.cpp`
- 不建议先背所有 CVar
- 不建议先研究 cluster 细节

因为那样容易把“局部机制”误当成“整体框架”。

---

## 十一、学习笔记建议

如果你是自学, 建议在自己的笔记里专门写下三列:

### 11.1 术语列

- RootSet
- KeepFlags
- Reachability
- Schema
- FullPurge
- Cluster

### 11.2 实体列

- `CollectGarbage`
- `GUObjectArray`
- `FUObjectItem`
- `FGCObject`
- `FSchemaView`

### 11.3 问题列

- GC 看得见什么引用?
- 什么情况下对象不会被回收?
- 一个对象为什么会在编辑器里活、在游戏里死?

这样做的好处是, 后面每节课都能挂接到这个框架里。

---

## 十二、本节课后的自检题

1. UE5 GC 为什么不能等同于“全内存自动回收”?
2. 为什么一个 `UObject*` 裸指针不一定能保活对象?
3. `GARBAGE_COLLECTION_KEEPFLAGS` 在编辑器和游戏态为什么不同?
4. `FGCObject` 解决的是哪类问题?
5. 为什么 `ReferenceSchema` 是 UE5 GC 课程中必须重点学习的部分?
6. 为什么“对象当前没被释放”不等于“对象仍然从根可达”?

---

## 十三、继续学习前你应该达到的状态

如果你现在已经能比较自然地说出下面这些话, 就可以进入第二课:

- “UE GC 只管理 UObject 生态”
- “GC 判活靠可达性, 不是引用计数”
- “GC 要不要保活一个对象, 关键是它是否从根可达”
- “GC 看不见的裸指针不能指望它保活对象”
- “ReferenceSchema 是 UE5 GC 的核心关键词之一”

如果这些话你还说不顺, 建议本节再过一遍。

---

## 源码锚点

- `Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectGlobals.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/GarbageCollection.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectArray.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/GCObject.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/GarbageCollectionSchema.h`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollection.cpp`
