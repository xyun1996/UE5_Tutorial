# 第十三课：网络架构深度问答

本课整理了UE网络架构中的常见深度问题，帮助理解底层设计原理。

---

## 目录

1. [PushModel系统的现状与未来](#1-pushmodel系统的现状与未来)
2. [为什么UE不在Packet级别实现可靠传输](#2-为什么ue不在packet级别实现可靠传输)
3. [Partial Bunch机制详解](#3-partial-bunch机制详解)
4. [可靠Bunch溢出问题](#4-可靠bunch溢出问题)

---

## 1. PushModel系统的现状与未来

### 1.1 问题背景

PushModel是UE4.20引入的属性脏标记系统，旨在替代传统的"每帧扫描所有属性"的方式。随着Iris系统的推出，开发者关心PushModel是否还被使用。

### 1.2 当前状态（UE5.7）

#### 编译时状态

```cpp
// TargetRules.cs
// 只有Editor目标才默认启用PushModel
public bool bWithPushModel
{
    get => bWithPushModelOverride ?? (Type == TargetType.Editor);
    set => bWithPushModelOverride = value;
}
```

#### 运行时状态

```cpp
// PushModel.cpp
bool bIsPushModelEnabled = false;  // 默认关闭

// Iris中的PushModel被重命名为LegacyPushModel
// LegacyPushModel.cpp
int IrisPushModelMode = 2;  // 可运行时切换
```

### 1.3 系统架构对比

```
┌─────────────────────────────────────────────────────────────────┐
│                    传统复制系统                                   │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │ PushModel   │───►│ RepLayout   │───►│ NetDriver   │         │
│  │ (属性脏标记) │    │ (属性比较)   │    │ (数据发送)   │         │
│  └─────────────┘    └─────────────┘    └─────────────┘         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    Iris复制系统                                  │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────┐ │
│  │ DirtyNetObject  │───►│ ReplicationState │───►│ IrisDriver  │ │
│  │ Tracker (内置)   │    │ (状态序列化)     │    │ (高效发送)  │ │
│  └─────────────────┘    └─────────────────┘    └─────────────┘ │
│                                                                  │
│  LegacyPushModel ──► 仅用于向后兼容                             │
└─────────────────────────────────────────────────────────────────┘
```

### 1.4 关键源码分析

```cpp
// Engine/Source/Runtime/Net/Iris/Private/Iris/ReplicationSystem/LegacyPushModel.h
namespace UE::Net::Private
{
    // Iris中使用LegacyPushModel作为兼容层
    extern IRISCORE_API bool bIsIrisPushModelForceEnabled;
    extern IRISCORE_API int IrisPushModelMode;

    // 检查是否启用
    inline bool IsIrisPushModelEnabled(bool bIsPushModelEnabled = IS_PUSH_MODEL_ENABLED())
    {
        return (bIsIrisPushModelForceEnabled | (IrisPushModelMode > 0)) & bIsPushModelEnabled;
    }
}
```

### 1.5 新项目建议

| 场景 | 建议 | 配置方式 |
|------|------|----------|
| **使用Iris** | 不需要PushModel | Iris有内置脏标记追踪 |
| **使用传统复制** | 可选启用PushModel | `Net.IsPushModelEnabled 1` |
| **完全新项目** | 推荐尝试Iris | `net.Iris.UseIrisReplication 1` |

### 1.6 配置方法

```ini
; DefaultEngine.ini

; 方式1：启用Iris（推荐）
[/Script/Engine.NetworkSettings]
Net.Iris.UseIrisReplication=1

; 方式2：传统复制 + PushModel
Net.IsPushModelEnabled=1
```

或命令行参数：
```
UseIrisReplication=1
```

### 1.7 总结

- **PushModel是遗留系统**，在Iris中被重命名为LegacyPushModel
- **Iris是Epic推动的新方向**，有更高效的脏标记追踪机制
- **新项目建议使用Iris**，不需要手动调用MARK_PROPERTY_DIRTY
- 传统复制系统仍可使用PushModel，但需要显式启用

---

## 2. 为什么UE不在Packet级别实现可靠传输

### 2.1 问题背景

UDP是不可靠传输，游戏需要可靠性保证。但UE选择在Bunch/Channel级别实现可靠传输，而非Packet级别，这是为什么？

### 2.2 数据层级结构

```
┌─────────────────────────────────────────────────────────────────┐
│                         Packet (UDP)                             │
│  可能丢失，包含多个Bunch，每个Bunch可能属于不同Channel           │
│                                                                  │
│  ┌───────────────┬───────────────┬───────────────┐              │
│  │    Bunch 1    │    Bunch 2    │    Bunch 3    │              │
│  │  (可靠RPC)    │ (不可靠属性)  │  (可靠属性)   │              │
│  │  Channel A    │  Channel B    │  Channel A    │              │
│  └───────────────┴───────────────┴───────────────┘              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 核心设计原因

#### 原因1：选择性可靠性

一个Packet中可能同时包含：
- **可靠Bunch**：需要重传（如RPC、关键属性）
- **不可靠Bunch**：直接丢弃（如高频位置更新）

```
Packet级别重传：     [可靠数据 + 不可靠数据] 全部重传 ❌
Bunch级别重传：      只重传 [可靠数据] ✓
```

#### 原因2：时效性需求

游戏数据有时效性，过时数据无意义：

```
帧1: PlayerLocation = (100, 0, 0)  ← 丢失
帧2: PlayerLocation = (105, 0, 0)  ← 已发送

如果重传帧1的位置：
- 客户端收到时已经过时
- 浪费带宽和处理时间
- 可能导致位置回跳
```

#### 原因3：带宽效率

```
场景：Packet包含100字节数据
- 可靠数据：30字节
- 不可靠数据：70字节

Packet级重传：重传100字节
Bunch级重传：只重传30字节（节省70%带宽）
```

#### 原因4：灵活的数据合并

```cpp
// DataChannel.cpp:1344
// 只有可靠性相同的Bunch才能合并
Connection->LastOut.bReliable == Bunch->bReliable
```

### 2.4 可靠性实现机制

```
发送方                                    接收方
  │                                         │
  │  Packet (Seq=100)                       │
  │  ├─ Bunch A (可靠, ChSeq=1)             │
  │  └─ Bunch B (不可靠)                    │
  │────────────────────────────────────────►│
  │                                         │ Packet丢失
  │                                         │
  │  NAK (Packet 100 丢失)                  │
  │◄────────────────────────────────────────│
  │                                         │
  │  重发 Packet (新Seq)                    │
  │  └─ Bunch A (可靠, ChSeq=1)  ──────────►│  只重传可靠的
  │                                         │ Bunch B直接丢弃
  │                                         │
  │  ACK (确认收到)                         │
  │◄────────────────────────────────────────│
```

### 2.5 关键源码分析

```cpp
// DataChannel.cpp:1648-1676
void UChannel::ReceivedNak(int32 NakPacketId)
{
    for(FOutBunch* Out=OutRec; Out; Out=Out->Next)
    {
        // 只重传可靠Bunch
        if(Out->PacketId==NakPacketId && !Out->ReceivedAck)
        {
            check(Out->bReliable);  // 确保是可靠的
            UE_LOG(LogNetTraffic, Log, TEXT("Channel %i nak; resending %i..."),
                   Out->ChIndex, Out->ChSequence);

            // 重传这个Bunch
            Connection->SendRawBunch(*Out, 0);
        }
    }
}
```

### 2.6 为什么不用TCP？

| 特性 | TCP | UE/UDP |
|------|-----|--------|
| 可靠性 | 全部可靠 | 选择性可靠 |
| 丢包处理 | 阻塞重传 | 继续发送新数据 |
| 延迟 | 丢包时延迟高 | 只有可靠数据有延迟 |
| 拥塞控制 | 严格拥塞控制 | 可自定义控制 |
| 适用场景 | 文件传输、Web | 实时游戏 |

### 2.7 总结

UE不使用Packet级可靠传输的原因：

| 原因 | 说明 |
|------|------|
| **效率** | 只重传需要的数据，不浪费带宽 |
| **时效性** | 过时数据不重传，保持游戏流畅 |
| **灵活性** | 同一Packet可混合可靠/不可靠数据 |
| **控制权** | 游戏逻辑决定什么需要可靠传输 |

---

## 3. Partial Bunch机制详解

### 3.1 问题背景

UDP有MTU限制（通常约1200字节），但游戏数据可能很大，比如Actor初始同步、大型数组等。Partial Bunch机制解决了Bunch大小超过Packet限制的问题。

### 3.2 核心问题

```
┌─────────────────────────────────────────────────────────────────┐
│                        Bunch (可能很大)                          │
│  例如：一个Actor的所有初始属性、大型数组、复杂RPC参数            │
│                        ↓                                        │
│  问题：单个Bunch可能超过Packet最大大小（约1200字节）              │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 分割机制

#### 发送端：分割大Bunch

```cpp
// DataChannel.cpp:1371-1401
if(Bunch->GetNumBits() > MAX_SINGLE_BUNCH_SIZE_BITS)
{
    uint8* data = Bunch->GetData();
    int64 bitsLeft = Bunch->GetNumBits();

    while(bitsLeft > 0)
    {
        FOutBunch* PartialBunch = new FOutBunch(this, false);
        int64 bitsThisBunch = FMath::Min<int64>(bitsLeft, MAX_PARTIAL_BUNCH_SIZE_BITS);
        PartialBunch->SerializeBits(data, bitsThisBunch);

        OutgoingBunches.Add(PartialBunch);

        bitsLeft -= bitsThisBunch;
        data += (bitsThisBunch >> 3);
    }
}
```

#### 标记分片状态

```cpp
// DataChannel.cpp:1470-1477
if (OutgoingBunches.Num() > 1)
{
    NextBunch->bPartial = 1;
    NextBunch->bPartialInitial = (PartialNum == 0 ? 1 : 0);   // 第一个
    NextBunch->bPartialFinal = (PartialNum == OutgoingBunches.Num() - 1 ? 1 : 0); // 最后一个
}
```

```
原始Bunch (3000字节)
        │
        ▼ 分割
┌─────────────┬─────────────┬─────────────┐
│ Partial #1  │ Partial #2  │ Partial #3  │
│ bPartial=1  │ bPartial=1  │ bPartial=1  │
│ Initial=1   │ Initial=0   │ Initial=0   │
│ Final=0     │ Final=0     │ Final=1     │
└─────────────┴─────────────┴─────────────┘
   Packet1       Packet2       Packet3
```

### 3.4 接收端：重组

```cpp
// DataChannel.cpp:849-922
// 合并条件：
//  - 有有效的InPartialBunch
//  - 当前InPartialBunch未完成
//  - ChSequence是下一个序列
//  - 可靠性标志匹配

if (InPartialBunch && !InPartialBunch->bPartialFinal && bSequenceMatches)
{
    // 合并数据
    InPartialBunch->AppendDataFromChecked(Bunch.GetDataPosChecked(), Bunch.GetBitsLeft());

    if (Bunch.bPartialFinal)
    {
        // 完整Bunch重组完成
        HandleBunch = InPartialBunch;
    }
}
```

### 3.5 关键设计要点

#### 1. 原子性保证

```cpp
// DataChannel.cpp:92-96
int32 GCVarNetPartialBunchReliableThreshold = 8;

// 如果分片超过8个，自动转为可靠传输
// 因为"Partial bunches are atomic and must all make it over to be used"
```

Partial Bunch必须**全部到达**才能处理，任何一个丢失都会导致整个Bunch无效。

#### 2. 字节对齐优化

```cpp
// DataChannel.cpp:878-889
// 非最终Partial Bunch必须是字节对齐的，便于快速复制
if (!Bunch.bPartialFinal && (Bunch.GetBitsLeft() % 8 != 0))
{
    // 错误：非最终分片必须字节对齐
}
```

#### 3. 可靠性一致性

```cpp
// DataChannel.cpp:867
// 所有Partial Bunch的可靠性必须一致
InPartialBunch->bReliable == Bunch->bReliable
```

#### 4. 序列验证

```cpp
// DataChannel.cpp:858-864
// 可靠：ChSequence必须连续+1
// 不可靠：允许相同ChSequence（多个Bunch可能在同一Packet中）
const bool bReliableSequencesMatches = Bunch.ChSequence == InPartialBunch->ChSequence + 1;
const bool bUnreliableSequenceMatches = bReliableSequencesMatches ||
                                        (Bunch.ChSequence == InPartialBunch->ChSequence);
```

### 3.6 典型使用场景

| 场景 | Bunch大小 | 是否需要Partial |
|------|-----------|-----------------|
| 简单RPC调用 | < 500字节 | 否 |
| Actor初始同步 | 可能 > 1KB | 经常需要 |
| 大型数组属性 | 可能很大 | 需要分片 |
| FastArray序列化 | 取决于数组大小 | 可能需要 |

### 3.7 流程图

```
发送方                                    接收方
  │                                         │
  │  大Bunch (3000字节)                     │
  │         │                               │
  │         ▼ 分割                          │
  │  ┌─────────────────┐                    │
  │  │ Partial #1      │───Packet 1────────►│ 收到，存储到InPartialBunch
  │  │ Initial=1       │                    │
  │  └─────────────────┘                    │
  │  ┌─────────────────┐                    │
  │  │ Partial #2      │───Packet 2────────►│ 合并到InPartialBunch
  │  │                 │   (丢失)           │ ✗ 未收到
  │  └─────────────────┘                    │
  │  ┌─────────────────┐                    │
  │  │ Partial #2      │───重传────────────►│ 合并到InPartialBunch
  │  │ (重传)          │                    │
  │  └─────────────────┘                    │
  │  ┌─────────────────┐                    │
  │  │ Partial #3      │───Packet 3────────►│ Final=1, 重组完成
  │  │ Final=1         │                    │ 处理完整Bunch
  │  └─────────────────┘                    │
```

### 3.8 总结

Partial Bunch机制解决了：

| 问题 | 解决方案 |
|------|----------|
| **MTU限制** | 突破单个Packet大小限制 |
| **透明传输** | 上层代码无需关心分片细节 |
| **可靠性支持** | 可选择可靠或不可靠传输 |
| **效率优化** | 字节对齐加速重组过程 |

---

## 4. 可靠Bunch溢出问题

### 4.1 问题背景

每个Channel有`RELIABLE_BUFFER = 512`的限制，这是等待确认的可靠Bunch数量上限。如果超过这个限制会发生什么？

### 4.2 两种溢出场景

#### 场景1：发送端溢出（Outgoing Overflow）

```cpp
// DataChannel.cpp:1414-1448
const bool bOverflowsReliable = (NumOutRec + OutgoingBunches.Num() >= RELIABLE_BUFFER + Bunch->bClose);

if (Bunch->bReliable && bOverflowsReliable)
{
    UE_LOG(LogNetPartialBunch, Warning, TEXT("SendBunch: Reliable partial bunch overflows reliable buffer!"));

    // 发送错误消息
    FString ErrorMsg = NSLOCTEXT("NetworkErrors", "ClientReliableBufferOverflow",
                                  "Outgoing reliable buffer overflow").ToString();
    Connection->SendCloseReason(ENetCloseResult::ReliableBufferOverflow);
    FNetControlMessage<NMT_Failure>::Send(Connection, ErrorMsg);

    // 强制关闭连接
    Connection->Close(ENetCloseResult::ReliableBufferOverflow);
}
```

**触发条件**：发送端有512个可靠Bunch等待ACK，又想发送新的可靠Bunch

**结果**：**连接被强制断开**

#### 场景2：接收端溢出（Incoming Overflow）

```cpp
// DataChannel.cpp:681-689
if (NumInRec >= RELIABLE_BUFFER)
{
    Bunch.SetError();
    AddToChainResultPtr(Bunch.ExtendedError, ENetCloseResult::MaxReliableExceeded);
    UE_LOG(LogNetTraffic, Error, TEXT("UChannel::ReceivedRawBunch: Too many reliable messages queued up"));
    return;
}
```

**触发条件**：接收端收到乱序Bunch（前面有丢失），缓存了512个等待处理

**结果**：Bunch标记错误，**后续处理失败**

### 4.3 正常流程 vs 异常流程

#### 正常流程

```
时间 ──────────────────────────────────────────────────────────────►

发送方:  [B1]──[B2]──[B3]──[B4]──[B5]──[B6]...
           │     │     │     │
ACK:       ◄─────┴─────┴─────┴─────┴─────────────────────────────
           (B1确认后释放缓冲区，可以继续发送)

NumOutRec: 1 → 2 → 3 → 2 → 1 → 2 → ... (正常波动，不会超过512)
```

#### 异常流程（网络拥堵/丢包严重）

```
发送方:  [B1]──[B2]──[B3]──[B4]──[B5]──[B6]──[B7]──...──[B512]──[B513]
           │     │     │     │     │     │     │          │
           ✗     ✗     ✗     ✗     ✗     ✗     ✗          (全部丢失/未ACK)
           │
           └── NumOutRec = 512 (缓冲区满)
           │
           尝试发送 B513
           │
           ▼
    ┌─────────────────────────────────────────────┐
    │           RELIABLE_BUFFER OVERFLOW!          │
    │                                              │
    │  1. 发送 NMT_Failure 消息给对方              │
    │  2. 发送 CloseReason: ReliableBufferOverflow │
    │  3. 强制关闭连接                             │
    │                                              │
    └─────────────────────────────────────────────┘
```

### 4.4 为什么是512？

```cpp
// NetConnection.h:81-83
enum { RELIABLE_BUFFER = 512 };    // Power of 2 >= 1
enum { MAX_CHSEQUENCE = 1024 };    // Power of 2 > RELIABLE_BUFFER
```

设计考量：

| 因素 | 说明 |
|------|------|
| **2的幂次** | 便于位运算处理序列号回绕 |
| **足够大** | 512个Bunch通常足够应对正常网络波动 |
| **内存限制** | 每个Channel缓存512个Bunch已是较大开销 |
| **序列号空间** | MAX_CHSEQUENCE=1024是RELIABLE_BUFFER的2倍，避免回绕问题 |

### 4.5 流量控制机制

#### IsNetReady检查

```cpp
// DataChannel.cpp:1620-1641
int32 UChannel::IsNetReady(bool Saturate)
{
    // 如果缓冲区接近满，返回false阻止发送
    if (NumOutRec >= RELIABLE_BUFFER - 1)
    {
        return false;
    }
    return Connection->IsNetReady(Saturate);
}

bool UChannel::IsNetReady() const
{
    if(NumOutRec >= RELIABLE_BUFFER - 1)
    {
        return false;
    }
    return Connection->IsNetReady();
}
```

#### 监控指标

```cpp
// 网络指标会记录最大队列大小
Connection->GetDriver()->GetMetrics()->SetMaxInt(
    UE::Net::Metric::OutgoingReliableMessageQueueMaxSize, NumOutRec);

// 也会记录因溢出关闭的连接数
GetMetrics()->IncrementInt(UE::Net::Metric::ClosedConnectionsDueToReliableBufferOverflow, 1);
```

### 4.6 如何避免溢出

#### 设计层面

| 策略 | 说明 |
|------|------|
| 减少可靠RPC | 可靠RPC会占用缓冲区，考虑用不可靠方式 |
| 使用不可靠更新 | 高频数据（如位置）用不可靠 |
| 控制Actor数量 | 每个Actor一个Channel，减少Channel数量 |
| 监控网络质量 | 丢包率高时降低发送频率 |

#### 代码层面

```cpp
// 发送前检查
if (!MyChannel->IsNetReady())
{
    // 缓冲区快满了，不要发送
    return;
}

// 或者检查队列大小
if (MyChannel->GetNumOutRec() > 400)  // 预留缓冲空间
{
    // 延迟发送
    return;
}
```

#### 配置层面

```ini
; DefaultEngine.ini

; 调整NetUpdateFrequency降低发送频率
[/Script/Engine.Engine]
NetUpdateFrequency=60  ; 默认100，降低可减少数据量
```

### 4.7 总结

| 场景 | 触发条件 | 后果 | 恢复方式 |
|------|----------|------|----------|
| 发送端溢出 | NumOutRec ≥ 512 | 连接断开 | 重新连接 |
| 接收端溢出 | NumInRec ≥ 512 | Bunch错误 | 可能断开 |

**UE选择直接断开连接的原因**：

1. 超过512个Bunch未确认说明网络已严重问题
2. 继续发送只会加剧拥塞
3. 断开重连是更干净的恢复方式
4. 避免无效数据堆积

---

## 课程总结

本课深入探讨了UE网络架构的四个核心问题：

| 问题 | 核心要点 |
|------|----------|
| PushModel现状 | 遗留系统，Iris是未来方向 |
| Bunch级可靠性 | 选择性重传，时效性优先 |
| Partial Bunch | 突破MTU限制，透明分片 |
| 缓冲区溢出 | 512上限，超出则断开连接 |

理解这些底层机制有助于：
- 编写更高效的网络代码
- 排查网络相关问题
- 做出正确的架构决策

---

## 参考资料

- Engine/Source/Runtime/Net/Core/Public/Net/Core/PushModel/PushModel.h
- Engine/Source/Runtime/Net/Iris/Private/Iris/ReplicationSystem/LegacyPushModel.cpp
- Engine/Source/Runtime/Engine/Private/DataChannel.cpp
- Engine/Source/Runtime/Engine/Classes/Engine/NetConnection.h
