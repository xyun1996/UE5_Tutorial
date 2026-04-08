# UnrealEngine 网络模块课程大纲

> **课程时长**: 12节课
> **难度级别**: 从入门到进阶
> **前置知识**: C++基础、UnrealEngine基础概念

---

## 课程概述

本课程系统性地讲解UnrealEngine网络模块的核心原理与实现，从基础概念到高级优化技巧，帮助学员全面理解UE网络复制系统。

---

## 第一阶段：网络基础（第1-3课）

### 第1课：网络架构概览

**学习目标**: 理解UE网络架构的整体设计

**核心内容**:
- 客户端-服务器模型基础
- 网络模块目录结构
  - `Engine/Source/Runtime/Net/` - 网络核心
  - `Engine/Source/Runtime/Sockets/` - Socket子系统
  - `Engine/Classes/Engine/NetDriver.h` - 网络驱动
- 核心类关系图
  - UNetDriver (网络驱动器)
  - UNetConnection (网络连接)
  - UChannel (通道)
  - UPackageMapClient (包映射)
- 网络Tick流程概览

**实践环节**: 阅读NetDriver.h头部注释，理解架构设计文档

**参考文件**:
- `Engine/Classes/Engine/NetDriver.h`
- `Engine/Source/Runtime/Networking/Public/Networking.h`

---

### 第2课：网络驱动与连接管理

**学习目标**: 掌握NetDriver和NetConnection的工作机制

**核心内容**:
- **UNetDriver (网络驱动器)**
  - 服务器端：管理所有客户端连接
  - 客户端：管理到服务器的连接
  - TickDispatch() / TickFlush() 流程
  - 创建和管理Channel
- **UNetConnection (网络连接)**
  - 连接状态管理
  - 数据包组装与拆解
  - 可靠性保证与重传机制
  - 连接握手流程
- **UIpNetDriver** - 基于IP的实现
- **UDemoNetDriver** - 回放录制/播放

**关键函数**:
```cpp
// NetDriver
virtual void TickDispatch(float DeltaTime);
virtual void TickFlush();

// NetConnection
void SendPackage(FOutBunch* Bunch);
void ReceivedRawPacket(void* Data, int32 Count);
```

**参考文件**:
- `Engine/Classes/Engine/NetDriver.h`
- `Engine/Classes/Engine/NetConnection.h`
- `Engine/Private/Net/NetDriver.cpp`

---

### 第3课：通道系统 (Channel)

**学习目标**: 理解Channel的设计与使用

**核心内容**:
- **UChannel (通道基类)**
  - 可靠/不可靠传输
  - 数据束(DataBunch)序列化
  - 通道生命周期
- **通道类型 (EChannelType)**
  ```
  CHTYPE_Control   = 1  // 控制通道 - 连接状态管理
  CHTYPE_Actor     = 2  // Actor通道 - Actor复制
  CHTYPE_File      = 3  // 文件通道 - 文件传输
  CHTYPE_Voice     = 4  // 语音通道 - VoIP
  ```
- **UControlChannel** - 控制通道
  - 连接建立/断开
  - 控制消息处理
- **UActorChannel** - Actor通道（预告）
  - Actor复制载体
  - 属性同步
  - RPC传输

**关键结构**:
```cpp
// DataBunch.h
struct FOutBunch : public FOutArchive
{
    int32 PacketId;
    int32 Channel;
    bool bReliable;
    // ...
};
```

**参考文件**:
- `Engine/Classes/Engine/Channel.h`
- `Engine/Classes/Engine/ControlChannel.h`
- `Engine/Public/Net/DataBunch.h`

---

## 第二阶段：Actor复制（第4-7课）

### 第4课：Actor网络复制基础

**学习目标**: 掌握Actor网络复制的基本原理

**核心内容**:
- **AActor网络属性**
  ```cpp
  ENetRole RemoteRole;        // 远程角色
  ENetRole Role;              // 本地角色
  ENetDormancy NetDormancy;   // 休眠状态
  bool bReplicates;           // 是否复制
  float NetUpdateFrequency;   // 更新频率
  float NetPriority;          // 网络优先级
  ```
- **网络角色 (ENetRole)**
  ```
  ROLE_None            // 无角色
  ROLE_SimulatedProxy  // 模拟代理
  ROLE_AutonomousProxy // 自治代理
  ROLE_Authority       // 权威
  ```
- **UActorChannel详解**
  - 创建远程Actor
  - 销毁远程Actor
  - 属性同步入口
- **复制生命周期**
  - PreReplication()
  - ReplicateSubobjects()

