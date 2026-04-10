# 第36课：微服务架构搭建

> 学习目标：构建完整的企业级游戏后端微服务架构。

---

## 36.1 微服务架构规划

### 36.1.1 服务拆分策略

```
游戏后端微服务架构:

┌─────────────────────────────────────────────────────────────────────┐
│                        API Gateway (Kong)                           │
│                     路由 / 限流 / 认证 / 监控                         │
└─────────────────────────────────────────────────────────────────────┘
                              │
    ┌─────────────────────────┼─────────────────────────┐
    │                         │                         │
    ▼                         ▼                         ▼
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│ Auth Service│       │Match Service│       │ Game Service│
│   认证服务   │       │   匹配服务   │       │   游戏服务   │
│  Port: 8001 │       │  Port: 8002 │       │  Port: 8003 │
└─────────────┘       └─────────────┘       └─────────────┘

┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│Social Service│      │ Leaderboard │       │ Analytics   │
│   社交服务   │       │   排行榜     │       │   分析服务   │
│  Port: 8004 │       │  Port: 8005 │       │  Port: 8006 │
└─────────────┘       └─────────────┘       └─────────────┘

┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│Inventory Svc│       │Payment Svc  │       │Notification │
│   库存服务   │       │   支付服务   │       │   通知服务   │
│  Port: 8007 │       │  Port: 8008 │       │  Port: 8009 │
└─────────────┘       └─────────────┘       └─────────────┘
```

---

## 36.2 服务模板实现

### 36.2.1 基础服务模板

