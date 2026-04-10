# 第40课：生产环境部署

> 学习目标：完成生产环境的完整搭建、监控配置和运维体系建立。

---

## 40.1 生产环境架构

### 40.1.1 基础设施架构

```
生产环境架构:

┌──────────────────────────────────────────────────────────────┐
│                        用户层                                  │
│                    CDN / DNS / SSL                            │
└───────────────────────────┬──────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────┐
│                      接入层                                    │
│              Load Balancer / API Gateway                      │
│              Kong / Nginx / AWS ALB                          │
└───────────────────────────┬──────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────┐
│                      应用层                                    │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐        │
│  │  Auth    │ │  Match   │ │  Game    │ │  Social  │        │
│  │  Service │ │  Service │ │  Service │ │  Service │        │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘        │
│                      Kubernetes Cluster                        │
└───────────────────────────┬──────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────┐
│                      数据层                                    │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐        │
│  │PostgreSQL│ │  Redis   │ │ S3/MinIO │ │Elastic   │        │
│  │ Cluster  │ │ Cluster  │ │ Storage  │ │ Search   │        │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘        │
└───────────────────────────┬──────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────┐
│                    监控与运维                                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐        │
│  │Prometheus│ │ Grafana  │ │  Alert   │ │  Log     │        │
│  │          │ │          │ │ Manager  │ │Aggregation│       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘        │
└──────────────────────────────────────────────────────────────┘
```

---

## 40.2 Kubernetes生产配置

### 40.2.1 命名空间与配额

```yaml
# k8s/namespaces.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: arena-production
  labels:
    name: arena-production
    environment: production
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: arena-quota
  namespace: arena-production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
    services: "50"
    secrets: "50"
    configmaps: "50"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: arena-limits
  namespace: arena-production
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "4"
        memory: "8Gi"
      min:
        cpu: "50m"
        memory: "64Mi"
```

### 40.2.2 密钥与配置

```yaml
# k8s/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: arena-secrets
  namespace: arena-production
type: Opaque
stringData:
  DB_PASSWORD: "your-secure-password"
  REDIS_PASSWORD: "your-redis-password"
  JWT_SECRET: "your-jwt-secret"
  JWT_REFRESH_SECRET: "your-refresh-secret"
  API_KEY: "your-api-key"
---
apiVersion: v1
kind: Secret
metadata:
  name: docker-registry
  namespace: arena-production
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
---
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: arena-config
  namespace: arena-production
data:
  NODE_ENV: "production"
  LOG_LEVEL: "info"
  DB_HOST: "arena-postgresql"
  DB_PORT: "5432"
  DB_NAME: "arena"
  DB_USER: "arena"
  REDIS_HOST: "arena-redis-master"
  REDIS_PORT: "6379"
```

### 40.2.3 Ingress配置

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: arena-ingress
  namespace: arena-production
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
spec:
  tls:
    - hosts:
        - api.arena-game.com
        - game.arena-game.com
      secretName: arena-tls
  rules:
    - host: api.arena-game.com
      http:
        paths:
          - path: /auth
            pathType: Prefix
            backend:
              service:
                name: auth-service
                port:
                  number: 8000
          - path: /match
            pathType: Prefix
            backend:
              service:
                name: match-service
                port:
                  number: 8000
          - path: /social
            pathType: Prefix
            backend:
              service:
                name: social-service
                port:
                  number: 8000
          - path: /leaderboard
            pathType: Prefix
            backend:
              service:
                name: leaderboard-service
                port:
                  number: 8000
          - path: /
            pathType: Prefix
            backend:
              service:
                name: game-service
                port:
                  number: 8000
```

---

## 40.3 监控系统

### 40.3.1 Prometheus配置

```yaml
# monitoring/prometheus-values.yaml
prometheus:
  prometheusSpec:
    retention: 30d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: standard
          resources:
            requests:
              storage: 50Gi

    serviceMonitorSelector:
      matchLabels:
        release: arena

    resources:
      requests:
        cpu: 500m
        memory: 1Gi
      limits:
        cpu: 2
        memory: 4Gi

    additionalScrapeConfigs:
      - job_name: 'arena-services'
        kubernetes_sd_configs:
          - role: pod
            namespaces:
              names:
                - arena-production
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
          - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
            action: replace
            regex: ([^:]+)(?::\d+)?;(\d+)
            replacement: $1:$2
            target_label: __address__

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: standard
          resources:
            requests:
              storage: 10Gi

grafana:
  persistence:
    enabled: true
    size: 10Gi
  adminPassword: "secure-admin-password"
```

### 40.3.2 告警规则

```yaml
# monitoring/alert-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: arena-alerts
  namespace: arena-production
