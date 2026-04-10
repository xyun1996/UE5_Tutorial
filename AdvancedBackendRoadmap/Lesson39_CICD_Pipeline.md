# 第39课：CI/CD流水线

> 学习目标：掌握自动化构建、测试和部署流程的实现。

---

## 39.1 CI/CD架构设计

### 39.1.1 流水线概述

```
CI/CD流水线:

代码提交 -> 构建 -> 测试 -> 打包 -> 部署 -> 监控
    │         │       │       │       │       │
    ▼         ▼       ▼       ▼       ▼       ▼
  Git Hook  编译    单元测试  Docker镜像  Kubernetes  Prometheus
  Lint检查   链接    集成测试  制品上传    Helm部署    Grafana
  PR检查    资源    E2E测试  版本标记   金丝雀发布   告警
```

---

## 39.2 GitLab CI配置

### 39.2.1 完整流水线配置

```yaml
# .gitlab-ci.yml

stages:
  - lint
  - build
  - test
  - package
  - deploy

variables:
  DOCKER_REGISTRY: registry.gitlab.com
  IMAGE_PREFIX: arena-game
  KUBE_NAMESPACE: arena-production

# 全局配置
default:
  image: node:18-alpine
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
      - .npm/

# Lint阶段
lint:backend:
  stage: lint
  script:
    - cd services
    - npm ci
    - npm run lint
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"
    - if: $CI_COMMIT_BRANCH == "develop"

lint:game:
  stage: lint
  image: ue5-build:latest
  script:
    - ./Scripts/RunLinter.sh
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - changes:
        - Source/**/*
        - Content/**/*

# 构建阶段
build:services:
  stage: build
  script:
    - cd services
    - npm ci
    - npm run build
  artifacts:
    paths:
      - services/dist/
      - services/package.json
    expire_in: 1 week

build:game-server:
  stage: build
  image: ue5-build:latest
  tags:
    - ue5
    - linux
  script:
    - ./Scripts/BuildGameServer.sh
  artifacts:
    paths:
      - build/LinuxServer/
    expire_in: 1 week
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
    - if: $CI_COMMIT_BRANCH == "develop"
    - changes:
        - Source/**/*
        - Config/**/*

# 测试阶段
test:unit:
  stage: test
  script:
    - cd services
    - npm ci
    - npm run test:unit
  coverage: '/Lines\s*:\s*(\d+\.?\d*)%/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: services/coverage/cobertura-coverage.xml
      junit: services/test-results/junit.xml

test:integration:
  stage: test
  services:
    - postgres:15-alpine
    - redis:7-alpine
  variables:
    POSTGRES_DB: test_db
    POSTGRES_USER: test
    POSTGRES_PASSWORD: test
    DB_HOST: postgres
    REDIS_HOST: redis
  script:
    - cd services
    - npm ci
    - npm run test:integration
  artifacts:
    reports:
      junit: services/test-results/integration-junit.xml

test:e2e:
  stage: test
  script:
    - cd services
    - npm ci
    - npm run test:e2e
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
    - if: $CI_COMMIT_BRANCH == "develop"
  artifacts:
    when: always
    paths:
      - services/test-results/e2e/
    expire_in: 1 week

# 打包阶段
package:docker:
  stage: package
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $DOCKER_REGISTRY

    # 构建所有服务镜像
    - |
      for service in auth match game social leaderboard; do
        docker build \
          -t $DOCKER_REGISTRY/$IMAGE_PREFIX/${service}-service:$CI_COMMIT_SHA \
          -t $DOCKER_REGISTRY/$IMAGE_PREFIX/${service}-service:latest \
          -f services/${service}-service/Dockerfile \
          services/${service}-service
        docker push $DOCKER_REGISTRY/$IMAGE_PREFIX/${service}-service:$CI_COMMIT_SHA
        docker push $DOCKER_REGISTRY/$IMAGE_PREFIX/${service}-service:latest
      done
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
    - if: $CI_COMMIT_BRANCH == "develop"

package:game-server:
  stage: package
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $DOCKER_REGISTRY
    - docker build -t $DOCKER_REGISTRY/$IMAGE_PREFIX/game-server:$CI_COMMIT_SHA -f Dockerfile.game-server .
    - docker push $DOCKER_REGISTRY/$IMAGE_PREFIX/game-server:$CI_COMMIT_SHA
    - docker push $DOCKER_REGISTRY/$IMAGE_PREFIX/game-server:latest
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
    - if: $CI_COMMIT_BRANCH == "develop"
    - changes:
        - build/LinuxServer/**/*

# 部署阶段
deploy:staging:
  stage: deploy
  image: bitnami/kubectl:latest
  environment:
    name: staging
    url: https://staging.arena-game.com
  script:
    - kubectl config use-context staging-cluster
    - kubectl set image deployment/auth-service auth-service=$DOCKER_REGISTRY/$IMAGE_PREFIX/auth-service:$CI_COMMIT_SHA -n staging
    - kubectl set image deployment/match-service match-service=$DOCKER_REGISTRY/$IMAGE_PREFIX/match-service:$CI_COMMIT_SHA -n staging
    - kubectl set image deployment/game-service game-service=$DOCKER_REGISTRY/$IMAGE_PREFIX/game-service:$CI_COMMIT_SHA -n staging
    - kubectl rollout status deployment/auth-service -n staging
    - kubectl rollout status deployment/match-service -n staging
    - kubectl rollout status deployment/game-service -n staging
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"

deploy:production:
  stage: deploy
  image: bitnami/kubectl:latest
  environment:
    name: production
    url: https://arena-game.com
  script:
    - kubectl config use-context production-cluster

    # 使用Helm部署
    - helm upgrade --install arena ./helm/arena \
        --namespace $KUBE_NAMESPACE \
        --set image.tag=$CI_COMMIT_SHA \
        --set image.registry=$DOCKER_REGISTRY/$IMAGE_PREFIX \
        --values ./helm/arena/values-production.yaml
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
  when: manual

# 金丝雀部署
deploy:canary:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl config use-context production-cluster
    - kubectl apply -f k8s/canary.yaml
    - kubectl set image deployment/arena-canary arena=$DOCKER_REGISTRY/$IMAGE_PREFIX/game-server:$CI_COMMIT_SHA -n $KUBE_NAMESPACE
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
  when: manual
  allow_failure: true
```

