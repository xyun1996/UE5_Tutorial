# 第三课：通道系统 (Channel)

> **学习目标**: 深入理解UChannel的设计与实现，掌握数据束(Bunch)机制
> **时长**: 约60分钟
> **前置知识**: 第一课、第二课内容

---

## 一、通道系统概述

### 1.1 什么是通道 (Channel)?

通道是UE网络架构中**数据传输的基本单位**。每个通道负责特定类型的数据传输：

- **控制通道**: 管理连接状态
- **Actor通道**: 复制Actor及其子对象
- **语音通道**: 传输VoIP数据
- **文件通道**: 传输文件数据

### 1.2 通道与连接的关系

```
┌─────────────────────────────────────────────────────────────────────┐
│                        UNetConnection                               │
│                                                                     │
│  Channels[] 数组                                                    │
│  ┌───────┬───────┬───────┬───────┬───────┬───────┬───────┐          │
│  │  [0]  │  [1]  │  [2]  │  [3]  │  [4]  │  ...  │  [N]  │          │
│  │Control│ Voice │Actor 1│Actor 2│Actor 3│       │Actor N│          │
│  └───────┴───────┴───────┴───────┴───────┴───────┴───────┘          │
│                                                                     │
│  ChIndex = 通道在数组中的索引                                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**关键点**:

- 每个连接有独立的通道数组
- 通道索引(ChIndex)是通道在数组中的位置
- ChIndex=0 永远是控制通道
- ChIndex=1 通常是语音通道

---

## 二、通道类型

### 2.1 EChannelType 枚举

**位置**: `Engine/Classes/Engine/Channel.h:24`

```cpp
enum EChannelType
{
    CHTYPE_None     = 0,  // 无效类型
    CHTYPE_Control  = 1,  // 控制通道 - 连接状态管理
    CHTYPE_Actor    = 2,  // Actor通道 - Actor复制
    CHTYPE_File     = 3,  // 文件通道 - 文件传输
    CHTYPE_Voice    = 4,  // 语音通道 - VoIP
    CHTYPE_MAX      = 8,  // 最大值
};
```

### 2.2 各类型通道详解

| 类型      | 数量     | 主要职责     | 生命周期          |
| --------- | -------- | ------------ | ----------------- |
| `Control` | 1个/连接 | 连接状态管理 | 与连接同生命周期  |
| `Actor`   | N个/连接 | Actor复制    | 与Actor相关性绑定 |
| `Voice`   | 1个/连接 | VoIP数据     | 连接期间持续存在  |
| `File`    | 按需创建 | 文件传输     | 传输完成后关闭    |

---

## 三、UChannel 基类

### 3.1 类定义

**位置**: `Engine/Classes/Engine/Channel.h:62`

```cpp
UCLASS(abstract, transient, MinimalAPI)
class UChannel : public UObject
{
    GENERATED_BODY()
};
```

### 3.2 核心成员变量

```cpp
class UChannel : public UObject
{
public:
    // ========== 基础信息 ==========

    // 所属连接
    UPROPERTY()
    TObjectPtr<class UNetConnection> Connection;

    // 通道索引
    int32 ChIndex;

    // 通道类型名称
    FName ChName;


    // ========== 状态标志 ==========

    uint32 OpenAcked:1;        // Open包已确认
    uint32 Closing:1;          // 正在关闭
    uint32 Dormant:1;          // 休眠状态
    uint32 OpenTemporary:1;    // 临时通道
    uint32 Broken:1;           // 已出错
    uint32 bTornOff:1;         // Actor已被撕裂(TornOff)
    uint32 bPendingDormancy:1; // 等待进入休眠
    uint32 bIsInDormancyHysteresis:1; // 休眠迟滞中
    uint32 bPausedUntilReliableACK:1; // 暂停直到可靠包确认
    uint32 SentClosingBunch:1; // 已发送关闭Bunch
    uint32 bPooled:1;          // 在通道池中
    uint32 OpenedLocally:1;    // 本地打开
    uint32 bOpenedForCheckpoint:1; // 回放检查点打开


