# Lesson 2: 引用图如何被 UE5 看见

## 本节定位

如果说第一课解决的是“GC 管什么”, 那这一课解决的就是:

> GC 到底靠什么知道“谁引用了谁”?

这是整套课程里最核心的一节之一。  
因为只要“引用图怎么建立”这件事搞懂了, 以后你分析:

- 对象为什么活着
- 对象为什么死掉
- 为什么裸指针会出事
- 为什么 `TObjectPtr` / `UPROPERTY` 很重要

都会轻松很多。

---

## 学习目标

学完本节, 你应该能掌握:

- GC 语义上的强引用到底是什么
- 为什么“有个指针”不等于“GC 能看见”
- `ReferenceSchema` 在 UE5 里的核心作用
- `AddReferencedObjects` 在新架构中的位置
- `FGCObject` 如何把非 UObject 世界接进 GC 图
- `TObjectPtr`、裸指针、弱引用、软引用的 GC 语义差异

---

## 一、GC 眼中的引用, 和 C++ 眼中的引用不是一回事

### 1.1 从 C++ 看

下面这些都像“我引用了一个对象”:

- 成员指针
- 局部变量指针
- `TArray<UObject*>`
- `TWeakObjectPtr`
- `TSharedPtr<SomeWrapperHoldingUObject>`

### 1.2 从 GC 看

GC 只关心一个问题:

> 这条引用会不会在 GC 遍历时被发现?

如果不会, 那你逻辑上就算“觉得自己还拿着对象”, GC 也不会理你。

### 1.3 这为什么重要

因为很多初学者的思维还是:

- “这个指针还在”
- “这个对象应该没问题”

但在 UE5 里, 更准确的说法应该是:

- “这条引用是否进入了 GC 可见图?”

---

## 二、GC 语义上的强引用是什么

### 2.1 一个实用定义

你可以把“GC 强引用”定义成:

> GC 在做可达性分析时, 能沿着它继续走到下一个对象的引用。

### 2.2 常见强引用来源

- 反射系统里的 `UObject*`
- 反射系统里的 `TObjectPtr`
- 反射可见容器中的对象引用
- `AddReferencedObjects` 手动上报的引用
- `FGCObject` 上报的引用
- 根集合对象扩展出的引用
- cluster 内有效对象关系

### 2.3 你应该形成的心智模型

GC 看的是一张图:

```text
Root
  -> Object A
      -> Object B
          -> Object C
```

只有图上的边, 才算“GC 真正承认的引用”。

---

## 三、哪些东西通常不是 GC 强引用

### 3.1 裸成员指针

```cpp
UObject* RawPtr;
```

如果它没有通过反射体系、ARO 或 `FGCObject` 暴露出来, GC 默认看不见。

### 3.2 弱引用

```cpp
TWeakObjectPtr<UObject> WeakPtr;
```

它的目标是:

- 安全观测
- 避免悬空指针

不是:

- 阻止 GC 回收

### 3.3 软引用

```cpp
TSoftObjectPtr<UObject> SoftPtr;
```

它更偏向:

- 延迟加载入口
- 路径引用

默认不是强保活工具。

### 3.4 Native 对象中的普通容器

```cpp
class FManager
{
public:
    TArray<UObject*> Cached;
};
```

如果这个 `FManager` 不是反射对象, 也没走 `FGCObject`, 那这个数组并不会自动接进 GC。

---

## 四、为什么“裸指针明明还能访问, 但对象却被 GC 了”

这是一个非常典型的问题。

你可以把整个误区拆成三步:

### 4.1 业务逻辑视角

“我还留着地址, 所以对象应该还活着。”

### 4.2 GC 视角

“这条地址引用没有进入可达性图。”

### 4.3 结果

- 本轮 GC 无法从根走到该对象
- 对象被判为不可达
- 回收后你的裸指针变成危险地址

所以正确结论不是:

- “UE GC 不稳定”

而是:

- “我没有把引用关系用 GC 能看见的方式表达出来”

---

## 五、UE5 为什么要引入 `ReferenceSchema`

### 5.1 源码位置

