# Lesson 2: 对象追踪与引用图

## 课程概述

本节课深入讲解UE5如何追踪对象和构建引用图，包括FUObjectArray、引用Schema系统、以及FGCObject机制。

**学习目标：**
- 掌握FUObjectArray全局对象池的工作原理
- 理解引用Schema系统的设计与优势
- 学会使用FGCObject让非UObject参与GC
- 了解并行引用遍历的实现机制

---

## 一、FUObjectArray - 全局对象池

### 1.1 设计原理

```cpp
// FUObjectArray 是所有UObject的中央注册表
// Source: Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectArray.h

class FUObjectArray
{
public:
    // 对象池 - 所有UObject的指针
    TArray<struct FUObjectItem*> ObjectArray;

    // 空闲索引列表 - 用于对象复用
    TArray<int32> AvailableIndices;

    // 统计信息
    int32 MaxObjectsEver;           // 历史最大对象数
    int32 ObjFirstGCIndex;          // 第一个可GC对象索引

    // 对象创建
    int32 AllocateUObjectIndex(UObjectBase* Object, bool bMappable);

    // 对象销毁
    void FreeUObjectIndex(UObjectBase* Object);

    // 遍历接口
    FUObjectItem* IndexToObject(int32 Index);

    // 全局实例声明
    extern COREUOBJECT_API FUObjectArray GUObjectArray;
};
```

### 1.2 FUObjectItem 结构

```cpp
// 每个UObject对应的元数据条目
// Source: UObjectArray.h

struct FUObjectItem
{
    // ===== 核心数据 =====
    // 打包的标志位和引用计数 (原子操作)
    mutable std::atomic<int64> FlagsAndRefCount;

    // 对象指针 (可能与标志位打包)
    UObjectBase* Object;

    // 弱引用序列号 - 防止索引复用问题
    int32 SerialNumber;

    // 聚类根索引 - 用于聚类系统
    int32 ClusterRootIndex;

    // ===== 标志位操作 =====
    EInternalObjectFlags GetFlags() const;
    void SetFlags(EInternalObjectFlags Flags);
    void ClearFlags(EInternalObjectFlags Flags);

    // ===== 引用计数操作 =====
    int32 GetRefCount() const;
    void AddRef();
    void Release();

    // ===== 可达性检查 =====
    bool IsReachable() const;
    bool IsUnreachable() const;
};
```

### 1.3 对象索引分配机制

```
┌─────────────────────────────────────────────────────────────────┐
│                    FUObjectArray 索引分配机制                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  索引范围划分:                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  0 - ObjFirstGCIndex-1  │  永久对象 (Engine, Core等)    │    │
│  │  - 永不参与GC            │  - 引擎核心对象              │    │
│  │  - 始终存活              │                              │    │
│  ├─────────────────────────┼────────────────────────────────┤    │
│  │  ObjFirstGCIndex+       │  可GC对象 (普通UObject)       │    │
│  │  - 参与GC标记            │  - 游戏运行时对象            │    │
│  │  - 可被回收              │                              │    │
│  └─────────────────────────┴────────────────────────────────┘    │
│                                                                 │
│  索引分配流程:                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  NewObject() 调用                                      │    │
│  │         │                                               │    │
│  │         ▼                                               │    │
│  │  AllocateUObjectIndex()                                 │    │
│  │         │                                               │    │
│  │         ├── 检查 AvailableIndices                      │    │
│  │         │      │                                        │    │
│  │         │      ├── 有空闲索引 → 复用                    │    │
│  │         │      │                                        │    │
│  │         │      └── 无空闲索引 → 扩展数组                │    │
│  │         │                                               │    │
│  │         └── 检查 MaxObjectsInGame 限制                  │    │
│  │                │                                        │    │
│  │                └── 超过限制 → 报错                       │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.4 索引分配源码

```cpp
// 对象索引分配实现
// Source: UObjectArray.cpp