```javascript
// shared/BaseService.js
const express = require('express');
const cors = require('cors');
const helmet = require('helmet');
const compression = require('compression');
const morgan = require('morgan');
const { v4: uuidv4 } = require('uuid');
const Redis = require('ioredis');
const { Pool } = require('pg');

class BaseService {
    constructor(serviceName, port) {
        this.serviceName = serviceName;
        this.port = port;
        this.app = express();
        this.redis = null;
        this.db = null;

        this.setupMiddleware();
        this.setupHealthCheck();
        this.setupErrorHandling();
    }

    setupMiddleware() {
        // 安全中间件
        this.app.use(helmet());
        this.app.use(cors());

        // 性能中间件
        this.app.use(compression());

        // 日志中间件
        this.app.use(morgan('combined', {
            skip: (req, res) => process.env.NODE_ENV === 'test'
        }));

        // 请求ID
        this.app.use((req, res, next) => {
            req.id = uuidv4();
            res.setHeader('X-Request-Id', req.id);
            next();
        });

        // JSON解析
        this.app.use(express.json({ limit: '10mb' }));
        this.app.use(express.urlencoded({ extended: true }));
    }

    setupHealthCheck() {
        // 健康检查端点
        this.app.get('/health', async (req, res) => {
            const health = {
                service: this.serviceName,
                status: 'healthy',
                timestamp: new Date().toISOString(),
                uptime: process.uptime(),
                checks: {}
            };

            // 检查数据库连接
            if (this.db) {
                try {
                    await this.db.query('SELECT 1');
                    health.checks.database = 'connected';
                } catch (error) {
                    health.checks.database = 'disconnected';
                    health.status = 'degraded';
                }
            }

            // 检查Redis连接
            if (this.redis) {
                try {
                    await this.redis.ping();
                    health.checks.redis = 'connected';
                } catch (error) {
                    health.checks.redis = 'disconnected';
                    health.status = 'degraded';
                }
            }

            const statusCode = health.status === 'healthy' ? 200 : 503;
            res.status(statusCode).json(health);
        });

        // 就绪检查
        this.app.get('/ready', async (req, res) => {
            const ready = await this.isReady();
            res.status(ready ? 200 : 503).json({ ready });
        });

        // 指标端点
        this.app.get('/metrics', async (req, res) => {
            const metrics = await this.getMetrics();
            res.set('Content-Type', 'text/plain');
            res.send(metrics);
        });
    }

    setupErrorHandling() {
        // 404处理
        this.app.use((req, res, next) => {
            res.status(404).json({
                error: 'Not Found',
                message: `Route ${req.method} ${req.path} not found`,
                requestId: req.id
            });
        });

        // 错误处理
        this.app.use((err, req, res, next) => {
            console.error(`[${req.id}] Error:`, err);

            const statusCode = err.statusCode || 500;
            const response = {
                error: err.name || 'InternalServerError',
                message: err.message || 'An unexpected error occurred',
                requestId: req.id
            };

            // 开发环境显示堆栈
            if (process.env.NODE_ENV === 'development') {
                response.stack = err.stack;
            }

            res.status(statusCode).json(response);
        });
    }

    async initializeDatabase() {
        this.db = new Pool({
            host: process.env.DB_HOST || 'localhost',
            port: process.env.DB_PORT || 5432,
            database: process.env.DB_NAME || 'gamedb',
            user: process.env.DB_USER || 'admin',
            password: process.env.DB_PASSWORD,
            max: 20,
            idleTimeoutMillis: 30000,
            connectionTimeoutMillis: 2000
        });

        // 测试连接
        try {
            await this.db.query('SELECT 1');
            console.log(`[${this.serviceName}] Database connected`);
        } catch (error) {
            console.error(`[${this.serviceName}] Database connection failed:`, error);
            throw error;
        }
    }

    async initializeRedis() {
        this.redis = new Redis({
            host: process.env.REDIS_HOST || 'localhost',
            port: process.env.REDIS_PORT || 6379,
            password: process.env.REDIS_PASSWORD,
            db: process.env.REDIS_DB || 0,
            retryDelayOnFailover: 100,
            maxRetriesPerRequest: 3
        });

        this.redis.on('connect', () => {
            console.log(`[${this.serviceName}] Redis connected`);
        });

        this.redis.on('error', (error) => {
            console.error(`[${this.serviceName}] Redis error:`, error);
        });
    }

    async isReady() {
        // 检查所有依赖是否就绪
        let ready = true;

        if (this.db) {
            try {
                await this.db.query('SELECT 1');
            } catch {
                ready = false;
            }
        }

        if (this.redis) {
            try {
                await this.redis.ping();
            } catch {
                ready = false;
            }
        }

        return ready;
    }

    async getMetrics() {
        // Prometheus格式指标
        const metrics = [];

        metrics.push(`# HELP service_info Service information`);
        metrics.push(`# TYPE service_info gauge`);
        metrics.push(`service_info{name="${this.serviceName}"} 1`);

        metrics.push(`# HELP service_uptime_seconds Service uptime in seconds`);
        metrics.push(`# TYPE service_uptime_seconds gauge`);
        metrics.push(`service_uptime_seconds{name="${this.serviceName}"} ${process.uptime()}`);

        metrics.push(`# HELP service_memory_usage_bytes Memory usage in bytes`);
        metrics.push(`# TYPE service_memory_usage_bytes gauge`);
        const memUsage = process.memoryUsage();
        metrics.push(`service_memory_usage_bytes{name="${this.serviceName}",type="heapUsed"} ${memUsage.heapUsed}`);
        metrics.push(`service_memory_usage_bytes{name="${this.serviceName}",type="heapTotal"} ${memUsage.heapTotal}`);
        metrics.push(`service_memory_usage_bytes{name="${this.serviceName}",type="rss"} ${memUsage.rss}`);

        return metrics.join('\n');
    }

    registerRoutes(routes) {
        routes(this.app);
    }

    async start() {
        return new Promise((resolve) => {
            this.server = this.app.listen(this.port, () => {
                console.log(`[${this.serviceName}] Service started on port ${this.port}`);
                resolve();
            });
        });
    }

    async stop() {
        if (this.server) {
            await new Promise((resolve) => this.server.close(resolve));
        }

        if (this.db) {
            await this.db.end();
        }

        if (this.redis) {
            this.redis.disconnect();
        }

        console.log(`[${this.serviceName}] Service stopped`);
    }
}

