# 第38课：缓存层设计

> 学习目标：掌握Redis缓存策略、数据一致性保证和缓存高可用设计。

---

## 38.1 缓存架构设计

### 38.1.1 缓存层次结构

```
缓存架构:

┌─────────────────────────────────────────────────────────┐
│                      客户端                              │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                   CDN / 边缘缓存                         │
│              (静态资源、API响应缓存)                      │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                   应用层缓存                             │
│              (本地内存缓存 L1)                           │
│              - 热点数据                                  │
│              - 会话数据                                  │
│              - 频繁访问的小数据                          │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                   分布式缓存                             │
│              (Redis集群 L2)                             │
│              - 用户数据                                  │
│              - 排行榜                                    │
│              - 会话存储                                  │
│              - 匹配队列                                  │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                   数据库                                 │
│              (PostgreSQL持久化)                         │
└─────────────────────────────────────────────────────────┘
```

---

## 38.2 Redis缓存管理器

### 38.2.1 缓存服务实现

```javascript
// cache/CacheManager.js
const Redis = require('ioredis');

class CacheManager {
    constructor() {
        // Redis集群配置
        const nodes = (process.env.REDIS_CLUSTER_NODES || 'localhost:6379')
            .split(',')
            .map(node => {
                const [host, port] = node.split(':');
                return { host, port: parseInt(port) || 6379 };
            });

        // 创建Redis客户端
        if (nodes.length > 1) {
            this.redis = new Redis.Cluster(nodes, {
                redisOptions: {
                    password: process.env.REDIS_PASSWORD,
                    enableReadyCheck: true,
                    maxRedirections: 16
                }
            });
        } else {
            this.redis = new Redis({
                host: nodes[0].host,
                port: nodes[0].port,
                password: process.env.REDIS_PASSWORD,
                enableReadyCheck: true
            });
        }

        // 本地缓存（L1）
        this.localCache = new Map();
        this.localCacheMaxSize = 10000;
        this.localCacheTTL = 60000; // 60秒

        // 缓存键前缀
        this.keyPrefix = process.env.CACHE_KEY_PREFIX || 'arena:';

        // 统计
        this.stats = {
            hits: 0,
            misses: 0,
            localHits: 0,
            sets: 0,
            deletes: 0
        };
    }

    // 生成缓存键
    buildKey(key) {
        return `${this.keyPrefix}${key}`;
    }

    // 获取缓存
    async get(key, options = {}) {
        const fullKey = this.buildKey(key);

        // 先检查本地缓存
        if (!options.skipLocal) {
            const localData = this.getLocalCache(fullKey);
            if (localData !== null) {
                this.stats.hits++;
                this.stats.localHits++;
                return localData;
            }
        }

        // 从Redis获取
        try {
            const data = await this.redis.get(fullKey);

            if (data !== null) {
                const parsed = JSON.parse(data);

                // 存入本地缓存
                if (!options.skipLocal && parsed !== null) {
                    this.setLocalCache(fullKey, parsed, options.localTTL || this.localCacheTTL);
                }

                this.stats.hits++;
                return parsed;
            }

            this.stats.misses++;
            return null;

        } catch (error) {
            console.error(`Cache get error for key ${key}:`, error);
            return null;
        }
    }

    // 设置缓存
    async set(key, value, ttl = 3600, options = {}) {
        const fullKey = this.buildKey(key);

        try {
            const serialized = JSON.stringify(value);

            // 存入Redis
            if (ttl > 0) {
                await this.redis.setex(fullKey, ttl, serialized);
            } else {
                await this.redis.set(fullKey, serialized);
            }

            // 存入本地缓存
            if (!options.skipLocal) {
                this.setLocalCache(fullKey, value, Math.min(ttl * 1000, this.localCacheTTL));
            }

            this.stats.sets++;
            return true;

        } catch (error) {
            console.error(`Cache set error for key ${key}:`, error);
            return false;
        }
    }

    // 删除缓存
    async delete(key) {
        const fullKey = this.buildKey(key);

        try {
            await this.redis.del(fullKey);
            this.deleteLocalCache(fullKey);
            this.stats.deletes++;
            return true;

        } catch (error) {
            console.error(`Cache delete error for key ${key}:`, error);
            return false;
        }
    }

    // 批量删除
    async deletePattern(pattern) {
        const fullPattern = this.buildKey(pattern);

        try {
            const keys = await this.scanKeys(fullPattern);

            if (keys.length > 0) {
                await this.redis.del(...keys);

                // 清理本地缓存
                for (const key of keys) {
                    this.deleteLocalCache(key);
                }
            }

            return keys.length;

        } catch (error) {
            console.error(`Cache delete pattern error for ${pattern}:`, error);
            return 0;
        }
    }

    // 扫描键
    async scanKeys(pattern, count = 100) {
        const keys = [];
        let cursor = '0';

        do {
            const result = await this.redis.scan(cursor, 'MATCH', pattern, 'COUNT', count);
            cursor = result[0];
            keys.push(...result[1]);
        } while (cursor !== '0');

        return keys;
    }

    // 获取或设置（Cache-Aside模式）
    async getOrSet(key, fetcher, ttl = 3600, options = {}) {
        // 尝试获取缓存
        const cached = await this.get(key, options);

        if (cached !== null) {
            return cached;
        }

        // 缓存未命中，调用fetcher
        const value = await fetcher();

        // 存入缓存
        if (value !== null && value !== undefined) {
            await this.set(key, value, ttl, options);
        }

        return value;
    }

    // 本地缓存操作
    getLocalCache(key) {
        const item = this.localCache.get(key);

        if (!item) {
            return null;
        }

        if (Date.now() > item.expiry) {
            this.localCache.delete(key);
            return null;
        }

        return item.value;
    }

    setLocalCache(key, value, ttl) {
        // 检查容量
        if (this.localCache.size >= this.localCacheMaxSize) {
            // 删除最旧的条目
            const oldestKey = this.localCache.keys().next().value;
            this.localCache.delete(oldestKey);
        }

        this.localCache.set(key, {
            value,
            expiry: Date.now() + ttl
        });
    }

    deleteLocalCache(key) {
        this.localCache.delete(key);
    }

    // 清空本地缓存
    clearLocalCache() {
        this.localCache.clear();
    }

    // 获取统计信息
    getStats() {
        const total = this.stats.hits + this.stats.misses;
        const hitRate = total > 0 ? (this.stats.hits / total * 100).toFixed(2) : 0;

        return {
            ...this.stats,
            hitRate: `${hitRate}%`,
            localCacheSize: this.localCache.size
        };
    }

    // 重置统计
    resetStats() {
        this.stats = {
            hits: 0,
            misses: 0,
            localHits: 0,
            sets: 0,
            deletes: 0
        };
    }
}

module.exports = new CacheManager();
```