    // ========== 可靠性相关 ==========

    // Open包的PacketId范围
    FPacketIdRange OpenPacketId;

    // 等待依赖的传入数据
    class FInBunch* InRec;
    int32 NumInRec;

    // 未确认的传出可靠数据
    class FOutBunch* OutRec;
    int32 NumOutRec;

    // 正在接收的部分Bunch
    class FInBunch* InPartialBunch;
};
```

### 3.3 状态标志详解

| 标志               | 含义         | 触发条件                |
| ------------------ | ------------ | ----------------------- |
| `OpenAcked`        | Open包已确认 | 收到Open包的ACK         |
| `Closing`          | 正在关闭     | 调用Close()             |
| `Dormant`          | 休眠状态     | Actor失去相关性         |
| `Broken`           | 已出错       | 遇到错误，忽略后续包    |
| `bTornOff`         | 撕裂状态     | Actor销毁但保留最后状态 |
| `bPendingDormancy` | 等待休眠     | 准备进入休眠            |

### 3.4 核心虚函数

```cpp
class UChannel : public UObject
{
public:
    // 初始化通道
    virtual void Init(UNetConnection* InConnection, int32 InChIndex, EChannelCreateFlags CreateFlags);

    // 关闭通道
    virtual int64 Close(EChannelCloseReason Reason);

    // 处理接收到的Bunch (纯虚函数，子类必须实现)
    virtual void ReceivedBunch(FInBunch& Bunch) PURE_VIRTUAL(UChannel::ReceivedBunch,);

    // 处理ACK
    virtual void ReceivedAck(int32 AckPacketId);

    // 处理NAK
    virtual void ReceivedNak(int32 NakPacketId);

    // Tick
    virtual void Tick();

    // 发送Bunch
    virtual FPacketIdRange SendBunch(FOutBunch* Bunch, bool Merge);

    // 描述通道 (调试用)
    virtual FString Describe();

    // 是否准备好进入休眠
    virtual bool ReadyForDormancy(bool suppressLogs=false) { return false; }

    // 开始进入休眠状态
    virtual void StartBecomingDormant() { }
};
```

---

## 四、数据束 (Bunch)

### 4.1 Bunch概念

**Bunch是通道级别数据传输的基本单位**。

```
┌─────────────────────────────────────────────────────────────────────┐
│                        数据传输层级                                  │
└─────────────────────────────────────────────────────────────────────┘

    Packet (连接级别)
    ┌──────────────────────────────────────────────────────────────┐
    │  Header  │  Bunch 1  │  Bunch 2  │  Bunch 3  │  ...  │ Tail  │
    └──────────────────────────────────────────────────────────────┘
                 │
                 ▼
    Bunch (通道级别)
    ┌──────────────────────────────────────────────────────────────┐
    │ Header │ Channel Data (属性/RPC/控制信息)                     │
    └──────────────────────────────────────────────────────────────┘
```

### 4.2 FOutBunch (输出数据束)

**位置**: `Engine/Public/Net/DataBunch.h:23`

```cpp
class FOutBunch : public FNetBitWriter
{
public:
    // ========== 链表结构 ==========
    FOutBunch* Next;          // 下一个Bunch (用于重传队列)

    // ========== 通道信息 ==========
    UChannel* Channel;        // 所属通道
    int32 ChIndex;            // 通道索引
    FName ChName;             // 通道类型名
    int32 ChSequence;         // 通道序列号 (可靠传输)
    int32 PacketId;           // 所属Packet的ID