module.exports = BaseService;
```

### 36.2.2 服务发现与注册

```javascript
// shared/ServiceRegistry.js
const Redis = require('ioredis');

class ServiceRegistry {
    constructor() {
        this.redis = new Redis({
            host: process.env.REDIS_HOST || 'localhost',
            port: process.env.REDIS_PORT || 6379
        });
        this.heartbeatInterval = null;
    }

    // 注册服务
    async register(serviceName, serviceId, host, port, metadata = {}) {
        const serviceKey = `service:${serviceName}:${serviceId}`;

        const serviceData = {
            id: serviceId,
            name: serviceName,
            host,
            port,
            metadata,
            registeredAt: Date.now(),
            lastHeartbeat: Date.now()
        };

        await this.redis.hset(serviceKey, serviceData);
        await this.redis.expire(serviceKey, 30); // 30秒过期

        // 添加到服务列表
        await this.redis.sadd(`services:${serviceName}`, serviceId);

        console.log(`Service registered: ${serviceName}/${serviceId} at ${host}:${port}`);
    }

    // 注销服务
    async deregister(serviceName, serviceId) {
        const serviceKey = `service:${serviceName}:${serviceId}`;

        await this.redis.del(serviceKey);
        await this.redis.srem(`services:${serviceName}`, serviceId);

        console.log(`Service deregistered: ${serviceName}/${serviceId}`);
    }

    // 发现服务
    async discover(serviceName) {
        const serviceIds = await this.redis.smembers(`services:${serviceName}`);

        const services = [];
        for (const serviceId of serviceIds) {
            const serviceKey = `service:${serviceName}:${serviceId}`;
            const data = await this.redis.hgetall(serviceKey);

            if (data && Object.keys(data).length > 0) {
                services.push(data);
            } else {
                // 清理过期服务
                await this.redis.srem(`services:${serviceName}`, serviceId);
            }
        }

        return services;
    }

    // 获取单个服务实例（负载均衡）
    async getOne(serviceName, strategy = 'round-robin') {
        const services = await this.discover(serviceName);

        if (services.length === 0) {
            return null;
        }

        switch (strategy) {
            case 'round-robin':
                return services[Math.floor(Math.random() * services.length)];

            case 'random':
                return services[Math.floor(Math.random() * services.length)];

            default:
                return services[0];
        }
    }

    // 心跳
    startHeartbeat(serviceName, serviceId, intervalSeconds = 10) {
        this.heartbeatInterval = setInterval(async () => {
            const serviceKey = `service:${serviceName}:${serviceId}`;

            // 更新心跳时间
            await this.redis.hset(serviceKey, 'lastHeartbeat', Date.now());
            await this.redis.expire(serviceKey, 30);
        }, intervalSeconds * 1000);
    }

    stopHeartbeat() {
        if (this.heartbeatInterval) {
            clearInterval(this.heartbeatInterval);
            this.heartbeatInterval = null;
        }
    }
}

module.exports = ServiceRegistry;
```

---

## 36.3 服务间通信

### 36.3.1 服务客户端

```javascript
// shared/ServiceClient.js
const axios = require('axios');
const ServiceRegistry = require('./ServiceRegistry');

class ServiceClient {
    constructor(serviceName) {
        this.serviceName = serviceName;
        this.registry = new ServiceRegistry();
        this.circuitBreaker = {
            failures: 0,
            lastFailure: 0,
            state: 'closed', // closed, open, half-open
            threshold: 5,
            timeout: 30000
        };
    }

