# Lesson 1: 对象模型与 GC 边界

## 这节课解决什么问题

如果你想深入理解 UE5 GC, 第一件事不是去背 `CollectGarbage` 的调用链, 而是先回答两个更根本的问题:

1. UE5 GC 到底在管理什么
2. UE5 GC 刻意不管理什么

很多后续误解, 本质都出在这里。  
比如:

- 为什么普通 `new` 出来的对象不会被 UE GC 自动接管
- 为什么 `UObject*` 裸指针不能天然保活对象
- 为什么 UE 项目里既有 GC, 又仍然大量使用 `TUniquePtr`、`TSharedPtr`、手动释放资源

如果这节课学明白, 你后面看源码时就不会把“UObject 世界”和“整个 native 内存世界”混为一谈。

---

## 一、UE5 为什么需要 GC

### 1.1 游戏引擎对象生命周期比普通应用复杂得多

UE 中的对象经常要经历下面这些时序:

- 关卡加载
- 关卡卸载
- Actor 运行时生成
- Actor 运行时销毁
- Component 动态挂接
- Asset 异步加载
- 蓝图运行时创建临时 UObject
- 编辑器、PIE、热重载、重新实例化

如果所有这一切都依赖“谁创建谁 delete”, 代价会非常高:

- 你很难维护跨系统对象图
- 蓝图和反射系统很难自然接入
- 引用环和悬空指针问题会成倍增加
- 关卡切换和资源装卸的正确性非常脆弱

### 1.2 但 UE5 又不能把一切都交给托管运行时

UE5 不是 Java 或 C#。  
它仍然是一个以 native C++ 为核心的引擎, 所以它必须同时满足:

- 让 `UObject` 世界有一套统一的生命周期语义
- 又不接管所有 native 内存
- 还能和渲染资源、线程、IO、异步加载、编辑器系统协作

因此 UE5 的 GC 目标从来不是:

> 自动回收整个程序里所有内存

而是:

> 管理 `UObject` 图谱的可达性与生命周期收尾

这句话你要反复记。

---

## 二、GC 的管理边界

### 2.1 GC 主要管理什么

直接进入 GC 视野的典型对象:

- `UObject`
- `AActor`
- `UActorComponent`
- `UClass`
- 大多数引擎资源对象, 如 `UTexture`、`UStaticMesh`、`UMaterial`

如果你要用一句更抽象的话来概括:

> 所有进入 UE 反射对象系统的核心运行时对象

### 2.2 GC 不直接管理什么

以下内容一般不由 UObject GC 直接托管:

- 普通 C++ `new/delete` 对象
- `TUniquePtr` / `TSharedPtr` 管理的 native 对象
- Slate 对象
- 线程对象和锁对象
- RHI / GPU 资源
- 普通容器自身的堆内存

### 2.3 这为什么重要

因为 UE5 的内存管理本来就是多层次并存的:

- UObject 生命周期: 主要靠 GC
- 资源 / native 内存: 主要靠 RAII、引用计数、显式释放等机制

所以项目里同时出现下面这些并不矛盾:

- `UPROPERTY`
- `TObjectPtr`
- `TWeakObjectPtr`
- `TUniquePtr`
- `TSharedPtr`
- 手动 `Release` / `Destroy`

它们解决的不是同一类问题。

---

## 三、GC 判活的核心标准不是“有没有指针”, 而是“是否可达”

### 3.1 最容易犯的错误

很多人刚接触 UE 时会下意识地认为:

```cpp
UObject* Ptr = SomeObject;
```

只要这个指针还在, 对象就应该活着。  
这是错误的。

### 3.2 GC 的视角

GC 不关心你“逻辑上觉得还在引用它”, 它关心的是:

> 这条引用是否进入 GC 可见引用图

如果没有进入, 那么对 GC 来说:

- 这条边根本不存在
- 对象依旧可能被判定为不可达

### 3.3 一个简单示意

```text
Root -> PlayerController -> InventoryComponent -> CurrentItem
```

如果这整条链上的引用都是 GC 可见的, `CurrentItem` 就可达。  
但如果中间某一段是 GC 看不见的裸指针:

```text
Root -> PlayerController -> [native raw ptr, GC不可见] -> CurrentItem
```

那么图在 GC 看来就断了。

---

## 四、`CollectGarbage` 的真实位置

### 4.1 源码位置

- `Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectGlobals.h`

### 4.2 当前公开签名

```cpp
COREUOBJECT_API void CollectGarbage(EObjectFlags KeepFlags, bool bPerformFullPurge = true);
COREUOBJECT_API bool TryCollectGarbage(EObjectFlags KeepFlags, bool bPerformFullPurge = true);
```

这里先只看三个结论:

1. GC 的公开入口并不神秘
2. `KeepFlags` 会影响本轮保留策略
3. full purge 是一维, 不是“GC 唯一模式”

### 4.3 `TryCollectGarbage` 为什么存在

这说明 GC 还是受引擎实时同步条件约束的:

- 有时当前线程状态不适合立刻跑 GC
- 某些时候宁可延后, 也不马上强推

这进一步证明:

> GC 不只是算法问题, 也是运行时调度问题

---

## 五、`GARBAGE_COLLECTION_KEEPFLAGS`

### 5.1 源码位置

- `Engine/Source/Runtime/CoreUObject/Public/UObject/GarbageCollection.h`

### 5.2 当前定义

```cpp
#define GARBAGE_COLLECTION_KEEPFLAGS (GIsEditor ? RF_Standalone : RF_NoFlags)
```

### 5.3 这说明什么

- 编辑器环境下, 默认保留策略更宽
- 游戏环境下, 默认 `KeepFlags` 更收敛