**关键函数**:
```cpp
virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps);
virtual bool ReplicateSubobjects(UActorChannel* Channel, ...);
virtual void PreReplication(IRepChangedPropertyTracker& ChangedPropertyTracker);
```

**参考文件**:
- `Engine/Classes/GameFramework/Actor.h`
- `Engine/Classes/Engine/ActorChannel.h`

---

### 第5课：属性复制详解

**学习目标**: 熟练使用属性复制系统

**核心内容**:
- **复制宏详解**
  ```cpp
  DOREPLIFETIME(c, v)                    // 基础复制
  DOREPLIFETIME_CONDITION(c, v, cond)    // 条件复制
  DOREPLIFETIME_CONDITION_NOTIFY(c, v, cond, rncond) // 带RepNotify
  ```
- **FObjectReplicator (对象复制器)**
  - 属性比较与脏检测
  - 属性序列化
  - 复制状态管理
- **FRepLayout (复制布局)**
  - 属性序列化布局
  - 类型处理器
  - 数组和容器支持
- **支持的属性类型**
  - 基础类型 (int, float, bool, enum)
  - 字符串 (FString, FName, FText)
  - 向量和旋转 (FVector, FRotator, FQuat)
  - 容器 (TArray, TMap, TSet)

**代码示例**:
```cpp
void AMyActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps)
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(AMyActor, Health);
    DOREPLIFETIME_CONDITION(AMyActor, bIsVisible, COND_OwnerOnly);
    DOREPLIFETIME_CONDITION_NOTIFY(AMyActor, Score, COND_None, REPNOTIFY_OnChanged);
}
```

**参考文件**:
- `Engine/Public/Net/DataReplication.h`
- `Engine/Public/Net/RepLayout.h`
- `Engine/Public/Net/UnrealNetwork.h`

---

### 第6课：条件复制与RepNotify

**学习目标**: 掌握条件复制和属性通知机制

**核心内容**:
- **复制条件 (ELifetimeCondition)**
  ```
  COND_None               // 无条件
  COND_InitialOnly        // 仅初始同步
  COND_OwnerOnly          // 仅所有者
  COND_SkipOwner          // 跳过所有者
  COND_SimulatedOnly      // 仅模拟代理
  COND_AutonomousOnly     // 仅自治代理
  COND_SimulatedOrPhysics // 模拟或物理
  COND_ReplayOrOwner      // 回放或所有者
  ```
- **RepNotify (属性通知)**
  ```cpp
  UPROPERTY(ReplicatedUsing=OnRep_Health)
  float Health;

  UFUNCTION()
  void OnRep_Health(float OldValue);
  ```
- **REPNOTIFY 标志**
  - REPNOTIFY_OnChanged - 值变化时触发
  - REPNOTIFY_Always - 每次同步都触发
- **自定义条件函数**
  ```cpp
  virtual bool IsRelevancyOwnerFor(AActor* ReplicatedActor);
  ```

**实践环节**: 实现一个条件复制的武器系统

**参考文件**:
- `Engine/Public/Net/UnrealNetwork.h`
- `Net/Core/Public/Net/Core/PropertyConditions/`

---

### 第7课：RPC远程过程调用

**学习目标**: 熟练使用RPC进行网络通信

**核心内容**:
- **RPC类型**
  ```cpp
  UFUNCTION(Server, Reliable)
  void ServerFire();

  UFUNCTION(Client, Reliable)
  void ClientShowDamage(float Damage);

  UFUNCTION(NetMulticast, Unreliable)
  void MulticastPlayEffect(FVector Location);
  ```
- **RPC执行规则**
  | 调用位置 | Server RPC | Client RPC | Multicast |
  |---------|-----------|------------|-----------|
  | 服务器   | 无效      | 发送到目标客户端 | 发送到所有客户端 |
  | 客户端   | 发送到服务器 | 无效 | 无效 |
- **可靠性与验证**
  ```cpp
  UFUNCTION(Server, Reliable, WithValidation)
  void ServerMove(FVector Location);

  bool ServerMove_Validate(FVector Location);
  void ServerMove_Implementation(FVector Location);
  ```
- **RPC底层实现**
  - RPC参数序列化
  - RPC排队与发送
  - 在ActorChannel上传输

**最佳实践**:
- 服务端RPC需验证参数合法性
- 避免频繁的可靠RPC
- 合理使用Multicast

**参考文件**:
- `Engine/Public/Net/DataReplication.h`
- `Engine/Public/Net/UnrealNetwork.h`

---

## 第三阶段：高级主题（第8-10课）

