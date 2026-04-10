# Lesson 2: 引用发现机制与 Schema

## 这节课解决什么问题

如果第一节回答了“GC 管什么”, 那第二节要回答的就是:

> GC 如何知道“谁引用了谁”

这是整套 GC 原理里最关键的核心之一。  
因为后面所有问题最终都会落回这里:

- 对象为什么没被回收
- 对象为什么被错误回收
- 为什么 `UPROPERTY` / `TObjectPtr` 重要
- 为什么 native 容器里的 `UObject*` 默认不安全

---

## 一、GC 眼中的“引用”与 C++ 眼中的“引用”

### 1.1 从 C++ 看, 你可能觉得这些都在“引用对象”

- 成员指针
- 局部变量指针
- `TArray<UObject*>`
- `TWeakObjectPtr`
- 某个 native 管理器里的缓存指针

### 1.2 从 GC 看, 问题只有一个

> 这条边会不会在 GC 遍历时被发现?

如果不会, 那么这条边在 GC 视角里就等于不存在。

### 1.3 一个极其重要的学习结论

GC 语义上的强引用不是“语法上有个指针”, 而是:

> GC 在做可达性分析时能沿着它继续走到目标对象

---

## 二、GC 强引用的主要来源

### 2.1 反射可见成员

这是最常规、最推荐的一层:

- 反射系统中的对象成员
- `TObjectPtr`
- 反射容器中的对象引用

### 2.2 `AddReferencedObjects`

当反射表达不了引用关系时, 由对象自己手动向 GC 报告额外引用。

### 2.3 `FGCObject`

让非 `UObject` 类型参与引用上报。

### 2.4 其他系统级路径

- RootSet
- KeepFlags
- cluster
- 外部 tracer / 某些桥接系统

如果你想用一句话总括:

> UE5 通过“反射可见成员 + schema + ARO + FGCObject + 系统级根”共同构成可达性图。

---

## 三、为什么裸指针是 GC 课程里的高频坑

### 3.1 典型误区

```cpp
UObject* CachedObject;
```

很多人会觉得:

- “我都缓存起来了, 对象当然还活着”

但 GC 不会这么想。

### 3.2 GC 的判断方式

GC 会问:

- 这个成员是否属于 GC 可见引用布局?
- 是否会在 traversal 中被枚举出来?

如果答案是否定的:

- 这条边不在图里
- 对象一样可能被判定为不可达

### 3.3 这为什么尤其危险

因为裸指针问题通常不是“立刻崩”。  
而是:

- 平时偶尔还能用
- 某轮 GC 后突然出问题

这类 bug 非常具有迷惑性。

---

## 四、`ReferenceSchema` 是 UE5 GC 现代化的核心

### 4.1 源码位置