int32 FUObjectArray::AllocateUObjectIndex(UObjectBase* Object, bool bMappable)
{
    // 1. 尝试复用空闲索引
    int32 Index;
    if (AvailableIndices.Num() > 0)
    {
        // 从空闲列表取出
        Index = AvailableIndices.Pop();

        // 重置FUObjectItem
        FUObjectItem* Item = ObjectArray[Index];
        Item->Object = Object;
        Item->FlagsAndRefCount = 0;
        // 递增序列号，防止弱引用误用
        Item->SerialNumber++;
    }
    else
    {
        // 2. 扩展数组
        Index = ObjectArray.Num();

        // 检查对象上限
        int32 MaxObjects = GIsEditor ?
            gc.MaxObjectsInEditor : gc.MaxObjectsInGame;

        if (Index >= MaxObjects)
        {
            UE_LOG(LogGarbage, Error,
                TEXT("Object count exceeded limit: %d >= %d"),
                Index, MaxObjects);
            return INDEX_NONE;
        }

        // 创建新的FUObjectItem
        FUObjectItem* NewItem = new FUObjectItem();
        NewItem->Object = Object;
        NewItem->FlagsAndRefCount = 0;
        NewItem->SerialNumber = 0;
        ObjectArray.Add(NewItem);
    }

    // 更新统计
    MaxObjectsEver = FMath::Max(MaxObjectsEver, Index + 1);

    return Index;
}
```

---

## 二、引用Schema系统

### 2.1 传统序列化 vs Schema

```cpp
// ===== 传统方式: 使用Archive序列化遍历引用 =====
// 问题: 每次GC都需要创建Archive, 效率低

void UObject::Serialize(FArchive& Ar)
{
    Super::Serialize(Ar);
    Ar << MyObjectPtr;
    Ar << MyObjectArray;
    // 每次遍历都要执行这些代码
}

// ===== UE5 Schema方式: 预计算引用布局 =====
// 优势: 零运行时开销

struct FSchemaView
{
    // 成员描述符数组 (编译期生成)
    const FMemberWord* Members;
    int32 NumMembers;

    // 快速遍历所有引用
    template<typename Func>
    void ForEachReference(const void* Container, Func&& Func) const
    {
        for (int32 i = 0; i < NumMembers; ++i)
        {
            const FMemberWord& Member = Members[i];
            ProcessMember(Container, Member, Func);
        }
    }
};
```

### 2.2 引用类型枚举

```cpp
// Source: GarbageCollectionSchema.h

enum class EMemberType : uint8
{
    // ===== 基本引用类型 =====
    Reference,           // UObject*, TObjectPtr<T>
    ReferenceArray,      // TArray<UObject*>
    ReferenceSet,        // TSet<UObject*>
    ReferenceMap,        // TMap<Key, UObject*>

    // ===== 结构体容器 =====
    Struct,              // 包含引用的结构体
    StructArray,         // TArray<FStructWithRefs>

    // ===== 特殊类型 =====
    FieldPath,           // 强引用到Field所有者
    ARO,                 // AddReferencedObjects回调
    SlowARO,             // 慢速ARO (批量处理)

    // ===== 指针包装器 =====
    Optional,            // TOptional<T>
    UniquePtr,           // TUniquePtr<T>
    SharedPtr,           // TSharedPtr<T>

    // ===== 非引用类型 =====
    NonReference,        // 非引用数据
};
```

### 2.3 FMemberWord 结构

```cpp
// 打包的成员描述符
// 使用紧凑位域存储，最小化内存占用

struct FMemberWord
{
    uint32 Offset;          // 成员在类中的字节偏移量
    EMemberType Type : 8;   // 引用类型
    uint8 Depth;            // 嵌套深度 (用于结构体)
    uint16 Count;           // 固定数组元素数
    FSchemaView* SubSchema; // 子Schema (用于嵌套结构体)

    // ===== 便捷方法 =====