    async request(method, path, data = null, options = {}) {
        // 检查熔断器状态
        if (this.circuitBreaker.state === 'open') {
            const timeSinceFailure = Date.now() - this.circuitBreaker.lastFailure;

            if (timeSinceFailure > this.circuitBreaker.timeout) {
                this.circuitBreaker.state = 'half-open';
            } else {
                throw new Error(`Circuit breaker is open for ${this.serviceName}`);
            }
        }

        // 发现服务
        const service = await this.registry.getOne(this.serviceName);

        if (!service) {
            throw new Error(`No available instances for ${this.serviceName}`);
        }

        const url = `http://${service.host}:${service.port}${path}`;

        try {
            const response = await axios({
                method,
                url,
                data,
                timeout: options.timeout || 5000,
                headers: {
                    'Content-Type': 'application/json',
                    ...options.headers
                }
            });

            // 成功，重置熔断器
            this.circuitBreaker.failures = 0;
            this.circuitBreaker.state = 'closed';

            return response.data;

        } catch (error) {
            // 失败，更新熔断器
            this.circuitBreaker.failures++;
            this.circuitBreaker.lastFailure = Date.now();

            if (this.circuitBreaker.failures >= this.circuitBreaker.threshold) {
                this.circuitBreaker.state = 'open';
            }

            throw error;
        }
    }

    async get(path, options) {
        return this.request('GET', path, null, options);
    }

    async post(path, data, options) {
        return this.request('POST', path, data, options);
    }

    async put(path, data, options) {
        return this.request('PUT', path, data, options);
    }

    async delete(path, options) {
        return this.request('DELETE', path, null, options);
    }
}

module.exports = ServiceClient;
```

### 36.3.2 消息队列通信

```javascript
// shared/MessageQueue.js
const Redis = require('ioredis');

class MessageQueue {
    constructor(channel) {
        this.publisher = new Redis();
        this.subscriber = new Redis();
        this.channel = channel;
        this.handlers = new Map();
    }

    // 发布消息
    async publish(event, data) {
        const message = JSON.stringify({
            event,
            data,
            timestamp: Date.now(),
            id: require('uuid').v4()
        });

        await this.publisher.publish(this.channel, message);
    }

    // 订阅消息
    async subscribe(event, handler) {
        if (!this.handlers.has(event)) {
            this.handlers.set(event, []);
        }

        this.handlers.get(event).push(handler);

        // 确保已订阅通道
        if (this.handlers.size === 1) {
            await this.subscriber.subscribe(this.channel);

            this.subscriber.on('message', (channel, message) => {
                if (channel === this.channel) {
                    try {
                        const parsed = JSON.parse(message);
                        const handlers = this.handlers.get(parsed.event);

                        if (handlers) {
                            for (const handler of handlers) {
                                handler(parsed.data, parsed);
                            }
                        }
                    } catch (error) {
                        console.error('Failed to parse message:', error);
                    }
                }
            });
        }
    }

    // 取消订阅
    async unsubscribe(event) {
        this.handlers.delete(event);

        if (this.handlers.size === 0) {
            await this.subscriber.unsubscribe(this.channel);
        }
    }
}

// 事件类型定义
const GameEvents = {
    MATCH_CREATED: 'match:created',
    MATCH_STARTED: 'match:started',
    MATCH_ENDED: 'match:ended',
    PLAYER_JOINED: 'player:joined',
    PLAYER_LEFT: 'player:left',
    PLAYER_KILLED: 'player:killed',
    ACHIEVEMENT_UNLOCKED: 'achievement:unlocked',
    RANK_UPDATED: 'rank:updated'
};

module.exports = { MessageQueue, GameEvents };
```

---

## 36.4 Docker Compose 配置

### 36.4.1 开发环境配置

```yaml
# docker-compose.yml
version: '3.8'