### 38.2.2 缓存装饰器

```javascript
// cache/CacheDecorator.js
const cache = require('./CacheManager');

// 方法缓存装饰器
function cached(keyTemplate, ttl = 3600, options = {}) {
    return function (target, propertyKey, descriptor) {
        const originalMethod = descriptor.value;

        descriptor.value = async function (...args) {
            // 构建缓存键
            const key = typeof keyTemplate === 'function'
                ? keyTemplate(...args)
                : keyTemplate;

            // 尝试从缓存获取
            return cache.getOrSet(
                key,
                () => originalMethod.apply(this, args),
                ttl,
                options
            );
        };

        return descriptor;
    };
}

// 类方法缓存
class CachedRepository {
    constructor(repository, keyPrefix, ttl = 3600) {
        this.repository = repository;
        this.keyPrefix = keyPrefix;
        this.ttl = ttl;
    }

    async findById(id) {
        const key = `${this.keyPrefix}:id:${id}`;
        return cache.getOrSet(key, () => this.repository.findById(id), this.ttl);
    }

    async findMany(ids) {
        // 批量获取，合并缓存命中和未命中
        const results = new Map();
        const missed = [];

        // 检查缓存
        for (const id of ids) {
            const key = `${this.keyPrefix}:id:${id}`;
            const cached = await cache.get(key);

            if (cached !== null) {
                results.set(id, cached);
            } else {
                missed.push(id);
            }
        }

        // 获取未命中的数据
        if (missed.length > 0) {
            const fetched = await this.repository.findMany(missed);

            for (const item of fetched) {
                results.set(item.id, item);

                // 存入缓存
                const key = `${this.keyPrefix}:id:${item.id}`;
                await cache.set(key, item, this.ttl);
            }
        }

        return ids.map(id => results.get(id)).filter(Boolean);
    }

    async update(id, data) {
        // 更新数据库
        const result = await this.repository.update(id, data);

        // 使缓存失效
        await cache.delete(`${this.keyPrefix}:id:${id}`);

        return result;
    }

    async delete(id) {
        // 删除数据库记录
        await this.repository.delete(id);

        // 删除缓存
        await cache.delete(`${this.keyPrefix}:id:${id}`);
    }
}

module.exports = { cached, CachedRepository };
```

---

## 38.3 缓存策略实现

### 38.3.1 数据一致性策略

