# 第35课：结果统计与排行榜

> 学习目标：实现比赛数据统计、排行榜系统和赛季管理功能。

---

## 35.1 统计系统架构

### 35.1.1 数据流设计

```
比赛结束数据流:

[游戏服务器] -> [统计收集] -> [数据验证] -> [数据存储] -> [排行榜更新]
       │              │              │              │              │
       ▼              ▼              ▼              ▼              ▼
    比赛结果       玩家统计        反作弊检查     持久化存储     排名计算
    生成事件       聚合计算        数据清洗       分布式存储     缓存更新
```

---

## 35.2 比赛统计系统

### 35.2.1 统计数据管理器

```csharp
// MatchStatisticsSubsystem.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "MatchStatisticsSubsystem.generated.h"

USTRUCT(BlueprintType)
struct FMatchStatistics
{
    GENERATED_BODY()

    // 基础统计
    UPROPERTY(BlueprintReadOnly)
    int32 Kills = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 Deaths = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 Assists = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 Headshots = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 Score = 0;

    // 伤害统计
    UPROPERTY(BlueprintReadOnly)
    float DamageDealt = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    float DamageTaken = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    float DamageBlocked = 0.0f;

    // 时间统计
    UPROPERTY(BlueprintReadOnly)
    float TimeAlive = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    float TimeInMatch = 0.0f;

    // 武器统计
    UPROPERTY(BlueprintReadOnly)
    int32 ShotsFired = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 ShotsHit = 0;

    UPROPERTY(BlueprintReadOnly)
    float Accuracy = 0.0f;

    // 连杀统计
    UPROPERTY(BlueprintReadOnly)
    int32 HighestKillStreak = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 CurrentKillStreak = 0;

    // 目标统计
    UPROPERTY(BlueprintReadOnly)
    int32 ObjectivesCompleted = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 ObjectivesFailed = 0;

    // 结果
    UPROPERTY(BlueprintReadOnly)
    bool bWon = false;

    UPROPERTY(BlueprintReadOnly)
    int32 TeamId = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 TeamPlacement = 0;

    // 计算
    float GetKDRatio() const;
    float GetKDARatio() const;
    FString GetPerformanceRating() const;
};

USTRUCT(BlueprintType)
struct FWeaponStatistics
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString WeaponId;

    UPROPERTY(BlueprintReadOnly)
    int32 Kills = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 ShotsFired = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 ShotsHit = 0;

    UPROPERTY(BlueprintReadOnly)
    float DamageDealt = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    float Accuracy = 0.0f;
};

USTRUCT(BlueprintType)
struct FMatchResultSummary
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString MatchId;

    UPROPERTY(BlueprintReadOnly)
    FString GameMode;

    UPROPERTY(BlueprintReadOnly)
    FString MapName;

    UPROPERTY(BlueprintReadOnly)
    FDateTime MatchTime;

    UPROPERTY(BlueprintReadOnly)
    int32 MatchDurationSeconds = 0;

    UPROPERTY(BlueprintReadOnly)
    FMatchStatistics PlayerStats;

    UPROPERTY(BlueprintReadOnly)
    TArray<FWeaponStatistics> WeaponStats;

    UPROPERTY(BlueprintReadOnly)
    TArray<FString> TeamMates;

    UPROPERTY(BlueprintReadOnly)
    TArray<FString> Opponents;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnMatchStatsRecorded, const FMatchResultSummary&, Summary);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnPlayerStatsUpdated, const FMatchStatistics&, Stats);

UCLASS()
class MATCHSTATS_API UMatchStatisticsSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 记录比赛
    UFUNCTION(BlueprintCallable, Category = "Statistics")
    void RecordMatchResult(const FMatchResultSummary& Result);

    // 获取历史统计
    UFUNCTION(BlueprintCallable, Category = "Statistics")
    void LoadPlayerStatistics();

    UFUNCTION(BlueprintPure, Category = "Statistics")
    FMatchStatistics GetTotalStatistics() const { return TotalStats; }

    UFUNCTION(BlueprintPure, Category = "Statistics")
    TArray<FMatchResultSummary> GetRecentMatches() const { return RecentMatches; }

    // 统计查询
    UFUNCTION(BlueprintCallable, Category = "Statistics")
    void LoadMatchHistory(int32 Page = 1, int32 PageSize = 10);

    UFUNCTION(BlueprintCallable, Category = "Statistics")
    void LoadWeaponStatistics();

    // 更新统计
    UFUNCTION(BlueprintCallable, Category = "Statistics")
    void UpdateFromMatch(const FMatchStatistics& MatchStats);

    // 计算评级
    UFUNCTION(BlueprintPure, Category = "Statistics")
    FString CalculatePerformanceRating(const FMatchStatistics& Stats) const;

    // 事件
    UPROPERTY(BlueprintAssignable, Category = "Statistics")
    FOnMatchStatsRecorded OnMatchStatsRecorded;

    UPROPERTY(BlueprintAssignable, Category = "Statistics")
    FOnPlayerStatsUpdated OnPlayerStatsUpdated;

private:
    FString ApiBaseUrl;
    FMatchStatistics TotalStats;
    TArray<FMatchResultSummary> RecentMatches;
    TMap<FString, FWeaponStatistics> WeaponStats;

    void HandleStatsResponse(const FString& ResponseJson);
    void HandleMatchHistoryResponse(const FString& ResponseJson);
    FMatchStatistics ParseStatsJson(const TSharedPtr<FJsonObject>& JsonObject);
    FMatchResultSummary ParseMatchSummaryJson(const TSharedPtr<FJsonObject>& JsonObject);
};

// 内联实现
inline float FMatchStatistics::GetKDRatio() const
{
    return Deaths > 0 ? (float)Kills / Deaths : (float)Kills;
}

inline float FMatchStatistics::GetKDARatio() const
{
    return Deaths > 0 ? (float)(Kills + Assists) / Deaths : (float)(Kills + Assists);
}

inline FString FMatchStatistics::GetPerformanceRating() const
{
    float Score = GetKDARatio() * 2.0f + Accuracy * 0.5f + (bWon ? 1.0f : 0.0f);

    if (Score >= 5.0f) return TEXT("S+");
    if (Score >= 4.0f) return TEXT("S");
    if (Score >= 3.5f) return TEXT("A+");
    if (Score >= 3.0f) return TEXT("A");
    if (Score >= 2.5f) return TEXT("B+");
    if (Score >= 2.0f) return TEXT("B");
    if (Score >= 1.5f) return TEXT("C+");
    if (Score >= 1.0f) return TEXT("C");
    return TEXT("D");
}
```