- `Engine/Source/Runtime/CoreUObject/Public/UObject/GarbageCollectionSchema.h`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollectionSchema.cpp`

### 5.2 传统想象 vs UE5 实际

很多人脑中的 GC 遍历是:

- 遇到对象
- 反射看它有哪些字段
- 逐字段判断有没有 UObject 引用

这不是完全错, 但对 UE5 来说已经太粗了。

UE5 更强调:

- 把对象引用布局提前组织成 schema
- 运行时尽量直接按 schema 遍历

### 5.3 `FSchemaView` 在表达什么

你可以把 `FSchemaView` 理解成:

- 一个类或结构体的 GC 成员访问计划

它描述的内容包括:

- 普通对象引用
- 引用数组
- 结构体数组
- set / map 等结构中的成员
- `FieldPath`
- `Optional`
- `AddReferencedObjects` 调用点
- Verse value 等特殊值类型

这意味着 GC 在遍历对象时, 不是“边想边看”, 而是“按预定路线走”。

---

## 六、`EMemberType` 能帮你读懂 schema 的世界观

在 `GarbageCollectionSchema.h` 中, `EMemberType` 包含多种成员类别:

- `Reference`
- `ReferenceArray`
- `StructArray`
- `StructSet`
- `FieldPath`
- `Optional`
- `DynamicallyTypedValue`
- `ARO`
- `SlowARO`
- `MemberARO`

### 6.1 这说明 GC 处理的不是单一场景

GC 不只是处理:

- “这个类里有个 `UObject*`”

它还要处理:

- 数组
- 嵌套结构体
- 容器中的结构元素
- 需要手写引用收集逻辑的对象

### 6.2 为什么这对学习很重要

因为很多 GC 性能问题, 本质上跟“类和结构体的引用布局复杂度”有关, 而不只是对象总数。

---

## 七、`AddReferencedObjects` 在新架构里仍然很重要

### 7.1 一个常见误解

“既然有 schema 了, `AddReferencedObjects` 是不是已经过时了?”

不是。

### 7.2 更准确的理解

- 普通、规则化的成员引用, 由 schema 直接描述
- 反射不方便表达的复杂引用, 仍然通过 `AddReferencedObjects` 暴露
- schema 会记录哪里需要调用这些 ARO 路径

### 7.3 什么时候适合用 `AddReferencedObjects`

适合:

- 自定义容器关系
- 不好用普通反射字段表达的对象集合
- 一些特殊聚合对象

不适合:

- 本来就能写成正常反射字段, 却因为偷懒全塞到 ARO

过度依赖 ARO 的代价是:

- 可读性下降
- 调试难度增加
- 遍历成本可能更难控制

---

## 八、`FGCObject` 是非 UObject 世界进入 GC 图的桥

### 8.1 源码位置

- `Engine/Source/Runtime/CoreUObject/Public/UObject/GCObject.h`

### 8.2 它解决的是什么问题

场景如下:

- 你有一个普通 C++ 类型
- 它不是 `UObject`
- 但它内部又持有若干 `UObject*`
- 这些对象在它存活期间必须被正确保活

此时反射系统帮不到你, 就需要 `FGCObject`。

### 8.3 核心思想

不是“让 native 对象也变成 UObject”, 而是:

- 让 native 对象声明“我引用了哪些 UObject”

这样 GC 才能把它接进整张可达性图。

### 8.4 教学示意代码

下面是示意代码, 不是引擎源码:

```cpp
class FInventoryCache : public FGCObject
{
public:
    TObjectPtr<UObject> MainItem;
    TArray<TObjectPtr<UObject>> LoadedItems;

    virtual void AddReferencedObjects(FReferenceCollector& Collector) override
    {
        Collector.AddReferencedObject(MainItem);
        Collector.AddReferencedObjects(LoadedItems);
    }