---

## 39.3 GitHub Actions配置

### 39.3.1 工作流配置

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  release:
    types: [ published ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # 代码质量检查
  code-quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
          cache-dependency-path: services/package-lock.json

      - name: Install dependencies
        run: cd services && npm ci

      - name: Run ESLint
        run: cd services && npm run lint

      - name: Run TypeScript check
        run: cd services && npm run typecheck

      - name: SonarCloud Scan
        uses: SonarSource/sonarcloud-github-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

  # 测试
  test:
    runs-on: ubuntu-latest
    needs: code-quality

    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
          cache-dependency-path: services/package-lock.json

      - name: Install dependencies
        run: cd services && npm ci

      - name: Run unit tests
        run: cd services && npm run test:unit -- --coverage
        env:
          DB_HOST: localhost
          DB_PORT: 5432
          DB_USER: test
          DB_PASSWORD: test
          DB_NAME: test_db
          REDIS_HOST: localhost

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./services/coverage/lcov.info
          fail_ci_if_error: true

      - name: Run integration tests
        run: cd services && npm run test:integration
        env:
          DB_HOST: localhost
          REDIS_HOST: localhost

  # 构建镜像
  build:
    runs-on: ubuntu-latest
    needs: test
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha,prefix=
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - name: Build and push auth service
        uses: docker/build-push-action@v5
        with:
          context: ./services/auth-service
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/auth-service:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Build and push all services
        run: |
          for service in auth match game social leaderboard; do
            docker build \
              -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/${service}-service:${{ github.sha }} \
              ./services/${service}-service
            docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/${service}-service:${{ github.sha }}
          done

  # 部署到预发布环境
  deploy-staging:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging.arena-game.com

    steps:
      - uses: actions/checkout@v4

      - name: Configure kubectl
        uses: azure/k8s-set-context@v3
        with:
          kubeconfig: ${{ secrets.KUBE_CONFIG_STAGING }}

      - name: Deploy to staging
        run: |
          kubectl set image deployment/auth-service \
            auth-service=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/auth-service:${{ github.sha }} \
            -n staging
          kubectl rollout status deployment/auth-service -n staging --timeout=300s

  # 部署到生产环境
  deploy-production:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://arena-game.com

    steps:
      - uses: actions/checkout@v4

      - name: Configure kubectl
        uses: azure/k8s-set-context@v3
        with:
          kubeconfig: ${{ secrets.KUBE_CONFIG_PRODUCTION }}

      - name: Deploy to production
        run: |
          helm upgrade --install arena ./helm/arena \
            --namespace production \
            --set image.tag=${{ github.sha }} \
            --values ./helm/arena/values-production.yaml

      - name: Verify deployment
        run: |
          kubectl rollout status deployment/auth-service -n production --timeout=300s
          kubectl rollout status deployment/match-service -n production --timeout=300s
```

---

## 39.4 构建脚本

### 39.4.1 游戏服务器构建脚本

```bash
#!/bin/bash
# Scripts/BuildGameServer.sh

set -e

# 配置
PROJECT_NAME="ArenaChampions"
UE_PATH="${UE_PATH:-/opt/UnrealEngine}"
BUILD_CONFIGURATION="${BUILD_CONFIGURATION:-Development}"
TARGET_PLATFORM="Linux"
OUTPUT_DIR="build/LinuxServer"

echo "========================================"
echo "Building ${PROJECT_NAME} Dedicated Server"
echo "========================================"

# 清理旧构建
echo "Cleaning previous build..."
rm -rf ${OUTPUT_DIR}

# 生成项目文件
echo "Generating project files..."
${UE_PATH}/Engine/Build/BatchFiles/Linux/GenerateProjectFiles.sh \
    -project="$(pwd)/${PROJECT_NAME}.uproject" \
    -game \
    -engine

# 构建服务器
echo "Building server..."
${UE_PATH}/Engine/Build/BatchFiles/Linux/Build.sh \
    ${PROJECT_NAME}Server \
    Linux \
    ${BUILD_CONFIGURATION} \
    "$(pwd)/${PROJECT_NAME}.uproject" \
    -waitmutex

# 构建客户端（用于Cook）
echo "Building client for cooking..."
${UE_PATH}/Engine/Build/BatchFiles/Linux/Build.sh \
    ${PROJECT_NAME} \
    Linux \
    ${BUILD_CONFIGURATION} \
    "$(pwd)/${PROJECT_NAME}.uproject" \
    -waitmutex

# Cook内容
echo "Cooking content..."
${UE_PATH}/Engine/Binaries/Linux/UE4Editor-Cmd \
    "$(pwd)/${PROJECT_NAME}.uproject" \
    -run=Cook \
    -targetplatform=${TARGET_PLATFORM} \
    -cookonthefly \
    -unversioned \
    -stdout \
    -CrashForUAT \
    -unattended \
    -NoLogTimes \
    -UTF8Output

# 打包服务器
echo "Packaging server..."
${UE_PATH}/Engine/Build/BatchFiles/RunUAT.sh \
    BuildCookRun \
    -project="$(pwd)/${PROJECT_NAME}.uproject" \
    -noP4 \
    -platform=${TARGET_PLATFORM} \
    -clientconfig=${BUILD_CONFIGURATION} \
    -serverconfig=${BUILD_CONFIGURATION} \
    -cook \
    -build \
    -stage \
    -pak \
    -archive \
    -archivedirectory="$(pwd)/${OUTPUT_DIR}"

echo "========================================"
echo "Build completed successfully!"
echo "Output: ${OUTPUT_DIR}"
echo "========================================"

# 创建Docker构建上下文
echo "Creating Docker build context..."
cp Dockerfile.game-server ${OUTPUT_DIR}/Dockerfile

echo "Docker build context ready at ${OUTPUT_DIR}"
```

---

## 39.5 部署自动化

### 39.5.1 Helm Chart配置

```yaml
# helm/arena/Chart.yaml
apiVersion: v2
name: arena
description: Arena Game Backend Services
type: application
version: 1.0.0
appVersion: "1.0.0"

# helm/arena/values.yaml
replicaCount:
  auth: 2
  match: 2
  game: 3
  social: 1
  leaderboard: 1

image:
  registry: ghcr.io/arena-game
  tag: latest
  pullPolicy: Always

service:
  type: ClusterIP
  port: 80

resources:
  auth:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
  match:
    requests:
      cpu: 200m
      memory: 256Mi
    limits:
      cpu: 1000m
      memory: 1Gi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: api.arena-game.com
      paths:
        - path: /
          pathType: Prefix

postgresql:
  enabled: true
  auth:
    database: arena
    username: arena
    password: changeme

redis:
  enabled: true
  auth:
    enabled: true
    password: changeme
```

```yaml
# helm/arena/templates/deployment.yaml
{{- range $service := (list "auth" "match" "game" "social" "leaderboard") }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ $service }}-service
  labels:
    app: {{ $service }}-service