### 第8课：对象引用与NetGUID系统

**学习目标**: 理解网络对象引用机制

**核心内容**:
- **UPackageMapClient**
  - 管理NetGUID分配
  - 对象序列化/反序列化
  - 动态对象导出/导入
- **FNetGUIDCache**
  - GUID到对象的映射缓存
  - 异步加载处理
- **对象导出流程**
  ```
  服务器 → ExportObject → 客户端
  客户端 → ImportObject → 服务器
  ```
- **静态对象 vs 动态对象**
  - 静态对象：使用稳定的NetGUID
  - 动态对象：运行时分配GUID
- **NetGUID结构**
  ```cpp
  struct FNetworkGUID
  {
      uint32 Value;
      bool IsStatic() const;
      bool IsDynamic() const;
  };
  ```

**参考文件**:
- `Engine/Classes/Engine/PackageMapClient.h`
- `Engine/Classes/Engine/NetworkGuid.h`

---

### 第9课：子对象复制

**学习目标**: 掌握子对象(Subobject)的网络复制

**核心内容**:
- **为什么需要子对象复制**
  - 组件复制 (UActorComponent)
  - 嵌套对象复制
  - 减少Actor数量
- **ReplicateSubobjects()**
  ```cpp
  bool AMyActor::ReplicateSubobjects(UActorChannel* Channel,
                                      FOutBunch* Bunch,
                                      FReplicationFlags* RepFlags)
  {
      bool bWrote = Super::ReplicateSubobjects(Channel, Bunch, RepFlags);

      for (UObject* Subobject : ReplicatedSubobjects)
      {
          bWrote |= Channel->ReplicateSubobject(Subobject, Bunch, RepFlags);
      }

      return bWrote;
  }
  ```
- **FObjectReplicator管理**
  - ActorChannel维护子对象复制器映射
  - 子对象生命周期管理
- **组件网络复制**
  - UActorComponent的bReplicates
  - SetIsReplicated()
  - 组件RPC

**代码示例**: 实现一个复制子对象的武器组件

**参考文件**:
- `Engine/Classes/Engine/ActorChannel.h`
- `Engine/Public/Net/DataReplication.h`

---

### 第10课：网络性能优化

**学习目标**: 掌握网络性能优化技巧

**核心内容**:
- **Push Model系统**
  ```cpp
  UPROPERTY(ReplicatedUsing=OnRep_Count)
  int32 Count;

  void IncrementCount()
  {
      Count++;
      MARK_PROPERTY_DIRTY(AMyActor, Count);
  }
  ```
  - 避免每帧检查所有属性
  - 手动标记脏属性
  - 减少CPU开销
- **NetDormancy (网络休眠)**
  ```cpp
  // 设置休眠状态
  SetNetDormancy(DORM_DormantAll);

  // 唤醒
  SetNetDormancy(DORM_Awake);
  ```
- **Actor相关性 (Relevancy)**
  ```cpp
  virtual bool IsNetRelevantFor(AActor* RealViewer,
                                 AActor* ViewTarget,
                                 const FVector& SrcLocation);
  ```
- **带宽管理**
  - NetUpdateFrequency 控制更新频率
  - NetPriority 设置优先级
  - CullDistanceSquared 距离剔除
- **属性条件优化**
  - 合理使用COND_*条件
  - 减少不必要的同步

**参考文件**:
- `Net/Core/Public/Net/Core/PushModel/`
- `Engine/Classes/Engine/ReplicationDriver.h`

---

## 第四阶段：实战与进阶（第11-12课）

### 第11课：网络调试与最佳实践

**学习目标**: 掌握网络调试工具和开发规范

**核心内容**:
- **网络统计命令**
  ```
  stat net         // 网络统计
  stat packet      // 数据包统计
  net.profile      // 网络性能分析
  PacketRecord     // 数据包录制
  ```
- **控制台变量**
  ```
  net.PktDraw      // 绘制数据包
  net.PktOrder     // 数据包排序
  net.Replication.DebugProperty
  ```
- **常用调试方法**
  - LogNet driver日志
  - OnRep断点调试
  - RPC验证断点
- **常见问题排查**
  - 属性不同步
  - RPC丢失
  - 连接断开
- **最佳实践总结**
  - 最小化复制属性
  - 使用Push Model
  - 合理的NetDormancy
  - 参数验证

**参考文件**:
- `Engine/Classes/Engine/NetworkSettings.h`
- `Engine/Private/Net/`

---

### 第12课：Iris新一代网络系统（进阶）

