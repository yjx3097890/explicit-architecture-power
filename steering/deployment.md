# 项目部署指南

本指南定义了 Go 后端项目的部署方案，包含 Docker Compose 和 Kubernetes 两种方式。

---

## deploy 目录结构

```text
deploy/
├── docker-compose/              # Docker Compose 部署
│   ├── docker-compose.yaml      # 编排文件（后端 + MySQL + Redis）
│   ├── docker-compose.prod.yaml # 生产环境覆盖配置
│   ├── .env.example             # 环境变量模板
│   └── configs/                 # 挂载配置文件
│       └── config.yaml
├── k8s/                         # Kubernetes 部署
│   ├── test/                    # 测试环境
│   │   ├── app/
│   │   │   ├── app.yaml         # Deployment + Service + HPA
│   │   │   ├── configmap.yaml   # 配置文件
│   │   │   └── secret.yaml.example
│   │   └── argo.yaml            # Argo Workflows CI/CD
│   └── production/              # 生产环境
│       └── app/
│           ├── app.yaml
│           ├── configmap.yaml
│           ├── ingress.yaml
│           └── tls.yaml
├── .gitignore                   # 忽略 secret.yaml 等敏感文件
└── README.md                    # 部署文档
```

---

## 方案一：Docker Compose 部署

适用于开发环境、单机部署、小规模生产环境。

### deploy/docker-compose/.env.example

```bash
# 应用配置
APP_NAME={app_name}
APP_PORT=8080

# 数据库配置
DB_HOST=mysql
DB_PORT=3306
DB_NAME={db_name}
DB_USER=root
DB_PASSWORD=change_me_in_production

# Redis 配置（可选）
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=change_me_in_production

# JWT 配置
JWT_SECRET=change_me_in_production
```

### deploy/docker-compose/docker-compose.yaml

```yaml
version: "3.8"

services:
  app:
    build:
      context: ../../
      dockerfile: Dockerfile
    container_name: ${APP_NAME:-app}
    restart: unless-stopped
    ports:
      - "${APP_PORT:-8080}:8080"
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./configs/config.yaml:/app/configs/config.yaml:ro
      - app-logs:/app/logs
      - app-temp:/app/temp
    environment:
      - TZ=Asia/Shanghai
      - GIN_MODE=release
    networks:
      - backend

  mysql:
    image: mysql:8.0
    container_name: ${APP_NAME:-app}-mysql
    restart: unless-stopped
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_CHARSET: utf8mb4
      TZ: Asia/Shanghai
    volumes:
      - mysql-data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

  redis:
    image: redis:7-alpine
    container_name: ${APP_NAME:-app}-redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    command: redis-server --requirepass ${REDIS_PASSWORD} --appendonly yes
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

volumes:
  mysql-data:
  redis-data:
  app-logs:
  app-temp:

networks:
  backend:
    driver: bridge
```

### deploy/docker-compose/docker-compose.prod.yaml

生产环境覆盖配置（不暴露数据库端口、调整资源限制）：

```yaml
version: "3.8"

services:
  app:
    restart: always
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 2G
        reservations:
          cpus: "0.5"
          memory: 512M

  mysql:
    ports: []  # 生产环境不暴露数据库端口
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 4G

  redis:
    ports: []  # 生产环境不暴露 Redis 端口
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 1G
```

### deploy/docker-compose/configs/config.yaml

```yaml
server:
  host: "0.0.0.0"
  port: 8080
  mode: "release"

database:
  driver: "mysql"
  host: "mysql"
  port: 3306
  username: "root"
  password: "${DB_PASSWORD}"
  database: "${DB_NAME}"
  charset: "utf8mb4"
  max_idle_conns: 10
  max_open_conns: 100

redis:
  host: "redis"
  port: 6379
  password: "${REDIS_PASSWORD}"
  db: 0

jwt:
  secret: "${JWT_SECRET}"
  expire_time: 24h
  issuer: "{app_name}"

log:
  level: "info"
  format: "json"
  output: "file"
  file_path: "logs/app.log"
```

### Docker Compose 使用方式

```bash
# 开发环境启动
cp deploy/docker-compose/.env.example deploy/docker-compose/.env
# 编辑 .env 填入实际配置
docker compose -f deploy/docker-compose/docker-compose.yaml --env-file deploy/docker-compose/.env up -d

# 生产环境启动（叠加 prod 覆盖）
docker compose \
  -f deploy/docker-compose/docker-compose.yaml \
  -f deploy/docker-compose/docker-compose.prod.yaml \
  --env-file deploy/docker-compose/.env up -d

# 查看日志
docker compose -f deploy/docker-compose/docker-compose.yaml logs -f app

# 停止服务
docker compose -f deploy/docker-compose/docker-compose.yaml down

# 停止并清除数据卷
docker compose -f deploy/docker-compose/docker-compose.yaml down -v
```

---

## 方案二：Kubernetes 部署

适用于集群化生产环境，支持自动扩缩容、滚动更新、Argo Workflows CI/CD。

### deploy/.gitignore

```gitignore
# 忽略包含敏感信息的 Secret 文件
**/secret.yaml

# 保留示例文件
!**/secret.yaml.example
```