    // ========== 标志位 ==========
    uint8 ReceivedAck:1;      // 已收到ACK
    uint8 bOpen:1;            // 打开通道的Bunch
    uint8 bClose:1;           // 关闭通道的Bunch
    uint8 bReliable:1;        // 可靠Bunch
    uint8 bPartial:1;         // 部分Bunch (未完成)
    uint8 bPartialInitial:1;  // 部分Bunch的第一个
    uint8 bPartialFinal:1;    // 部分Bunch的最后一个
    uint8 bHasPackageMapExports:1;  // 包含NetGUID导出
    uint8 bHasMustBeMappedGUIDs:1;  // 包含必须映射的GUID

    // ========== 关闭原因 ==========
    EChannelCloseReason CloseReason;

    // ========== 导出数据 ==========
    TArray<FNetworkGUID> ExportNetGUIDs;  // 导出的GUID列表
    TArray<uint64> NetFieldExports;       // 网络字段导出

    // ========== 构造函数 ==========
    FOutBunch();
    FOutBunch(UChannel* InChannel, bool bClose);
    FOutBunch(UPackageMap* PackageMap, int64 InMaxBits = 1024);
};
```

### 4.3 FInBunch (输入数据束)

**位置**: `Engine/Public/Net/DataBunch.h:126`

```cpp
class FInBunch : public FNetBitReader
{
public:
    // ========== 基本信息 ==========
    int32 PacketId;           // 所属Packet的ID
    FInBunch* Next;           // 链表下一个
    UNetConnection* Connection; // 所属连接
    int32 ChIndex;            // 通道索引
    FName ChName;             // 通道类型名
    int32 ChSequence;         // 通道序列号

    // ========== 标志位 ==========
    uint8 bOpen:1;            // 打开通道
    uint8 bClose:1;           // 关闭通道
    uint8 bReliable:1;        // 可靠Bunch
    uint8 bPartial:1;         // 部分Bunch
    uint8 bPartialInitial:1;  // 部分Bunch的第一个
    uint8 bPartialFinal:1;    // 部分Bunch的最后一个
    uint8 bHasPackageMapExports:1;
    uint8 bHasMustBeMappedGUIDs:1;
    uint8 bIgnoreRPCs:1;      // 忽略RPC

    EChannelCloseReason CloseReason;

    // ========== 构造函数 ==========
    FInBunch(UNetConnection* InConnection, uint8* Src=NULL, int64 CountBits=0);
    FInBunch(FInBunch& InBunch, bool CopyBuffer);
};
```

### 4.4 Bunch类型与标志

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Bunch类型与标志组合                             │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────┬──────────────────────────────────────────────┐
│      Bunch类型      │                   标志组合                    │
├─────────────────────┼──────────────────────────────────────────────┤
│ Open Bunch          │ bOpen = 1                                    │
│ (打开通道)           │ 通常 bReliable = 1                           │
├─────────────────────┼──────────────────────────────────────────────┤
│ Close Bunch         │ bClose = 1                                   │
│ (关闭通道)           │ 通常 bReliable = 1                           │
│                     │ 包含 CloseReason                             │
├─────────────────────┼──────────────────────────────────────────────┤
│ Normal Bunch        │ bOpen = 0, bClose = 0                        │
│ (正常数据)           │ 包含属性复制或RPC数据                         │
├─────────────────────┼──────────────────────────────────────────────┤
│ Partial Bunch       │ bPartial = 1                                 │
│ (分割传输)           │ bPartialInitial = 1 (第一个)                 │
│                     │ bPartial = 1, bPartialInitial = 0 (中间)     │
│                     │ bPartialFinal = 1 (最后一个)                  │
├─────────────────────┼──────────────────────────────────────────────┤
│ Reliable Bunch      │ bReliable = 1                                │
│ (可靠传输)           │ 有ChSequence序号                             │
│                     │ 会重传直到确认                                │
└─────────────────────┴──────────────────────────────────────────────┘
```

---

## 五、通道生命周期

### 5.1 通道状态转换