---

## 35.3 排行榜系统

### 35.3.1 排行榜子系统

```csharp
// LeaderboardSubsystem.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"

#include "LeaderboardSubsystem.generated.h"

UENUM(BlueprintType)
enum class ELeaderboardType : uint8
{
    Global,
    Regional,
    Friends,
    Weekly,
    Monthly,
    AllTime
};

UENUM(BlueprintType)
enum class ELeaderboardCategory : uint8
{
    Rating,
    Wins,
    Kills,
    KDRatio,
    Score,
    PlayTime
};

USTRUCT(BlueprintType)
struct FLeaderboardEntry
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    int32 Rank = 0;

    UPROPERTY(BlueprintReadOnly)
    FString PlayerId;

    UPROPERTY(BlueprintReadOnly)
    FString DisplayName;

    UPROPERTY(BlueprintReadOnly)
    FString AvatarUrl;

    UPROPERTY(BlueprintReadOnly)
    float Score = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    FString RankTier;

    UPROPERTY(BlueprintReadOnly)
    int32 Wins = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 Losses = 0;

    UPROPERTY(BlueprintReadOnly)
    bool bIsCurrentUser = false;
};

USTRUCT(BlueprintType)
struct FSeasonInfo
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString SeasonId;

    UPROPERTY(BlueprintReadOnly)
    FString SeasonName;

    UPROPERTY(BlueprintReadOnly)
    FDateTime StartTime;

    UPROPERTY(BlueprintReadOnly)
    FDateTime EndTime;

    UPROPERTY(BlueprintReadOnly)
    bool bIsActive = false;

    UPROPERTY(BlueprintReadOnly)
    TArray<FString> RankTiers;
};

USTRUCT(BlueprintType)
struct FPlayerRankInfo
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    FString RankTier;

    UPROPERTY(BlueprintReadOnly)
    int32 RankPoints = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 Wins = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 Losses = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 GlobalRank = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 RegionalRank = 0;

    UPROPERTY(BlueprintReadOnly)
    FString HighestRankTier;

    UPROPERTY(BlueprintReadOnly)
    int32 PointsToNextTier = 0;

    UPROPERTY(BlueprintReadOnly)
    int32 PointsToDemotion = 0;
};

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnLeaderboardLoaded, ELeaderboardType, Type, const TArray<FLeaderboardEntry>&, Entries);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnRankInfoUpdated, const FPlayerRankInfo&, Info);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnSeasonInfoLoaded, const FSeasonInfo&, Season);

UCLASS()
class LEADERBOARD_API ULeaderboardSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    // 排行榜查询
    UFUNCTION(BlueprintCallable, Category = "Leaderboard")
    void LoadLeaderboard(ELeaderboardType Type, ELeaderboardCategory Category, int32 Limit = 100, int32 Offset = 0);

    UFUNCTION(BlueprintCallable, Category = "Leaderboard")
    void LoadLeaderboardAroundPlayer(ELeaderboardCategory Category, int32 Range = 10);

    UFUNCTION(BlueprintCallable, Category = "Leaderboard")
    void LoadFriendsLeaderboard(ELeaderboardCategory Category);

    // 玩家排名
    UFUNCTION(BlueprintCallable, Category = "Leaderboard")
    void LoadPlayerRankInfo();

    UFUNCTION(BlueprintPure, Category = "Leaderboard")
    FPlayerRankInfo GetPlayerRankInfo() const { return CurrentRankInfo; }

    // 赛季
    UFUNCTION(BlueprintCallable, Category = "Leaderboard")
    void LoadCurrentSeason();

    UFUNCTION(BlueprintCallable, Category = "Leaderboard")
    void LoadSeasonHistory();

    UFUNCTION(BlueprintPure, Category = "Leaderboard")
    FSeasonInfo GetCurrentSeason() const { return CurrentSeason; }

    // 缓存
    UFUNCTION(BlueprintPure, Category = "Leaderboard")
    TArray<FLeaderboardEntry> GetCachedLeaderboard(ELeaderboardType Type) const;

    // 更新
    UFUNCTION(BlueprintCallable, Category = "Leaderboard")
    void UpdateRankPoints(int32 PointsChange, bool bWon);

    // 事件
    UPROPERTY(BlueprintAssignable, Category = "Leaderboard")
    FOnLeaderboardLoaded OnLeaderboardLoaded;

    UPROPERTY(BlueprintAssignable, Category = "Leaderboard")
    FOnRankInfoUpdated OnRankInfoUpdated;

    UPROPERTY(BlueprintAssignable, Category = "Leaderboard")
    FOnSeasonInfoLoaded OnSeasonInfoLoaded;

private:
    FString ApiBaseUrl;
    FPlayerRankInfo CurrentRankInfo;
    FSeasonInfo CurrentSeason;
    TMap<ELeaderboardType, TArray<FLeaderboardEntry>> CachedLeaderboards;

    void HandleLeaderboardResponse(const FString& ResponseJson, ELeaderboardType Type);
    void HandleRankInfoResponse(const FString& ResponseJson);
    void HandleSeasonInfoResponse(const FString& ResponseJson);

    TArray<FLeaderboardEntry> ParseLeaderboardJson(const TArray<TSharedPtr<FJsonValue>>& JsonArray);
    FLeaderboardEntry ParseLeaderboardEntry(const TSharedPtr<FJsonObject>& JsonObject);
    FPlayerRankInfo ParseRankInfoJson(const TSharedPtr<FJsonObject>& JsonObject);
    FSeasonInfo ParseSeasonInfoJson(const TSharedPtr<FJsonObject>& JsonObject);

    FString GetRankTierFromPoints(int32 Points) const;
    int32 GetPointsForTier(const FString& Tier) const;
};
```