    // 获取单个引用指针
    UObject** GetReference(void* Container) const
    {
        return (UObject**)((uint8*)Container + Offset);
    }

    // 获取引用数组
    TArray<UObject*>* GetReferenceArray(void* Container) const
    {
        return (TArray<UObject*>*)((uint8*)Container + Offset);
    }

    // 获取结构体指针
    void* GetStruct(void* Container) const
    {
        return (void*)((uint8*)Container + Offset);
    }
};
```

### 2.4 Schema自动生成示例

```cpp
// UCLASS自动生成Schema
// 编译器在构建时生成

UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

    UPROPERTY()
    UObject* MyObject;                    // 单个引用

    UPROPERTY()
    TArray<UObject*> MyArray;             // 引用数组

    UPROPERTY()
    FMyDataStruct MyStruct;               // 包含引用的结构体

    UPROPERTY()
    TMap<FName, UObject*> MyMap;          // 引用Map
};

// 编译器生成的Schema (简化版)
static const FMemberWord AMyCharacter_Members[] =
{
    { offsetof(AMyCharacter, MyObject),  EMemberType::Reference,      0, 1, nullptr            },
    { offsetof(AMyCharacter, MyArray),   EMemberType::ReferenceArray, 0, 0, nullptr            },
    { offsetof(AMyCharacter, MyStruct),  EMemberType::Struct,         0, 0, &FMyDataStruct::Schema },
    { offsetof(AMyCharacter, MyMap),     EMemberType::ReferenceMap,   0, 0, nullptr            },
};

static const FSchemaView AMyCharacter_Schema =
{
    AMyCharacter_Members,
    ARRAY_COUNT(AMyCharacter_Members)
};
```

---

## 三、引用遍历实现

### 3.1 TFastReferenceCollector

```cpp
// 快速引用收集器模板
// Source: FastReferenceCollector.h

template<typename CollectorType>
class TFastReferenceCollector
{
public:
    // 遍历单个对象的所有引用
    FORCEINLINE void ProcessObject(UObject* Object)
    {
        // 1. 获取对象的Schema
        FSchemaView Schema = Object->GetClass()->GetSchema();

        // 2. 遍历所有成员引用
        Schema.ForEachReference(Object, [this](UObject*& Ref)
        {
            if (Ref && !Ref->IsUnreachable())
            {
                // 处理引用 (由CollectorType决定具体行为)
                static_cast<CollectorType*>(this)->HandleReference(Ref);
            }
        });

        // 3. 处理AddReferencedObjects回调
        if (Object->HasARO())
        {
            Object->AddReferencedObjects(
                static_cast<CollectorType*>(this)->GetCollector());
        }
    }

    // 批量处理对象 (用于并行)
    void ProcessObjects(TArrayView<UObject*> Objects)
    {
        for (UObject* Obj : Objects)
        {
            ProcessObject(Obj);
        }
    }
};
```

### 3.2 预取优化

```cpp
// 缓存友好的遍历
// 使用预取减少内存延迟

class FPrefetchingObjectIterator
{
    static constexpr int32 Lookahead = 16;  // 预取距离

    void PrefetchNext(UObject** Objects, int32 CurrentIndex, int32 Total)
    {
        // 预取未来Lookahead个对象的元数据
        for (int32 i = 1; i <= Lookahead && (CurrentIndex + i) < Total; ++i)
        {
            UObject* FutureObj = Objects[CurrentIndex + i];
            if (FutureObj)
            {
                // 预取对象本身
                FPlatformMisc::Prefetch(FutureObj);

                // 预取相关数据 (这些数据马上要用到)
                FPlatformMisc::Prefetch(FutureObj->GetClass());
                FPlatformMisc::Prefetch(FutureObj->GetOuter());

                // 预取Schema
                FPlatformMisc::Prefetch(
                    FutureObj->GetClass()->GetSchema().Members);
            }
        }
    }
};
```

### 3.3 引用批处理

```cpp
// 批量处理引用减少缓存失效
// Source: FastReferenceCollector.h

