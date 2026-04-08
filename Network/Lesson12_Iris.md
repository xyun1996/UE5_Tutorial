# 第十二课：Iris新一代网络系统（进阶）

> **学习目标**: 了解UE5+的Iris网络系统架构与使用
> **时长**: 约60分钟
> **前置知识**: 第一至第十一课内容

---

## 一、Iris概述

### 1.1 什么是Iris?

Iris是Unreal Engine 5引入的**新一代网络复制系统**，旨在提供更高效、更可扩展的网络解决方案。

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Iris vs 传统系统                                  │
└─────────────────────────────────────────────────────────────────────┘

传统系统                          Iris系统
┌────────────────────┐            ┌────────────────────┐
│  ReplicationDriver │            │  ReplicationSystem │
│  (单线程处理)       │            │  (多线程支持)       │
└────────────────────┘            └────────────────────┘
        │                                  │
        ▼                                  ▼
┌────────────────────┐            ┌────────────────────┐
│  属性逐个比较       │            │  批量属性处理       │
│  O(n)复杂度        │            │  更高效的数据结构    │
└────────────────────┘            └────────────────────┘
        │                                  │
        ▼                                  ▼
┌────────────────────┐            ┌────────────────────┐
│  ActorChannel传输   │            │  ReplicationBridge │
│  (固定开销)         │            │  (优化传输)         │
└────────────────────┘            └────────────────────┘
```

### 1.2 Iris的优势

| 特性       | 传统系统 | Iris     |
| ---------- | -------- | -------- |
| 多线程支持 | 有限     | 完全支持 |
| 扩展性     | 中等     | 高       |
| 大规模复制 | 性能瓶颈 | 优化支持 |
| 带宽效率   | 良好     | 更优     |
| 内存使用   | 较高     | 优化     |

---

## 二、Iris架构

### 2.1 核心组件

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Iris核心组件                                    │
└─────────────────────────────────────────────────────────────────────┘

                    ┌──────────────────────┐
                    │  ReplicationSystem   │
                    │   (复制系统核心)      │
                    └──────────┬───────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
        ▼                      ▼                      ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│Replication    │    │ DataStream    │    │ Filtering/    │
│Bridge         │    │ (数据流)       │    │ Prioritization│
│(桥接引擎)      │    │               │    │ (过滤/优先级)  │
└───────────────┘    └───────────────┘    └───────────────┘
        │                      │                      │
        ▼                      ▼                      ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ Replication   │    │ Serialization │    │ NetObject     │
│ State         │    │ (序列化)       │    │ Filter        │
│ (复制状态)     │    │               │    │               │
└───────────────┘    └───────────────┘    └───────────────┘
```

### 2.2 关键类

```cpp
// Iris核心类位置
// Engine/Source/Runtime/Net/Iris/Public/Iris/

// 复制系统
class UReplicationSystem;

// 复制桥接
class UReplicationBridge;

// 复制状态描述
struct FReplicationStateDescriptor;

// 复制片段
class FReplicationFragment;

// 数据流
class IDataStream;

// 网络对象过滤器
class INetObjectFilter;

// 网络对象优先级器
class INetObjectPrioritizer;
```

---

## 三、ReplicationSystem

### 3.1 创建与初始化

```cpp
// Iris复制系统配置
// DefaultEngine.ini

[/Script/Engine.NetDriver]
; 启用Iris
NetConnectionClassName=/Script/Engine.IpConnection
ReplicationDriverClassName=/Script/Iris.IrisReplicationDriver

; Iris配置
[/Script/Iris.ReplicationSystemConfig]
bEnableIrisReplication=True
```

### 3.2 复制系统流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Iris复制流程                                       │
└─────────────────────────────────────────────────────────────────────┘

服务器端:
┌─────────────────────────────────────────────────────────────────┐
│ 1. 收集需要复制的对象                                             │
│    ReplicationBridge::GatherReplicatedObjects()                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. 计算每个对象对每个连接的可见性                                  │
│    INetObjectFilter::FilterNetObject()                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. 计算复制优先级                                                │
│    INetObjectPrioritizer::PrioritizeNetObjects()                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. 序列化对象状态                                                │
│    ReplicationStateDescriptor::Serialize()                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. 通过DataStream发送                                            │
│    IDataStream::Write()                                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## 四、ReplicationBridge