services:
  # API Gateway
  gateway:
    image: kong:latest
    container_name: arena-gateway
    environment:
      KONG_DATABASE: "off"
      KONG_DECLARATIVE_CONFIG: /etc/kong/kong.yml
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_PROXY_LISTEN: "0.0.0.0:8000"
      KONG_ADMIN_LISTEN: "0.0.0.0:8001"
    ports:
      - "8000:8000"
      - "8443:8443"
      - "8001:8001"
    volumes:
      - ./gateway/kong.yml:/etc/kong/kong.yml
    networks:
      - arena-network
    depends_on:
      - auth-service
      - match-service
      - game-service

  # Auth Service
  auth-service:
    build:
      context: ./services/auth-service
      dockerfile: Dockerfile
    container_name: arena-auth
    environment:
      - NODE_ENV=development
      - PORT=8001
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=arena_auth
      - DB_USER=admin
      - DB_PASSWORD=password
      - REDIS_HOST=redis
      - JWT_SECRET=your-jwt-secret
    ports:
      - "8001:8001"
    networks:
      - arena-network
    depends_on:
      - postgres
      - redis

  # Match Service
  match-service:
    build:
      context: ./services/match-service
      dockerfile: Dockerfile
    container_name: arena-match
    environment:
      - NODE_ENV=development
      - PORT=8002
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=arena_match
      - DB_USER=admin
      - DB_PASSWORD=password
      - REDIS_HOST=redis
    ports:
      - "8002:8002"
    networks:
      - arena-network
    depends_on:
      - postgres
      - redis

  # Game Service
  game-service:
    build:
      context: ./services/game-service
      dockerfile: Dockerfile
    container_name: arena-game
    environment:
      - NODE_ENV=development
      - PORT=8003
      - DB_HOST=postgres
      - REDIS_HOST=redis
    ports:
      - "8003:8003"
    networks:
      - arena-network
    depends_on:
      - postgres
      - redis

  # Social Service
  social-service:
    build:
      context: ./services/social-service
      dockerfile: Dockerfile
    container_name: arena-social
    environment:
      - NODE_ENV=development
      - PORT=8004
      - DB_HOST=postgres
      - REDIS_HOST=redis
    ports:
      - "8004:8004"
    networks:
      - arena-network
    depends_on:
      - postgres
      - redis

  # Leaderboard Service
  leaderboard-service:
    build:
      context: ./services/leaderboard-service
      dockerfile: Dockerfile
    container_name: arena-leaderboard
    environment:
      - NODE_ENV=development
      - PORT=8005
      - DB_HOST=postgres
      - REDIS_HOST=redis
    ports:
      - "8005:8005"
    networks:
      - arena-network
    depends_on:
      - postgres
      - redis

  # PostgreSQL
  postgres:
    image: postgres:15-alpine
    container_name: arena-postgres
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password
      POSTGRES_DB: arena
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./database/init:/docker-entrypoint-initdb.d
    networks:
      - arena-network

  # Redis
  redis:
    image: redis:7-alpine
    container_name: arena-redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - arena-network

  # Prometheus
  prometheus:
    image: prom/prometheus:latest
    container_name: arena-prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
    networks:
      - arena-network

  # Grafana
  grafana:
    image: grafana/grafana:latest
    container_name: arena-grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
    networks:
      - arena-network
    depends_on:
      - prometheus

networks:
  arena-network:
    driver: bridge

volumes:
  postgres-data:
  redis-data:
  grafana-data:
```

---

## 36.5 实践任务

### 任务1：创建微服务

基于模板创建新服务：
- 实现健康检查
- 添加数据库连接
- 实现基本CRUD

### 任务2：配置服务发现

实现服务注册与发现：
- 服务注册
- 心跳机制
- 负载均衡

### 任务3：搭建开发环境

使用Docker Compose：
- 启动所有服务
- 验证服务通信
- 查看监控面板

---

## 36.6 总结

本课完成了：
- 微服务架构规划
- 基础服务模板
- 服务发现机制
- 服务间通信
- Docker Compose配置

**下一课预告**：数据库集群部署 - PostgreSQL高可用配置和读写分离。

---

*课程版本：1.0*
*最后更新：2026-04-10*