class TReferenceBatcher
{
    // 验证队列 - 已验证可达的对象
    TArray<UObject*> ValidatedQueue;

    // 未验证队列 - 待验证的对象
    TArray<UObject*> UnvalidatedQueue;

    // 批处理阈值
    static constexpr int32 BatchSize = 64;

public:
    void AddReference(UObject* Ref)
    {
        UnvalidatedQueue.Add(Ref);

        // 达到批处理阈值时处理
        if (UnvalidatedQueue.Num() >= BatchSize)
        {
            ProcessBatch();
        }
    }

    void ProcessBatch()
    {
        // 批量验证 (更好的缓存利用率)
        for (UObject* Obj : UnvalidatedQueue)
        {
            if (Obj && !Obj->IsUnreachable())
            {
                ValidatedQueue.Add(Obj);
            }
        }
        UnvalidatedQueue.Reset();

        // 批量标记
        for (UObject* Obj : ValidatedQueue)
        {
            if (MarkAsReachable(Obj))
            {
                // 加入工作队列继续遍历
                WorkQueue.Push(Obj);
            }
        }
        ValidatedQueue.Reset();
    }
};
```

---

## 四、FGCObject 机制

### 4.1 为什么需要FGCObject？

```
┌─────────────────────────────────────────────────────────────────┐
│                    FGCObject 使用场景                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  问题: 非UObject类如何持有UObject引用?                           │
│                                                                 │
│  场景示例:                                                       │
│  ├── 子系统内部类                                               │
│  │   └── UGameInstanceSubsystem的内部管理器                     │
│  │                                                              │
│  ├── 单例管理器                                                 │
│  │   └── 非UObject的全局Manager类                               │
│  │                                                              │
│  ├── 工具类                                                     │
│  │   └── 持有UObject引用的工具类                                │
│  │                                                              │
│  ├── 插件内部类                                                 │
│  │   └── 第三方插件的内部数据结构                               │
│  │                                                              │
│  └── 异步任务                                                   │
│      └── 后台线程持有的临时引用                                  │
│                                                                 │
│  危险: 如果不继承FGCObject                                       │
│  ├── GC不知道这些引用的存在                                     │
│  ├── 对象可能被错误回收                                         │
│  └── 导致访问无效内存 → 崩溃!                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 FGCObject 实现

```cpp
// Source: GCObject.h

class COREUOBJECT_API FGCObject
{
public:
    // 构造时自动注册到全局链表
    FGCObject()
    {
        NextGCObject = FirstGCObject;
        FirstGCObject = this;
    }

    // 析构时自动从链表移除
    virtual ~FGCObject()
    {
        // 从链表移除
        FGCObject** Prev = &FirstGCObject;
        while (*Prev)
        {
            if (*Prev == this)
            {
                *Prev = NextGCObject;
                break;
            }
            Prev = &((*Prev)->NextGCObject);
        }
    }

    // ===== 必须实现 =====
    // 报告此对象持有的所有UObject引用
    virtual void AddReferencedObjects(FReferenceCollector& Collector) = 0;

    // ===== 可选实现 =====
    // 返回引用者名称 (调试用)
    virtual FString GetReferencerName() const
    {
        return TEXT("Unknown FGCObject");
    }

private:
    // 注册链表
    FGCObject* NextGCObject;
    static FGCObject* FirstGCObject;

    // 友元: 允许UGCObjectReferencer访问
    friend class UGCObjectReferencer;
};

// 全局引用收集器
class UGCObjectReferencer : public UObject
{
    virtual void AddReferencedObjects(FReferenceCollector& Collector) override
    {
        // 遍历所有FGCObject
        for (FGCObject* Obj = FGCObject::FirstGCObject;
             Obj != nullptr;
             Obj = Obj->NextGCObject)
        {
            Obj->AddReferencedObjects(Collector);
        }
    }
};
```