### 4.1 概述

ReplicationBridge是Iris与传统引擎代码之间的**桥接层**，负责将Actor转换为Iris可处理的复制对象。

```cpp
/**
 * 复制桥接 - 连接引擎和Iris
 */
class UReplicationBridge : public UObject
{
public:
    // 收集需要复制的对象
    virtual void GatherReplicatedObjects(
        FReplicatedObjectCollector& Collector
    );

    // 处理对象删除
    virtual void ProcessObjectDeletion(
        const FNetObjectReference& Reference
    );

    // 处理远程函数调用
    virtual void ProcessRemoteFunction(
        const FNetObjectReference& Reference,
        const FNetFunctionInfo& FunctionInfo,
        FNetBitReader& Reader
    );
};
```

### 4.2 自定义桥接

```cpp
// 自定义ReplicationBridge
UCLASS()
class UMyReplicationBridge : public UReplicationBridge
{
    GENERATED_BODY()

public:
    virtual void GatherReplicatedObjects(
        FReplicatedObjectCollector& Collector) override
    {
        // 自定义收集逻辑
        Super::GatherReplicatedObjects(Collector);

        // 添加自定义对象
        for (UObject* Obj : CustomReplicatedObjects)
        {
            Collector.AddObject(Obj);
        }
    }
};
```

---

## 五、过滤与优先级

### 5.1 NetObjectFilter

```cpp
/**
 * 网络对象过滤器接口
 */
class INetObjectFilter
{
public:
    // 过滤对象是否对连接可见
    virtual void FilterNetObjects(
        FNetObjectFilteringParams& Params
    ) = 0;

    // 检查对象是否对特定连接可见
    virtual bool IsNetObjectRelevant(
        FNetObjectRelevanceParams& Params
    ) const = 0;
};
```

### 5.2 内置过滤器

```
┌─────────────────────────────────────────────────────────────────────┐
│                    内置过滤器类型                                    │
└─────────────────────────────────────────────────────────────────────┘

AlwaysRelevantNetObjectFilter
    - 对所有连接始终相关
    - 用于：游戏状态、规则

LocationBasedNetObjectFilter
    - 基于位置的相关性
    - 用于：常规Actor

NetObjectGridFilter
    - 基于网格的空间划分
    - 高效处理大量空间对象

FieldOfViewNetObjectFilter
    - 基于视野范围
    - 只复制玩家可见的对象
```

### 5.3 自定义过滤器

```cpp
// 自定义过滤器
class FMyNetObjectFilter : public INetObjectFilter
{
public:
    virtual void FilterNetObjects(
        FNetObjectFilteringParams& Params) override
    {
        for (FNetObjectHandle Handle : Params.ObjectsToFilter)
        {
            // 自定义过滤逻辑
            if (ShouldBeVisible(Handle, Params.Connection))
            {
                Params.AddRelevantObject(Handle);
            }
        }
    }

private:
    bool ShouldBeVisible(
        FNetObjectHandle Object,
        FNetObjectConnectionHandle Connection)
    {
        // 团队可见性检查等
        // ...
        return true;
    }
};
```

### 5.4 NetObjectPrioritizer

```cpp
/**
 * 网络对象优先级器
 */
class INetObjectPrioritizer
{
public:
    // 计算对象优先级
    virtual void PrioritizeNetObjects(
        FNetObjectPrioritizationParams& Params
    ) = 0;
};

// 使用示例
void FMyPrioritizer::PrioritizeNetObjects(
    FNetObjectPrioritizationParams& Params)
{
    for (FNetObjectHandle Handle : Params.ObjectsToPrioritize)
    {
        float Priority = CalculatePriority(Handle, Params.ViewerLocation);
        Params.SetPriority(Handle, Priority);
    }
}
```

---

## 六、序列化系统

### 6.1 ReplicationStateDescriptor