```csharp
// LeaderboardSubsystem.cpp
#include "LeaderboardSubsystem.h"
#include "JsonUtilities.h"
#include "HttpModule.h"

void ULeaderboardSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
    ApiBaseUrl = "http://localhost:8080/api";
}

void ULeaderboardSubsystem::LoadLeaderboard(ELeaderboardType Type, ELeaderboardCategory Category, int32 Limit, int32 Offset)
{
    FString TypeStr;
    switch (Type)
    {
        case ELeaderboardType::Global: TypeStr = TEXT("global"); break;
        case ELeaderboardType::Regional: TypeStr = TEXT("regional"); break;
        case ELeaderboardType::Friends: TypeStr = TEXT("friends"); break;
        case ELeaderboardType::Weekly: TypeStr = TEXT("weekly"); break;
        case ELeaderboardType::Monthly: TypeStr = TEXT("monthly"); break;
        case ELeaderboardType::AllTime: TypeStr = TEXT("all_time"); break;
    }

    FString CategoryStr;
    switch (Category)
    {
        case ELeaderboardCategory::Rating: CategoryStr = TEXT("rating"); break;
        case ELeaderboardCategory::Wins: CategoryStr = TEXT("wins"); break;
        case ELeaderboardCategory::Kills: CategoryStr = TEXT("kills"); break;
        case ELeaderboardCategory::KDRatio: CategoryStr = TEXT("kd"); break;
        case ELeaderboardCategory::Score: CategoryStr = TEXT("score"); break;
        case ELeaderboardCategory::PlayTime: CategoryStr = TEXT("playtime"); break;
    }

    FString Url = FString::Printf(TEXT("%s/leaderboard/%s/%s?limit=%d&offset=%d"),
        *ApiBaseUrl, *TypeStr, *CategoryStr, Limit, Offset);

    TSharedRef<IHttpRequest> Request = FHttpModule::Get().CreateRequest();
    Request->SetURL(Url);
    Request->SetVerb("GET");
    Request->SetHeader("Authorization", "Bearer " + GetAccessToken());

    Request->OnProcessRequestComplete().BindLambda([this, Type](FHttpRequestPtr Req, FHttpResponsePtr Resp, bool bSuccess)
    {
        if (bSuccess && Resp.IsValid() && Resp->GetResponseCode() == 200)
        {
            HandleLeaderboardResponse(Resp->GetContentAsString(), Type);
        }
    });

    Request->ProcessRequest();
}

void ULeaderboardSubsystem::LoadPlayerRankInfo()
{
    TSharedRef<IHttpRequest> Request = FHttpModule::Get().CreateRequest();
    Request->SetURL(ApiBaseUrl + "/rank/me");
    Request->SetVerb("GET");
    Request->SetHeader("Authorization", "Bearer " + GetAccessToken());

    Request->OnProcessRequestComplete().BindLambda([this](FHttpRequestPtr Req, FHttpResponsePtr Resp, bool bSuccess)
    {
        if (bSuccess && Resp.IsValid() && Resp->GetResponseCode() == 200)
        {
            HandleRankInfoResponse(Resp->GetContentAsString());
        }
    });

    Request->ProcessRequest();
}

void ULeaderboardSubsystem::HandleLeaderboardResponse(const FString& ResponseJson, ELeaderboardType Type)
{
    TArray<TSharedPtr<FJsonValue>> JsonArray;
    TJsonReaderFactory<TCHAR>::Create(ResponseJson) >> JsonArray;

    TArray<FLeaderboardEntry> Entries = ParseLeaderboardJson(JsonArray);
    CachedLeaderboards.Add(Type, Entries);

    OnLeaderboardLoaded.Broadcast(Type, Entries);
}

void ULeaderboardSubsystem::HandleRankInfoResponse(const FString& ResponseJson)
{
    TSharedPtr<FJsonObject> JsonObject;
    TJsonReaderFactory<TCHAR>::Create(ResponseJson) >> JsonObject;

    if (JsonObject.IsValid())
    {
        CurrentRankInfo = ParseRankInfoJson(JsonObject);
        OnRankInfoUpdated.Broadcast(CurrentRankInfo);
    }
}

TArray<FLeaderboardEntry> ULeaderboardSubsystem::ParseLeaderboardJson(const TArray<TSharedPtr<FJsonValue>>& JsonArray)
{
    TArray<FLeaderboardEntry> Entries;

    int32 Rank = 1;
    for (const auto& Value : JsonArray)
    {
        TSharedPtr<FJsonObject> Obj = Value->AsObject();
        if (Obj.IsValid())
        {
            FLeaderboardEntry Entry = ParseLeaderboardEntry(Obj);
            Entry.Rank = Rank++;
            Entries.Add(Entry);
        }
    }

    return Entries;
}

FLeaderboardEntry ULeaderboardSubsystem::ParseLeaderboardEntry(const TSharedPtr<FJsonObject>& JsonObject)
{
    FLeaderboardEntry Entry;

    Entry.PlayerId = JsonObject->GetStringField("playerId");
    Entry.DisplayName = JsonObject->GetStringField("displayName");
    Entry.Score = JsonObject->GetNumberField("score");

    if (JsonObject->HasField("avatarUrl"))
    {
        Entry.AvatarUrl = JsonObject->GetStringField("avatarUrl");
    }

    if (JsonObject->HasField("rankTier"))
    {
        Entry.RankTier = JsonObject->GetStringField("rankTier");
    }

    if (JsonObject->HasField("wins"))
    {
        Entry.Wins = JsonObject->GetNumberField("wins");
    }

    if (JsonObject->HasField("losses"))
    {
        Entry.Losses = JsonObject->GetNumberField("losses");
    }

    // 检查是否是当前用户
    // Entry.bIsCurrentUser = (Entry.PlayerId == GetCurrentUserId());

    return Entry;
}

FPlayerRankInfo ULeaderboardSubsystem::ParseRankInfoJson(const TSharedPtr<FJsonObject>& JsonObject)
{
    FPlayerRankInfo Info;

    Info.RankTier = JsonObject->GetStringField("rankTier");
    Info.RankPoints = JsonObject->GetNumberField("rankPoints");
    Info.Wins = JsonObject->GetNumberField("wins");
    Info.Losses = JsonObject->GetNumberField("losses");

    if (JsonObject->HasField("globalRank"))
    {
        Info.GlobalRank = JsonObject->GetNumberField("globalRank");
    }

    if (JsonObject->HasField("regionalRank"))
    {
        Info.RegionalRank = JsonObject->GetNumberField("regionalRank");
    }

    if (JsonObject->HasField("highestRankTier"))
    {
        Info.HighestRankTier = JsonObject->GetStringField("highestRankTier");
    }

    // 计算距离下一段位/降级的点数
    // Info.PointsToNextTier = CalculatePointsToNextTier(Info.RankTier, Info.RankPoints);
    // Info.PointsToDemotion = CalculatePointsToDemotion(Info.RankTier, Info.RankPoints);

    return Info;
}

FString ULeaderboardSubsystem::GetRankTierFromPoints(int32 Points) const
{
    // 段位划分
    if (Points >= 3000) return TEXT("Champion");
    if (Points >= 2600) return TEXT("Diamond");
    if (Points >= 2200) return TEXT("Platinum");
    if (Points >= 1800) return TEXT("Gold");
    if (Points >= 1400) return TEXT("Silver");
    if (Points >= 1000) return TEXT("Bronze");
    return TEXT("Unranked");
}

int32 ULeaderboardSubsystem::GetPointsForTier(const FString& Tier) const
{
    if (Tier == TEXT("Champion")) return 3000;
    if (Tier == TEXT("Diamond")) return 2600;
    if (Tier == TEXT("Platinum")) return 2200;
    if (Tier == TEXT("Gold")) return 1800;
    if (Tier == TEXT("Silver")) return 1400;
    if (Tier == TEXT("Bronze")) return 1000;
    return 0;
}

void ULeaderboardSubsystem::UpdateRankPoints(int32 PointsChange, bool bWon)
{
    CurrentRankInfo.RankPoints += PointsChange;

    if (bWon)
    {
        CurrentRankInfo.Wins++;
    }
    else
    {
        CurrentRankInfo.Losses++;
    }

    // 更新段位
    FString NewTier = GetRankTierFromPoints(CurrentRankInfo.RankPoints);

    if (NewTier != CurrentRankInfo.RankTier)
    {
        // 段位变化
        CurrentRankInfo.RankTier = NewTier;

        // 记录最高段位
        int32 NewTierPoints = GetPointsForTier(NewTier);
        int32 HighestTierPoints = GetPointsForTier(CurrentRankInfo.HighestRankTier);

        if (NewTierPoints > HighestTierPoints)
        {
            CurrentRankInfo.HighestRankTier = NewTier;
        }
    }

    OnRankInfoUpdated.Broadcast(CurrentRankInfo);
}

TArray<FLeaderboardEntry> ULeaderboardSubsystem::GetCachedLeaderboard(ELeaderboardType Type) const
{
    const TArray<FLeaderboardEntry>* Cached = CachedLeaderboards.Find(Type);
    return Cached ? *Cached : TArray<FLeaderboardEntry>();
}
```