### 4.3 正确使用示例

```cpp
// ===== 正确示例 =====

class FAssetCache : public FGCObject
{
    // 持有的UObject引用
    TMap<FName, UObject*> CachedAssets;
    TArray<UObject*> LoadingAssets;
    UObject* CurrentAsset;

public:
    // 实现AddReferencedObjects
    virtual void AddReferencedObjects(FReferenceCollector& Collector) override
    {
        // 报告Map中的所有值
        for (auto& Pair : CachedAssets)
        {
            Collector.AddReferencedObject(Pair.Value);
        }

        // 报告数组中的所有对象
        Collector.AddReferencedObjects(LoadingAssets);

        // 报告单个对象
        Collector.AddReferencedObject(CurrentAsset);
    }

    virtual FString GetReferencerName() const override
    {
        return TEXT("FAssetCache");
    }
};

// ===== 错误示例 =====

class FBadManager
{
    UObject* Ref;  // 危险! GC不知道这个引用!

public:
    void SetRef(UObject* InRef)
    {
        Ref = InRef;
        // 如果Ref是唯一引用, GC会错误地回收它!
        // 后续访问Ref会导致崩溃!
    }
};
```

---

## 五、引用类型详解

### 5.1 强引用 vs 弱引用

```cpp
// ===== 强引用 (阻止GC回收) =====

// 方式1: UPROPERTY宏
UPROPERTY()
UObject* StrongRef1;  // 对象不会被GC

// 方式2: TObjectPtr (UE5推荐)
UPROPERTY()
TObjectPtr<UObject> StrongRef2;  // 同样阻止GC

// ===== 弱引用 (不阻止GC回收) =====

// TWeakObjectPtr - 不阻止GC
UPROPERTY()
TWeakObjectPtr<UObject> WeakRef;  // 对象可能被GC

// 安全访问方式
if (UObject* Obj = WeakRef.Get())
{
    // 对象仍然有效
    Obj->DoSomething();
}

// ===== 软引用 (延迟加载) =====

// TSoftObjectPtr - 可能未加载
UPROPERTY()
TSoftObjectPtr<UObject> SoftRef;

// 需要异步加载
SoftRef.LoadSynchronous();  // 同步加载
// 或使用 StreamableManager 异步加载
```

### 5.2 TObjectPtr 智能指针

```cpp
// UE5推荐的UObject指针类型
// Source: UObjectPtr.h

template<typename T>
class TObjectPtr
{
    // 存储对象索引而非直接指针
    mutable int32 ObjectIndex;

public:
    // 代理访问
    T* Get() const
    {
        // 从全局数组获取对象
        FUObjectItem* Item = GUObjectArray.IndexToObject(ObjectIndex);
        if (Item && Item->Object)
        {
            return static_cast<T*>(Item->Object);
        }
        return nullptr;
    }

    // 重载操作符
    T* operator->() const { return Get(); }
    T& operator*() const { return *Get(); }
    bool IsValid() const { return Get() != nullptr; }

    // 隐式转换
    operator T*() const { return Get(); }
};

// 优势:
// 1. 更快的NULL检查 (直接检查索引)
// 2. 更好的缓存局部性 (索引比指针小)
// 3. 支持对象重定位 (间接访问)
// 4. 编译时类型安全
```

### 5.3 引用类型对比表

| 类型 | 阻止GC | 序列化 | 加载时机 | 使用场景 |
|------|--------|--------|----------|----------|
| `UObject*` | 是 | 是 | 同步 | 普通引用 |
| `TObjectPtr<T>` | 是 | 是 | 同步 | UE5推荐替代 |
| `TWeakObjectPtr<T>` | 否 | 否 | - | 观察者模式 |
| `TSoftObjectPtr<T>` | 是 | 是 | 异步 | 跨关卡引用 |
| `TLazyObjectPtr<T>` | 否 | 是 | 按需 | 延迟加载 |

---

## 六、并行引用遍历