这件事实际很重要。  
因为很多“编辑器里对象怎么没回收”的问题, 不只是引用图问题, 也和保留策略有关。

---

## 六、GC 系统里的几个核心实体

这一节先做第一次认识, 后面课程再逐个展开。

### 6.1 `GUObjectArray`

源码位置:

- `Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectArray.h`

它是 UObject 的全局目录。  
GC 不是靠在内存里盲找对象, 而是依托这张全局表。

### 6.2 `FUObjectItem`

同样在:

- `UObjectArray.h`

每个 UObject 在全局数组中都对应一个 `FUObjectItem`。  
它保存的是 GC / UObject 运行时关心的元数据, 比如:

- 对象指针
- 内部状态位
- 弱引用序列信息
- cluster 归属信息

不要把它理解成“引用计数节点”。  
UE5 GC 的核心逻辑不是引用计数。

### 6.3 `FSchemaView`

源码位置:

- `Engine/Source/Runtime/CoreUObject/Public/UObject/GarbageCollectionSchema.h`

这是 UE5 GC 理解现代遍历方式的关键。  
你可以暂时把它记成:

> 一个类或结构体的 GC 引用布局描述

后面第二课会详细展开。

### 6.4 `FGCObject`

源码位置:

- `Engine/Source/Runtime/CoreUObject/Public/UObject/GCObject.h`

它负责把非 `UObject` 类型中持有的 `UObject` 引用桥接进 GC 图。  
也就是说:

- native 世界不是 GC 天然盲区
- 但必须显式接桥

### 6.5 `GUObjectClusters`

同样在:

- `UObjectArray.h`

它是 cluster 系统的全局容器。  
先记一句:

> cluster 是 GC 图压缩优化器, 不是另一套独立 GC

---

## 七、一次 GC 周期的高层图景

这一节只看大框架, 不深入每个阶段细节。

```text
CollectGarbage / TryCollectGarbage
    -> 预处理与锁管理
    -> MarkObjectsAsUnreachable
    -> 标记 keep / root / cluster root
    -> PerformReachabilityAnalysis
    -> 收集不可达对象
    -> 清理弱引用
    -> BeginDestroy / FinishDestroy / Purge
    -> 回调与统计
```

这里最重要的三个认识:

1. GC 是流水线, 不是单个动作
2. Reachability Analysis 是核心, 但不是全部
3. “对象被判死” 与 “对象内存立刻释放” 不是同一时刻

---

## 八、为什么 UE5 GC 不是“简单引用计数”

很多人一看到 `FUObjectItem`、内部标记、序列号, 会下意识误解:

- UE 大概是“引用计数 + 一点额外回收”

不是。

更准确的说法是:

- UE5 通过可达性分析判定对象是否应该存活
- 关键是对象能否从根集合沿着 GC 可见引用路径被到达

### 8.1 这个差异为什么重要

因为它决定你的思维方式要从:

- “我是不是还握着某个地址”

切换到:

- “这条引用是否进入了 GC 图”

这也是为什么 UE5 GC 课程如果只是讲“对象数量”和“手动触发 GC”, 基本是不够的。

---

## 九、三个你现在就该建立的心智模型

### 9.1 模型 1: UObject 世界和 native 世界并存

UE 不是纯 GC 语言。  
UObject 世界有 GC, native 世界仍然大量靠传统 C++ 管理。

### 9.2 模型 2: GC 看的是图

保活靠的是:

- root
- 可见引用边
- reachability

不是“你脑中觉得对象还在被用”。

### 9.3 模型 3: GC 是流程, 不是瞬时事件

一个对象经历:

- 活着
- 不可达
- 进入销毁
- 最终释放

这些阶段之间可能跨越多步甚至多帧。

---

## 十、第一轮源码导读

如果你现在就想配合源码开始读, 推荐顺序如下:

1. `UObjectGlobals.h`
2. `GarbageCollection.h`
3. `UObjectArray.h`
4. `GCObject.h`
5. `GarbageCollectionSchema.h`

### 读完这一轮, 至少应该能回答

1. GC 入口 API 是什么
2. `GUObjectArray` 是什么
3. `FUObjectItem` 是什么
4. `FGCObject` 在解决什么问题
5. `FSchemaView` 是什么

---

## 十一、常见面试题

### 面试题 1

UE5 GC 管理什么? 不管理什么?

### 回答要点

- 主要管理 `UObject` 生态中的对象生命周期
- 不直接管理普通 native 对象和 GPU 等资源
- UE 的内存管理是多套机制并存

### 面试题 2

UE5 GC 是引用计数吗?

### 回答要点

- 不是核心依赖引用计数
- 核心是基于 root 和引用图的可达性判活

### 面试题 3

为什么一个 `UObject*` 裸指针不一定能保活对象?

### 回答要点

- 因为 GC 看的不是“有没有地址变量”
- 而是“引用是否进入 GC 可见图”

---

## 十二、本节自测

1. 为什么 UE5 不能把所有对象都交给 UObject GC 管?
2. `TryCollectGarbage` 的存在说明了什么?
3. `GARBAGE_COLLECTION_KEEPFLAGS` 为什么会区分编辑器和游戏环境?
4. 为什么“对象当前没被释放”不等于“对象仍然从根可达”?
5. `FUObjectItem` 为什么不能简单理解成“引用计数槽位”?

---

## 十三、本节的一个结论句

如果你要用一句话总结本节, 可以记成:

> UE5 GC 不是“自动回收一切内存”的黑盒, 而是专门管理 UObject 图谱可达性的分阶段系统。

---

## 源码锚点

- `Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectGlobals.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/GarbageCollection.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectArray.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/GCObject.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/GarbageCollectionSchema.h`