```
┌─────────────────────────────────────────────────────────────────────┐
│                      通道状态转换图                                   │
└─────────────────────────────────────────────────────────────────────┘

                    ┌──────────────┐
                    │    未创建     │
                    └──────────────┘
                           │
                           │ 创建通道
                           ▼
                    ┌──────────────┐
                    │   已创建      │
                    │ OpenAcked=0  │
                    └──────────────┘
                           │
                           │ 发送/接收 Open Bunch
                           │ 收到 ACK
                           ▼
                   ┌──────────────┐
           ┌───────│    活跃       │◄──────────┐
           │       │ OpenAcked=1  │            │
           │       │ Closing=0    │            │
           │       └──────────────┘            │
           │              │                    │
           │              │ Actor失去相关性     │ 唤醒
           │              ▼                    │
           │       ┌──────────────┐            │
           │       │    休眠      │────────────┘
           │       │ Dormant=1    │
           │       └──────────────┘
           │              │
           │              │ 开始关闭
           │              ▼
           │       ┌──────────────┐
           │       │    关闭中     │
           │       │ Closing=1    │
           │       │ 等待可靠数据  │
           │       │ 确认         │
           │       └──────────────┘
           │              │
           │              │ 确认完成/超时
           │              ▼
           │       ┌──────────────┐
           └──────►│   已销毁      │
                   │ Broken=1     │
                   └──────────────┘
```

### 5.2 通道打开流程

```
服务器                                              客户端
  │                                                   │
  │  1. 创建ActorChannel                              │
  │     ActorChannel = NewObject<UActorChannel>()     │
  │     ActorChannel->Init(Connection, ChIndex, ...)  │
  │                                                   │
  │  2. 发送 Open Bunch                               │
  │     FOutBunch Bunch(Channel, false)               │
  │     Bunch.bOpen = true                            │
  │     Bunch.bReliable = true                        │
  │  ─────────── Open Bunch ────────────────────────► │
  │                                                   │  3. 接收 Open Bunch
  │                                                   │     创建对应通道
  │                                                   │     OpenAcked = 1
  │                                                   │
  │  4. 收到ACK                                       │
  │     OpenAcked = 1                                 │
  │                                                   │
  │         === 通道建立，开始正常通信 ===              │
  │                                                   │
```

### 5.3 通道关闭流程

```cpp
// 关闭通道的调用流程
int64 UChannel::Close(EChannelCloseReason Reason)
{
    // 1. 设置Closing标志
    Closing = 1;

    // 2. 发送Close Bunch
    FOutBunch CloseBunch(this, true);  // bClose = true
    CloseBunch.CloseReason = Reason;
    CloseBunch.bReliable = true;

    SendBunch(&CloseBunch, false);

    // 3. 等待所有可靠数据确认
    // ...

    // 4. 清理资源
    CleanUp(false, Reason);

    return BitsWritten;
}
```

---

## 六、UActorChannel 详解

### 6.1 类定义

**位置**: `Engine/Classes/Engine/ActorChannel.h:77`

```cpp
/**
 * Actor通道：用于交换Actor及其子对象的属性和RPC
 *
 * ActorChannel管理复制Actor的创建和生命周期。
 * 实际的属性和RPC复制发生在FObjectReplicator中。
 */
UCLASS(transient, customConstructor, MinimalAPI)
class UActorChannel : public UChannel
{
    GENERATED_BODY()
};
```

### 6.2 ActorChannel数据结构

