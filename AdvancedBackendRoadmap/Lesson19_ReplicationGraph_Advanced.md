# 第19课：ReplicationGraph进阶

> 本课深入讲解UE5的ReplicationGraph高级应用，包括自定义策略、性能优化和分析工具。

---

## 课程目标

- 深入理解ReplicationGraph架构
- 掌握自定义SpatialGrid策略
- 学会Critical Classes设计
- 理解流式距离优化
- 掌握性能分析工具

---

## 一、ReplicationGraph架构深入

### 1.1 核心概念

```
┌─────────────────────────────────────────────────────────────┐
│                  ReplicationGraph架构                        │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │               ReplicationGraph                       │    │
│  │                                                      │    │
│  │  ┌───────────────────────────────────────────────┐  │    │
│  │  │            Global Nodes                        │  │    │
│  │  │  ┌─────────────────────────────────────────┐  │  │    │
│  │  │  │ AlwaysRelevant                          │  │  │    │
│  │  │  │ - GameState                             │  │  │    │
│  │  │  │ - GameMode                              │  │  │    │
│  │  │  │ - Important Actors                      │  │  │    │
│  │  │  └─────────────────────────────────────────┘  │  │    │
│  │  │  ┌─────────────────────────────────────────┐  │  │    │
│  │  │  │ PlayerStateFrequencyLimiter             │  │  │    │
│  │  │  │ - 控制PlayerState更新频率               │  │  │    │
│  │  │  └─────────────────────────────────────────┘  │  │    │
│  │  └───────────────────────────────────────────────┘  │    │
│  │                                                      │    │
│  │  ┌───────────────────────────────────────────────┐  │    │
│  │  │            Spatial Nodes                       │  │    │
│  │  │  ┌─────────────────────────────────────────┐  │  │    │
│  │  │  │ GridSpatialization2D                    │  │  │    │
│  │  │  │ ┌─────┐ ┌─────┐ ┌─────┐               │  │  │    │
│  │  │  │ │Cell │ │Cell │ │Cell │ ...           │  │  │    │
│  │  │  │ └─────┘ └─────┘ └─────┘               │  │  │    │
│  │  │  │ 根据位置自动分配Actor到格子           │  │  │    │
│  │  │  └─────────────────────────────────────────┘  │  │    │
│  │  └───────────────────────────────────────────────┘  │    │
│  │                                                      │    │
│  │  ┌───────────────────────────────────────────────┐  │    │
│  │  │          Connection-Specific Nodes             │  │    │
│  │  │  ┌─────────────────────────────────────────┐  │  │    │
│  │  │  │ AlwaysRelevant_ForConnection            │  │  │    │
│  │  │  │ - PlayerController                      │  │  │    │
│  │  │  │ - Owned Actors                          │  │  │    │
│  │  │  └─────────────────────────────────────────┘  │  │    │
│  │  └───────────────────────────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 节点类型详解

```cpp
// ReplicationGraphNode类型

// 1. 空间化节点
// UReplicationGraphNode_GridSpatialization2D
// - 将世界划分为2D网格
// - Actor根据位置自动分配到格子
// - 只复制玩家所在格子及周围格子的Actor

// 2. 始终相关节点
// UReplicationGraphNode_AlwaysRelevant_AlwaysRelevantNode
// - 所有玩家始终看到这些Actor
// - 用于GameState、重要游戏对象

// 3. 玩家状态节点
// UReplicationGraphNode_PlayerStateFrequencyLimiter
// - 控制PlayerState的更新频率
// - 避免频繁更新影响性能

// 4. 连接专属节点
// UReplicationGraphNode_AlwaysRelevant_ForConnection
// - 每个连接独立的节点
// - 存放该连接特有的Actor（如PlayerController）
```

---

## 二、自定义ReplicationGraph

### 2.1 自定义Graph类

```cpp
// MyReplicationGraph.h
#pragma once

#include "ReplicationGraph.h"
#include "MyReplicationGraph.generated.h"

// 自定义网格空间化节点
UCLASS()
class UMyGridSpatializationNode : public UReplicationGraphNode_GridSpatialization2D
{
    GENERATED_BODY()

public:
    UMyGridSpatializationNode();

