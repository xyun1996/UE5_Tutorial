# 第23课：容器化与编排

> 学习目标：掌握Docker容器化游戏服务器技术，学习Kubernetes集群部署与编排策略。

---

## 23.1 Docker 基础

### 23.1.1 UE5 Dedicated Server Docker化

**创建Dockerfile：**

```dockerfile
# UE5 Dedicated Server Dockerfile
# 基础镜像 - Ubuntu 22.04
FROM ubuntu:22.04

# 设置环境变量
ENV DEBIAN_FRONTEND=noninteractive
ENV UE_SERVER_PORT=7777
ENV UE_QUERY_PORT=27015

# 安装依赖
RUN apt-get update && apt-get install -y \
    libicu70 \
    libssl3 \
    libcurl4-openssl-dev \
    libgl1-mesa-glx \
    libglu1-mesa \
    libxcursor1 \
    libxinerama1 \
    libxrandr2 \
    libxi6 \
    libsdl2-2.0-0 \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# 创建非root用户
RUN useradd -m -s /bin/bash ueadmin

# 创建服务器目录
WORKDIR /home/ueadmin/Server

# 复制服务器文件（需要先构建Linux服务器）
# 假设服务器文件在build/LinuxServer目录
COPY build/LinuxServer/ ./

# 设置权限
RUN chown -R ueadmin:ueadmin /home/ueadmin/Server

# 切换到非root用户
USER ueadmin

# 暴露端口
EXPOSE ${UE_SERVER_PORT}/udp
EXPOSE ${UE_SERVER_PORT}/tcp
EXPOSE ${UE_QUERY_PORT}/udp

# 健康检查
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD netstat -an | grep ${UE_SERVER_PORT} || exit 1

# 启动命令
ENTRYPOINT ["./MyProjectServer.sh"]
CMD ["MyMap", "-log", "-port=7777"]
```

### 23.1.2 多阶段构建优化

```dockerfile
# 多阶段构建 - 编译阶段
FROM ue5-build:latest AS builder

WORKDIR /build
COPY . .

# 编译服务器
RUN ./BuildMyProjectServer.sh

# 运行时镜像
FROM ubuntu:22.04 AS runtime

# 安装最小运行时依赖
RUN apt-get update && apt-get install -y --no-install-recommends \
    libicu70 \
    libssl3 \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# 创建用户和目录
RUN useradd -m -s /bin/bash ueadmin
WORKDIR /home/ueadmin/Server

# 从编译阶段复制文件
COPY --from=builder /build/LinuxServer/ ./
COPY --from=builder /build/DefaultGame.ini ./MyProject/Saved/Config/LinuxServer/

# 设置权限
RUN chown -R ueadmin:ueadmin /home/ueadmin/Server

USER ueadmin

EXPOSE 7777/udp 7777/tcp 27015/udp

ENTRYPOINT ["./MyProjectServer.sh"]
```

### 23.1.3 Docker Compose配置

```yaml
# docker-compose.yml
version: '3.8'

services:
  # 游戏服务器
  game-server:
    build:
      context: .
      dockerfile: Dockerfile.server
    image: myproject-server:latest
    container_name: myproject-server
    restart: unless-stopped
    ports:
      - "7777:7777/udp"
      - "7777:7777/tcp"
      - "27015:27015/udp"
    environment:
      - SERVER_NAME=MyGameServer
      - MAX_PLAYERS=16
      - GAME_MODE=Deathmatch
      - LOG_LEVEL=info
    volumes:
      - ./Server/Saved:/home/ueadmin/Server/MyProject/Saved
      - ./Server/Logs:/home/ueadmin/Server/MyProject/Saved/Logs
    networks:
      - game-network
    healthcheck:
      test: ["CMD", "netstat", "-an", "|", "grep", "7777"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
        reservations:
          cpus: '1'
          memory: 2G

  # 匹配服务器
  matchmaker:
    build:
      context: ./Matchmaker
      dockerfile: Dockerfile.matchmaker
    image: myproject-matchmaker:latest
    container_name: myproject-matchmaker
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - SERVER_POOL_SIZE=10
    depends_on:
      - redis
    networks:
      - game-network

  # Redis缓存
  redis:
    image: redis:7-alpine
    container_name: myproject-redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - game-network

  # 数据库
  postgres:
    image: postgres:15-alpine
    container_name: myproject-postgres
    restart: unless-stopped
    environment:
      - POSTGRES_USER=gameadmin
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=gamedb
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - game-network

networks:
  game-network:
    driver: bridge

volumes:
  redis-data:
  postgres-data:
```

---

## 23.2 Docker 网络配置

### 23.2.1 网络模式