spec:
  groups:
    - name: arena-service-alerts
      rules:
        # 服务不可用
        - alert: ServiceDown
          expr: up == 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "Service {{ $labels.job }} is down"
            description: "{{ $labels.instance }} has been down for more than 1 minute"

        # 高错误率
        - alert: HighErrorRate
          expr: |
            sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
            /
            sum(rate(http_requests_total[5m])) by (service)
            > 0.05
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High error rate for {{ $labels.service }}"
            description: "Error rate is {{ $value | humanizePercentage }}"

        # 高延迟
        - alert: HighLatency
          expr: |
            histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)) > 1
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High latency for {{ $labels.service }}"
            description: "P99 latency is {{ $value }}s"

        # 内存使用过高
        - alert: HighMemoryUsage
          expr: |
            (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes)
            / node_memory_MemTotal_bytes > 0.9
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High memory usage"
            description: "Memory usage is {{ $value | humanizePercentage }}"

        # CPU使用过高
        - alert: HighCPUUsage
          expr: |
            100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High CPU usage on {{ $labels.instance }}"
            description: "CPU usage is {{ $value }}%"

    - name: arena-database-alerts
      rules:
        # 数据库连接数过高
        - alert: HighDatabaseConnections
          expr: pg_stat_activity_count > 100
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High number of database connections"
            description: "{{ $value }} connections"

        # 复制延迟
        - alert: ReplicationLag
          expr: pg_replication_lag_seconds > 10
          for: 1m
          labels:
            severity: warning
          annotations:
            summary: "PostgreSQL replication lag is high"
            description: "Lag is {{ $value }}s"

    - name: arena-game-alerts
      rules:
        # 匹配队列积压
        - alert: MatchQueueBacklog
          expr: match_queue_size > 100
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Match queue backlog"
            description: "{{ $value }} players waiting in queue"

        # 游戏服务器不足
        - alert: InsufficientGameServers
          expr: |
            game_servers_available / game_servers_total < 0.2
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Insufficient game servers"
            description: "Only {{ $value | humanizePercentage }} servers available"
```

### 40.3.3 Grafana仪表盘

```json
{
  "dashboard": {
    "title": "Arena Game Backend",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (service)",
            "legendFormat": "{{ service }}"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) by (service) / sum(rate(http_requests_total[5m])) by (service)",
            "legendFormat": "{{ service }}"
          }
        ]
      },
      {
        "title": "Response Time (P99)",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))",
            "legendFormat": "{{ service }}"
          }
        ]
      },
      {
        "title": "Active Players",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(active_players)",
            "legendFormat": "Players"
          }
        ]
      },
      {
        "title": "Match Queue Size",
        "type": "graph",
        "targets": [
          {
            "expr": "match_queue_size",
            "legendFormat": "Queue Size"
          }
        ]
      },
      {
        "title": "Database Connections",
        "type": "graph",
        "targets": [
          {
            "expr": "pg_stat_activity_count",
            "legendFormat": "Connections"
          }
        ]
      },
      {
        "title": "Cache Hit Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(cache_hits_total[5m]) / (rate(cache_hits_total[5m]) + rate(cache_misses_total[5m]))",
            "legendFormat": "Hit Rate"
          }
        ]
      }
    ]
  }
}
```

---

## 40.4 日志系统

### 40.4.1 ELK Stack配置

```yaml
# logging/elasticsearch-values.yaml
elasticsearch:
  replicas: 3
  resources:
    requests:
      cpu: "1"
      memory: "2Gi"
    limits:
      cpu: "2"
      memory: "4Gi"

  volumeClaimTemplate:
    storageClassName: standard
    resources:
      requests:
        storage: 100Gi

  esJavaOpts: "-Xms1g -Xmx1g"

---
# logging/kibana-values.yaml
kibana:
  resources:
    requests:
      cpu: "500m"
      memory: "512Mi"
    limits:
      cpu: "1"
      memory: "1Gi"

  elasticsearchHosts: "http://elasticsearch-master:9200"

---
# logging/fluent-bit-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf

    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Parser            docker
        DB                /var/log/flb_kube.db
        Mem_Buf_Limit     5MB
        Skip_Long_Lines   On
        Refresh_Interval  10

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Merge_Log           On
        K8S-Logging.Parser  On
        K8S-Logging.Exclude On

    [OUTPUT]
        Name            elasticsearch
        Match           *
        Host            elasticsearch-master
        Port            9200
        Logstash_Format On
        Retry_Limit     False

  parsers.conf: |
    [PARSER]
        Name        docker
        Format      json
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L
        Time_Keep   On
```

---

## 40.5 运维手册

### 40.5.1 部署检查清单

```markdown
# 生产部署检查清单

## 部署前检查

### 基础设施
- [ ] Kubernetes集群健康状态
- [ ] 节点资源充足
- [ ] 网络策略配置
- [ ] 存储类配置

### 配置
- [ ] 密钥已正确配置
- [ ] ConfigMap已更新
- [ ] 环境变量正确
- [ ] 数据库连接已验证

### 镜像
- [ ] 镜像版本正确
- [ ] 镜像已推送到仓库
- [ ] 镜像拉取策略配置

### 网络
- [ ] Ingress配置正确
- [ ] SSL证书有效
- [ ] DNS记录已配置
- [ ] 防火墙规则已设置