### K8s 测试环境

#### deploy/k8s/test/app/app.yaml

包含 Namespace、Deployment、Service（NodePort）、HPA：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: {namespace}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {app_name}
  namespace: {namespace}
  labels:
    app: {app_name}
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: {app_name}
  template:
    metadata:
      labels:
        app: {app_name}
        version: v1
    spec:
      containers:
        - name: {app_name}
          image: ${IMAGE}
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          resources:
            limits:
              cpu: 2000m
              memory: 4Gi
            requests:
              cpu: 1000m
              memory: 2Gi
          env:
            - name: TZ
              value: Asia/Shanghai
            - name: GIN_MODE
              value: release
          volumeMounts:
            - name: app-config
              mountPath: /app/configs/config.yaml
              subPath: config.yaml
              readOnly: true
            - name: logs
              mountPath: /app/logs
            - name: temp
              mountPath: /app/temp
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
      volumes:
        - name: app-config
          configMap:
            name: {app_name}-config
        - name: logs
          emptyDir: {}
        - name: temp
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: {app_name}-service
  namespace: {namespace}
spec:
  type: NodePort
  selector:
    app: {app_name}
  ports:
    - name: http
      protocol: TCP
      port: 8080
      targetPort: 8080
      nodePort: {node_port}
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {app_name}-hpa
  namespace: {namespace}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {app_name}
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

#### deploy/k8s/test/app/secret.yaml.example

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {app_name}-secrets
  namespace: {namespace}
type: Opaque
stringData:
  DB_USERNAME: "admin"
  DB_PASSWORD: "your-database-password"
  REDIS_PASSWORD: "your-redis-password"
  JWT_SECRET: "your-jwt-secret"
```

#### deploy/k8s/test/argo.yaml

Argo Workflows CI/CD 工作流（clone → build → deploy 三步流水线）：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: {app_name}-
  namespace: argo
spec:
  entrypoint: git-build-deploy
  arguments:
    parameters:
      - name: app-name
        value: {app_name}
      - name: git-url
        value: {git_url}
      - name: container-registry
        value: {registry}/{app_name}:latest
      - name: git-branch
        value: develop

  templates:
    - name: git-build-deploy
      steps:
        - - name: clone-repo
            template: clone-git
        - - name: build-image
            template: build-docker
        - - name: deploy
            template: deploy-k8s

    - name: clone-git
      # Git clone 步骤（需要 SSH Secret）
      # ...

    - name: build-docker
      # Docker build + push 步骤（需要 Registry 凭证）
      # ...

    - name: deploy-k8s
      # kubectl apply 步骤
      # ...
```

### K8s 生产环境

#### deploy/k8s/production/app/app.yaml

与测试环境类似，区别：
- Service 类型改为 `ClusterIP`（通过 Ingress 暴露）
- `imagePullPolicy: IfNotPresent`
- 资源限制适当调整
- HPA `minReplicas: 2`，`maxReplicas: 10`

#### deploy/k8s/production/app/ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {app_name}-ingress
  namespace: {namespace}
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
spec:
  tls:
    - hosts:
        - {domain}
      secretName: {app_name}-tls
  rules:
    - host: {domain}
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: {app_name}-service
                port:
                  number: 8080
```

---

## deploy/README.md 模板

```markdown
# {app_name} - 部署文档

## 部署方式

### Docker Compose（开发/单机）

\```bash
# 1. 复制环境变量
cp deploy/docker-compose/.env.example deploy/docker-compose/.env

# 2. 编辑 .env 填入实际配置

# 3. 启动服务
docker compose -f deploy/docker-compose/docker-compose.yaml --env-file deploy/docker-compose/.env up -d

# 4. 查看日志
docker compose -f deploy/docker-compose/docker-compose.yaml logs -f app
\```

### Kubernetes（集群）

\```bash
# 1. 创建命名空间
kubectl create namespace {namespace}

# 2. 创建 Secret
cp deploy/k8s/test/app/secret.yaml.example deploy/k8s/test/app/secret.yaml
kubectl apply -f deploy/k8s/test/app/secret.yaml

# 3. 应用配置和部署
kubectl apply -f deploy/k8s/test/app/configmap.yaml
kubectl apply -f deploy/k8s/test/app/app.yaml

# 4. 验证
kubectl get pods -n {namespace}
\```
```

---

## Makefile 集成

在项目 Makefile 中添加部署相关目标：

```makefile
# ==================== 部署 ====================

## Docker Compose 启动（开发环境）
compose-up:
	docker compose -f deploy/docker-compose/docker-compose.yaml \
		--env-file deploy/docker-compose/.env up -d

## Docker Compose 停止
compose-down:
	docker compose -f deploy/docker-compose/docker-compose.yaml down

## Docker Compose 查看日志
compose-logs:
	docker compose -f deploy/docker-compose/docker-compose.yaml logs -f app

## Docker Compose 生产环境启动
compose-prod:
	docker compose \
		-f deploy/docker-compose/docker-compose.yaml \
		-f deploy/docker-compose/docker-compose.prod.yaml \
		--env-file deploy/docker-compose/.env up -d
```