```cpp
/**
 * 复制状态描述符 - 描述如何序列化对象
 */
struct FReplicationStateDescriptor
{
    // 成员列表
    TArray<FReplicationStateMember> Members;

    // 序列化函数
    void Serialize(
        FNetBitWriter& Writer,
        const uint8* SourceData,
        const FReplicationFlags& Flags
    );

    // 反序列化函数
    void Deserialize(
        FNetBitReader& Reader,
        uint8* TargetData,
        const FReplicationFlags& Flags
    );
};
```

### 6.2 NetSerializer

```cpp
/**
 * 网络序列化器基类
 */
class FNetSerializer
{
public:
    // 序列化
    virtual void Serialize(
        FNetSerializationContext& Context,
        const void* Source
    );

    // 反序列化
    virtual void Deserialize(
        FNetSerializationContext& Context,
        void* Target
    );
};
```

### 6.3 内置序列化器

```
┌─────────────────────────────────────────────────────────────────────┐
│                    内置序列化器类型                                   │
└─────────────────────────────────────────────────────────────────────┘

基础类型序列化器:
    - Int8/16/32/64Serializer
    - FloatSerializer, DoubleSerializer
    - BoolSerializer

向量序列化器:
    - VectorSerializer (压缩向量)
    - QuantizedVectorSerializer (量化向量)

字符串序列化器:
    - StringSerializer
    - NameSerializer
    - TextSerializer

对象序列化器:
    - ObjectReferenceSerializer
    - SoftObjectPtrSerializer
```

### 6.4 自定义序列化器

```cpp
// 自定义序列化器
class FMyCustomSerializer : public FNetSerializer
{
public:
    virtual void Serialize(
        FNetSerializationContext& Context,
        const void* Source) override
    {
        const FMyCustomStruct* Data = static_cast<const FMyCustomStruct*>(Source);

        // 自定义序列化逻辑
        Context.WriteInt32(Data->Value);
        Context.WriteFloat(Data->Scale);
    }

    virtual void Deserialize(
        FNetSerializationContext& Context,
        void* Target) override
    {
        FMyCustomStruct* Data = static_cast<FMyCustomStruct*>(Target);

        Data->Value = Context.ReadInt32();
        Data->Scale = Context.ReadFloat();
    }
};

// 注册序列化器
REGISTER_NET_SERIALIZER(FMyCustomStruct, FMyCustomSerializer);
```

---

## 七、迁移到Iris

### 7.1 兼容性

Iris与传统系统**可以共存**，大部分代码无需修改：

```cpp
// 传统复制代码仍然有效
void AMyActor::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // 这些宏在Iris下同样工作
    DOREPLIFETIME(AMyActor, Score);
    DOREPLIFETIME_CONDITION(AMyActor, Health, COND_None);
}
```

### 7.2 配置迁移

```ini
; DefaultEngine.ini

; 启用Iris
[/Script/Engine.NetDriver]
NetConnectionClassName=/Script/Engine.IpConnection
ReplicationDriverClassName=/Script/Iris.IrisReplicationDriver
ReplicationBridgeClassName=/Script/Iris.ReplicationBridge

; 可选：配置Iris参数
[/Script/Iris.ReplicationSystemConfig]
bEnableIrisReplication=True
MaxReplicatedObjects=10000
MaxBandwidthPerConnection=65536
```

### 7.3 需要修改的部分

```cpp
// 1. 自定义ReplicationDriver需要迁移到ReplicationBridge
// 旧:
class UMyReplicationDriver : public UReplicationDriver { ... };

// 新:
class UMyReplicationBridge : public UReplicationBridge { ... };

// 2. 自定义相关性逻辑
// 旧:
bool AMyActor::IsNetRelevantFor(...);

// 新:
// 实现INetObjectFilter
class FMyFilter : public INetObjectFilter { ... };

// 3. 高级场景
// 使用ReplicationFragment进行细粒度控制
class FMyFragment : public FReplicationFragment { ... };
```

---

## 八、Iris高级特性

### 8.1 ReplicationFragment