## 部署中监控

### 服务状态
- [ ] 所有Pod正常运行
- [ ] 服务健康检查通过
- [ ] 数据库迁移完成
- [ ] 缓存预热完成

### 功能验证
- [ ] API端点响应正常
- [ ] 用户认证功能正常
- [ ] 匹配功能正常
- [ ] 游戏功能正常

## 部署后验证

### 性能指标
- [ ] 响应时间在预期范围内
- [ ] 错误率低于阈值
- [ ] 资源使用正常
- [ ] 数据库查询性能正常

### 监控告警
- [ ] Prometheus指标正常
- [ ] Grafana仪表盘正常
- [ ] 告警规则生效
- [ ] 日志收集正常
```

### 40.5.2 应急响应流程

```markdown
# 应急响应流程

## 1. 事故识别

### 监控告警触发
1. 检查告警详情
2. 确认影响范围
3. 评估严重程度

### 用户报告
1. 记录问题描述
2. 收集用户信息
3. 复现问题

## 2. 事故分类

### P0 - 紧急
- 服务完全不可用
- 数据丢失风险
- 安全漏洞

### P1 - 严重
- 核心功能不可用
- 性能严重下降
- 影响大量用户

### P2 - 一般
- 非核心功能异常
- 性能轻微下降
- 影响少量用户

## 3. 应急处理

### 服务不可用
```bash
# 检查Pod状态
kubectl get pods -n arena-production

# 查看日志
kubectl logs -f deployment/auth-service -n arena-production

# 重启服务
kubectl rollout restart deployment/auth-service -n arena-production

# 回滚部署
kubectl rollout undo deployment/auth-service -n arena-production
```

### 数据库问题
```bash
# 检查连接数
psql -c "SELECT count(*) FROM pg_stat_activity;"

# 检查锁
psql -c "SELECT * FROM pg_locks WHERE NOT granted;"

# 终止阻塞查询
psql -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state = 'active' AND query_start < now() - interval '5 minutes';"
```

### Redis问题
```bash
# 检查内存使用
redis-cli info memory

# 检查连接数
redis-cli info clients

# 清理过期键
redis-cli --scan --pattern "cache:*" | xargs redis-cli del
```

## 4. 恢复验证

### 功能测试
- [ ] 核心API正常
- [ ] 用户登录正常
- [ ] 游戏功能正常
- [ ] 数据一致性确认

### 监控确认
- [ ] 错误率恢复正常
- [ ] 响应时间正常
- [ ] 资源使用正常
- [ ] 无新告警

## 5. 事后复盘

### 报告内容
- 事故时间线
- 影响范围
- 根因分析
- 解决方案
- 预防措施
```

---

## 40.6 实践任务

### 任务1：部署生产环境

完整部署流程：
- 配置Kubernetes
- 部署所有服务
- 验证功能

### 任务2：配置监控系统

设置完整监控：
- Prometheus指标
- Grafana仪表盘
- 告警规则

### 任务3：编写运维文档

完善运维手册：
- 部署流程
- 应急响应
- 日常维护

---

## 40.7 总结

本课完成了：
- 生产环境架构设计
- Kubernetes生产配置
- 监控系统搭建
- 日志系统配置
- 运维手册编写

---

## 课程完结总结

恭喜你完成了《UE5后端高级开发学习路线》全部40节课！

### 你已经掌握的技能

**第一阶段：网络架构基础**
- UE5网络架构核心概念
- Actor复制系统
- RPC深度应用
- 属性复制与OnRep回调
- 网络同步策略

**第二阶段：游戏服务器架构**
- Dedicated Server开发
- 服务器框架设计
- 游戏会话管理
- 服务器架构模式
- 高级服务器功能

**第三阶段：数据持久化**
- SaveGame系统
- 关系型数据库集成
- NoSQL数据库集成
- 用户数据管理
- 云存储集成

**第四阶段：高级网络特性**
- 映射系统
- 回放系统
- 网络预测系统
- ReplicationGraph进阶
- 网络同步高级技巧

**第五阶段：云服务与微服务**
- AWS集成
- PlayFab集成
- 容器化与编排
- 微服务架构设计
- 实时通信服务

**第六阶段：性能优化与安全**
- 网络性能优化
- 服务器性能调优
- 服务器安全
- 防盗版设计
- 监控与运维

**第七阶段：实战项目**
- 完整项目架构
- 用户系统
- 匹配系统
- 游戏房间逻辑
- 结果统计与排行榜
- 微服务架构搭建
- 数据库集群部署
- 缓存层设计
- CI/CD流水线
- 生产环境部署

### 继续学习建议

1. **深入研究源码**：阅读UE5网络模块源码
2. **参与开源项目**：贡献游戏后端项目
3. **实践项目开发**：完成完整的多人游戏
4. **关注社区动态**：UE5官方文档和论坛
5. **持续技术更新**：关注最新版本特性

---

*课程版本：1.0*
*最后更新：2026-04-10*