    virtual FString GetReferencerName() const override
    {
        return TEXT("FInventoryCache");
    }
};
```

真正要学到的是:

- native 世界不是 GC 盲区
- 但你必须显式接桥

---

## 九、`FGCObject` 不是“偷懒不用 UPROPERTY 的万能替代品”

优先级应该是:

1. 能放在 `UObject` 反射字段里, 先放进去
2. 反射表达不了, 再考虑 `AddReferencedObjects`
3. 对象根本不是 `UObject`, 再考虑 `FGCObject`

如果你反过来:

- 什么都想塞进 `FGCObject`

那往往意味着你的对象建模有问题。

---

## 十、根集合到底从哪里来

要理解引用图, 必须理解图从哪里开始。

根来源大致包括:

- `RF_RootSet`
- 本轮 `KeepFlags` 命中的对象
- cluster root
- 某些全局保活对象
- `FGCObject` / `UGCObjectReferencer` 暴露出的引用
- 外部 tracer 提供的根

### 10.1 为什么这个概念重要

因为“对象能活着”真正意味着:

> 能从某个根出发, 一路沿 GC 可见引用走到它。

这比“谁还有一个变量指向它”准确得多。

---

## 十一、增量 GC 为什么逼着我们关注 barrier

### 11.1 问题背景

如果 reachability analysis 是跨多帧做的:

- 第 1 帧 GC 开始遍历
- 第 2 帧业务逻辑又写入了一个新引用

那 GC 怎么知道这个新引用?

### 11.2 UObject 侧的答案

源码位置:

- `Engine/Source/Runtime/CoreUObject/Public/UObject/ObjectPtr.h`

当:

```cpp
UE::GC::GIsIncrementalReachabilityPending
```

为真时, `FObjectPtr` / `TObjectPtr` 某些路径会触发:

```cpp
UE::GC::MarkAsReachable(...)
```

### 11.3 这件事揭示了什么

incremental 不是简单的“做一半停一下”。  
它还意味着:

- 运行中对象图可能变化
- GC 需要动态修正 reachable 集合

所以 barrier 机制是增量 GC 正确性的组成部分。

---

## 十二、四个典型错误案例

### 12.1 错误案例 1: 裸指针成员

```cpp
UObject* RawPtr;
```

GC 看不见。

### 12.2 错误案例 2: 弱引用误当强引用

```cpp
TWeakObjectPtr<UObject> WeakPtr;
```

能观测, 不能保活。

### 12.3 错误案例 3: native 容器缓存 UObject

```cpp
TArray<UObject*> Cached;
```

如果所在对象没接入 GC, 一样不保活。

### 12.4 错误案例 4: 以为“局部变量没出作用域就安全”

GC 看的是全局可达性图, 不是你当前函数栈上的主观感觉。

---

## 十三、学习时的源码阅读顺序

建议这样读:

1. `GarbageCollectionSchema.h`
   先看 `EMemberType`、`FSchemaView`
2. `GarbageCollectionSchema.cpp`
   看 schema 如何构建
3. `GCObject.h`
   看 native 如何接桥
4. `ObjectPtr.h`
   看 incremental 下的 reachable barrier
5. `FastReferenceCollector.h`
   最后再看 traversal 如何消费这些信息

---

## 十四、本节后的自检题

1. GC 语义上的强引用和 C++ 语义上的“有个指针”有什么区别?
2. 为什么 `ReferenceSchema` 是 UE5 GC 的主干?
3. `AddReferencedObjects` 在新版 GC 里为什么依然重要?
4. `FGCObject` 适合解决什么问题?
5. 为什么增量 GC 课程里必须讲 `MarkAsReachable` 这类 barrier 逻辑?
6. 为什么“native 容器里有 UObject*”默认不等于“对象会被保活”?

---

## 十五、进入下一课前的完成标准

如果你现在已经能顺着讲清下面这句话, 就可以进入第三课:

> UE5 的对象存活, 不是靠“谁还拿着地址”, 而是靠“是否能从根沿着 GC 看得见的引用图走到它”; 这张图主要由 `ReferenceSchema`、`AddReferencedObjects`、`FGCObject` 等机制共同组成。

---

## 源码锚点

- `Engine/Source/Runtime/CoreUObject/Public/UObject/GarbageCollectionSchema.h`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollectionSchema.cpp`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/FastReferenceCollector.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/GCObject.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/ObjectPtr.h`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollection.cpp`