- `Engine/Source/Runtime/CoreUObject/Public/UObject/GarbageCollectionSchema.h`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollectionSchema.cpp`

### 4.2 它在解决什么问题

最原始的想法是:

- 每遇到一个对象
- 临时用反射把字段挨个检查一遍

这在工程上成本高, 也不够稳定。  
UE5 的做法更现代:

- 把类和结构体的引用布局提前组织为 schema
- traversal 时尽量直接按 schema 访问

### 4.3 `FSchemaView` 是什么

可以把它理解成:

> 一个类或结构体的 GC 成员访问计划

它告诉遍历器:

- 哪些偏移是对象引用
- 哪些是引用数组
- 哪些地方要继续进入 struct
- 哪些地方需要调用 ARO

---

## 五、`EMemberType` 透露了 GC 实际会处理哪些成员形态

在 `GarbageCollectionSchema.h` 中, 当前版本可以看到这些成员类别:

- `Reference`
- `ReferenceArray`
- `StructArray`
- `StridedArray`
- `StructSet`
- `FieldPath`
- `FieldPathArray`
- `Optional`
- `DynamicallyTypedValue`
- `ARO`
- `SlowARO`
- `MemberARO`

### 5.1 这说明什么

GC 不只是在扫:

- “单个 `UObject*` 字段”

它还要处理:

- 数组里的引用
- 容器中的结构体成员
- 字段路径
- 需要手写上报的复杂引用

### 5.2 为什么这个细节重要

因为它直接决定了:

- 为什么对象数量不是 GC 成本的唯一来源
- 为什么布局复杂度和容器嵌套会显著影响 traversal 成本

---

## 六、`AddReferencedObjects` 在今天依然很重要

### 6.1 一个很常见的误解

“既然 UE5 有 schema, 那 `AddReferencedObjects` 应该已经不重要了。”

不是。

### 6.2 更准确的理解

- schema 负责大多数规则化、可描述的强引用
- `AddReferencedObjects` 负责反射不方便表达的复杂关系
- 两者不是竞争关系, 而是协作关系

### 6.3 什么时候应该考虑 ARO

适合:

- 复杂自定义容器
- 特殊聚合对象
- 反射不方便直接表达的引用关系

不适合:

- 本来可以用正常反射成员, 却为了偷懒全部塞进 ARO

---

## 七、`FGCObject`：让 native 世界接入 GC 图

### 7.1 源码位置

- `Engine/Source/Runtime/CoreUObject/Public/UObject/GCObject.h`

### 7.2 它解决的真实问题

场景:

- 你有一个普通 C++ 类型
- 它不是 `UObject`
- 但它内部持有若干 `UObject*`

此时反射系统帮不了你, 但你又确实需要保活这些对象。  
`FGCObject` 就是这条桥。

### 7.3 它的核心接口

你至少会实现:

```cpp
virtual void AddReferencedObjects(FReferenceCollector& Collector) = 0;
virtual FString GetReferencerName() const = 0;
```

### 7.4 学习重点

不要把它理解成:

- “另一种写 UObject 成员的方式”

更准确的理解是:

- “非 UObject 类型进入 GC 图的桥接机制”

---

## 八、`TObjectPtr`、裸指针、弱引用、软引用的 GC 语义差异

### 8.1 `TObjectPtr`

更接近 UE5 推荐的现代对象成员表达形式。  
它在增量 GC 语境下也更自然地协同 reachable barrier 路径。

### 8.2 裸指针

只有在特定上下文中才能安全使用。  
如果你想依靠它表达 GC 强引用, 默认是不可靠的。

### 8.3 `TWeakObjectPtr`

负责“安全观测对象是否还活着”。  
不负责保活。

### 8.4 `TSoftObjectPtr`

偏资源定位与延迟加载。  
不是常规强引用。

---

## 九、增量 GC 为什么会迫使我们更重视引用表达方式

当 GC 是增量进行时, 对象图会在 traversal 进行中继续变化。  
这时:

- `TObjectPtr` 等路径可以更自然参与 reachable 修正机制
- 裸指针则更容易彻底游离在 GC 可见性之外

这也是为什么在现代 UE5 工程实践中:

- “对象引用表达方式”不再只是代码风格问题
- 它直接关联 GC 正确性和增量协作能力

---

## 十、第二轮源码导读

建议阅读顺序:

1. `GarbageCollectionSchema.h`
2. `GarbageCollectionSchema.cpp`
3. `GCObject.h`
4. `FastReferenceCollector.h`
5. `ObjectPtr.h`

### 带着这些问题去看

1. schema 到底描述了哪些成员种类?
2. ARO 在 schema 体系里扮演什么角色?
3. `FGCObject` 为什么必须存在?
4. 增量期间为什么 `ObjectPtr.h` 很重要?

---

## 十一、常见面试题

### 面试题 1

什么是 `ReferenceSchema`?

### 回答要点

- 类或结构体的 GC 引用布局描述
- 帮助 traversal 高效访问强引用成员
- UE5 GC 现代化的重要基础之一

### 面试题 2

为什么 `AddReferencedObjects` 在 UE5 里仍然重要?

### 回答要点

- schema 不能替代所有复杂引用表达
- ARO 仍然负责补足特殊引用关系

### 面试题 3

`FGCObject` 解决什么问题?

### 回答要点

- 让非 UObject 持有者把自己的 UObject 引用接入 GC 图

### 面试题 4

为什么裸指针不能天然保活对象?

### 回答要点

- 因为 GC 看的是可见引用图
- 裸指针默认不在图里

---

## 十二、本节自测

1. 什么是 GC 语义上的强引用?
2. 为什么“有个指针”不等于“GC 能看见”?
3. schema 与 ARO 的关系是什么?
4. `FGCObject` 为什么不是 `UPROPERTY` 的替代品?
5. 为什么对象引用的表达方式会影响增量 GC 的正确性?

---

## 十三、本节的一个结论句

> UE5 GC 不是靠“临时猜字段”建立引用图, 而是通过 schema、ARO、FGCObject 与系统级根共同构造一张可遍历的可达性图。

---

## 源码锚点

- `Engine/Source/Runtime/CoreUObject/Public/UObject/GarbageCollectionSchema.h`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/GarbageCollectionSchema.cpp`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/FastReferenceCollector.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/GCObject.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/ObjectPtr.h`