根据源码注释 (ActorChannel.h:52-74)，ActorChannel Bunch结构如下：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ActorChannel Bunch 结构                          │
├─────────────────────┬───────────────────────────────────────────────┤
│ SpawnInfo           │ (仅初始Bunch)                                  │
│  - Actor Class      │   由ActorChannel创建                           │
│  - Spawn Loc/Rot    │                                               │
│ NetGUID assigns     │                                               │
│  - Actor NetGUID    │                                               │
│  - Component NetGUIDs│                                              │
├─────────────────────┼───────────────────────────────────────────────┤
│ NetGUID ObjRef      │ (内容块) x N个复制对象 (Actor + 组件)           │
│                     │   每个块由独立的FObjectReplicator创建           │
├─────────────────────┼───────────────────────────────────────────────┤
│ Properties...       │ 复制的属性数据                                 │
├─────────────────────┼───────────────────────────────────────────────┤
│ RPCs...             │ RPC调用数据                                    │
├─────────────────────┼───────────────────────────────────────────────┤
│ </End Tag>          │ 结束标记                                       │
└─────────────────────┴───────────────────────────────────────────────┘
```

### 6.3 核心成员

```cpp
class UActorChannel : public UChannel
{
public:
    // ========== Actor关联 ==========

    // 关联的Actor
    UPROPERTY()
    TObjectPtr<AActor> Actor;

    // Actor的NetGUID
    FNetworkGUID ActorNetGUID;


    // ========== 复制状态 ==========

    // Actor复制器
    TSharedPtr<FObjectReplicator> ActorReplicator;

    // 子对象复制器映射
    TMap<UObject*, TSharedRef<FObjectReplicator>> ReplicationMap;


    // ========== 时间追踪 ==========

    // 最后相关时间
    double RelevantTime;

    // 最后更新时间
    double LastUpdateTime;


    // ========== 状态标志 ==========

    uint32 SpawnAcked:1;              // Spawn已确认
    uint32 bForceCompareProperties:1; // 强制比较属性
    uint32 bIsReplicatingActor:1;     // 正在复制Actor


    // ========== 异步加载支持 ==========

    // 等待处理的Bunch队列
    TArray<FInBunch*> QueuedBunches;

    // 等待解析的GUID
    TSet<FNetworkGUID> PendingGuidResolves;

    // 创建的子对象列表
    UPROPERTY()
    TArray<TObjectPtr<UObject>> CreateSubObjects;
};
```

### 6.4 核心函数

```cpp
class UActorChannel : public UChannel
{
public:
    // 复制Actor - 返回复制的比特数
    int64 ReplicateActor();

    // 设置通道关联的Actor
    void SetChannelActor(AActor* InActor, ESetChannelActorFlags Flags);

    // 复制子对象
    bool ReplicateSubobject(UObject* Obj, FOutBunch& Bunch, FReplicationFlags RepFlags);

    // 处理接收到的Bunch
    virtual void ReceivedBunch(FInBunch& Bunch) override;

    // 关闭通道
    virtual int64 Close(EChannelCloseReason Reason) override;

    // 准备进入休眠
    virtual bool ReadyForDormancy(bool debug=false) override;

    // 开始进入休眠
    virtual void StartBecomingDormant() override;

    // 获取Actor复制数据
    FObjectReplicator& GetActorReplicationData();
};
```

### 6.5 ReplicateActor 流程

```cpp
int64 UActorChannel::ReplicateActor()
{
    // 1. 检查Actor是否准备好复制
    if (!IsActorReadyForReplication())
        return 0;

    // 2. 创建输出Bunch
    FOutBunch Bunch(this, false);

    // 3. 写入内容块
    //    - 写入属性数据
    //    - 写入RPC数据
    //    - 写入子对象数据

    // 4. 复制子对象
    DoSubObjectReplication(Bunch, RepFlags);

    // 5. 发送Bunch
    SendBunch(&Bunch, false);

    // 6. 更新时间戳
    LastUpdateTime = Driver->GetElapsedTime();

    return Bunch.GetNumBits();
}
```

---

## 七、可靠性传输机制

### 7.1 序列号机制

```
┌─────────────────────────────────────────────────────────────────────┐
│                      序列号机制                                      │
└─────────────────────────────────────────────────────────────────────┘

Packet序列号 (连接级别):
    - 每个Packet递增
    - 用于检测丢包
    - 不重复使用