```yaml
# docker-compose.network.yml
version: '3.8'

services:
  # 使用Host网络模式（最佳性能，但端口冲突风险高）
  game-server-host:
    image: myproject-server:latest
    network_mode: host
    environment:
      - SERVER_NAME=HostModeServer

  # 使用Bridge网络模式（默认，隔离性好）
  game-server-bridge:
    image: myproject-server:latest
    networks:
      - game-bridge
    ports:
      - "7777:7777/udp"

  # 使用自定义网络配置
  game-server-custom:
    image: myproject-server:latest
    networks:
      game-network:
        ipv4_address: 172.20.0.10
    ports:
      - "7778:7777/udp"

networks:
  game-bridge:
    driver: bridge
  game-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
          gateway: 172.20.0.1
```

### 23.2.2 服务发现

```yaml
# 服务发现配置
version: '3.8'

services:
  game-server:
    image: myproject-server:latest
    networks:
      - game-network
    # 容器名称作为主机名
    container_name: game-server-1
    # DNS配置
    dns:
      - 8.8.8.8
      - 8.8.4.4
    # 额外主机映射
    extra_hosts:
      - "api.mygame.local:192.168.1.100"

  # 服务注册（Consul示例）
  consul:
    image: consul:latest
    networks:
      - game-network
    ports:
      - "8500:8500"
    command: agent -server -ui -bootstrap-expect=1 -client=0.0.0.0

  # Registrator - 自动服务注册
  registrator:
    image: gliderlabs/registrator:latest
    networks:
      - game-network
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock
    command: -internal consul://consul:8500
    depends_on:
      - consul

networks:
  game-network:
    driver: bridge
```

---

## 23.3 Kubernetes 集群部署

### 23.3.1 基础概念

**Kubernetes核心组件：**
- **Pod**: 最小部署单元，一个或多个容器
- **Deployment**: 管理Pod副本和更新策略
- **Service**: 提供稳定的网络端点
- **ConfigMap**: 配置数据存储
- **Secret**: 敏感数据存储
- **Ingress**: HTTP路由入口

### 23.3.2 游戏服务器Deployment

```yaml
# game-server-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: game-server-deployment
  namespace: game-servers
  labels:
    app: myproject-server
    version: v1.0.0
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myproject-server
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: myproject-server
        version: v1.0.0
    spec:
      # 使用专用节点池
      nodeSelector:
        node-type: game-server
      # 容忍GPU节点污点（如需要）
      tolerations:
      - key: "nvidia.com/gpu"
        operator: "Exists"
        effect: "NoSchedule"
      # 反亲和性 - 分散到不同节点
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: myproject-server
              topologyKey: "kubernetes.io/hostname"
      containers:
      - name: game-server
        image: myregistry.azurecr.io/myproject-server:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 7777
          name: game-port
          protocol: UDP
        - containerPort: 7777
          name: game-port-tcp
          protocol: TCP
        - containerPort: 27015
          name: query-port
          protocol: UDP
        # 环境变量
        env:
        - name: SERVER_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: MAX_PLAYERS
          value: "16"
        - name: LOG_LEVEL
          value: "info"
        # 从ConfigMap加载配置
        envFrom:
        - configMapRef:
            name: game-server-config
        # 资源限制
        resources:
          requests:
            cpu: "1000m"
            memory: "2Gi"
          limits:
            cpu: "2000m"
            memory: "4Gi"
        # 存活探针
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - "netstat -an | grep 7777"
          initialDelaySeconds: 60
          periodSeconds: 30
          timeoutSeconds: 10
          failureThreshold: 3
        # 就绪探针
        readinessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - "netstat -an | grep 7777 | grep LISTEN"
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        # 存储卷
        volumeMounts:
        - name: server-data
          mountPath: /home/ueadmin/Server/MyProject/Saved
        - name: logs
          mountPath: /home/ueadmin/Server/MyProject/Saved/Logs
      volumes:
      - name: server-data
        emptyDir: {}
      - name: logs
        emptyDir: {}
---
# ConfigMap配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-server-config
  namespace: game-servers
data:
  GAME_MODE: "Deathmatch"
  MAP_NAME: "MyMap"
  TICK_RATE: "60"
```

### 23.3.3 Service配置

```yaml
# game-server-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: game-server-service
  namespace: game-servers
  annotations:
    # 外部负载均衡器配置
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local  # 保留源IP
  selector:
    app: myproject-server
  ports:
  - name: game-udp
    port: 7777
    targetPort: 7777
    protocol: UDP
  - name: game-tcp
    port: 7777
    targetPort: 7777
    protocol: TCP
  - name: query-udp
    port: 27015
    targetPort: 27015
    protocol: UDP
---
# Headless Service 用于直接访问Pod
apiVersion: v1
kind: Service
metadata:
  name: game-server-headless
  namespace: game-servers
spec:
  type: ClusterIP
  clusterIP: None  # Headless
  selector:
    app: myproject-server
  ports:
  - name: game-udp
    port: 7777
    targetPort: 7777
    protocol: UDP
```