### 6.1 并行架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    并行引用遍历架构                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    ┌─────────────┐                              │
│                    │  主线程      │                              │
│                    │  分配工作    │                              │
│                    └──────┬──────┘                              │
│                           │                                     │
│           ┌───────────────┼───────────────┐                    │
│           │               │               │                    │
│           ▼               ▼               ▼                    │
│    ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│    │ Worker 0 │    │ Worker 1 │    │ Worker N │              │
│    │ 本地队列  │    │ 本地队列  │    │ 本地队列  │              │
│    │ 遍历引用  │    │ 遍历引用  │    │ 遍历引用  │              │
│    └────┬─────┘    └────┬─────┘    └────┬─────┘              │
│         │               │               │                      │
│         └───────────────┼───────────────┘                      │
│                         │                                      │
│                         ▼                                      │
│                ┌─────────────────┐                             │
│                │ 工作窃取队列     │                             │
│                │ (Work-Stealing) │                             │
│                │ 负载均衡         │                             │
│                └─────────────────┘                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 工作窃取实现

```cpp
// 工作窃取队列
// Source: GarbageCollection.cpp

class FWorkstealingQueue
{
    static constexpr int32 MaxWorkers = 16;
    static constexpr int32 StealThreshold = 64;

    // 每个Worker的本地队列
    TArray<UObject*> LocalQueues[MaxWorkers];

    // 全局共享队列
    TArray<UObject*> SharedQueue;

    // 并发控制
    std::atomic<int32> ActiveWorkers{0};

public:
    void Push(UObject* Obj, int32 WorkerIndex)
    {
        LocalQueues[WorkerIndex].Add(Obj);
    }

    UObject* Pop(int32 WorkerIndex)
    {
        // 1. 先从本地队列取
        if (LocalQueues[WorkerIndex].Num() > 0)
        {
            return LocalQueues[WorkerIndex].Pop();
        }

        // 2. 本地空了, 尝试从全局队列取
        if (SharedQueue.Num() > 0)
        {
            return SharedQueue.Pop();
        }

        // 3. 最后才窃取其他Worker的任务
        return Steal(WorkerIndex);
    }

private:
    UObject* Steal(int32 WorkerIndex)
    {
        // 从其他Worker的队列偷一半
        for (int32 i = 0; i < MaxWorkers; ++i)
        {
            int32 TargetIdx = (WorkerIndex + i + 1) % MaxWorkers;

            if (LocalQueues[TargetIdx].Num() > StealThreshold)
            {
                int32 StealCount = LocalQueues[TargetIdx].Num() / 2;

                for (int32 j = 0; j < StealCount; ++j)
                {
                    LocalQueues[WorkerIndex].Add(
                        LocalQueues[TargetIdx].Pop());
                }

                return LocalQueues[WorkerIndex].Pop();
            }
        }
        return nullptr;
    }
};
```

### 6.3 配置参数

```ini
; 并行GC配置
; DefaultEngine.ini 或运行时控制台

; 启用并行GC
gc.AllowParallelGC=1

; Worker数量 (0=自动, 基于CPU核心数)
gc.NumParallelGCWorkers=0

; 每个工作块的对象数
gc.WorkBlockSize=512

; 触发并行GC的最小对象数
gc.MinObjectsForParallelGC=1000
```

---

## 七、实践练习

### 练习1: 实现自定义FGCObject