Bunch序列号 (通道级别):
    - 仅可靠Bunch有
    - 用于可靠传输
    - 可以重传 (相同序号)

示例:

Packet 100:
    - Bunch A (ChSequence=1, Reliable)
    - Bunch B (ChSequence=2, Reliable)
    - Bunch C (Unreliable)

Packet 101:
    - Bunch D (ChSequence=3, Reliable)

如果Packet 100丢失:
    - 重传 Bunch A (ChSequence=1) 到新Packet
    - 重传 Bunch B (ChSequence=2) 到新Packet
    - Bunch C 不重传 (不可靠)
```

### 7.2 可靠性保证流程

```
发送端                                             接收端
  │                                                 │
  │  1. 创建可靠Bunch                                │
  │     Bunch.bReliable = true                      │
  │     Bunch.ChSequence = NextSeq++                │
  │                                                 │
  │  2. 添加到OutRec队列                             │
  │     OutRec = Bunch                              │
  │                                                 │
  │  3. 发送Bunch                                   │
  │  ─────────── Bunch ────────────────────────────►│
  │                                                 │  4. 接收Bunch
  │                                                 │     检查ChSequence
  │                                                 │
  │  5. 收到ACK                                     │
  │ ◄─────────── ACK ────────────────────────────── │
  │                                                 │
  │  6. 从OutRec移除                                 │
  │     OutRec = nullptr                            │
  │                                                 │


如果收到NAK:

发送端                                             接收端
  │                                                  │
  │  ─────────── Bunch (丢失) ────────────────────►  │
  │                                                  │
  │ ◄─────────── NAK ──────────────────────────────  │
  │                                                  │
  │  重传Bunch                                       │
  │  (相同ChSequence)                                │
  │  ─────────── Bunch (重传) ─────────────────────► │
  │                                                  │
```

### 7.3 OutRec和InRec

```cpp
// 发送端重传缓冲
class UChannel
{
    // 未确认的传出可靠数据
    class FOutBunch* OutRec;
    int32 NumOutRec;
};

// 接收端排队缓冲
class UChannel
{
    // 等待依赖的传入数据
    class FInBunch* InRec;
    int32 NumInRec;
};
```

**工作原理**:

```
接收端处理乱序Bunch:

期望ChSequence = 5

收到 ChSequence = 6
    │
    ▼
检测到缺失 ChSequence = 5
    │
    ▼
将 ChSequence = 6 添加到 InRec
    │
    ▼
等待 ChSequence = 5
    │
    ▼
收到 ChSequence = 5
    │
    ▼
处理 ChSequence = 5
    │
    ▼
处理 InRec 中的 ChSequence = 6
```

---

## 八、Partial Bunch 机制

### 8.1 为什么需要Partial Bunch?

当Bunch数据过大，超过单个Packet的限制时，需要分割传输。

```cpp
// 常量定义
extern const int32 MAX_BUNCH_SIZE;  // 最大Bunch大小
```

### 8.2 分割与重组

```
发送端:

原始大数据Bunch (10KB)
        │
        ▼
┌───────────────────────────────────────────┐
│ 分割为多个Partial Bunch                    │
│                                           │
│ Partial 1: bPartial=1, bPartialInitial=1  │
│ Partial 2: bPartial=1                     │
│ Partial 3: bPartial=1                     │
│ Partial 4: bPartial=1, bPartialFinal=1    │
└───────────────────────────────────────────┘


接收端:

收到 Partial 1 (Initial)
        │
        ▼
创建 InPartialBunch 缓冲区
        │
        ▼
收到 Partial 2, 3
        │
        ▼
追加到 InPartialBunch
        │
        ▼
收到 Partial 4 (Final)
        │
        ▼