**学习目标**: 了解UE5+的Iris网络系统

**核心内容**:
- **Iris系统概览**
  - 新一代网络架构
  - 与传统系统对比
  - 启用方式
- **核心组件**
  - ReplicationSystem - 复制系统
  - ReplicationBridge - 桥接引擎
  - ReplicationFragment - 模块化复制
- **过滤系统 (Filtering)**
  - NetObjectFilter
  - 网格过滤 (GridFilter)
  - 视野过滤 (FOVFilter)
- **优先级系统 (Prioritization)**
  - NetObjectPrioritizer
  - 基于位置的优先级
  - 自定义优先级计算
- **序列化系统**
  - NetSerializer架构
  - 自定义序列化器
  - BitStream优化
- **迁移指南**
  - 从传统系统迁移
  - 兼容性考虑

**参考文件**:
- `Net/Iris/Public/Iris/ReplicationSystem/ReplicationSystem.h`
- `Net/Iris/Public/Iris/ReplicationSystem/ReplicationBridge.h`
- `Net/Iris/Public/Iris/Serialization/NetSerializer.h`

---

## 附录

### 推荐学习路径

```
第1-3课 (基础) → 第4-7课 (Actor复制) → 第8-10课 (高级) → 第11-12课 (实战/进阶)
```

### 核心文件清单

| 文件 | 说明 | 重要性 |
|-----|------|-------|
| `Engine/Classes/Engine/NetDriver.h` | 网络架构文档 | ★★★★★ |
| `Engine/Classes/Engine/NetConnection.h` | 连接管理 | ★★★★★ |
| `Engine/Classes/Engine/Channel.h` | 通道基类 | ★★★★ |
| `Engine/Classes/Engine/ActorChannel.h` | Actor复制 | ★★★★★ |
| `Engine/Public/Net/DataReplication.h` | 复制核心 | ★★★★★ |
| `Engine/Public/Net/UnrealNetwork.h` | 复制宏 | ★★★★★ |
| `Engine/Classes/Engine/PackageMapClient.h` | GUID管理 | ★★★★ |
| `Engine/Classes/Engine/ReplicationDriver.h` | 复制驱动 | ★★★★ |

### 延伸阅读

1. [Unreal Engine Networking Official Docs](https://docs.unrealengine.com/5.0/en-US/gameplay/networking/)
2. `Engine/Classes/Engine/NetDriver.h` 头部注释（非常详细的架构说明）
3. Epic Games GitHub - Networking 标签的源码提交历史

---

*课程版本: 1.0*
*适用引擎版本: UnrealEngine 5.x*

---

## 课程文件列表

| 课程 | 文件名 | 状态 |
|-----|-------|------|
| 第一课：网络架构概览 | [Lesson01_NetworkArchitecture.md](./Lesson01_NetworkArchitecture.md) | ✅ 已完成 |
| 第二课：网络驱动与连接管理 | [Lesson02_NetDriverAndConnection.md](./Lesson02_NetDriverAndConnection.md) | ✅ 已完成 |
| 第三课：通道系统 | [Lesson03_ChannelSystem.md](./Lesson03_ChannelSystem.md) | ✅ 已完成 |
| 第四课：Actor网络复制基础 | [Lesson04_ActorReplicationBasic.md](./Lesson04_ActorReplicationBasic.md) | ✅ 已完成 |
| 第五课：属性复制详解 | [Lesson05_PropertyReplication.md](./Lesson05_PropertyReplication.md) | ✅ 已完成 |
| 第六课：条件复制与RepNotify | [Lesson06_ConditionalReplication.md](./Lesson06_ConditionalReplication.md) | ✅ 已完成 |
| 第七课：RPC远程过程调用 | [Lesson07_RPC.md](./Lesson07_RPC.md) | ✅ 已完成 |
| 第八课：对象引用与NetGUID系统 | [Lesson08_NetGUID.md](./Lesson08_NetGUID.md) | ✅ 已完成 |
| 第九课：子对象复制 | [Lesson09_SubobjectReplication.md](./Lesson09_SubobjectReplication.md) | ✅ 已完成 |
| 第十课：网络性能优化 | [Lesson10_NetworkOptimization.md](./Lesson10_NetworkOptimization.md) | ✅ 已完成 |
| 第十一课：网络调试与最佳实践 | [Lesson11_DebugAndBestPractices.md](./Lesson11_DebugAndBestPractices.md) | ✅ 已完成 |
| 第十二课：Iris新一代网络系统 | [Lesson12_Iris.md](./Lesson12_Iris.md) | ✅ 已完成 |