---

## 35.4 后端排行榜服务

### 35.4.1 排行榜服务实现

```javascript
// leaderboard-service/src/services/LeaderboardService.js
const Redis = require('ioredis');
const database = require('../database');

class LeaderboardService {
    constructor() {
        this.redis = new Redis();
        this.rankTiers = [
            { name: 'Champion', minPoints: 3000 },
            { name: 'Diamond', minPoints: 2600 },
            { name: 'Platinum', minPoints: 2200 },
            { name: 'Gold', minPoints: 1800 },
            { name: 'Silver', minPoints: 1400 },
            { name: 'Bronze', minPoints: 1000 },
            { name: 'Unranked', minPoints: 0 }
        ];
    }

    // 更新玩家排名
    async updatePlayerRank(userId, pointsChange, won) {
        // 获取当前排名
        const currentRank = await database.query(
            'SELECT rank_points, wins, losses, highest_rank_tier FROM player_ranks WHERE user_id = $1',
            [userId]
        );

        let rankPoints = currentRank.rows[0]?.rank_points || 1000;
        let wins = currentRank.rows[0]?.wins || 0;
        let losses = currentRank.rows[0]?.losses || 0;
        let highestTier = currentRank.rows[0]?.highest_rank_tier || 'Unranked';

        // 更新点数
        rankPoints += pointsChange;
        rankPoints = Math.max(0, rankPoints); // 不能为负

        if (won) {
            wins++;
        } else {
            losses++;
        }

        // 计算段位
        const newTier = this.getRankTier(rankPoints);

        // 更新最高段位
        const newTierPoints = this.getTierPoints(newTier);
        const highestTierPoints = this.getTierPoints(highestTier);
        if (newTierPoints > highestTierPoints) {
            highestTier = newTier;
        }

        // 保存到数据库
        await database.query(
            `INSERT INTO player_ranks (user_id, season_id, rank_points, rank_tier, wins, losses, highest_rank_tier, updated_at)
             VALUES ($1, $2, $3, $4, $5, $6, $7, NOW())
             ON CONFLICT (user_id, season_id)
             DO UPDATE SET rank_points = $3, rank_tier = $4, wins = $5, losses = $6,
                           highest_rank_tier = $7, updated_at = NOW()`,
            [userId, await this.getCurrentSeasonId(), rankPoints, newTier, wins, losses, highestTier]
        );

        // 更新Redis排行榜
        await this.redis.zadd('leaderboard:global', rankPoints, userId);
        await this.redis.zadd(`leaderboard:regional:${await this.getPlayerRegion(userId)}`, rankPoints, userId);

        return {
            rankPoints,
            rankTier: newTier,
            wins,
            losses,
            highestRankTier: highestTier
        };
    }

    // 获取排行榜
    async getLeaderboard(type, category, limit, offset, region = null) {
        let key = `leaderboard:${type}`;

        if (type === 'regional' && region) {
            key = `leaderboard:regional:${region}`;
        }

        // 从Redis获取排名
        const entries = await this.redis.zrevrange(key, offset, offset + limit - 1, 'WITHSCORES');

        const leaderboard = [];
        for (let i = 0; i < entries.length; i += 2) {
            const userId = entries[i];
            const score = parseInt(entries[i + 1]);

            // 获取玩家信息
            const playerInfo = await this.getPlayerInfo(userId);

            leaderboard.push({
                rank: offset + (i / 2) + 1,
                playerId: userId,
                displayName: playerInfo.display_name,
                avatarUrl: playerInfo.avatar_url,
                score: score,
                rankTier: this.getRankTier(score),
                wins: playerInfo.wins,
                losses: playerInfo.losses
            });
        }

        return leaderboard;
    }

    // 获取玩家排名
    async getPlayerRank(userId, region = null) {
        // 全球排名
        const globalRank = await this.redis.zrevrank('leaderboard:global', userId);
        const globalScore = await this.redis.zscore('leaderboard:global', userId);

        // 区域排名
        let regionalRank = null;
        if (region) {
            regionalRank = await this.redis.zrevrank(`leaderboard:regional:${region}`, userId);
        }

        // 从数据库获取详细信息
        const result = await database.query(
            `SELECT rank_points, rank_tier, wins, losses, highest_rank_tier
             FROM player_ranks
             WHERE user_id = $1 AND season_id = $2`,
            [userId, await this.getCurrentSeasonId()]
        );

        const rankData = result.rows[0] || {
            rank_points: 1000,
            rank_tier: 'Unranked',
            wins: 0,
            losses: 0,
            highest_rank_tier: 'Unranked'
        };

        return {
            rankPoints: rankData.rank_points,
            rankTier: rankData.rank_tier,
            wins: rankData.wins,
            losses: rankData.losses,
            globalRank: globalRank !== null ? globalRank + 1 : null,
            regionalRank: regionalRank !== null ? regionalRank + 1 : null,
            highestRankTier: rankData.highest_rank_tier
        };
    }

    // 辅助方法
    getRankTier(points) {
        for (const tier of this.rankTiers) {
            if (points >= tier.minPoints) {
                return tier.name;
            }
        }
        return 'Unranked';
    }

    getTierPoints(tierName) {
        const tier = this.rankTiers.find(t => t.name === tierName);
        return tier ? tier.minPoints : 0;
    }

    async getCurrentSeasonId() {
        const result = await database.query(
            "SELECT id FROM seasons WHERE is_active = true ORDER BY start_time DESC LIMIT 1"
        );
        return result.rows[0]?.id || 'default';
    }

    async getPlayerRegion(userId) {
        const result = await database.query(
            'SELECT region FROM users WHERE id = $1',
            [userId]
        );
        return result.rows[0]?.region || 'us-east';
    }

    async getPlayerInfo(userId) {
        const result = await database.query(
            `SELECT u.display_name, u.avatar_url,
                    COALESCE(pr.wins, 0) as wins, COALESCE(pr.losses, 0) as losses
             FROM users u
             LEFT JOIN player_ranks pr ON u.id = pr.user_id AND pr.season_id = $2
             WHERE u.id = $1`,
            [userId, await this.getCurrentSeasonId()]
        );
        return result.rows[0] || { display_name: 'Unknown', avatar_url: null, wins: 0, losses: 0 };
    }

    // 记录比赛结果
    async recordMatchResult(matchId, results) {
        for (const playerResult of results) {
            // 计算点数变化
            const pointsChange = this.calculatePointsChange(playerResult);

            // 更新排名
            await this.updatePlayerRank(
                playerResult.playerId,
                pointsChange,
                playerResult.isWinner
            );

            // 更新统计
            await this.updatePlayerStats(playerResult);
        }

        // 记录比赛到历史
        await database.query(
            `INSERT INTO match_history (match_id, results, created_at)
             VALUES ($1, $2, NOW())`,
            [matchId, JSON.stringify(results)]
        );
    }

    calculatePointsChange(result) {
        let change = 0;

        // 基础变化
        if (result.isWinner) {
            change += 25;
        } else {
            change -= 15;
        }

        // 根据表现调整
        const kda = result.deaths > 0 ? (result.kills + result.assists) / result.deaths : result.kills + result.assists;

        if (kda > 3.0) {
            change += 5;  // 表现优异
        } else if (kda < 1.0) {
            change -= 5;  // 表现不佳
        }

        return change;
    }

    async updatePlayerStats(result) {
        await database.query(
            `INSERT INTO player_stats (user_id, total_matches, wins, losses, kills, deaths, assists, updated_at)
             VALUES ($1, 1, $2, $3, $4, $5, $6, NOW())
             ON CONFLICT (user_id)
             DO UPDATE SET
                 total_matches = player_stats.total_matches + 1,
                 wins = player_stats.wins + $2,
                 losses = player_stats.losses + $3,
                 kills = player_stats.kills + $4,
                 deaths = player_stats.deaths + $5,
                 assists = player_stats.assists + $6,
                 updated_at = NOW()`,
            [result.playerId, result.isWinner ? 1 : 0, result.isWinner ? 0 : 1,
             result.kills, result.deaths, result.assists]
        );
    }
}

module.exports = new LeaderboardService();
```

---

## 35.5 实践任务

### 任务1：实现统计界面

创建比赛统计UI：
- 比赛结果展示
- 玩家表现评分
- 历史战绩查看

### 任务2：实现排行榜UI

创建排行榜界面：
- 全球/区域排行榜
- 好友排行榜
- 玩家排名卡片

### 任务3：实现段位系统

完成段位系统：
- 段位图标显示
- 段位变化动画
- 晋级/降级提示

---

## 35.6 总结

本课完成了：
- 比赛统计系统设计
- 统计数据管理器
- 排行榜子系统
- 后端排行榜服务
- 段位计算系统

**下一课预告**：微服务架构搭建 - 完整的企业级后端服务架构。

---

*课程版本：1.0*
*最后更新：2026-04-10*