    virtual void NotifyAddNetworkActor(const FNewReplicatedActorInfo& ActorInfo) override;
    virtual void NotifyRemoveNetworkActor(const FNewReplicatedActorInfo& ActorInfo) override;
    virtual void GatherActorListsForConnection(const FConnectionGatherActorListParameters& Params) override;

    // 配置
    void SetGridSize(float CellSize);
    void SetSpatialBias(const FVector2D& Bias);
    void SetViewDistance(float Distance);

protected:
    // 自定义数据
    float CustomCullDistance;
    TMap<FIntVector, TArray<FNewReplicatedActorInfo>> CellActors;
};

// 自定义优先级节点
UCLASS()
class UMyPrioritizationNode : public UReplicationGraphNode
{
    GENERATED_BODY()

public:
    virtual void GatherActorListsForConnection(const FConnectionGatherActorListParameters& Params) override;

    void SetPrioritySettings(const TMap<FString, float>& Settings);

private:
    TMap<FString, float> PrioritySettings;
    TArray<FNewReplicatedActorInfo> Actors;

    float CalculatePriority(const FNewReplicatedActorInfo& Actor, const FConnectionGatherActorListParameters& Params);
};

// 主Graph类
UCLASS()
class UMyReplicationGraph : public UReplicationGraph
{
    GENERATED_BODY()

public:
    virtual void InitGlobalGraphNodes() override;
    virtual void InitConnectionGraphNodes(UNetReplicationGraphConnection* Connection) override;

    // 节点访问
    UMyGridSpatializationNode* GetSpatializationNode() { return GridNode; }

    // 配置
    UPROPERTY(EditDefaultsOnly, Category = "Configuration")
    float GridCellSize = 10000.0f;

    UPROPERTY(EditDefaultsOnly, Category = "Configuration")
    float SpatialBiasX = -100000.0f;

    UPROPERTY(EditDefaultsOnly, Category = "Configuration")
    float SpatialBiasY = -100000.0f;

    UPROPERTY(EditDefaultsOnly, Category = "Configuration")
    float ViewDistance = 20000.0f;

    UPROPERTY(EditDefaultsOnly, Category = "Configuration")
    float PlayerStateUpdateFrequency = 1.0f;

private:
    // 全局节点
    UPROPERTY()
    UMyGridSpatializationNode* GridNode;

    UPROPERTY()
    UReplicationGraphNode_AlwaysRelevant_AlwaysRelevantNode* AlwaysRelevantNode;

    UPROPERTY()
    UReplicationGraphNode_PlayerStateFrequencyLimiter* PlayerStateNode;

    UPROPERTY()
    UMyPrioritizationNode* PrioritizationNode;
};

// MyReplicationGraph.cpp
#include "MyReplicationGraph.h"
#include "Engine/World.h"

UMyGridSpatializationNode::UMyGridSpatializationNode()
{
    CustomCullDistance = 20000.0f;
}

void UMyGridSpatializationNode::NotifyAddNetworkActor(const FNewReplicatedActorInfo& ActorInfo)
{
    Super::NotifyAddNetworkActor(ActorInfo);

    // 自定义添加逻辑
    AActor* Actor = ActorInfo.Actor;
    if (Actor)
    {
        UE_LOG(LogTemp, Verbose, TEXT("Added actor to grid: %s"), *Actor->GetName());
    }
}

void UMyGridSpatializationNode::NotifyRemoveNetworkActor(const FNewReplicatedActorInfo& ActorInfo)
{
    Super::NotifyRemoveNetworkActor(ActorInfo);

    AActor* Actor = ActorInfo.Actor;
    if (Actor)
    {
        UE_LOG(LogTemp, Verbose, TEXT("Removed actor from grid: %s"), *Actor->GetName());
    }
}

void UMyGridSpatializationNode::GatherActorListsForConnection(const FConnectionGatherActorListParameters& Params)
{
    Super::GatherActorListsForConnection(Params);

    // 自定义收集逻辑
    // 可以在这里添加额外的过滤或优先级计算
}

void UMyGridSpatializationNode::SetGridSize(float CellSize)
{
    GridCellSize = CellSize;
}

void UMyGridSpatializationNode::SetSpatialBias(const FVector2D& Bias)
{
    SpatialBias = Bias;
}