```cpp
// 实现一个资产缓存管理器

class FAssetCacheManager : public FGCObject
{
    TMap<FName, TObjectPtr<UObject>> CachedAssets;
    TArray<TWeakObjectPtr<UObject>> WeakReferences;

public:
    virtual void AddReferencedObjects(FReferenceCollector& Collector) override
    {
        // 报告强引用
        for (auto& Pair : CachedAssets)
        {
            Collector.AddReferencedObject(Pair.Value);
        }

        // 注意: TWeakObjectPtr不需要报告!
        // 弱引用不阻止GC, 所以GC不需要追踪
    }

    virtual FString GetReferencerName() const override
    {
        return TEXT("FAssetCacheManager");
    }

    // 缓存资产
    void CacheAsset(FName Name, UObject* Asset)
    {
        CachedAssets.Add(Name, Asset);
    }

    // 获取缓存的资产
    UObject* GetCachedAsset(FName Name) const
    {
        if (TObjectPtr<UObject>* Found = CachedAssets.Find(Name))
        {
            return *Found;
        }
        return nullptr;
    }

    // 清除缓存
    void ClearCache()
    {
        CachedAssets.Empty();
        // 下次GC时会回收未使用的资产
    }
};
```

### 练习2: 调试引用问题

```cpp
// 追踪对象引用关系

void DebugObjectReferences(UObject* Target)
{
    UE_LOG(LogTemp, Log,
        TEXT("=== Tracing references for: %s ==="),
        *Target->GetName());

    // 1. 查看对象的Schema
    FSchemaView Schema = Target->GetClass()->GetSchema();
    UE_LOG(LogTemp, Log, TEXT("Schema has %d members"), Schema.NumMembers);

    // 2. 遍历对象的所有引用
    int32 RefCount = 0;
    Schema.ForEachReference(Target, [&RefCount](UObject* Ref)
    {
        if (Ref)
        {
            RefCount++;
            UE_LOG(LogTemp, Log, TEXT("  -> %s (%s)"),
                *Ref->GetName(),
                *Ref->GetClass()->GetName());
        }
    });

    UE_LOG(LogTemp, Log, TEXT("Total references: %d"), RefCount);

    // 3. 使用控制台命令查看引用关系
    // obj refs <ObjectName>
}

// 检查对象是否被正确追踪
void VerifyObjectTracked(UObject* Obj, UObject* Owner)
{
    // 检查Owner的Schema是否追踪了这个对象
    FSchemaView Schema = Owner->GetClass()->GetSchema();
    bool bFound = false;

    Schema.ForEachReference(Owner, [&bFound, Obj](UObject* Ref)
    {
        if (Ref == Obj)
        {
            bFound = true;
        }
    });

    if (bFound)
    {
        UE_LOG(LogTemp, Log, TEXT("Object is properly tracked"));
    }
    else
    {
        UE_LOG(LogTemp, Warning,
            TEXT("Object is NOT tracked! Did you forget UPROPERTY?"));
    }
}
```

---

## 八、总结

### 本节课要点

1. **FUObjectArray**
   - 全局对象池管理所有UObject
   - 使用索引而非指针标识对象
   - 支持索引复用和序列号防误用

2. **引用Schema**
   - 编译期预计算引用布局
   - 零运行时开销的引用遍历
   - 支持多种引用类型

3. **引用遍历**
   - TFastReferenceCollector高效遍历
   - 预取优化减少缓存失效
   - 批处理提高吞吐量

4. **FGCObject**
   - 让非UObject参与GC
   - 必须实现AddReferencedObjects
   - 自动注册/注销机制

5. **引用类型**
   - 强引用阻止GC
   - 弱引用不阻止GC
   - 软引用支持延迟加载

6. **并行遍历**
   - 工作窃取负载均衡
   - 多Worker并行处理
   - 可配置并行参数

### 下一节课预告

下一节课将深入讲解**标记-清除算法实现**，包括：
- 根对象集合的构成
- 可达性分析的实现细节
- 清除阶段的执行流程
- 弱引用的处理机制

---

## 参考资料

### 源码文件
- `Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectArray.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/GarbageCollectionSchema.h`
- `Engine/Source/Runtime/CoreUObject/Public/UObject/GCObject.h`
- `Engine/Source/Runtime/CoreUObject/Private/UObject/FastReferenceCollector.h`

### 相关课程
- Lesson 1: UE5 垃圾回收基础架构
- Lesson 3: 标记-清除算法实现
