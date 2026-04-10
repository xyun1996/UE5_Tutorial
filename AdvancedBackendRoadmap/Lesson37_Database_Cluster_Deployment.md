# 第37课：数据库集群部署

> 学习目标：掌握PostgreSQL高可用配置、读写分离和数据同步技术。

---

## 37.1 数据库架构设计

### 37.1.1 高可用架构

```
PostgreSQL高可用架构:

                    ┌─────────────────┐
                    │   HAProxy       │
                    │   负载均衡器     │
                    └────────┬────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
            ▼                ▼                ▼
     ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
     │   Primary    │ │  Standby 1   │ │  Standby 2   │
     │   主服务器    │ │   从服务器    │ │   从服务器    │
     │   读写       │ │   只读        │ │   只读        │
     └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
            │                │                │
            └────────────────┼────────────────┘
                             │
                      流复制 (Streaming Replication)
```

---

## 37.2 PostgreSQL主从配置

### 37.2.1 主服务器配置

```ini
# postgresql.conf (Primary)

# 连接设置
listen_addresses = '*'
port = 5432
max_connections = 200

# 复制设置
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
wal_keep_segments = 64
wal_keep_size = 1GB
synchronous_commit = on

# 日志设置
logging_collector = on
log_directory = 'pg_log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_statement = 'ddl'
log_replication_commands = on

# 性能优化
shared_buffers = 256MB
effective_cache_size = 768MB
maintenance_work_mem = 64MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 2621kB
min_wal_size = 1GB
max_wal_size = 4GB
```

```bash
# pg_hba.conf (Primary)

# 允许复制连接
host    replication     replicator      10.0.0.0/8               md5
host    replication     replicator      172.16.0.0/12            md5
host    replication     replicator      192.168.0.0/16           md5

# 允许应用连接
host    all             all             0.0.0.0/0                md5
```

### 37.2.2 从服务器配置

```ini
# postgresql.conf (Standby)

# 从主服务器继承大部分配置
primary_conninfo = 'host=primary.server port=5432 user=replicator password=yourpassword'

# 热备份模式
hot_standby = on
hot_standby_feedback = on

# 从服务器设置
max_standby_streaming_delay = 30s
wal_receiver_status_interval = 10s
```

### 37.2.3 初始化从服务器

```bash
#!/bin/bash
# setup_standby.sh

# 停止从服务器
sudo systemctl stop postgresql

# 清空数据目录
sudo rm -rf /var/lib/postgresql/15/main/*

# 从主服务器复制数据
sudo -u postgres pg_basebackup \
    -h primary.server \
    -D /var/lib/postgresql/15/main \
    -U replicator \
    -P \
    -v \
    -R \
    -X stream \
    -C -S replica_slot_1

# 创建standby.signal文件（PostgreSQL 12+）
touch /var/lib/postgresql/15/main/standby.signal

# 设置权限
sudo chown -R postgres:postgres /var/lib/postgresql/15/main
sudo chmod 700 /var/lib/postgresql/15/main

# 启动从服务器
sudo systemctl start postgresql
```

---

## 37.3 读写分离实现

### 37.3.1 连接池管理