void UMyGridSpatializationNode::SetViewDistance(float Distance)
{
    CustomCullDistance = Distance;
}

// PrioritizationNode实现
void UMyPrioritizationNode::GatherActorListsForConnection(const FConnectionGatherActorListParameters& Params)
{
    // 按优先级排序Actor
    Actors.Sort([this, &Params](const FNewReplicatedActorInfo& A, const FNewReplicatedActorInfo& B)
    {
        return CalculatePriority(A, Params) > CalculatePriority(B, Params);
    });

    // 添加到输出
    for (const FNewReplicatedActorInfo& ActorInfo : Actors)
    {
        Params.OutGatheredReplicationLists.AddActor(ActorInfo);
    }
}

void UMyPrioritizationNode::SetPrioritySettings(const TMap<FString, float>& Settings)
{
    PrioritySettings = Settings;
}

float UMyPrioritizationNode::CalculatePriority(const FNewReplicatedActorInfo& ActorInfo, const FConnectionGatherActorListParameters& Params)
{
    float Priority = 1.0f;

    AActor* Actor = ActorInfo.Actor;
    if (!Actor)
    {
        return Priority;
    }

    // 基于距离
    FVector ActorLocation = Actor->GetActorLocation();
    FVector ViewerLocation = Params.ViewerLocation;
    float Distance = FVector::Dist(ActorLocation, ViewerLocation);

    if (Distance < 5000.0f)
    {
        Priority *= 2.0f; // 近距离高优先级
    }
    else if (Distance > 15000.0f)
    {
        Priority *= 0.5f; // 远距离低优先级
    }

    // 基于类型
    FString ClassName = Actor->GetClass()->GetName();
    if (float* TypePriority = PrioritySettings.Find(ClassName))
    {
        Priority *= *TypePriority;
    }

    return Priority;
}

// 主Graph初始化
void UMyReplicationGraph::InitGlobalGraphNodes()
{
    Super::InitGlobalGraphNodes();

    // 创建网格节点
    GridNode = CreateNewNode<UMyGridSpatializationNode>();
    GridNode->SetGridSize(GridCellSize);
    GridNode->SetSpatialBias(FVector2D(SpatialBiasX, SpatialBiasY));
    GridNode->SetViewDistance(ViewDistance);
    AddGlobalGraphNode(GridNode);

    // 创建始终相关节点
    AlwaysRelevantNode = CreateNewNode<UReplicationGraphNode_AlwaysRelevant_AlwaysRelevantNode>();
    AddGlobalGraphNode(AlwaysRelevantNode);

    // 创建玩家状态节点
    PlayerStateNode = CreateNewNode<UReplicationGraphNode_PlayerStateFrequencyLimiter>();
    PlayerStateNode->SetNetUpdateFrequency(PlayerStateUpdateFrequency);
    AddGlobalGraphNode(PlayerStateNode);

    // 创建优先级节点
    PrioritizationNode = CreateNewNode<UMyPrioritizationNode>();
    AddGlobalGraphNode(PrioritizationNode);

    UE_LOG(LogTemp, Log, TEXT("MyReplicationGraph initialized"));
}

void UMyReplicationGraph::InitConnectionGraphNodes(UNetReplicationGraphConnection* Connection)
{
    Super::InitConnectionGraphNodes(Connection);

    // 创建连接专属节点
    UReplicationGraphNode_AlwaysRelevant_ForConnection* ConnectionNode =
        CreateNewNode<UReplicationGraphNode_AlwaysRelevant_ForConnection>();

    AddConnectionGraphNode(ConnectionNode, Connection);
}
```

---

## 三、Critical Classes设计

### 3.1 Critical Classes概念

```cpp
// Critical Classes是需要优先复制的Actor类

// 配置Critical Classes
UCLASS(Config = Engine)
class UMyReplicationGraphSettings : public UDeveloperSettings
{
    GENERATED_BODY()

public:
    // Critical Classes列表
    UPROPERTY(Config, EditAnywhere, Category = "Critical Classes")
    TArray<TSoftClassPtr<AActor>> CriticalClasses;