### 23.3.4 游戏服务器专用配置（Agones）

```yaml
# Agones GameServer配置
apiVersion: agones.dev/v1
kind: GameServer
metadata:
  name: myproject-server-1
  namespace: game-servers
  labels:
    game: myproject
    mode: deathmatch
spec:
  # 端口配置
  ports:
  - name: gameport
    portPolicy: Dynamic
    container: game-server
    containerPort: 7777
    protocol: UDP
  # 健康检查
  health:
    initialDelaySeconds: 30
    periodSeconds: 10
    failureThreshold: 3
  # 镜像配置
  template:
    spec:
      containers:
      - name: game-server
        image: myregistry.azurecr.io/myproject-server:latest
        env:
        - name: READY
          value: "false"
        resources:
          requests:
            cpu: "1000m"
            memory: "2Gi"
          limits:
            cpu: "2000m"
            memory: "4Gi"
---
# Fleet配置 - 管理多个GameServer
apiVersion: agones.dev/v1
kind: Fleet
metadata:
  name: myproject-fleet
  namespace: game-servers
spec:
  replicas: 5
  allocation:
    enabled: true
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      ports:
      - name: gameport
        portPolicy: Dynamic
        container: game-server
        containerPort: 7777
        protocol: UDP
      health:
        initialDelaySeconds: 30
        periodSeconds: 10
        failureThreshold: 3
      template:
        spec:
          containers:
          - name: game-server
            image: myregistry.azurecr.io/myproject-server:latest
            resources:
              requests:
                cpu: "1000m"
                memory: "2Gi"
              limits:
                cpu: "2000m"
                memory: "4Gi"
---
# FleetAutoscaler - 自动扩缩容
apiVersion: autoscaling.agones.dev/v1
kind: FleetAutoscaler
metadata:
  name: myproject-autoscaler
  namespace: game-servers
spec:
  fleetName: myproject-fleet
  policy:
    type: Buffer
    buffer:
      bufferSize: 2
      minReplicas: 2
      maxReplicas: 20
```

---

## 23.4 自动扩缩容设计

### 23.4.1 Horizontal Pod Autoscaler

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: matchmaker-hpa
  namespace: game-servers
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: matchmaker
  minReplicas: 2
  maxReplicas: 10
  metrics:
  # CPU使用率
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  # 内存使用率
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  # 自定义指标 - 等待队列长度
  - type: Pods
    pods:
      metric:
        name: matchmaking_queue_length
      target:
        type: AverageValue
        averageValue: "100"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 1
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60
```

### 23.4.2 自定义指标服务器

```yaml
# 自定义指标部署
apiVersion: apps/v1
kind: Deployment
metadata:
  name: custom-metrics-server
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: custom-metrics-server
  template:
    metadata:
      labels:
        app: custom-metrics-server
    spec:
      containers:
      - name: metrics-server
        image: custom-metrics-server:latest
        env:
        - name: REDIS_HOST
          value: "redis.monitoring.svc.cluster.local"
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: custom-metrics-server
  namespace: monitoring
spec:
  selector:
    app: custom-metrics-server
  ports:
  - port: 8080
    targetPort: 8080
---
# APIService注册自定义指标
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1beta1.custom.metrics.k8s.io
spec:
  service:
    name: custom-metrics-server
    namespace: monitoring
  group: custom.metrics.k8s.io
  version: v1beta1
  insecureSkipTLSVerify: true
  groupPriorityMinimum: 100
  versionPriority: 100
```

---

## 23.5 容器日志与监控

### 23.5.1 日志收集架构

```yaml
# Fluent Bit日志收集器
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      serviceAccountName: fluent-bit
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:2.0
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: dockerlogs
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: config
          mountPath: /fluent-bit/etc/
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: dockerlogs
        hostPath:
          path: /var/lib/docker/containers
      - name: config
        configMap:
          name: fluent-bit-config
---
# Fluent Bit配置
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
        Host            elasticsearch.logging.svc.cluster.local
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

### 23.5.2 Prometheus监控配置

```yaml
# Prometheus部署
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:v2.45.0
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus/"
        - "--storage.tsdb.retention.time=15d"
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: config
          mountPath: /etc/prometheus
        - name: storage
          mountPath: /prometheus
      volumes:
      - name: config
        configMap:
          name: prometheus-config
      - name: storage
        emptyDir: {}
---
# Prometheus配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s

    scrape_configs:
    - job_name: 'prometheus'
      static_configs:
      - targets: ['localhost:9090']

    - job_name: 'game-servers'
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
          - game-servers
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: keep
        regex: myproject-server

    - job_name: 'kube-state-metrics'
      static_configs:
      - targets: ['kube-state-metrics.monitoring.svc.cluster.local:8080']

    - job_name: 'node-exporter'
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_endpoints_name]
        action: keep
        regex: node-exporter
---
# 游戏服务器指标暴露
apiVersion: v1
kind: Service
metadata:
  name: game-server-metrics
  namespace: game-servers
  labels:
    app: myproject-server
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9091"
spec:
  selector:
    app: myproject-server
  ports:
  - name: metrics
    port: 9091
    targetPort: 9091
```