spec:
  replicas: {{ index $.Values.replicaCount $service }}
  selector:
    matchLabels:
      app: {{ $service }}-service
  template:
    metadata:
      labels:
        app: {{ $service }}-service
    spec:
      containers:
        - name: {{ $service }}
          image: "{{ $.Values.image.registry }}/{{ $service }}-service:{{ $.Values.image.tag }}"
          imagePullPolicy: {{ $.Values.image.pullPolicy }}
          ports:
            - containerPort: 8000
          env:
            - name: NODE_ENV
              value: production
            - name: DB_HOST
              value: "{{ $.Release.Name }}-postgresql"
            - name: DB_PORT
              value: "5432"
            - name: DB_NAME
              value: "{{ $.Values.postgresql.auth.database }}"
            - name: DB_USER
              value: "{{ $.Values.postgresql.auth.username }}"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: "{{ $.Release.Name }}-postgresql"
                  key: password
            - name: REDIS_HOST
              value: "{{ $.Release.Name }}-redis-master"
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: "{{ $.Release.Name }}-redis"
                  key: redis-password
          resources:
            {{- toYaml (index $.Values.resources $service) | nindent 12 }}
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 5
---
{{- end }}
```

---

## 39.6 实践任务

### 任务1：配置CI流水线

设置基本CI流程：
- 代码检查
- 单元测试
- 构建打包

### 任务2：配置CD流水线

设置部署流程：
- Docker镜像构建
- Kubernetes部署
- 滚动更新

### 任务3：实现金丝雀发布

配置金丝雀部署：
- 流量分割
- 自动回滚
- 监控集成

---

## 39.7 总结

本课完成了：
- CI/CD架构设计
- GitLab CI配置
- GitHub Actions配置
- 构建脚本实现
- Helm Chart配置
- 部署自动化

**下一课预告**：生产环境部署 - 完整的生产环境搭建和监控配置。

---

*课程版本：1.0*
*最后更新：2026-04-10*