完成重组，处理完整Bunch
```

### 8.3 代码示例

```cpp
// 发送Partial Bunch (伪代码)
void SendLargeBunch(FOutBunch& LargeBunch)
{
    int32 TotalBits = LargeBunch.GetNumBits();
    int32 BitsSent = 0;

    bool bFirst = true;

    while (BitsSent < TotalBits)
    {
        FOutBunch PartialBunch;

        PartialBunch.bPartial = true;
        PartialBunch.bPartialInitial = bFirst;
        PartialBunch.bPartialFinal = (BitsSent + MAX_PARTIAL_SIZE >= TotalBits);

        // 复制数据...
        // ...

        SendBunch(&PartialBunch, false);

        BitsSent += MAX_PARTIAL_SIZE;
        bFirst = false;
    }
}
```

---

## 九、通道休眠 (Dormancy)

### 9.1 什么是通道休眠?

当Actor失去相关性(比如距离过远)时，通道可以进入休眠状态，停止复制但不销毁远程Actor。

### 9.2 休眠状态转换

```
┌─────────────────────────────────────────────────────────────────────┐
│                      休眠状态转换                                    │
└─────────────────────────────────────────────────────────────────────┘

活跃状态
    │
    │ Actor失去相关性
    │ 调用 StartBecomingDormant()
    ▼
等待休眠 (bPendingDormancy = 1)
    │
    │ ReadyForDormancy() == true
    │ 所有可靠数据已确认
    ▼
休眠状态 (Dormant = 1)
    │
    │ Actor重新获得相关性
    ▼
唤醒，恢复活跃

休眠的好处:
    - 减少带宽消耗
    - 保持客户端Actor状态
    - 快速恢复复制
```

### 9.3 休眠相关函数

```cpp
// 开始进入休眠
void UActorChannel::StartBecomingDormant()
{
    bPendingDormancy = true;
}

// 检查是否准备好休眠
bool UActorChannel::ReadyForDormancy(bool debug)
{
    // 检查是否有未确认的可靠数据
    if (NumOutRec > 0)
        return false;

    // 检查其他条件...

    return true;
}

// 进入休眠状态
void UActorChannel::BecomeDormant()
{
    Dormant = true;
    bPendingDormancy = false;

    // 通知连接保存休眠状态
    // ...
}
```

---

## 十、通道池 (Channel Pool)

### 10.1 为什么需要通道池?

ActorChannel的创建和销毁开销较大，使用对象池可以提高性能。

```cpp
// NetDriver中的通道池
class UNetDriver
{
    // 已使用并回收的通道列表
    UPROPERTY()
    TArray<TObjectPtr<UChannel>> ActorChannelPool;
};
```

### 10.2 通道池使用流程

```
获取通道:

GetOrCreateChannelByName(ChName)
        │
        ▼
检查通道池
        │
   ┌────┴────┐
   │         │
 有可用     池为空
   │         │
   ▼         ▼
从池取出   创建新通道
   │
   ▼
重置通道状态
   │
   ▼
返回通道


释放通道:

ReleaseToChannelPool(Channel)
        │
        ▼
调用 AddedToChannelPool()
        │
        ▼
重置通道到初始状态
        │
        ▼