```javascript
// cache/CacheStrategies.js
const cache = require('./CacheManager');

class CacheStrategies {
    // Cache-Aside Pattern（旁路缓存）
    static async cacheAside(key, fetcher, ttl = 3600) {
        // 先读缓存
        let data = await cache.get(key);

        if (data === null) {
            // 缓存未命中，从数据库获取
            data = await fetcher();

            // 写入缓存
            if (data !== null) {
                await cache.set(key, data, ttl);
            }
        }

        return data;
    }

    // Write-Through Pattern（写穿透）
    static async writeThrough(key, data, writer, ttl = 3600) {
        // 先更新数据库
        const result = await writer(data);

        // 同时更新缓存
        await cache.set(key, result, ttl);

        return result;
    }

    // Write-Behind Pattern（写回）
    static async writeBehind(key, data, writer, ttl = 3600) {
        // 先更新缓存
        await cache.set(key, data, ttl);

        // 异步更新数据库
        setImmediate(async () => {
            try {
                await writer(data);
            } catch (error) {
                console.error('Write-behind failed:', error);
                // 可以加入重试队列
            }
        });

        return data;
    }

    // Read-Through Pattern（读穿透）
    static async readThrough(key, fetcher, ttl = 3600) {
        return cache.getOrSet(key, fetcher, ttl);
    }

    // Refresh-Ahead Pattern（预刷新）
    static async refreshAhead(key, fetcher, ttl = 3600, refreshThreshold = 0.8) {
        const data = await cache.get(key);

        if (data !== null) {
            // 检查TTL，如果接近过期则异步刷新
            const ttlRemaining = await cache.redis.ttl(cache.buildKey(key));

            if (ttlRemaining > 0 && ttlRemaining < ttl * (1 - refreshThreshold)) {
                // 异步刷新
                setImmediate(async () => {
                    try {
                        const freshData = await fetcher();
                        await cache.set(key, freshData, ttl);
                    } catch (error) {
                        console.error('Refresh-ahead failed:', error);
                    }
                });
            }
        }

        return data || cache.getOrSet(key, fetcher, ttl);
    }
}

// 失效策略
class InvalidationStrategies {
    // 使单个键失效
    static async invalidate(key) {
        await cache.delete(key);
    }

    // 使相关键失效
    static async invalidateRelated(baseKey, relatedKeys = []) {
        const keys = [baseKey, ...relatedKeys];

        for (const key of keys) {
            await cache.delete(key);
        }
    }

    // 使模式匹配的键失效
    static async invalidatePattern(pattern) {
        await cache.deletePattern(pattern);
    }

    // 使标签相关的键失效
    static async invalidateByTags(tags) {
        for (const tag of tags) {
            const keys = await cache.redis.smembers(`tag:${tag}`);

            if (keys.length > 0) {
                await cache.redis.del(...keys);
                await cache.redis.del(`tag:${tag}`);
            }
        }
    }

    // 添加键到标签
    static async addToTag(tag, key) {
        await cache.redis.sadd(`tag:${tag}`, key);
    }
}

module.exports = { CacheStrategies, InvalidationStrategies };
```

### 38.3.2 分布式锁

```javascript
// cache/DistributedLock.js
const Redis = require('ioredis');
const { v4: uuidv4 } = require('uuid');

class DistributedLock {
    constructor() {
        this.redis = new Redis();
        this.locks = new Map();
    }

    // 获取锁
    async acquire(lockKey, ttlMs = 10000, maxRetries = 10, retryDelayMs = 100) {
        const lockId = uuidv4();
        const key = `lock:${lockKey}`;

        for (let attempt = 0; attempt < maxRetries; attempt++) {
            // 尝试设置锁
            const result = await this.redis.set(key, lockId, 'PX', ttlMs, 'NX');

            if (result === 'OK') {
                // 记录锁信息
                this.locks.set(key, {
                    lockId,
                    acquiredAt: Date.now(),
                    ttl: ttlMs
                });

                return {
                    success: true,
                    lockId,
                    key
                };
            }

            // 等待后重试
            await new Promise(resolve => setTimeout(resolve, retryDelayMs));
        }

        return {
            success: false,
            error: 'Failed to acquire lock after max retries'
        };
    }

    // 释放锁
    async release(lockKey, lockId) {
        const key = `lock:${lockKey}`;

        // Lua脚本确保只有锁的持有者才能释放
        const luaScript = `
            if redis.call("get", KEYS[1]) == ARGV[1] then
                return redis.call("del", KEYS[1])
            else
                return 0
            end
        `;

        const result = await this.redis.eval(luaScript, 1, key, lockId);

        if (result === 1) {
            this.locks.delete(key);
            return true;
        }

        return false;
    }

    // 自动续期
    async renew(lockKey, lockId, ttlMs = 10000) {
        const key = `lock:${lockKey}`;

        // 检查锁是否仍然持有
        const currentLockId = await this.redis.get(key);

        if (currentLockId !== lockId) {
            return false;
        }

        // 续期
        await this.redis.pexpire(key, ttlMs);

        const lock = this.locks.get(key);
        if (lock) {
            lock.ttl = ttlMs;
        }

        return true;
    }

    // 使用锁执行操作
    async withLock(lockKey, callback, options = {}) {
        const { ttlMs = 10000, maxRetries = 10, retryDelayMs = 100 } = options;

        const lock = await this.acquire(lockKey, ttlMs, maxRetries, retryDelayMs);

        if (!lock.success) {
            throw new Error(`Failed to acquire lock: ${lockKey}`);
        }

        try {
            // 启动自动续期
            const renewInterval = setInterval(async () => {
                await this.renew(lockKey, lock.lockId, ttlMs);
            }, ttlMs / 2);

            // 执行回调
            const result = await callback();

            // 停止续期
            clearInterval(renewInterval);

            return result;

        } finally {
            // 释放锁
            await this.release(lockKey, lock.lockId);
        }
    }
}

module.exports = new DistributedLock();
```