    // Critical距离
    UPROPERTY(Config, EditAnywhere, Category = "Critical Classes")
    float CriticalDistance = 5000.0f;

    // Critical更新频率倍率
    UPROPERTY(Config, EditAnywhere, Category = "Critical Classes")
    float CriticalUpdateFrequencyMultiplier = 2.0f;
};

// Custom Critical处理
UCLASS()
class UMyCriticalActorNode : public UReplicationGraphNode
{
    GENERATED_BODY()

public:
    virtual void GatherActorListsForConnection(const FConnectionGatherActorListParameters& Params) override;

    void AddCriticalClass(TSubclassOf<AActor> ActorClass);
    void RemoveCriticalClass(TSubclassOf<AActor> ActorClass);

    void SetCriticalDistance(float Distance) { CriticalDistance = Distance; }

private:
    TArray<TSubclassOf<AActor>> CriticalClasses;
    TArray<FNewReplicatedActorInfo> CriticalActors;
    float CriticalDistance = 5000.0f;

    bool IsCriticalActor(const AActor* Actor) const;
    float GetDistanceToViewer(const AActor* Actor, const FVector& ViewerLocation) const;
};

void UMyCriticalActorNode::GatherActorListsForConnection(const FConnectionGatherActorListParameters& Params)
{
    FVector ViewerLocation = Params.ViewerLocation;

    for (const FNewReplicatedActorInfo& ActorInfo : CriticalActors)
    {
        AActor* Actor = ActorInfo.Actor;
        if (!Actor)
        {
            continue;
        }

        // 检查是否是Critical Class
        if (!IsCriticalActor(Actor))
        {
            continue;
        }

        // 检查距离
        float Distance = GetDistanceToViewer(Actor, ViewerLocation);

        if (Distance <= CriticalDistance)
        {
            // 高优先级添加
            Params.OutGatheredReplicationLists.AddActor(ActorInfo);
        }
    }
}

bool UMyCriticalActorNode::IsCriticalActor(const AActor* Actor) const
{
    if (!Actor)
    {
        return false;
    }

    UClass* ActorClass = Actor->GetClass();

    for (TSubclassOf<AActor> CriticalClass : CriticalClasses)
    {
        if (ActorClass->IsChildOf(CriticalClass))
        {
            return true;
        }
    }

    return false;
}

float UMyCriticalActorNode::GetDistanceToViewer(const AActor* Actor, const FVector& ViewerLocation) const
{
    if (!Actor)
    {
        return FLT_MAX;
    }

    return FVector::Dist(Actor->GetActorLocation(), ViewerLocation);
}
```

---

## 四、流式距离优化

### 4.1 动态流式距离

```cpp
// MyStreamingDistanceOptimizer.h
#pragma once

#include "CoreMinimal.h"
#include "MyStreamingDistanceOptimizer.generated.h"

// 流式距离配置
USTRUCT()
struct FStreamingDistanceConfig
{
    GENERATED_BODY()

    float BaseDistance = 15000.0f;
    float MaxDistance = 50000.0f;
    float MinDistance = 5000.0f;

    // 玩家数量影响
    float PlayerCountScale = 0.9f;
    int32 BasePlayerCount = 10;

    // 性能影响
    float FrameTimeThreshold = 33.0f; // ms
    float PerformanceScale = 0.8f;
};

UCLASS()
class UMyStreamingDistanceOptimizer : public UObject
{
    GENERATED_BODY()

public:
    void Initialize(UWorld* World, const FStreamingDistanceConfig& Config);

    // 更新流式距离
    void UpdateStreamingDistance(float DeltaTime);

    // 获取当前推荐距离
    float GetCurrentStreamingDistance() const { return CurrentStreamingDistance; }

    // 配置
    void SetConfig(const FStreamingDistanceConfig& Config) { CurrentConfig = Config; }

private:
    UWorld* TargetWorld;
    FStreamingDistanceConfig CurrentConfig;
    float CurrentStreamingDistance;

    // 性能追踪
    float AverageFrameTime;
    TArray<float> FrameTimeHistory;

    // 计算方法
    float CalculateOptimalDistance();
    float ApplyPlayerCountFactor(float BaseDistance);
    float ApplyPerformanceFactor(float Distance);
    void SmoothTransition(float TargetDistance, float Speed = 100.0f);
};