```javascript
// database/ConnectionPool.js
const { Pool } = require('pg');

class DatabaseConnectionPool {
    constructor() {
        // 主库连接池（读写）
        this.primaryPool = new Pool({
            host: process.env.PRIMARY_DB_HOST || 'localhost',
            port: process.env.DB_PORT || 5432,
            database: process.env.DB_NAME || 'gamedb',
            user: process.env.DB_USER || 'admin',
            password: process.env.DB_PASSWORD,
            max: 20,
            idleTimeoutMillis: 30000,
            connectionTimeoutMillis: 2000
        });

        // 从库连接池（只读）
        this.replicaPools = [];
        this.currentReplicaIndex = 0;

        const replicas = (process.env.REPLICA_HOSTS || '').split(',').filter(h => h);

        for (const replicaHost of replicas) {
            this.replicaPools.push(new Pool({
                host: replicaHost,
                port: process.env.DB_PORT || 5432,
                database: process.env.DB_NAME || 'gamedb',
                user: process.env.DB_USER || 'admin',
                password: process.env.DB_PASSWORD,
                max: 10,
                idleTimeoutMillis: 30000,
                connectionTimeoutMillis: 2000
            }));
        }

        // 如果没有从库，使用主库
        if (this.replicaPools.length === 0) {
            this.replicaPools.push(this.primaryPool);
        }
    }

    // 获取从库连接（轮询）
    getReplicaPool() {
        if (this.replicaPools.length === 0) {
            return this.primaryPool;
        }

        const pool = this.replicaPools[this.currentReplicaIndex];
        this.currentReplicaIndex = (this.currentReplicaIndex + 1) % this.replicaPools.length;

        return pool;
    }

    // 执行写操作（主库）
    async write(query, params = []) {
        const client = await this.primaryPool.connect();
        try {
            const result = await client.query(query, params);
            return result;
        } finally {
            client.release();
        }
    }

    // 执行读操作（从库）
    async read(query, params = []) {
        const pool = this.getReplicaPool();
        const client = await pool.connect();
        try {
            const result = await client.query(query, params);
            return result;
        } finally {
            client.release();
        }
    }

    // 事务（主库）
    async transaction(callback) {
        const client = await this.primaryPool.connect();
        try {
            await client.query('BEGIN');
            const result = await callback(client);
            await client.query('COMMIT');
            return result;
        } catch (error) {
            await client.query('ROLLBACK');
            throw error;
        } finally {
            client.release();
        }
    }

    // 关闭所有连接
    async close() {
        await this.primaryPool.end();
        for (const pool of this.replicaPools) {
            if (pool !== this.primaryPool) {
                await pool.end();
            }
        }
    }
}

module.exports = new DatabaseConnectionPool();
```

### 37.3.2 数据访问层

```javascript
// database/UserRepository.js
const db = require('./ConnectionPool');

class UserRepository {
    // 创建用户（写操作 -> 主库）
    async create(userData) {
        const query = `
            INSERT INTO users (id, email, display_name, created_at)
            VALUES ($1, $2, $3, NOW())
            RETURNING *
        `;

        const result = await db.write(query, [
            userData.id,
            userData.email,
            userData.displayName
        ]);

        return result.rows[0];
    }

    // 更新用户（写操作 -> 主库）
    async update(userId, updates) {
        const setClauses = [];
        const values = [userId];
        let paramIndex = 2;

        for (const [key, value] of Object.entries(updates)) {
            setClauses.push(`${key} = $${paramIndex}`);
            values.push(value);
            paramIndex++;
        }

        const query = `
            UPDATE users
            SET ${setClauses.join(', ')}, updated_at = NOW()
            WHERE id = $1
            RETURNING *
        `;

        const result = await db.write(query, values);
        return result.rows[0];
    }

    // 查询用户（读操作 -> 从库）
    async findById(userId) {
        const query = 'SELECT * FROM users WHERE id = $1';
        const result = await db.read(query, [userId]);
        return result.rows[0];
    }

    // 查询用户列表（读操作 -> 从库）
    async findMany(filter = {}, limit = 100, offset = 0) {
        const conditions = [];
        const values = [];
        let paramIndex = 1;

        if (filter.email) {
            conditions.push(`email = $${paramIndex}`);
            values.push(filter.email);
            paramIndex++;
        }

        if (filter.displayName) {
            conditions.push(`display_name ILIKE $${paramIndex}`);
            values.push(`%${filter.displayName}%`);
            paramIndex++;
        }

        const whereClause = conditions.length > 0
            ? `WHERE ${conditions.join(' AND ')}`
            : '';

        values.push(limit, offset);

        const query = `
            SELECT * FROM users
            ${whereClause}
            ORDER BY created_at DESC
            LIMIT $${paramIndex} OFFSET $${paramIndex + 1}
        `;

        const result = await db.read(query, values);
        return result.rows;
    }

    // 批量查询（读操作 -> 从库）
    async findByIds(userIds) {
        const query = `
            SELECT * FROM users
            WHERE id = ANY($1)
        `;

        const result = await db.read(query, [userIds]);
        return result.rows;
    }

    // 更新用户统计（写操作 -> 主库，带事务）
    async updateStats(userId, statsUpdate) {
        return db.transaction(async (client) => {
            // 更新统计
            const updateQuery = `
                UPDATE player_stats
                SET
                    total_matches = total_matches + $2,
                    wins = wins + $3,
                    losses = losses + $4,
                    kills = kills + $5,
                    deaths = deaths + $6,
                    assists = assists + $7,
                    updated_at = NOW()
                WHERE user_id = $1
            `;

            await client.query(updateQuery, [
                userId,
                statsUpdate.matches || 0,
                statsUpdate.wins || 0,
                statsUpdate.losses || 0,
                statsUpdate.kills || 0,
                statsUpdate.deaths || 0,
                statsUpdate.assists || 0
            ]);

            // 更新用户等级（如果需要）
            const levelQuery = `
                UPDATE users
                SET experience = experience + $2,
                    level = LEAST(100, FLOOR(SQRT(experience + $2) / 10))
                WHERE id = $1
            `;

            await client.query(levelQuery, [userId, statsUpdate.experience || 0]);

            return true;
        });
    }
}

module.exports = new UserRepository();
```