---

## 23.6 CI/CD 流水线

### 23.6.1 GitLab CI配置

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - package
  - deploy

variables:
  DOCKER_REGISTRY: registry.gitlab.com
  IMAGE_NAME: myproject/game-server
  KUBE_NAMESPACE: game-servers

# 构建阶段
build:server:
  stage: build
  image: ue5-build:latest
  tags:
    - docker
    - linux
  script:
    - ./BuildMyProjectServer.sh
  artifacts:
    paths:
      - build/LinuxServer/
    expire_in: 1 week

# 测试阶段
test:unit:
  stage: test
  image: ue5-build:latest
  script:
    - ./RunTests.sh
  dependencies:
    - build:server

# 打包Docker镜像
package:docker:
  stage: package
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $DOCKER_REGISTRY
    - docker build -t $DOCKER_REGISTRY/$IMAGE_NAME:$CI_COMMIT_SHA -t $DOCKER_REGISTRY/$IMAGE_NAME:latest -f Dockerfile.server .
    - docker push $DOCKER_REGISTRY/$IMAGE_NAME:$CI_COMMIT_SHA
    - docker push $DOCKER_REGISTRY/$IMAGE_NAME:latest
  dependencies:
    - build:server
  only:
    - main
    - develop

# 部署到开发环境
deploy:dev:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl config use-context dev-cluster
    - kubectl set image deployment/game-server-deployment game-server=$DOCKER_REGISTRY/$IMAGE_NAME:$CI_COMMIT_SHA -n $KUBE_NAMESPACE
    - kubectl rollout status deployment/game-server-deployment -n $KUBE_NAMESPACE
  environment:
    name: development
    url: https://dev.mygame.com
  only:
    - develop

# 部署到生产环境
deploy:prod:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl config use-context prod-cluster
    - kubectl set image deployment/game-server-deployment game-server=$DOCKER_REGISTRY/$IMAGE_NAME:$CI_COMMIT_SHA -n $KUBE_NAMESPACE
    - kubectl rollout status deployment/game-server-deployment -n $KUBE_NAMESPACE
  environment:
    name: production
    url: https://mygame.com
  when: manual
  only:
    - main
```

### 23.6.2 GitHub Actions配置

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy Game Server

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/game-server

jobs:
  build:
    runs-on: self-hosted
    steps:
    - uses: actions/checkout@v3

    - name: Build Server
      run: |
        ./BuildMyProjectServer.sh

    - name: Upload Build Artifacts
      uses: actions/upload-artifact@v3
      with:
        name: server-build
        path: build/LinuxServer/

  docker:
    needs: build
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3

    - name: Download Build Artifacts
      uses: actions/download-artifact@v3
      with:
        name: server-build
        path: build/LinuxServer/

    - name: Log in to Container Registry
      uses: docker/login-action@v2
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        file: ./Dockerfile.server
        push: true
        tags: |
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest

  deploy:
    needs: docker
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
    - name: Deploy to Kubernetes
      run: |
        kubectl config use-context ${{ secrets.KUBE_CONTEXT }}
        kubectl set image deployment/game-server-deployment \
          game-server=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
          -n game-servers
        kubectl rollout status deployment/game-server-deployment -n game-servers
```

---

## 23.7 实践任务

### 任务1：创建Docker化的游戏服务器

为你的UE5项目创建：
- Dockerfile构建脚本
- docker-compose.yml配置
- 本地测试部署

### 任务2：配置Kubernetes集群

部署一个包含以下组件的K8s集群：
- 3个游戏服务器Pod
- 1个匹配服务器
- Redis缓存
- PostgreSQL数据库

### 任务3：实现自动扩缩容

配置HPA实现：
- CPU使用率超过70%时扩容
- 队列长度超过100时扩容
- 最少2个副本，最多10个副本

---

## 23.8 总结

本课学习了：
- Docker容器化UE5服务器
- 多阶段构建优化镜像大小
- Docker Compose本地开发环境
- Kubernetes集群部署配置
- Agones游戏服务器管理
- 自动扩缩容设计
- 日志收集与监控
- CI/CD流水线配置

**下一课预告**：微服务架构设计 - 游戏后端微服务拆分与服务间通信。

---

*课程版本：1.0*
*最后更新：2026-04-10*