添加到池中
```

---

## 十一、代码示例

### 11.1 获取Actor通道

```cpp
// 获取Actor对应的通道
UActorChannel* GetActorChannel(AActor* Actor, UNetConnection* Connection)
{
    if (!Actor || !Connection)
        return nullptr;

    // 从连接的ActorChannel映射中查找
    auto& ActorChannels = Connection->ActorChannels;
    UActorChannel** ChannelPtr = ActorChannels.Find(Actor);

    if (ChannelPtr)
    {
        return *ChannelPtr;
    }

    return nullptr;
}
```

### 11.2 监控通道状态

```cpp
void MonitorChannels(UNetConnection* Connection)
{
    UE_LOG(LogTemp, Log, TEXT("=== Channel Status ==="));

    for (UChannel* Channel : Connection->OpenChannels)
    {
        FString Status = FString::Printf(
            TEXT("ChIndex=%d, ChName=%s, OpenAcked=%d, Closing=%d, Dormant=%d"),
            Channel->ChIndex,
            *Channel->ChName.ToString(),
            Channel->OpenAcked ? 1 : 0,
            Channel->Closing ? 1 : 0,
            Channel->Dormant ? 1 : 0
        );

        // 如果是ActorChannel，显示更多信息
        if (UActorChannel* ActorCh = Cast<UActorChannel>(Channel))
        {
            Status += FString::Printf(
                TEXT(", Actor=%s"),
                ActorCh->GetActor() ? *ActorCh->GetActor()->GetName() : TEXT("nullptr")
            );
        }

        UE_LOG(LogTemp, Log, TEXT("%s"), *Status);
    }
}
```

### 11.3 强制通道休眠

```cpp
// 强制Actor通道进入休眠
void ForceChannelDormant(AActor* Actor)
{
    if (!Actor || !Actor->GetNetConnection())
        return;

    UActorChannel* Channel = Actor->GetNetConnection()->ActorChannels.FindRef(Actor);
    if (Channel && !Channel->Dormant)
    {
        Channel->StartBecomingDormant();
    }
}
```

---

## 十二、调试技巧

### 12.1 控制台命令

```
; 显示网络统计
stat net

; 显示通道信息
net.channels

; 显示特定连接的通道
net.listclients
```

### 12.2 日志输出

```cpp
// Bunch信息输出 (DataBunch.h中的ToString方法)
FOutBunch Bunch;
// ...
UE_LOG(LogNet, Log, TEXT("%s"), *Bunch.ToString());

// 输出示例:
// FOutBunch: Channel[2] ChSequence: 5 NumBits: 256 PacketId: 100
//            bOpen: 0 bClose: 0 bReliable: 1 bPartial: 0/0/0
```

### 12.3 常见问题

| 问题       | 可能原因       | 解决方案             |
| ---------- | -------------- | -------------------- |
| 通道打不开 | Open Bunch丢失 | 检查可靠传输         |
| 属性不同步 | 通道休眠       | 唤醒通道或检查相关性 |
| 通道泄漏   | 未正确关闭     | 检查Close调用        |
| 高延迟     | OutRec堆积     | 检查可靠Bunch数量    |

---

## 十三、总结

### 13.1 核心要点

1. **通道是数据传输的基本单位**
   - 每种类型负责特定功能
   - 通过ChIndex标识

2. **Bunch是通道级数据**
   - 包含属性复制、RPC数据
   - 支持可靠/不可靠传输

3. **可靠性保证**
   - 序列号机制
   - ACK/NAK确认
   - 重传缓冲(OutRec)

4. **通道生命周期**
   - 创建 → 打开 → 活跃 → (休眠) → 关闭 → 销毁

### 13.2 下一课预告

**第四课：Actor网络复制基础**

将深入讲解：

- AActor网络属性
- 网络角色(Role/RemoteRole)
- Actor复制生命周期
- GetLifetimeReplicatedProps

---

## 十四、课后练习

### 练习1：追踪Bunch流程

在调试器中追踪以下流程：

```
SendBunch → PrepBunch → SendRawBunch → Connection::SendPackage
```

### 练习2：实现通道监控

创建一个控制台命令，输出当前所有通道的状态信息。

### 练习3：分析通道休眠

在游戏中使Actor远离玩家，观察通道如何进入休眠状态。

---

## 参考资料

1. **源码文件**:
   - `Engine/Classes/Engine/Channel.h`
   - `Engine/Classes/Engine/ActorChannel.h`
   - `Engine/Public/Net/DataBunch.h`

2. **架构文档**:
   - `Engine/Classes/Engine/NetDriver.h` 头部注释

---

_上一课: [第二课：网络驱动与连接管理](./Lesson02_NetDriverAndConnection.md)_
_下一课: [第四课：Actor网络复制基础](./Lesson04_ActorReplicationBasic.md)_