---

## 37.4 数据同步与备份

### 37.4.1 自动备份脚本

```bash
#!/bin/bash
# backup.sh

# 配置
BACKUP_DIR="/backup/postgresql"
DB_NAME="gamedb"
DB_USER="postgres"
RETENTION_DAYS=7
S3_BUCKET="s3://your-bucket/postgresql-backups"

# 时间戳
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/${DB_NAME}_${TIMESTAMP}.sql.gz"

# 创建备份目录
mkdir -p ${BACKUP_DIR}

# 执行备份
echo "Starting backup of ${DB_NAME}..."
pg_dump -U ${DB_USER} -h localhost -d ${DB_NAME} | gzip > ${BACKUP_FILE}

# 检查备份是否成功
if [ $? -eq 0 ]; then
    echo "Backup completed: ${BACKUP_FILE}"

    # 计算文件大小
    SIZE=$(du -h ${BACKUP_FILE} | cut -f1)
    echo "Backup size: ${SIZE}"

    # 上传到S3
    if command -v aws &> /dev/null; then
        echo "Uploading to S3..."
        aws s3 cp ${BACKUP_FILE} ${S3_BUCKET}/$(basename ${BACKUP_FILE})
        echo "S3 upload completed"
    fi

    # 清理旧备份
    echo "Cleaning old backups..."
    find ${BACKUP_DIR} -name "*.sql.gz" -type f -mtime +${RETENTION_DAYS} -delete
    echo "Old backups cleaned"

    # 发送通知
    curl -X POST "${SLACK_WEBHOOK_URL}" \
        -H 'Content-Type: application/json' \
        -d "{\"text\":\"PostgreSQL backup completed: ${BACKUP_FILE} (${SIZE})\"}"
else
    echo "Backup failed!"
    exit 1
fi
```

### 37.4.2 数据同步监控