// MyStreamingDistanceOptimizer.cpp
#include "MyStreamingDistanceOptimizer.h"
#include "Engine/World.h"

void UMyStreamingDistanceOptimizer::Initialize(UWorld* World, const FStreamingDistanceConfig& Config)
{
    TargetWorld = World;
    CurrentConfig = Config;
    CurrentStreamingDistance = Config.BaseDistance;
    AverageFrameTime = 16.67f; // 60fps
}

void UMyStreamingDistanceOptimizer::UpdateStreamingDistance(float DeltaTime)
{
    if (!TargetWorld)
    {
        return;
    }

    // 记录帧时间
    FrameTimeHistory.Add(DeltaTime * 1000.0f);
    if (FrameTimeHistory.Num() > 60)
    {
        FrameTimeHistory.RemoveAt(0);
    }

    // 计算平均帧时间
    float TotalTime = 0.0f;
    for (float Time : FrameTimeHistory)
    {
        TotalTime += Time;
    }
    AverageFrameTime = TotalTime / FrameTimeHistory.Num();

    // 计算最优距离
    float OptimalDistance = CalculateOptimalDistance();

    // 平滑过渡
    SmoothTransition(OptimalDistance);
}

float UMyStreamingDistanceOptimizer::CalculateOptimalDistance()
{
    float BaseDistance = CurrentConfig.BaseDistance;

    // 应用玩家数量因子
    BaseDistance = ApplyPlayerCountFactor(BaseDistance);

    // 应用性能因子
    BaseDistance = ApplyPerformanceFactor(BaseDistance);

    // 限制范围
    return FMath::Clamp(BaseDistance, CurrentConfig.MinDistance, CurrentConfig.MaxDistance);
}

float UMyStreamingDistanceOptimizer::ApplyPlayerCountFactor(float BaseDistance)
{
    if (!TargetWorld)
    {
        return BaseDistance;
    }

    int32 PlayerCount = TargetWorld->GetPlayerControllerIterator().Num();

    if (PlayerCount <= CurrentConfig.BasePlayerCount)
    {
        return BaseDistance;
    }

    // 玩家越多，距离越小
    float Scale = FMath::Pow(CurrentConfig.PlayerCountScale,
                             PlayerCount - CurrentConfig.BasePlayerCount);

    return BaseDistance * Scale;
}

float UMyStreamingDistanceOptimizer::ApplyPerformanceFactor(float Distance)
{
    // 如果帧时间超过阈值，减小距离
    if (AverageFrameTime > CurrentConfig.FrameTimeThreshold)
    {
        float Overhead = AverageFrameTime - CurrentConfig.FrameTimeThreshold;
        float Scale = FMath::Pow(CurrentConfig.PerformanceScale, Overhead / 5.0f);
        return Distance * Scale;
    }

    return Distance;
}

void UMyStreamingDistanceOptimizer::SmoothTransition(float TargetDistance, float Speed)
{
    float Diff = TargetDistance - CurrentStreamingDistance;

    if (FMath::Abs(Diff) < 100.0f)
    {
        CurrentStreamingDistance = TargetDistance;
    }
    else
    {
        CurrentStreamingDistance += FMath::Sign(Diff) * Speed * 0.016f;
    }
}
```

---

## 五、性能分析工具

### 5.1 ReplicationGraph分析器

```cpp
// MyReplicationProfiler.h
#pragma once

#include "CoreMinimal.h"
#include "MyReplicationProfiler.generated.h"

// 复制统计
USTRUCT()
struct FReplicationStats
{
    GENERATED_BODY()

    int32 TotalActors = 0;
    int32 ReplicatedActors = 0;
    int32 CulledActors = 0;

    float TotalReplicationTime = 0.0f;
    float AverageReplicationTime = 0.0f;

    int32 TotalBytesSent = 0;
    int32 TotalPacketsSent = 0;

    TMap<FString, int32> ActorsPerClass;
    TMap<FString, float> ReplicationTimePerClass;
    TMap<FString, int32> BytesPerClass;
};

// 连接统计
USTRUCT()
struct FConnectionStats
{
    GENERATED_BODY()

    FString ConnectionId;
    FString Address;