---

## 38.4 Redis集群配置

### 38.4.1 Redis Sentinel配置

```yaml
# redis/sentinel.conf

# Sentinel配置
port 26379
sentinel monitor mymaster 10.0.1.1 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 10000

# 认证
sentinel auth-pass mymaster yourpassword

# 日志
logfile "/var/log/redis/sentinel.log"
```

### 38.4.2 Redis Cluster配置

```yaml
# docker-compose-redis-cluster.yml
version: '3.8'

services:
  redis-node-1:
    image: redis:7-alpine
    container_name: redis-node-1
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --appendonly yes --port 7000
    ports:
      - "7000:7000"
      - "17000:17000"
    volumes:
      - redis-node-1-data:/data
    networks:
      - redis-cluster

  redis-node-2:
    image: redis:7-alpine
    container_name: redis-node-2
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --appendonly yes --port 7001
    ports:
      - "7001:7001"
      - "17001:17001"
    volumes:
      - redis-node-2-data:/data
    networks:
      - redis-cluster

  redis-node-3:
    image: redis:7-alpine
    container_name: redis-node-3
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --appendonly yes --port 7002
    ports:
      - "7002:7002"
      - "17002:17002"
    volumes:
      - redis-node-3-data:/data
    networks:
      - redis-cluster

  redis-node-4:
    image: redis:7-alpine
    container_name: redis-node-4
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --appendonly yes --port 7003
    ports:
      - "7003:7003"
      - "17003:17003"
    volumes:
      - redis-node-4-data:/data
    networks:
      - redis-cluster

  redis-node-5:
    image: redis:7-alpine
    container_name: redis-node-5
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --appendonly yes --port 7004
    ports:
      - "7004:7004"
      - "17004:17004"
    volumes:
      - redis-node-5-data:/data
    networks:
      - redis-cluster

  redis-node-6:
    image: redis:7-alpine
    container_name: redis-node-6
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --appendonly yes --port 7005
    ports:
      - "7005:7005"
      - "17005:17005"
    volumes:
      - redis-node-6-data:/data
    networks:
      - redis-cluster

  # 集群初始化
  redis-cluster-init:
    image: redis:7-alpine
    container_name: redis-cluster-init
    depends_on:
      - redis-node-1
      - redis-node-2
      - redis-node-3
      - redis-node-4
      - redis-node-5
      - redis-node-6
    command: >
      redis-cli --cluster create
      redis-node-1:7000
      redis-node-2:7001
      redis-node-3:7002
      redis-node-4:7003
      redis-node-5:7004
      redis-node-6:7005
      --cluster-replicas 1
      --cluster-yes
    networks:
      - redis-cluster

networks:
  redis-cluster:
    driver: bridge

volumes:
  redis-node-1-data:
  redis-node-2-data:
  redis-node-3-data:
  redis-node-4-data:
  redis-node-5-data:
  redis-node-6-data:
```

---

## 38.5 实践任务

### 任务1：实现热点数据缓存

为排行榜和用户数据实现缓存：
- 多级缓存
- 自动刷新
- 失效策略

### 任务2：实现分布式锁

使用Redis实现分布式锁：
- 锁获取和释放
- 自动续期
- 死锁预防

### 任务3：搭建Redis集群

配置高可用Redis：
- Sentinel配置
- Cluster配置
- 故障转移测试

---

## 38.6 总结

本课完成了：
- 缓存架构设计
- Redis缓存管理器
- 缓存策略实现
- 数据一致性保证
- 分布式锁实现
- Redis集群配置

**下一课预告**：CI/CD流水线 - 自动化构建、测试和部署流程。

---

*课程版本：1.0*
*最后更新：2026-04-10*