```javascript
// monitoring/ReplicationMonitor.js
const { Pool } = require('pg');
const Redis = require('ioredis');

class ReplicationMonitor {
    constructor() {
        this.primaryPool = new Pool({
            host: process.env.PRIMARY_DB_HOST,
            database: 'postgres',
            user: process.env.DB_USER,
            password: process.env.DB_PASSWORD
        });

        this.redis = new Redis();
    }

    // 检查复制状态
    async checkReplicationStatus() {
        try {
            // 查询主库复制状态
            const result = await this.primaryPool.query(`
                SELECT
                    client_addr,
                    state,
                    sent_lsn,
                    write_lsn,
                    flush_lsn,
                    replay_lsn,
                    EXTRACT(EPOCH FROM (now() - reply_time)) as lag_seconds
                FROM pg_stat_replication
            `);

            const replicas = [];

            for (const row of result.rows) {
                const lag = parseFloat(row.lag_seconds) || 0;
                const status = {
                    address: row.client_addr,
                    state: row.state,
                    lagSeconds: lag,
                    healthy: lag < 10 && row.state === 'streaming'
                };

                replicas.push(status);

                // 存储到Redis用于告警
                await this.redis.hset('replication:status', row.client_addr, JSON.stringify(status));
            }

            return {
                primary: 'healthy',
                replicas,
                timestamp: new Date().toISOString()
            };

        } catch (error) {
            console.error('Failed to check replication status:', error);

            return {
                primary: 'error',
                error: error.message,
                timestamp: new Date().toISOString()
            };
        }
    }

    // 检查数据库大小
    async checkDatabaseSize() {
        const result = await this.primaryPool.query(`
            SELECT
                datname as database,
                pg_size_pretty(pg_database_size(datname)) as size,
                pg_database_size(datname) as bytes
            FROM pg_database
            WHERE datname NOT IN ('template0', 'template1')
        `);

        return result.rows;
    }

    // 检查连接数
    async checkConnections() {
        const result = await this.primaryPool.query(`
            SELECT
                datname as database,
                count(*) as connections,
                max(state) as state
            FROM pg_stat_activity
            GROUP BY datname
        `);

        return result.rows;
    }

    // 检查锁等待
    async checkLocks() {
        const result = await this.primaryPool.query(`
            SELECT
                blocked_locks.pid AS blocked_pid,
                blocked_activity.usename AS blocked_user,
                blocking_locks.pid AS blocking_pid,
                blocking_activity.usename AS blocking_user,
                blocked_activity.query AS blocked_statement,
                blocking_activity.query AS current_statement_in_blocking_process
            FROM pg_catalog.pg_locks blocked_locks
            JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
            JOIN pg_catalog.pg_locks blocking_locks
                ON blocking_locks.locktype = blocked_locks.locktype
                AND blocking_locks.DATABASE IS NOT DISTINCT FROM blocked_locks.DATABASE
                AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
                AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
                AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
                AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
                AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
                AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
                AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
                AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
                AND blocking_locks.pid != blocked_locks.pid
            JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
            WHERE NOT blocked_locks.GRANTED
        `);

        return result.rows;
    }

    // 启动监控
    startMonitoring(intervalMs = 60000) {
        this.monitorInterval = setInterval(async () => {
            const status = await this.checkReplicationStatus();

            // 检查从库是否延迟
            for (const replica of status.replicas) {
                if (!replica.healthy) {
                    // 发送告警
                    await this.redis.publish('alerts', JSON.stringify({
                        type: 'replication_lag',
                        severity: 'warning',
                        message: `Replica ${replica.address} lag is ${replica.lagSeconds}s`,
                        timestamp: new Date().toISOString()
                    }));
                }
            }

            // 存储指标
            await this.redis.lpush('monitoring:replication', JSON.stringify(status));
            await this.redis.ltrim('monitoring:replication', 0, 99); // 保留最近100条

        }, intervalMs);
    }

    stopMonitoring() {
        if (this.monitorInterval) {
            clearInterval(this.monitorInterval);
        }
    }
}

module.exports = ReplicationMonitor;
```

---

## 37.5 实践任务

### 任务1：配置主从复制

设置PostgreSQL主从：
- 配置主服务器
- 初始化从服务器
- 验证数据同步

### 任务2：实现读写分离

在应用层实现：
- 写操作走主库
- 读操作走从库
- 负载均衡

### 任务3：配置备份策略

实现自动备份：
- 每日全量备份
- 备份到云存储
- 备份监控告警

---

## 37.6 总结

本课完成了：
- PostgreSQL高可用架构
- 主从复制配置
- 读写分离实现
- 数据备份策略
- 复制监控实现

**下一课预告**：缓存层设计 - Redis缓存策略和数据一致性保证。

---

*课程版本：1.0*
*最后更新：2026-04-10*