```cpp
/**
 * 复制片段 - 模块化复制数据
 */
class FReplicationFragment
{
public:
    // 收集脏数据
    virtual void CollectReplicatedData(
        FReplicationStateCollector& Collector
    );

    // 应用接收的数据
    virtual void ApplyReplicatedData(
        const FReplicationStateApplier& Applier
    );
};

// 使用示例
class FHealthFragment : public FReplicationFragment
{
public:
    float Health;
    float MaxHealth;
    bool bIsDead;

    virtual void CollectReplicatedData(
        FReplicationStateCollector& Collector) override
    {
        Collector.AddFloat(Health);
        Collector.AddFloat(MaxHealth);
        Collector.AddBool(bIsDead);
    }

    virtual void ApplyReplicatedData(
        const FReplicationStateApplier& Applier) override
    {
        Health = Applier.ReadFloat();
        MaxHealth = Applier.ReadFloat();
        bIsDead = Applier.ReadBool();
    }
};
```

### 8.2 多线程复制

```cpp
// Iris支持多线程属性复制
// 配置文件
[/Script/Iris.ReplicationSystemConfig]
bEnableMultithreadedReplication=True
ReplicationThreadCount=4

// 代码中使用
void UMyReplicationBridge::GatherReplicatedObjects(
    FReplicatedObjectCollector& Collector)
{
    // 这里的收集可以在多个线程并行执行
    // Iris会自动处理线程安全
}
```

### 8.3 增量序列化

```cpp
// Iris自动进行增量序列化
// 只发送变化的部分
// 无需手动使用Push Model

// 但仍可手动标记
void AMyActor::UpdateScore(int32 NewScore)
{
    Score = NewScore;
    // Iris会自动检测变化，无需手动标记
}
```

---

## 九、调试Iris

### 9.1 控制台命令

```
; Iris统计
stat iris

; Iris调试
iris.Debug 1
iris.DumpObjects
iris.DumpFilters

; 性能分析
iris.Profile 1
```

### 9.2 日志

```cpp
// Iris日志类别
DECLARE_LOG_CATEGORY_EXTERN(LogIris, Log, All);
DECLARE_LOG_CATEGORY_EXTERN(LogIrisReplication, Log, All);
DECLARE_LOG_CATEGORY_EXTERN(LogIrisSerialization, Log, All);

// 启用详细日志
LogIris Verbose
LogIrisReplication Verbose
```

---

## 十、最佳实践

### 10.1 何时使用Iris

```
推荐使用Iris的场景:
    - 大规模多人游戏 (100+玩家)
    - 高密度Actor场景
    - 需要多线程复制
    - 带宽敏感场景

传统系统仍然适合:
    - 小规模游戏 (<50玩家)
    - 简单网络需求
    - 已有稳定代码
```

### 10.2 性能优化

```cpp
// 1. 使用合适的过滤器
// 空间密集场景使用GridFilter
AddFilter<UNetObjectGridFilter>();

// 2. 合理设置优先级
// 重要对象高优先级
SetPriority(ImportantActor, 10.0f);

// 3. 使用Fragment模块化
// 将数据按更新频率分组
```

---

## 十一、总结

### 11.1 核心要点

1. **Iris是UE5的新一代网络系统**
2. **支持多线程、更高扩展性**
3. **与传统系统兼容，迁移成本低**
4. **提供Filtering和Prioritization机制**
5. **模块化的ReplicationFragment**

### 11.2 课程总结

通过12节课的学习，我们系统性地掌握了UnrealEngine网络模块的核心知识：

```
第一阶段 (1-3课): 基础架构
    - 网络架构概览
    - NetDriver与Connection
    - Channel系统

第二阶段 (4-7课): 核心机制
    - Actor复制基础
    - 属性复制详解
    - 条件复制与RepNotify
    - RPC远程过程调用

第三阶段 (8-10课): 高级主题
    - NetGUID系统
    - 子对象复制
    - 网络性能优化

第四阶段 (11-12课): 实战进阶
    - 调试与最佳实践
    - Iris新系统
```

---

## 参考资料

1. **源码文件**:
   - `Net/Iris/Public/Iris/ReplicationSystem/`
   - `Net/Iris/Public/Iris/Serialization/`
   - `Net/Iris/Public/Iris/Core/`

2. **官方文档**:
   - [Iris Networking](https://docs.unrealengine.com/5.0/en-US/iris-in-unreal-engine/)

---

_课程完结_

_感谢您的学习！_
