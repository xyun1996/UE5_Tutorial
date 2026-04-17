# UE5 GC 高级问题与解决方案

本文档整理了UE5垃圾回收系统中的常见问题与解决方案。

---

## 目录

1. [GC过程中的新对象分配](#1-gc过程中的新对象分配)
2. [GC清理阶段对象复活问题](#2-gc清理阶段对象复活问题)
3. [IsReadyForFinishDestroy阻塞问题](#3-isreadyforfinishdestroy阻塞问题)
4. [对象释放超过GC时间片](#4-对象释放超过gc时间片)
5. [GC速度跟不上对象创建速度](#5-gc速度跟不上对象创建速度)
6. [TWeakPtr与TWeakObjectPtr的关系](#6-tweakptr与tweakobjectptr的关系)
7. [TStrongObjectPtr原理](#7-tstrongobjectptr原理)
8. [引用计数与GC可达性的关系](#8-引用计数与gc可达性的关系)
9. [NewObject时标记可达的源码](#9-newobject时标记可达的源码)
10. [双标志交换机制](#10-双标志交换机制)
11. [GC阶段哈希表锁定](#11-gc阶段哈希表锁定)
12. [GIsIncrementalReachabilityPending标志](#12-gisincrementalreachabilitypending标志)

---

## 1. GC过程中的新对象分配

### 问题场景

当GC正在进行时，游戏逻辑尝试分配新对象，系统如何处理？

### 核心机制：GC锁

```cpp
// 引擎源码：FGCCSyncObject

// GC进行时，对象分配会被阻塞
void* operator new(size_t Size, UObject* Outer, FName Name, ...)
{
    // 如果GC正在进行，等待GC完成
    if (GIsGarbageCollecting)
    {
        // 当前线程等待GC锁释放
        FGCObject::WaitForGCComplete();
    }

    // 分配对象...
}
```

### 处理策略

#### 策略一：阻塞等待（默认）

```
线程A: GC开始 → 获取GC锁 → 标记阶段 → 清理阶段 → 释放锁
线程B: new对象 → 检测到GC → 阻塞等待 → GC完成 → 继续分配
```

```cpp
// UObjectGlobals.cpp
UObject* StaticAllocateObject(...)
{
    // 检查GC状态
    if (IsGarbageCollecting())
    {
        // 等待GC完成
        FGCSyncLock GCLock;
        // 阻塞直到GC结束
    }

    // 继续分配
    return AllocateObjectInternal(...);
}
```

#### 策略二：增量式GC

UE5默认使用增量式GC，将GC分成小片段：

```cpp
// 控制台变量
gc.IncrementalPauseTime 5.0f  // 每帧最多5ms用于GC

// Engine.ini
[/Script/Engine.GarbageCollectionSettings]
gc.IncrementalPauseTime=5.0f
```

```
帧1: GC标记阶段 (5ms) → 游戏逻辑执行 → 对象分配正常
帧2: GC继续标记 (5ms) → 游戏逻辑执行 → 对象分配正常
帧3: GC清理阶段 (5ms) → 游戏逻辑执行 → 对象分配正常
```

#### 策略三：并行GC

UE5.3+支持并行GC，减少主线程阻塞：

```cpp
// 控制台变量
gc.AllowParallelGC 1          // 启用并行GC
gc.NumParallelGCThreads 4     // 并行线程数
```

### 完整流程图

```
┌─────────────────────────────────────────────────────────────┐
│                    游戏线程                                  │
├─────────────────────────────────────────────────────────────┤
│  Tick开始                                                    │
│     │                                                       │
│     ▼                                                       │
│  检查是否需要GC ────────────────────────────────────────┐    │
│     │                                                   │    │
│     ▼                                                   ▼    │
│  [不需要GC]                                        [开始GC]   │
│     │                                                   │    │
│     ▼                                                   ▼    │
│  游戏逻辑执行                                      获取GC锁   │
│     │                                                   │    │
│     ▼                                                   ▼    │
│  新对象分配 ←─┐                              ┌─── 标记阶段   │
│     │        │                              │       │      │
│     ▼        │                              │       ▼      │
│  检查GC状态──┘                              │  [新对象分配] │
│                                             │       │      │
│                                             │       ▼      │
│                                             │  阻塞等待... │
│                                             │       │      │
│                                             ▼       ▼      │
│                                         清理阶段 ←───┘      │
│                                             │               │
│                                             ▼               │
│                                         释放GC锁            │
│                                             │               │
│                                             ▼               │
│                                      [阻塞的对象分配继续]    │
└─────────────────────────────────────────────────────────────┘
```

### 增量式GC细节

```cpp
void FGarbageCollection::RunIncrementalGC(float MaxTimeSlice)
{
    // 步骤1：标记阶段（分多次完成）
    do
    {
        // 每次只标记一部分对象
        MarkBatchOfObjects();

        // 检查时间片
        if (GetElapsedTime() >= MaxTimeSlice)
        {
            // 暂停GC，让游戏逻辑执行
            // 此时对象分配可以正常进行
            return;
        }
    } while (!AllObjectsMarked);

    // 步骤2：清理阶段（一次性完成）
    SweepUnreachableObjects();
}
```

### 特殊情况处理

| 情况 | 处理方式 |
|------|---------|
| GC进行中，游戏线程分配新对象 | 阻塞等待GC完成 |
| 增量式GC标记间隙 | 正常分配 |
| 并行GC中 | 主线程分配会阻塞 |
| 异步加载线程 | 特殊同步机制 |
| 构造函数/初始化 | GC不会打断 |

### 控制台变量

```ini
; DefaultEngine.ini
[/Script/Engine.GarbageCollectionSettings]
gc.IncrementalPauseTime=5.0f
gc.AllowParallelGC=1
gc.NumParallelGCThreads=4
gc.DelayBetweenGCCycles=60.0f
```

```cpp
// 控制台命令
stat gc              // 显示GC统计
gc.FrameTime         // 查看GC帧时间
```

### 最佳实践

```cpp
// 不好的做法：每帧分配
void Tick(float DeltaTime)
{
    UObject* Temp = NewObject<UObject>();  // 每帧分配
}

// 好的做法：对象池
TArray<UObject*> ObjectPool;
UObject* GetPooledObject()
{
    if (ObjectPool.Num() > 0)
    {
        return ObjectPool.Pop();
    }
    return NewObject<UObject>();
}
```

---

## 2. GC清理阶段对象复活问题

### 问题场景

```
时间线：
T1: GC标记阶段 → 对象A被标记为"不可达"
T2: 游戏逻辑执行 → 对象A被重新引用
T3: GC清理阶段 → 如何处理对象A？
```

### 核心原则：清理前二次确认

```cpp
// GarbageCollection.cpp

void FGarbageCollection::SweepPhase()
{
    for (UObject* Object : AllObjects)
    {
        // 再次检查可达性！
        if (!Object->IsMarked(RF_MarkReachable))
        {
            // 最终确认：真的不可达才回收
            DestroyObject(Object);
        }
        else
        {
            // 清除标记，下一轮GC使用
            Object->ClearFlags(RF_MarkReachable);
        }
    }
}
```

### 完整流程

```
┌─────────────────────────────────────────────────────────────┐
│                    GC完整流程                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 获取GC锁                                                │
│       │                                                     │
│       ▼                                                     │
│  2. 标记阶段                                                │
│       │                                                     │
│       ├── 遍历根集合                                        │
│       ├── 标记可达对象 RF_MarkReachable                     │
│       │                                                     │
│       │   此时：对象A被标记为不可达                          │
│       │                                                     │
│       ▼                                                     │
│  3. 【关键】暂停标记，等待所有异步操作完成                    │
│       │                                                     │
│       │   如果此时有新引用建立：                             │
│       │   - 会被阻塞（GC锁）                                 │
│       │   - 或者在增量GC中，下一帧处理                       │
│       │                                                     │
│       ▼                                                     │
│  4. 清理阶段                                                │
│       │                                                     │
│       ├── 再次遍历所有对象                                   │
│       ├── 检查 RF_MarkReachable 标记                        │
│       │                                                     │
│       │   【关键检查】                                       │
│       │   if (IsMarked(RF_MarkReachable))                   │
│       │       保留对象，清除标记                             │
│       │   else                                              │
│       │       真正销毁对象                                   │
│       │                                                     │
│       ▼                                                     │
│  5. 释放GC锁                                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 写屏障机制

UE5使用写屏障来捕获对象引用的变化：

```cpp
// 当对象引用被修改时触发
void UObject::SetReference(UObject* NewRef)
{
    // 写屏障检查
    if (GIsGarbageCollecting)
    {
        if (IsIncrementalGC())
        {
            // 增量GC：标记新引用的对象为可达
            if (NewRef && !NewRef->IsMarked(RF_MarkReachable))
            {
                NewRef->SetFlags(RF_MarkReachable);
                GWriteBarrier.AddModifiedObject(NewRef);
            }
        }
    }

    Reference = NewRef;
}
```

### 增量式GC的特殊处理

```cpp
void FGarbageCollection::RunIncrementalGC()
{
    // 帧N: 标记阶段
    if (bNeedsMarkPhase)
    {
        MarkBatchOfObjects();
    }

    // ⚠️ 这里游戏逻辑执行了一帧
    // 对象可能被重新引用！

    // 帧N+1: 继续标记或进入清理
    if (bMarkPhaseComplete)
    {
        // 关键：重新标记根集合
        RebuildReachability();

        // 然后清理
        SweepPhase();
    }
}
```

### RF_PendingKill标记

```cpp
// 对象被标记为待销毁
void MarkObjectForKill(UObject* Object)
{
    Object->SetFlags(RF_PendingKill);
    // 即使后来被引用，也会在清理阶段销毁
}

// 清理阶段的检查
bool ShouldKillObject(UObject* Object)
{
    // 即使可达，如果被标记为 PendingKill，也要销毁
    if (Object->IsMarked(RF_PendingKill))
    {
        return true;
    }
    return !Object->IsMarked(RF_MarkReachable);
}
```

### 对象生命周期

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│   对象创建 ──► 活跃状态 ──► 不可达 ──► 清理检查 ──► 销毁     │
│       │            │           │           │           │      │
│       ▼            ▼           ▼           ▼           ▼      │
│   NewObject()   被引用     无引用      二次确认    析构函数   │
│                  │           │           │           │        │
│                  │           │      ┌────┴────┐      │        │
│                  │           │      │         │      │        │
│                  │           │   仍不可达  被复活   │        │
│                  │           │      │         │      │        │
│                  │           │      ▼         ▼      │        │
│                  │           │    销毁    保留对象   │        │
│                  │           │                    │          │
│                  │           └────────────────────┘          │
│                  │                                           │
│                  └──────► 继续参与下一轮GC                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 总结

| 情况 | UE5的处理 |
|------|----------|
| 标记后、清理前被引用 | 写屏障捕获，重新标记为可达 |
| 清理阶段尝试引用 | 游戏线程被阻塞，不可能发生 |
| 增量GC间隙被引用 | 下一帧重新确认可达性 |
| 析构函数中复活 | 无效，已经确定销毁 |
| RF_PendingKill | 即使可达也强制销毁 |

---

## 3. IsReadyForFinishDestroy阻塞问题

### 问题场景

销毁阶段会调用 `IsReadyForFinishDestroy()`，如果其中包含阻塞操作，会导致GC变慢。

### 销毁流程

```
┌─────────────────────────────────────────────────────────────┐
│                    对象销毁流程                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 清理阶段发现不可达对象                                   │
│       │                                                     │
│       ▼                                                     │
│  2. 调用 BeginDestroy()                                     │
│       │                                                     │
│       ├── 释放渲染资源                                      │
│       ├── 释放物理资源                                      │
│       ├── 发起异步销毁请求                                  │
│       │                                                     │
│       ▼                                                     │
│  3. 调用 IsReadyForFinishDestroy()                          │
│       │                                                     │
│       ├── 返回 true  → 继续销毁                             │
│       │                                                     │
│       └── 返回 false → 暂停销毁，等待下一帧检查              │
│                                                             │
│  4. 【只有返回true才执行】                                   │
│       │                                                     │
│       ▼                                                     │
│  5. 调用 FinishDestroy()                                    │
│       │                                                     │
│       ├── 最终清理                                          │
│       ├── 调用析构函数                                      │
│       └── 释放内存                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 潜在问题

```cpp
// ❌ 错误示例：阻塞操作
bool UMyObject::IsReadyForFinishDestroy()
{
    // 阻塞等待网络请求
    HttpClient->WaitForResponse();  // 可能等待数秒！

    // 阻塞等待文件IO
    FileManager->WaitForFileLoad();  // 可能阻塞几十毫秒

    return true;
}
```

### 正确的实现方式

#### 非阻塞检查

```cpp
// ✅ 正确：快速检查状态
bool UMyObject::IsReadyForFinishDestroy()
{
    // 只检查标志位，不做任何等待
    return bResourceReleased && bAsyncTaskComplete;
}
```

#### 异步释放资源

```cpp
void UMyObject::BeginDestroy()
{
    Super::BeginDestroy();

    // 发起异步释放，不等待
    AsyncTask(ENamedThreads::AnyThread, [this]()
    {
        ReleaseResources();
        bResourceReleased = true;
    });
}

bool UMyObject::IsReadyForFinishDestroy()
{
    // 快速返回当前状态
    return bResourceReleased;
}
```

#### 超时保护

```cpp
bool UMyObject::IsReadyForFinishDestroy()
{
    if (bReadyForDestroy)
    {
        return true;
    }

    // 检查超时
    float ElapsedTime = FPlatformTime::Seconds() - DestroyStartTime;
    if (ElapsedTime > MaxDestroyTimeout)
    {
        UE_LOG(LogGC, Warning, TEXT("Object %s destroy timeout after %.2fs"),
            *GetName(), ElapsedTime);

        // 强制标记为就绪，避免无限等待
        bReadyForDestroy = true;
        return true;
    }

    return false;
}
```

### 典型的异步资源处理

#### 纹理/渲染资源

```cpp
void UMyTexture::BeginDestroy()
{
    Super::BeginDestroy();

    // 渲染资源在渲染线程释放
    FRenderCommandFence Fence;
    BeginReleaseResource(this);
    Fence.BeginFence();
    ReleaseFence = Fence;
}

bool UMyTexture::IsReadyForFinishDestroy()
{
    // 检查渲染线程是否完成
    return ReleaseFence.IsFenceComplete();
}
```

#### 物理/碰撞资源

```cpp
void UMyPhysicsObject::BeginDestroy()
{
    Super::BeginDestroy();

    FPhysicsCommand::ExecuteWrite(PhysicsScene, [this]()
    {
        ReleasePhysicsBody();
        bPhysicsReleased = true;
    });
}

bool UMyPhysicsObject::IsReadyForFinishDestroy()
{
    return bPhysicsReleased;
}
```

### 性能监控

```cpp
// 检测耗时对象
void TrackSlowDestroy(UObject* Object, float CheckTime)
{
    if (CheckTime > 0.001f)  // 超过1ms
    {
        UE_LOG(LogGC, Warning,
            TEXT("Slow IsReadyForFinishDestroy: %s took %.2fms"),
            *Object->GetClass()->GetName(),
            CheckTime * 1000.0f);
    }
}
```

### 最佳实践总结

| Do's ✅ | Don'ts ❌ |
|---------|----------|
| 快速检查标志位 | 阻塞等待 |
| 检查原子变量 | 同步IO |
| 检查Fence状态 | 网络请求 |
| 使用超时保护 | 复杂计算 |

---

## 4. 对象释放超过GC时间片

### 核心答案

**对象会被推迟到下一轮处理，不会无限延长本轮GC时间。**

### 完整机制

```
┌─────────────────────────────────────────────────────────────────┐
│                    增量式GC处理流程                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  第N帧                                                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ GC标记阶段 (最多5ms)                                     │   │
│  │   ├── 标记可达对象                                       │   │
│  │   └── 时间到 → 暂停，下一帧继续                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  第N+2帧                                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ GC清理阶段 (最多5ms)                                     │   │
│  │                                                          │   │
│  │   for (Obj : ObjectsToDestroy)                          │   │
│  │   {                                                      │   │
│  │       if (时间到了) break;  ← 不会无限等待               │   │
│  │                                                          │   │
│  │       BeginDestroy(Obj);                                 │   │
│  │                                                          │   │
│  │       if (Obj->IsReadyForFinishDestroy())                │   │
│  │           FinishDestroy(Obj);                            │   │
│  │       else                                               │   │
│  │           PendingQueue.Add(Obj);  ← 推迟到后续处理       │   │
│  │   }                                                      │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  第N+3帧及之后                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 每帧开始时检查PendingQueue                               │   │
│  │                                                          │   │
│  │   for (Obj : PendingQueue)                              │   │
│  │   {                                                      │   │
│  │       if (Obj->IsReadyForFinishDestroy())                │   │
│  │       {                                                  │   │
│  │           FinishDestroy(Obj);                            │   │
│  │           PendingQueue.Remove(Obj);                      │   │
│  │       }                                                  │   │
│  │   }                                                      │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 引擎源码逻辑

```cpp
void FGarbageCollection::IncrementalSweepPhase(float MaxTimeSlice)
{
    double StartTime = FPlatformTime::Seconds();

    while (CurrentDestroyIndex < ObjectsToDestroy.Num())
    {
        // 检查时间片
        double ElapsedTime = FPlatformTime::Seconds() - StartTime;
        if (ElapsedTime >= MaxTimeSlice)
        {
            // 时间到！本轮GC结束
            return;
        }

        UObject* Obj = ObjectsToDestroy[CurrentDestroyIndex++];

        if (!Obj->HasBegunDestroy())
        {
            Obj->BeginDestroy();
        }

        if (Obj->IsReadyForFinishDestroy())
        {
            FinishDestroyObject(Obj);
        }
        else
        {
            // 还没准备好，加入待处理队列
            // 不会阻塞等待！
            GObjPendingKillObjects.Add(Obj);
        }
    }
}
```

### 待处理队列处理

```cpp
void FGarbageCollection::TickPendingKillObjects()
{
    for (int32 i = GObjPendingKillObjects.Num() - 1; i >= 0; i--)
    {
        UObject* Obj = GObjPendingKillObjects[i];

        if (Obj->IsReadyForFinishDestroy())
        {
            FinishDestroyObject(Obj);
            GObjPendingKillObjects.RemoveAtSwap(i);
        }
    }

    if (GObjPendingKillObjects.Num() > WarningThreshold)
    {
        UE_LOG(LogGC, Warning, TEXT("Pending kill objects accumulating: %d"),
            GObjPendingKillObjects.Num());
    }
}
```

### 实际场景示例

```cpp
// 对象A需要释放渲染资源，耗时10ms
// 但GC每帧只有5ms时间片

// 第1帧（GC清理阶段）
void UMyTexture::BeginDestroy()
{
    BeginReleaseResource(TextureResource);
    bDestroyStarted = true;
}

bool UMyTexture::IsReadyForFinishDestroy()
{
    return false;  // 渲染线程还没完成，加入PendingQueue
}

// 第2帧、第3帧
bool UMyTexture::IsReadyForFinishDestroy()
{
    return false;  // 继续等待
}

// 第4帧
bool UMyTexture::IsReadyForFinishDestroy()
{
    return true;  // 终于完成了，可以销毁
}
```

### 超时检测机制

```cpp
struct FPendingKillInfo
{
    UObject* Object;
    double AddTime;
    int32 CheckCount;
};

void CheckPendingKillTimeouts()
{
    double CurrentTime = FPlatformTime::Seconds();

    for (int32 i = PendingKillInfos.Num() - 1; i >= 0; i--)
    {
        FPendingKillInfo& Info = PendingKillInfos[i];
        double WaitTime = CurrentTime - Info.AddTime;

        // 超过30秒还没销毁
        if (WaitTime > 30.0)
        {
            UE_LOG(LogGC, Error, TEXT("Object %s stuck for %.1fs, forcing destroy"),
                *Info.Object->GetName(), WaitTime);

            FinishDestroyObject(Info.Object);
            PendingKillInfos.RemoveAtSwap(i);
        }
    }
}
```

### 总结

| 问题 | 答案 |
|------|------|
| 对象释放超过GC时间片 | 对象被放入PendingQueue，下一帧继续检查 |
| 是否延长本轮GC时间 | **否**，严格遵循时间片限制 |
| 何时完成销毁 | 对象准备好时（IsReadyForFinishDestroy返回true） |
| 可能的风险 | 大量对象堆积导致内存无法释放 |

---

## 5. GC速度跟不上对象创建速度

### 问题表现

```
内存增长趋势：

帧数    对象数量    内存占用
100     10,000     100MB
200     15,000     150MB    ← 新增5,000对象，GC只清理了部分
300     22,000     220MB    ← 继续增长
400     30,000     300MB    ← 可能触发OOM
```

### 检测方法

#### 控制台命令

```cpp
stat gc              // GC统计
stat memory          // 内存统计
stat objects         // 对象数量统计
gc.DumpObjectStats   // 详细对象统计
gc.ForceFullGC       // 强制完整GC
```

#### 代码监控

```cpp
UCLASS()
class UGCHealthMonitor : public UObject
{
    GENERATED_BODY()

public:
    void TickMonitor()
    {
        int32 ObjectCount = GUObjectArray.GetObjectCount();
        int32 PendingKillCount = GObjPendingKillObjects.Num();

        ObjectCountHistory.Add(ObjectCount);
        if (ObjectCountHistory.Num() > 60)
        {
            ObjectCountHistory.RemoveAt(0);
        }

        // 检测增长趋势
        if (ObjectCountHistory.Num() >= 10)
        {
            int32 Growth = ObjectCountHistory.Last() - ObjectCountHistory[0];

            if (Growth > 5000)
            {
                UE_LOG(LogGC, Error,
                    TEXT("Object growth rate too high: %d in 10 frames"), Growth);
            }
        }

        if (PendingKillCount > 1000)
        {
            UE_LOG(LogGC, Warning,
                TEXT("PendingKill queue accumulating: %d objects"), PendingKillCount);
        }
    }

private:
    TArray<int32> ObjectCountHistory;
};
```

### 解决方案

#### 方案一：调整GC参数

```ini
; DefaultEngine.ini
[/Script/Engine.GarbageCollectionSettings]

; 增加每帧GC时间
gc.IncrementalPauseTime=10.0f

; 减少GC周期间隔
gc.DelayBetweenGCCycles=30.0f

; 增加并行GC线程
gc.AllowParallelGC=1
gc.NumParallelGCThreads=8
```

#### 方案二：使用对象池

```cpp
template<typename T>
class TObjectPool
{
public:
    T* Acquire(UObject* Outer)
    {
        if (Pool.Num() > 0)
        {
            T* Object = Pool.Pop();
            Object->Reset();
            return Object;
        }
        return NewObject<T>(Outer);
    }

    void Release(T* Object)
    {
        Object->Reset();
        Pool.Add(Object);
    }

    void PreAllocate(int32 Count, UObject* Outer)
    {
        for (int32 i = 0; i < Count; i++)
        {
            Pool.Add(NewObject<T>(Outer));
        }
    }

private:
    TArray<T*> Pool;
};
```

#### 方案三：减少对象创建频率

```cpp
// 不好的做法：每帧创建临时对象
void AMyActor::Tick(float DeltaTime)
{
    UTemporaryObject* Temp = NewObject<UTemporaryObject>();  // ❌
}

// 好的做法：复用对象
void AMyActor::BeginPlay()
{
    CachedTempObject = NewObject<UTemporaryObject>(this);
}

void AMyActor::Tick(float DeltaTime)
{
    CachedTempObject->Reset();
    CachedTempObject->DoSomething();  // ✅ 复用
}
```

#### 方案四：检查销毁阻塞

```cpp
void DumpStuckObjects()
{
    TMap<UClass*, int32> StuckClassCounts;

    for (UObject* Obj : GObjPendingKillObjects)
    {
        UClass* Class = Obj->GetClass();
        StuckClassCounts.FindOrAdd(Class)++;
    }

    StuckClassCounts.ValueSort([](int32 A, int32 B) { return A > B; });

    for (auto& Pair : StuckClassCounts)
    {
        UE_LOG(LogGC, Warning, TEXT("  %s: %d objects stuck"),
            *Pair.Key->GetName(), Pair.Value);
    }
}
```

#### 方案五：分时创建策略

```cpp
UCLASS()
class AObjectCreationLimiter : public AActor
{
    GENERATED_BODY()

    int32 ObjectsCreatedThisFrame = 0;
    int32 MaxObjectsPerFrame = 50;

public:
    bool CanCreateObject()
    {
        return ObjectsCreatedThisFrame < MaxObjectsPerFrame;
    }

    void Tick(float DeltaTime) override
    {
        ObjectsCreatedThisFrame = 0;
        ProcessPendingCreations();
    }

private:
    TArray<TFunction<void()>> PendingCreations;

    void ProcessPendingCreations()
    {
        while (PendingCreations.Num() > 0 && CanCreateObject())
        {
            PendingCreations.Pop()();
            ObjectsCreatedThisFrame++;
        }
    }
};
```

### 完整的监控响应系统

```cpp
UCLASS()
class UGCPerformanceManager : public UObject
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly)
    int32 WarningObjectCount = 50000;

    UPROPERTY(EditDefaultsOnly)
    int32 CriticalObjectCount = 100000;

    void Update()
    {
        int32 CurrentCount = GUObjectArray.GetObjectCount();
        int32 LastCount = ObjectCountHistory.Num() > 0
            ? ObjectCountHistory.Last() : CurrentCount;

        int32 Growth = CurrentCount - LastCount;
        ObjectCountHistory.Add(CurrentCount);

        if (Growth > MaxGrowthPerFrame)
        {
            HandleHighGrowth(Growth);
        }

        if (CurrentCount > CriticalObjectCount)
        {
            HandleCriticalState(CurrentCount);
        }
        else if (CurrentCount > WarningObjectCount)
        {
            HandleWarningState(CurrentCount);
        }
    }

private:
    void HandleWarningState(int32 Count)
    {
        UE_LOG(LogGC, Warning, TEXT("Object count warning: %d"), Count);
        GEngine->ForceGarbageCollection(false);
    }

    void HandleCriticalState(int32 Count)
    {
        UE_LOG(LogGC, Error, TEXT("Object count critical: %d"), Count);
        GEngine->ForceGarbageCollection(true);
        OnCreationPaused.Broadcast();
    }

    TArray<int32> ObjectCountHistory;

    UPROPERTY(BlueprintAssignable)
    FOnCreationPaused OnCreationPaused;
};
```

### 控制台命令扩展

```cpp
UFUNCTION(Exec)
void DumpGCHealth()
{
    int32 TotalObjects = GUObjectArray.GetObjectCount();
    int32 PendingKill = GObjPendingKillObjects.Num();

    UE_LOG(LogGC, Log, TEXT("=== GC Health Report ==="));
    UE_LOG(LogGC, Log, TEXT("Total Objects: %d"), TotalObjects);
    UE_LOG(LogGC, Log, TEXT("Pending Kill: %d"), PendingKill);
}

UFUNCTION(Exec)
void SetGCTimeSlice(float TimeMs)
{
    GIncrementalGCTimeSlice = TimeMs;
}

UFUNCTION(Exec)
void ForceFullGCNow()
{
    GEngine->ForceGarbageCollection(true);
}
```

### 策略对比

| 策略 | 适用场景 | 效果 |
|------|---------|------|
| 调整GC参数 | 临时缓解 | 中 |
| 对象池 | 高频创建/销毁对象 | 高 |
| 减少创建频率 | 每帧创建临时对象 | 高 |
| 批量管理 | 大量同类对象 | 高 |
| 检查销毁阻塞 | PendingKill堆积 | 中 |
| 分层策略 | 复杂场景 | 高 |
| 分时创建 | 峰值创建压力 | 中 |

### 核心原则

1. **预防优于治疗**：使用对象池避免频繁创建/销毁
2. **监控增长趋势**：实时检测对象增长速度
3. **分级响应**：警告→调整参数，严重→强制GC
4. **找出瓶颈**：定位阻塞销毁的对象类型

---

---

## 6. TWeakPtr与TWeakObjectPtr的关系

### 核心区别

| 特性 | TWeakPtr | TWeakObjectPtr |
|------|----------|----------------|
| **适用对象** | 任意C++对象（非UObject） | UObject及其子类 |
| **底层机制** | 引用计数（控制块） | 对象池索引 + 序列号 |
| **源码位置** | `SharedPointer.h` | `WeakObjectPtrTemplates.h` |
| **依赖** | 需要TSharedRef/TSharedPtr配合 | 直接使用，无需配合 |

### TWeakPtr - 引用计数方式

```cpp
// SharedPointer.h
template<class T>
class TWeakPtr
{
private:
    // 持有控制块的弱引用
    FWeakReferencer<T, ESPMode::ThreadSafe> WeakReferenceCount;

    // 原始指针（缓存）
    T* Object;
};
```

**工作原理**：
1. 与TSharedPtr共享控制块
2. 控制块中维护弱引用计数和强引用计数
3. 当强引用计数归零，对象被销毁
4. 当弱引用计数归零，控制块被释放

```cpp
// 使用示例
TSharedPtr<FMyStruct> SharedPtr = MakeShared<FMyStruct>();
TWeakPtr<FMyStruct> WeakPtr = SharedPtr;

if (TSharedPtr<FMyStruct> Locked = WeakPtr.Pin())
{
    // 对象仍然存活
}
```

### TWeakObjectPtr - 对象池索引方式

```cpp
// WeakObjectPtrTemplates.h
template<class T, class TWeakObjectPtrBase>
class TWeakObjectPtr : private TWeakObjectPtrBase
{
    // 继承自 TWeakObjectPtrBase（默认是 FWeakObjectPtr）
};

// WeakObjectPtrTemplatesFwd.h - 关键发现
template<class T = UObject, class TWeakObjectPtrBase = FWeakObjectPtr>
class TWeakObjectPtr;
```

**FWeakObjectPtr结构**：
```cpp
// WeakObjectPtr.h
struct FWeakObjectPtr
{
private:
    int32 ObjectIndex;        // GUObjectArray中的索引
    int32 ObjectSerialNumber; // 序列号（防止重用）
};
```

**工作原理**：
1. 记录对象在全局对象池中的索引
2. 使用序列号检测对象是否已被销毁并替换
3. 不增加引用计数，不影响对象生命周期

```cpp
// 使用示例
UObject* Obj = NewObject<UObject>();
TWeakObjectPtr<UObject> WeakObj(Obj);

if (UObject* Resolved = WeakObj.Get())
{
    // 对象仍然存活
}
```

### 为什么UE需要两种弱引用？

```
┌─────────────────────────────────────────────────────────────────┐
│                      UE对象体系                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   UObject体系                  │    普通C++对象                   │
│   ─────────────               │    ─────────────                │
│                               │                                 │
│   - 受GC管理                   │    - 不受GC管理                  │
│   - 全局对象池 GUObjectArray   │    - 无全局注册                  │
│   - 有ObjectIndex              │    - 无索引                      │
│                               │                                 │
│   ┌─────────────────┐         │    ┌─────────────────┐         │
│   │ TWeakObjectPtr  │         │    │    TWeakPtr     │         │
│   │                 │         │    │                 │         │
│   │ ObjectIndex     │         │    │ ControlBlock    │         │
│   │ SerialNumber    │         │    │ WeakRefCount    │         │
│   └─────────────────┘         │    └─────────────────┘         │
│                               │                                 │
│   轻量级，无额外内存          │    需要控制块内存               │
│   GC集成                      │    与TSharedPtr配合             │
│                               │                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 选择指南

| 场景 | 推荐类型 |
|------|---------|
| 引用UObject | TWeakObjectPtr |
| 引用普通C++类 | TWeakPtr |
| 需要与TSharedPtr交互 | TWeakPtr |
| 需要最小内存开销 | TWeakObjectPtr |
| 需要GC安全检查 | TWeakObjectPtr |

---

## 7. TStrongObjectPtr原理

### 定义

```cpp
// StrongObjectPtr.h
template <class T>
class TStrongObjectPtr
{
private:
    T* Object;  // 原始指针
};
```

### 核心机制：引用计数 + GC Root

```cpp
// UObjectArray.h - FUObjectItem
void AddRef()
{
    const int64 NewRefCount = FPlatformAtomics::InterlockedIncrement(&FlagsAndRefCount);

    // 当引用计数从0变为1时
    if ((NewRefCount & EInternalObjectFlags_RefCountMask) == 1)
    {
        // 如果增量可达性分析正在进行，立即标记为可达
        if (UE::GC::GIsIncrementalReachabilityPending)
        {
            GetObject()->MarkAsReachable();
        }

        // 关键：标记为GC Root！
        MarkRootAsDirty();
    }
}

void ReleaseRef()
{
    const int64 NewRefCount = FPlatformAtomics::InterlockedDecrement(&FlagsAndRefCount);

    // 当引用计数从1变为0时
    if ((NewRefCount & EInternalObjectFlags_RefCountMask) == 0)
    {
        // 从GC Root移除
        MarkRootAsDirty();
    }
}
```

### 工作原理图

```
┌─────────────────────────────────────────────────────────────────┐
│                 TStrongObjectPtr 防止GC回收原理                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   TStrongObjectPtr<A> StrongPtr;                               │
│                                                                 │
│   构造时：                                                       │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  Obj->AddRef()                                          │  │
│   │      │                                                  │  │
│   │      ▼                                                  │  │
│   │  RefCount: 0 → 1                                       │  │
│   │      │                                                  │  │
│   │      ▼                                                  │  │
│   │  MarkRootAsDirty() ──► 添加到 GC Root Set              │  │
│   │      │                                                  │  │
│   │      ▼                                                  │  │
│   │  对象被视为"可达"，不会被GC回收                          │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   析构时：                                                       │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  Obj->ReleaseRef()                                      │  │
│   │      │                                                  │  │
│   │      ▼                                                  │  │
│   │  RefCount: 1 → 0                                       │  │
│   │      │                                                  │  │
│   │      ▼                                                  │  │
│   │  MarkRootAsDirty() ──► 从 GC Root Set 移除             │  │
│   │      │                                                  │  │
│   │      ▼                                                  │  │
│   │  对象可被正常GC回收                                      │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 与TSharedPtr的区别

| 特性 | TStrongObjectPtr | TSharedPtr |
|------|------------------|------------|
| **适用对象** | UObject | 任意C++对象 |
| **控制生命周期** | 影响GC可达性 | 完全控制生命周期 |
| **引用计数位置** | FUObjectItem | 控制块 |
| **与GC关系** | 作为Root阻止回收 | 无关 |

### 使用场景

```cpp
// 场景1：跨线程持有UObject
void AsyncWorker()
{
    TStrongObjectPtr<UAssetData> AssetPtr(LoadedAsset);

    // 异步操作期间，确保资产不被GC
    DoAsyncWork();

    // 离开作用域，自动释放
}

// 场景2：防止临时GC回收
void PerformOperation()
{
    // 某些情况下对象可能暂时不可达
    TStrongObjectPtr<UObject> GuardPtr(ImportantObject);

    // 确保操作期间对象存活
    DoComplexOperation();
}
```

---

## 8. 引用计数与GC可达性的关系

### 问题本质

GC使用可达性分析判断对象是否存活，而TStrongObjectPtr使用引用计数。两者如何统一？

### 答案：引用计数 > 0 等同于加入 Root Set

```cpp
// GarbageCollection.cpp - 关键实现
void FUObjectItem::AddRef()
{
    // 原子增加引用计数
    const int64 NewRefCount = FPlatformAtomics::InterlockedIncrement(&FlagsAndRefCount);

    // 当引用计数从0变为1时
    if ((NewRefCount & EInternalObjectFlags_RefCountMask) == 1)
    {
        // 标记为根对象！
        MarkRootAsDirty();
    }
}
```

### 完整机制图

```
┌─────────────────────────────────────────────────────────────────┐
│                引用计数 与 GC 可达性 的统一                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                    GC Root Set                          │  │
│   │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │  │
│   │  │ Root 1  │ │ Root 2  │ │ Root 3  │ │ ...     │       │  │
│   │  └─────────┘ └─────────┘ └─────────┘ └─────────┘       │  │
│   │       ▲                                                 │  │
│   │       │                                                 │  │
│   │       │ AddRef() 时加入                                 │  │
│   │       │                                                 │  │
│   └───────┼─────────────────────────────────────────────────┘  │
│           │                                                    │
│           │                                                    │
│   ┌───────┴─────────────────────────────────────────────────┐  │
│   │              TStrongObjectPtr 持有对象                    │  │
│   │                                                          │  │
│   │   RefCount = 1  ──────► 对象在 Root Set 中              │  │
│   │   RefCount = 0  ──────► 对象不在 Root Set 中            │  │
│   │                                                          │  │
│   └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│   GC标记阶段：                                                   │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  for (Root : RootSet)                                   │  │
│   │  {                                                      │  │
│   │      MarkReachable(Root);  // 从根开始遍历               │  │
│   │  }                                                      │  │
│   │                                                          │  │
│   │  RefCount > 0 的对象被标记为可达                          │  │
│   │  因为它们在 Root Set 中                                  │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 关键源码位置

| 文件 | 行号 | 说明 |
|------|------|------|
| UObjectArray.h | 355-408 | FUObjectItem::AddRef/ReleaseRef |
| GarbageCollection.cpp | 616-668 | AddRef/ReleaseRef具体实现 |
| GarbageCollectionInternalFlags.h | - | Root Set 管理标志 |

### 总结

**引用计数本质上是通过操作GC Root Set来实现"防止回收"的效果**：
- AddRef() → 加入Root Set → 被视为可达 → 不会被回收
- ReleaseRef() → 离开Root Set → 可能变为不可达 → 可被回收

---

## 9. NewObject时标记可达的源码

### 调用链

```
NewObject<UClass>()
    │
    ▼
StaticConstructObject_Internal()
    │
    ▼
StaticAllocateObject()
    │
    ▼
UObjectBase::UObjectBase()  // 构造函数
    │
    ▼
UObjectBase::AddObject()
    │
    ▼
GUObjectArray.AllocateUObjectIndex()  // 分配对象索引
    │
    ▼
标记为可达
```

### 关键源码

```cpp
// UObjectArray.cpp:297-304
int32 AllocateUObjectIndex(UObjectBase* Object, ...)
{
    // ... 分配索引 ...

    // 标记为可达！
    if (!IsIndexDisregardForGC(Index))
    {
        ObjectItem->FlagsAndRefCount |=
            ((int64)UE::GC::Private::FGCFlags::GetReachableFlagValue_ForGC()) << 32;
    }

    return Index;
}
```

### 完整流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                    NewObject 时标记可达流程                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   NewObject<UAsset>(Outer, Name)                               │
│       │                                                        │
│       ▼                                                        │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  StaticConstructObject_Internal                         │  │
│   │      │                                                  │  │
│   │      ▼                                                  │  │
│   │  StaticAllocateObject                                   │  │
│   │      │                                                  │  │
│   │      ├── 从对象池分配槽位                                │  │
│   │      │                                                  │  │
│   │      ▼                                                  │  │
│   │  UObjectBase 构造函数                                    │  │
│   │      │                                                  │  │
│   │      ▼                                                  │  │
│   │  AddObject()                                            │  │
│   │      │                                                  │  │
│   │      ▼                                                  │  │
│   │  AllocateUObjectIndex()                                 │  │
│   │      │                                                  │  │
│   │      ▼                                                  │  │
│   │  【关键步骤】                                            │  │
│   │  ObjectItem->FlagsAndRefCount |= ReachableFlag << 32   │  │
│   │      │                                                  │  │
│   │      ▼                                                  │  │
│   │  新对象被标记为可达 ✓                                    │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 为什么新对象需要标记为可达？

1. **防止立即被回收**：新创建的对象可能还没有被其他对象引用
2. **给对象初始化时间**：构造函数执行期间，对象需要存活
3. **依赖后续引用**：初始化后，如果被其他可达对象引用，继续保持可达

### 特殊情况

```cpp
// 某些对象不需要标记为可达
if (IsIndexDisregardForGC(Index))
{
    // 这些对象：
    // - 永久存活（如某些引擎核心对象）
    // - 有特殊管理方式
}
```

---

## 10. 双标志交换机制

### 问题背景

传统的GC标记-清除需要遍历所有对象两次：
1. 第一次：清除所有对象的可达标记
2. 第二次：从根集合标记可达对象

这导致 **O(n)** 的遍历开销，n = 对象总数。

### 解决方案：双标志交换

```cpp
// GarbageCollectionInternalFlags.h
class FGCFlags
{
    // 两个标志，含义动态交换
    static EInternalObjectFlags ReachableObjectFlag;
    static EInternalObjectFlags MaybeUnreachableObjectFlag;

    // 交换标志含义
    static void SwapReachableAndMaybeUnreachable()
    {
        GUObjectArray.LockInternalArray();
        Swap(ReachableObjectFlag, MaybeUnreachableObjectFlag);
        GUObjectArray.UnlockInternalArray();
    }
};
```

### 原理图

```
┌─────────────────────────────────────────────────────────────────┐
│                    双标志交换机制原理                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   初始状态：                                                     │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  ReachableObjectFlag = RF_Reachable_A                  │  │
│   │  MaybeUnreachableObjectFlag = RF_Unreachable_B         │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   GC开始：交换标志含义                                           │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  Swap() 之后：                                           │  │
│   │                                                         │  │
│   │  ReachableObjectFlag = RF_Unreachable_B  (原来的不可达)  │  │
│   │  MaybeUnreachableObjectFlag = RF_Reachable_A (原来的可达)│  │
│   │                                                         │  │
│   │  效果：                                                  │  │
│   │  - 上一轮标记为可达的对象，现在变成了"可能不可达"        │  │
│   │  - 无需遍历清除！标志含义变了                            │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   标记阶段：                                                     │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  for (Obj : ReachableObjects)                          │  │
│   │  {                                                      │  │
│   │      Obj->Flags |= ReachableObjectFlag;  // 设置可达    │  │
│   │  }                                                      │  │
│   │                                                         │  │
│   │  现在标记的是 RF_Unreachable_B（新的可达标志）          │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   清理阶段：                                                     │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  for (Obj : AllObjects)                                │  │
│   │  {                                                      │  │
│   │      if (Obj 没有被标记为 ReachableObjectFlag)          │  │
│   │      {                                                  │  │
│   │          // 真正不可达，回收                            │  │
│   │      }                                                  │  │
│   │  }                                                      │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 性能对比

| 方法 | 时间复杂度 | 说明 |
|------|-----------|------|
| 传统清除+标记 | O(n) + O(m) | n=总对象数，m=可达数 |
| 双标志交换 | O(1) + O(m) | 只需交换操作 |

### 关键代码

```cpp
// GarbageCollection.cpp
void FRealtimeGC::PerformReachabilityAnalysis()
{
    // 步骤1：交换标志含义（O(1)操作）
    FGCFlags::SwapReachableAndMaybeUnreachable();

    // 步骤2：从根集合标记可达（只遍历可达对象）
    for (UObject* Root : RootSet)
    {
        MarkReachable(Root);
    }

    // 步骤3：清理不可达对象
    SweepUnreachableObjects();
}
```

### 总结

| 方面 | 说明 |
|------|------|
| **目的** | 避免O(n)遍历清除所有对象的标记 |
| **方法** | 交换两个标志位的含义 |
| **效果** | 上一轮的"可达"自动变成这一轮的"可能不可达" |
| **优势** | 只需O(1)的交换操作 + O(m)的标记操作 |

---

## 11. GC阶段哈希表锁定

### UObject哈希表的作用

```cpp
// UObjectHash.cpp
struct FUObjectHashTables
{
    // 1. 对象哈希表：快速按名称查找对象
    TMultiMap<FName, UObjectBase*> Hash;

    // 2. Outer哈希表：快速查找某Outer下的所有对象
    TMultiMap<UObjectBase*, UObjectBase*> HashOuter;

    // 3. 类到对象列表：快速查找某类的所有实例
    TMap<UClass*, TSet<UObjectBase*>> ClassToObjectListMap;

    // 4. 异步加载注册表
    TSet<FName> AsyncRegistrationSet;

    // 5. 附加的查找表
    TMap<UObjectBase*, FUObjectBaseSet*> ObjectToOuterMap;
};
```

### 哈希表的主要用途

```
┌─────────────────────────────────────────────────────────────────┐
│                    UObject哈希表用途                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   1. 按名称查找对象                                             │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  FindObject<UTexture>(ANY_PACKAGE, "MyTexture")        │  │
│   │      │                                                  │  │
│   │      ▼                                                  │  │
│   │  Hash.Find("MyTexture") → UObject*                     │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   2. 查找某Outer下的所有子对象                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  GetObjectsWithOuter(MyActor)                          │  │
│   │      │                                                  │  │
│   │      ▼                                                  │  │
│   │  HashOuter.Find(MyActor) → TArray<UObject*>            │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   3. 查找某类的所有实例                                         │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  GetObjectsOfClass(UActorComponent::StaticClass())     │  │
│   │      │                                                  │  │
│   │      ▼                                                  │  │
│   │  ClassToObjectListMap.Find(Class) → TSet<UObject*>     │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### GC期间的哈希表锁定

```cpp
// GarbageCollection.cpp:5638-5746
void FRealtimeGC::PerformReachabilityAnalysis()
{
    // 只在标记阶段锁定哈希表
    FUObjectHashTables& HashTables = FUObjectHashTables::Get();
    FRWScopeLock Lock(&HashTables->Critical, SLT_Write);

    // 标记可达对象...
    // 此时不能修改哈希表

    // 标记完成后释放锁
}
```

### 锁定时间优化

```
┌─────────────────────────────────────────────────────────────────┐
│                    GC锁定时间分析                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   传统GC（非增量）：                                            │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  [================ 锁定哈希表 ===================]      │  │
│   │   标记阶段（可能几十毫秒到几百毫秒）                     │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   增量GC：                                                      │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  [锁] 标记一批 [解锁] ... [锁] 标记一批 [解锁] ...      │  │
│   │   ↑              ↑                                       │  │
│   │   短时间锁定    游戏逻辑可执行                           │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 为什么需要锁定

1. **保持一致性**：GC遍历对象时，哈希表不能被修改
2. **防止并发问题**：其他线程可能同时创建/销毁对象
3. **确保引用完整**：标记过程依赖哈希表查找引用关系

### 锁定时间是否太长？

**取决于GC模式**：

| GC模式 | 锁定时间 | 影响 |
|--------|---------|------|
| 非增量GC | 整个标记阶段 | 可能影响帧率 |
| 增量GC | 每帧一小段 | 影响较小 |
| 并行GC | 部分时间可并行 | 影响更小 |

### 优化建议

```cpp
// 如果发现哈希表锁定影响性能
// 1. 使用增量GC
gc.IncrementalPauseTime=2.0f

// 2. 减少对象数量
// - 使用对象池
// - 及时销毁不需要的对象

// 3. 监控锁定时间
// stat gc 查看GC时间分布
```

---

## 12. GIsIncrementalReachabilityPending标志

### 定义

```cpp
// GarbageCollectionGlobals.h:26
/** true if incremental reachability analysis is in progress */
extern COREUOBJECT_API TSAN_ATOMIC(bool) GIsIncrementalReachabilityPending;
```

**核心含义**：表示**增量可达性分析正在进行中（被挂起/未完成）**。

### 设置时机

```cpp
// GarbageCollection.cpp:5907
GIsIncrementalReachabilityPending = GReachabilityState.IsSuspended();
```

- **true**：可达性分析因时间片用尽被挂起，还没完成
- **false**：可达性分析已完成

### 关键用途：保护新对象/新引用

当增量可达性分析被挂起时，GC还没遍历完所有对象。此时新创建的对象或新建立的引用需要被立即标记为可达。

### 具体应用场景

#### 1. FUObjectItem::AddRef() - 强引用增加时

```cpp
// UObjectArray.h:364
void AddRef()
{
    const int64 NewRefCount = FPlatformAtomics::InterlockedIncrement(&FlagsAndRefCount);

    if ((NewRefCount & EInternalObjectFlags_RefCountMask) == 1)
    {
        // 如果增量可达性分析正在进行，立即标记为可达
        if (UE::GC::GIsIncrementalReachabilityPending)
        {
            GetObject()->MarkAsReachable();
        }
        MarkRootAsDirty();
    }
}
```

#### 2. FObjectPtr赋值时 - 写屏障

```cpp
// ObjectPtr.h:310-318
void ConditionallyMarkAsReachable(const FObjectPtr& InPtr) const
{
    if (UE::GC::GIsIncrementalReachabilityPending && InPtr.IsResolved())
    {
        if (UObject* Obj = ReadObjectHandlePointerNoCheck(InPtr.GetHandleRef()))
        {
            UE::GC::MarkAsReachable(Obj);
        }
    }
}
```

#### 3. 新对象分配时

```cpp
// UObjectGlobals.cpp:1379
if (UE::GC::GIsIncrementalReachabilityPending)
{
    // 新分配的对象需要标记为可达
}
```

### 工作流程图

```
┌─────────────────────────────────────────────────────────────────┐
│               GIsIncrementalReachabilityPending 流程            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  GC Tick 1         GC Tick 2         GC Tick 3                  │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐               │
│  │ 遍历对象  │      │ 继续遍历  │      │ 继续遍历  │               │
│  │ 标记可达  │      │ 标记可达  │      │ 标记可达  │               │
│  └────┬─────┘      └────┬─────┘      └────┬─────┘               │
│       │                 │                 │                      │
│    时间片用尽        时间片用尽         遍历完成                   │
│       │                 │                 │                      │
│       ▼                 ▼                 ▼                      │
│  GIsIncremental     GIsIncremental    GIsIncremental             │
│  ReachabilityPending ReachabilityPending ReachabilityPending    │
│       = true             = true            = false               │
│                                                                 │
│  此时新对象/引用        此时新对象/引用     可以安全进行清理        │
│  需要立即标记可达       需要立即标记可达                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 总结

| 方面 | 说明 |
|------|------|
| **标志含义** | 增量可达性分析正在进行（被挂起/未完成） |
| **设置时机** | 每次增量GC迭代结束时，根据是否完成来设置 |
| **核心目的** | 保护在可达性分析期间产生的新对象/新引用 |
| **工作机制** | 当标志为true时，任何新对象创建或引用建立都会立即标记为可达 |

这是UE5增量GC的关键安全机制，确保增量模式下不会误回收正在使用的对象。

---

## 总结

| 问题 | 核心要点 | 解决方案 |
|------|---------|---------|
| GC中新对象分配 | GC锁阻塞 | 增量式GC / 对象池 |
| 清理阶段对象复活 | 二次确认 + 写屏障 | 自动处理，无需干预 |
| IsReadyForFinishDestroy阻塞 | 只检查不等待 | 异步释放 + 标志位 |
| 对象释放超时 | 推迟到下一帧 | 超时检测机制 |
| GC速度跟不上创建 | 内存持续增长 | 对象池 + 调整参数 + 监控 |
| TWeakPtr vs TWeakObjectPtr | 不同机制，不同适用对象 | UObject用TWeakObjectPtr |
| TStrongObjectPtr | 引用计数+加入Root Set | 防止UObject被GC回收 |
| 引用计数与GC关系 | RefCount>0 → Root Set | 两者通过Root Set统一 |
| NewObject标记可达 | AllocateUObjectIndex中设置 | 防止新对象立即被回收 |
| 双标志交换 | 避免O(n)遍历清除 | 交换标志含义 |
| 哈希表锁定 | 标记阶段需要一致性 | 增量GC减少锁定时间 |
| GIsIncrementalReachabilityPending | 增量GC安全机制 | 保护新对象/新引用 |