    float Ping = 0.0f;
    float LastUpdateTime = 0.0f;

    int32 ActorsReplicated = 0;
    int32 BytesSent = 0;
    int32 BytesReceived = 0;

    TArray<FString> RelevantActors;
};

UCLASS()
class UMyReplicationProfiler : public UObject
{
    GENERATED_BODY()

public:
    // 开始/停止分析
    void StartProfiling();
    void StopProfiling();

    // 记录数据
    void RecordActorReplication(const FString& ClassName, float Time, int32 Bytes);
    void RecordConnectionStats(const FString& ConnectionId, const FConnectionStats& Stats);
    void RecordCulledActor(const FString& ClassName);

    // 获取统计
    FReplicationStats GetOverallStats() const;
    TArray<FConnectionStats> GetConnectionStats() const;
    TMap<FString, int32> GetActorsPerClass() const;
    TMap<FString, float> GetReplicationTimePerClass() const;

    // 导出
    FString GenerateReport() const;
    void ExportToCSV(const FString& FilePath);

    // 控制台命令
    static void RegisterConsoleCommands();
    static void UnregisterConsoleCommands();

private:
    bool bIsProfiling = false;
    FReplicationStats OverallStats;
    TArray<FConnectionStats> ConnectionStatsList;
    FDateTime ProfileStartTime;

    void ResetStats();
};

// 控制台命令实现
static FAutoConsoleCommand StartProfilingCommand(
    TEXT("my.replication.StartProfiling"),
    TEXT("Start replication profiling"),
    FConsoleCommandDelegate::CreateStatic([]()
    {
        // UMyReplicationProfiler::Get()->StartProfiling();
    })
);

static FAutoConsoleCommand StopProfilingCommand(
    TEXT("my.replication.StopProfiling"),
    TEXT("Stop replication profiling and generate report"),
    FConsoleCommandDelegate::CreateStatic([]()
    {
        // UMyReplicationProfiler::Get()->StopProfiling();
        // FString Report = UMyReplicationProfiler::Get()->GenerateReport();
        // UE_LOG(LogTemp, Log, TEXT("%s"), *Report);
    })
);

// 生成报告
FString UMyReplicationProfiler::GenerateReport() const
{
    FString Report;

    Report += TEXT("=== Replication Profiling Report ===\n\n");

    Report += FString::Printf(TEXT("Total Actors: %d\n"), OverallStats.TotalActors);
    Report += FString::Printf(TEXT("Replicated Actors: %d\n"), OverallStats.ReplicatedActors);
    Report += FString::Printf(TEXT("Culled Actors: %d\n"), OverallStats.CulledActors);
    Report += FString::Printf(TEXT("Total Replication Time: %.2f ms\n"), OverallStats.TotalReplicationTime);
    Report += FString::Printf(TEXT("Average Replication Time: %.2f ms\n"), OverallStats.AverageReplicationTime);
    Report += FString::Printf(TEXT("Total Bytes Sent: %d\n"), OverallStats.TotalBytesSent);
    Report += FString::Printf(TEXT("Total Packets Sent: %d\n"), OverallStats.TotalPacketsSent);

    Report += TEXT("\n=== Actors Per Class ===\n");
    for (const auto& Pair : OverallStats.ActorsPerClass)
    {
        Report += FString::Printf(TEXT("  %s: %d\n"), *Pair.Key, Pair.Value);
    }

    Report += TEXT("\n=== Replication Time Per Class ===\n");
    for (const auto& Pair : OverallStats.ReplicationTimePerClass)
    {
        Report += FString::Printf(TEXT("  %s: %.2f ms\n"), *Pair.Key, Pair.Value);
    }

    return Report;
}
```

---

## 六、总结

本课我们学习了：

1. **架构深入**：ReplicationGraph核心节点类型
2. **自定义节点**：SpatialGrid和Prioritization
3. **Critical Classes**：优先复制机制
4. **流式距离**：动态优化策略
5. **性能分析**：统计和报告工具

---

## 七、下节预告

**第20课：网络同步高级技巧**

将深入学习：
- 时间同步机制
- 延迟补偿技术
- 防作弊同步验证

---

*课程版本：1.0*
*最后更新：2026-04-10*