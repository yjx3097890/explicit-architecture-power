# 新项目脚手架搭建工作流

本指南帮助你从零开始创建一个符合显式架构规范的 Go 后端项目。

---

## Step 1: 收集项目信息

通过对话了解以下信息：

1. **项目名称**（Go module 名，如 `github.com/username/project-name`）
2. **数据库选择**：MySQL 还是 PostgreSQL？
3. **是否需要 Redis**？
4. **初始限界上下文**：第一个业务领域是什么？（如 user、order、product）
5. **API 端口**：默认 8080

---

## Step 2: Git 初始化与提交规范

### 初始化 Git 仓库

```bash
git init
```

### 创建 .gitignore

```gitignore
# 二进制文件
*.exe
*.exe~
*.dll
*.so
*.dylib
/server
/tmp

# 测试覆盖
*.test
*.out
coverage.*

# 依赖
/vendor/

# IDE
.idea/
.vscode/
*.swp
*.swo
*~

# 系统文件
.DS_Store
Thumbs.db

# 配置文件（敏感信息）
configs/config.local.yaml
.env
.env.local

# 日志
logs/
*.log
```

### 配置 commit-msg Hook（零依赖方案）

创建 `.githooks/commit-msg`：

```bash
#!/bin/sh
# commit-msg hook - 校验提交信息格式
# 格式: <type>(<scope>): <subject>  或  <type>: <subject>

commit_msg_file=$1
commit_msg=$(head -1 "$commit_msg_file")

# 允许的 type 列表
types="feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert"

# 正则匹配
pattern="^($types)(\([a-z0-9_-]+\))?: .{1,100}$"

if ! echo "$commit_msg" | grep -qE "$pattern"; then
    echo "❌ 提交信息格式错误！"
    echo ""
    echo "正确格式: <type>(<scope>): <subject>"
    echo "允许的 type: feat, fix, docs, style, refactor, perf, test, build, ci, chore, revert"
    echo ""
    echo "示例:"
    echo "  feat(user): 添加用户注册接口"
    echo "  fix: 修复数据库连接泄漏"
    echo ""
    echo "你的提交信息: $commit_msg"
    exit 1
fi

# 检查 header 长度
header_length=${#commit_msg}
if [ "$header_length" -gt 100 ]; then
    echo "❌ 提交信息标题过长！（${header_length}/100）"
    exit 1
fi
```

激活 Hook：

```bash
chmod +x .githooks/commit-msg
git config core.hooksPath .githooks
```

> Makefile 中的 `setup-hooks` 目标将在 Step 11 统一创建。

---

## Step 3: 创建项目结构

根据收集的信息，创建以下目录结构：

```bash
mkdir -p cmd/server
mkdir -p internal/{context_name}/domain
mkdir -p internal/{context_name}/app/command
mkdir -p internal/{context_name}/app/query
mkdir -p internal/{context_name}/infra/persistence
mkdir -p internal/{context_name}/infra/adapter
mkdir -p internal/{context_name}/ui
mkdir -p internal/pkg/response
mkdir -p internal/pkg/database
mkdir -p configs
mkdir -p api
mkdir -p deploy
```

---

## Step 4: 初始化 Go Module

```bash
go mod init {module_name}
```

安装核心依赖：

```bash
go get github.com/gin-gonic/gin
go get gorm.io/gorm
```

MySQL 驱动：
```bash
go get gorm.io/driver/mysql
```

PostgreSQL 驱动：
```bash
go get gorm.io/driver/postgres
```

Redis（如需要）：
```bash
go get github.com/redis/go-redis/v9
```

配置管理：
```bash
go get github.com/spf13/viper
```

---

## Step 5: 创建配置文件

### configs/config.yaml

MySQL 版本：
```yaml
server:
  port: 8080
  mode: debug  # debug / release

database:
  driver: mysql
  host: localhost
  port: 3306
  name: {db_name}
  user: root
  password: ""
  charset: utf8mb4
  max_idle_conns: 10
  max_open_conns: 100

# redis:
#   addr: localhost:6379
#   password: ""
#   db: 0
```

PostgreSQL 版本：
```yaml
server:
  port: 8080
  mode: debug

database:
  driver: postgres
  host: localhost
  port: 5432
  name: {db_name}
  user: postgres
  password: ""
  sslmode: disable
  max_idle_conns: 10
  max_open_conns: 100
```

---

## Step 6: 创建共享基础代码

### internal/pkg/response/response.go

统一 API 响应格式：

```go
package response

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

// Response 统一响应结构
type Response struct {
	Code    int         `json:"code"`
	Data    interface{} `json:"data"`
	Message string      `json:"message"`
}

// Success 成功响应
func Success(c *gin.Context, data interface{}) {
	c.JSON(http.StatusOK, Response{
		Code:    200,
		Data:    data,
		Message: "成功",
	})
}

// Error 错误响应
func Error(c *gin.Context, code int, message string) {
	c.JSON(code, Response{
		Code:    code,
		Data:    nil,
		Message: message,
	})
}

// BadRequest 400 错误
func BadRequest(c *gin.Context, message string) {
	Error(c, http.StatusBadRequest, message)
}

// NotFound 404 错误
func NotFound(c *gin.Context, message string) {
	Error(c, http.StatusNotFound, message)
}

// InternalError 500 错误
func InternalError(c *gin.Context, message string) {
	Error(c, http.StatusInternalServerError, message)
}
```

### internal/pkg/database/database.go

数据库初始化（支持 MySQL 和 PostgreSQL）：

```go
package database

import (
	"fmt"

	"gorm.io/driver/mysql"
	"gorm.io/driver/postgres"
	"gorm.io/gorm"
)

// Config 数据库配置
type Config struct {
	Driver       string
	Host         string
	Port         int
	Name         string
	User         string
	Password     string
	Charset      string // MySQL 专用
	SSLMode      string // PostgreSQL 专用
	MaxIdleConns int
	MaxOpenConns int
}

// NewConnection 创建数据库连接
func NewConnection(cfg Config) (*gorm.DB, error) {
	var dialector gorm.Dialector

	switch cfg.Driver {
	case "mysql":
		dsn := fmt.Sprintf("%s:%s@tcp(%s:%d)/%s?charset=%s&parseTime=True&loc=Local",
			cfg.User, cfg.Password, cfg.Host, cfg.Port, cfg.Name, cfg.Charset)
		dialector = mysql.Open(dsn)
	case "postgres":
		dsn := fmt.Sprintf("host=%s port=%d user=%s password=%s dbname=%s sslmode=%s",
			cfg.Host, cfg.Port, cfg.User, cfg.Password, cfg.Name, cfg.SSLMode)
		dialector = postgres.Open(dsn)
	default:
		return nil, fmt.Errorf("不支持的数据库驱动: %s", cfg.Driver)
	}

	db, err := gorm.Open(dialector, &gorm.Config{})
	if err != nil {
		return nil, fmt.Errorf("连接数据库失败: %w", err)
	}

	sqlDB, err := db.DB()
	if err != nil {
		return nil, fmt.Errorf("获取数据库实例失败: %w", err)
	}

	sqlDB.SetMaxIdleConns(cfg.MaxIdleConns)
	sqlDB.SetMaxOpenConns(cfg.MaxOpenConns)

	return db, nil
}
```

---

## Step 7: 创建第一个限界上下文

以下以 `{context_name}` 为例，展示各层的骨架代码。

### Domain 层

`internal/{context_name}/domain/entity.go`:
```go
package domain

import "time"

// {EntityName} 业务实体（纯净，不依赖任何第三方库）
type {EntityName} struct {
	ID        string
	Name      string
	CreatedAt time.Time
	UpdatedAt time.Time
}
```

`internal/{context_name}/domain/repository.go`:
```go
package domain

import "context"

// {EntityName}Repository 仓储接口（定义在 Domain 层，实现在 Infra 层）
type {EntityName}Repository interface {
	Save(ctx context.Context, entity *{EntityName}) error
	FindByID(ctx context.Context, id string) (*{EntityName}, error)
	Delete(ctx context.Context, id string) error
}
```

### Application 层

`internal/{context_name}/app/command/create_{context_name}.go`:
```go
package command

import (
	"context"
	"fmt"

	"{module}/internal/{context_name}/domain"
)

// Create{EntityName}Command 创建命令
type Create{EntityName}Command struct {
	Name string
}

// Create{EntityName}Handler 创建命令处理器
type Create{EntityName}Handler struct {
	repo domain.{EntityName}Repository
}

// NewCreate{EntityName}Handler 构造函数
func NewCreate{EntityName}Handler(repo domain.{EntityName}Repository) *Create{EntityName}Handler {
	return &Create{EntityName}Handler{repo: repo}
}

// Handle 处理创建命令
func (h *Create{EntityName}Handler) Handle(ctx context.Context, cmd Create{EntityName}Command) (*domain.{EntityName}, error) {
	entity := &domain.{EntityName}{
		Name: cmd.Name,
	}

	if err := h.repo.Save(ctx, entity); err != nil {
		return nil, fmt.Errorf("创建失败: %w", err)
	}

	return entity, nil
}
```

`internal/{context_name}/app/query/get_{context_name}.go`:
```go
package query

import (
	"context"
	"fmt"

	"{module}/internal/{context_name}/domain"
)

// Get{EntityName}Handler 查询处理器
type Get{EntityName}Handler struct {
	repo domain.{EntityName}Repository
}

// NewGet{EntityName}Handler 构造函数
func NewGet{EntityName}Handler(repo domain.{EntityName}Repository) *Get{EntityName}Handler {
	return &Get{EntityName}Handler{repo: repo}
}

// Handle 处理查询
func (h *Get{EntityName}Handler) Handle(ctx context.Context, id string) (*domain.{EntityName}, error) {
	entity, err := h.repo.FindByID(ctx, id)
	if err != nil {
		return nil, fmt.Errorf("查询失败: %w", err)
	}
	return entity, nil
}
```

### Infrastructure 层

`internal/{context_name}/infra/persistence/model.go`:
```go
package persistence

import (
	"time"

	"{module}/internal/{context_name}/domain"
	"gorm.io/gorm"
)

// {EntityName}Model 数据库模型（包含 ORM 细节，与 Domain Entity 严格分离）
type {EntityName}Model struct {
	ID        string `gorm:"primaryKey;type:varchar(36)"`
	Name      string `gorm:"type:varchar(255);not null"`
	CreatedAt time.Time
	UpdatedAt time.Time
	DeletedAt gorm.DeletedAt `gorm:"index"`
}

// TableName 表名
func ({EntityName}Model) TableName() string {
	return "{table_name}"
}

// ToDomain 模型转实体
func (m *{EntityName}Model) ToDomain() *domain.{EntityName} {
	return &domain.{EntityName}{
		ID:        m.ID,
		Name:      m.Name,
		CreatedAt: m.CreatedAt,
		UpdatedAt: m.UpdatedAt,
	}
}

// FromDomain 实体转模型
func FromDomain(entity *domain.{EntityName}) *{EntityName}Model {
	return &{EntityName}Model{
		ID:   entity.ID,
		Name: entity.Name,
	}
}
```

`internal/{context_name}/infra/persistence/repository.go`:
```go
package persistence

import (
	"context"
	"fmt"

	"{module}/internal/{context_name}/domain"
	"github.com/google/uuid"
	"gorm.io/gorm"
)

// {EntityName}RepositoryImpl 仓储实现
type {EntityName}RepositoryImpl struct {
	db *gorm.DB
}

// New{EntityName}Repository 构造函数
func New{EntityName}Repository(db *gorm.DB) domain.{EntityName}Repository {
	return &{EntityName}RepositoryImpl{db: db}
}

// Save 保存实体
func (r *{EntityName}RepositoryImpl) Save(ctx context.Context, entity *domain.{EntityName}) error {
	model := FromDomain(entity)
	if model.ID == "" {
		model.ID = uuid.New().String()
	}

	if err := r.db.WithContext(ctx).Save(model).Error; err != nil {
		return fmt.Errorf("保存失败: %w", err)
	}

	entity.ID = model.ID
	return nil
}

// FindByID 根据 ID 查找
func (r *{EntityName}RepositoryImpl) FindByID(ctx context.Context, id string) (*domain.{EntityName}, error) {
	var model {EntityName}Model
	if err := r.db.WithContext(ctx).Where("id = ?", id).First(&model).Error; err != nil {
		return nil, fmt.Errorf("查询失败: %w", err)
	}
	return model.ToDomain(), nil
}

// Delete 删除实体
func (r *{EntityName}RepositoryImpl) Delete(ctx context.Context, id string) error {
	if err := r.db.WithContext(ctx).Where("id = ?", id).Delete(&{EntityName}Model{}).Error; err != nil {
		return fmt.Errorf("删除失败: %w", err)
	}
	return nil
}
```

### UI 层

`internal/{context_name}/ui/dto.go`:
```go
package ui

// Create{EntityName}Request 创建请求 DTO
type Create{EntityName}Request struct {
	Name string `json:"name" binding:"required"`
}

// {EntityName}Response 响应 DTO
type {EntityName}Response struct {
	ID        string `json:"id"`
	Name      string `json:"name"`
	CreatedAt string `json:"created_at"`
}
```

`internal/{context_name}/ui/handler.go`:
```go
package ui

import (
	"github.com/gin-gonic/gin"

	"{module}/internal/{context_name}/app/command"
	"{module}/internal/{context_name}/app/query"
	"{module}/internal/pkg/response"
)

// Handler HTTP 请求处理器
type Handler struct {
	createHandler *command.Create{EntityName}Handler
	getHandler    *query.Get{EntityName}Handler
}

// NewHandler 构造函数
func NewHandler(
	createHandler *command.Create{EntityName}Handler,
	getHandler *query.Get{EntityName}Handler,
) *Handler {
	return &Handler{
		createHandler: createHandler,
		getHandler:    getHandler,
	}
}

// RegisterRoutes 注册路由
func (h *Handler) RegisterRoutes(r *gin.RouterGroup) {
	group := r.Group("/{context_name}s")
	{
		group.POST("", h.Create)
		group.GET("/:id", h.GetByID)
	}
}

// Create 创建
func (h *Handler) Create(c *gin.Context) {
	var req Create{EntityName}Request
	if err := c.ShouldBindJSON(&req); err != nil {
		response.BadRequest(c, "参数错误: "+err.Error())
		return
	}

	cmd := command.Create{EntityName}Command{
		Name: req.Name,
	}

	entity, err := h.createHandler.Handle(c.Request.Context(), cmd)
	if err != nil {
		response.InternalError(c, "创建失败")
		return
	}

	response.Success(c, {EntityName}Response{
		ID:        entity.ID,
		Name:      entity.Name,
		CreatedAt: entity.CreatedAt.Format("2006-01-02 15:04:05"),
	})
}

// GetByID 根据 ID 查询
func (h *Handler) GetByID(c *gin.Context) {
	id := c.Param("id")

	entity, err := h.getHandler.Handle(c.Request.Context(), id)
	if err != nil {
		response.NotFound(c, "未找到")
		return
	}

	response.Success(c, {EntityName}Response{
		ID:        entity.ID,
		Name:      entity.Name,
		CreatedAt: entity.CreatedAt.Format("2006-01-02 15:04:05"),
	})
}
```

---

## Step 8: 创建 main.go

`cmd/server/main.go`:
```go
package main

import (
	"log"

	"github.com/gin-gonic/gin"

	"{module}/internal/{context_name}/app/command"
	"{module}/internal/{context_name}/app/query"
	"{module}/internal/{context_name}/infra/persistence"
	"{module}/internal/{context_name}/ui"
	"{module}/internal/pkg/database"
)

func main() {
	// 1. 初始化数据库
	db, err := database.NewConnection(database.Config{
		Driver:       "mysql", // 或 "postgres"
		Host:         "localhost",
		Port:         3306,
		Name:         "{db_name}",
		User:         "root",
		Password:     "",
		Charset:      "utf8mb4",
		MaxIdleConns: 10,
		MaxOpenConns: 100,
	})
	if err != nil {
		log.Fatalf("数据库连接失败: %v", err)
	}

	// 2. 依赖注入：Infrastructure → Application
	repo := persistence.New{EntityName}Repository(db)
	createHandler := command.NewCreate{EntityName}Handler(repo)
	getHandler := query.NewGet{EntityName}Handler(repo)

	// 3. 创建 HTTP Handler
	handler := ui.NewHandler(createHandler, getHandler)

	// 4. 启动 Gin
	r := gin.Default()
	api := r.Group("/api/v1")
	handler.RegisterRoutes(api)

	log.Println("服务启动在 :8080")
	if err := r.Run(":8080"); err != nil {
		log.Fatalf("服务启动失败: %v", err)
	}
}
```

---

## Step 9: 自动迁移（可选）

在 main.go 中添加自动迁移：

```go
// 自动迁移表结构
if err := db.AutoMigrate(&persistence.{EntityName}Model{}); err != nil {
    log.Fatalf("数据库迁移失败: %v", err)
}
```

---

## Step 10: 创建 Dockerfile

### Dockerfile（多阶段构建）

```dockerfile
# 构建阶段
FROM golang:1.21-alpine AS builder

WORKDIR /app

# 安装依赖
COPY go.mod go.sum ./
RUN go mod download

# 复制源码并构建
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server ./cmd/server

# 运行阶段
FROM alpine:3.19

RUN apk --no-cache add ca-certificates tzdata

WORKDIR /app

# 从构建阶段复制二进制文件
COPY --from=builder /app/server .
COPY --from=builder /app/configs ./configs

# 设置时区
ENV TZ=Asia/Shanghai

EXPOSE 8080

CMD ["./server"]
```

### .dockerignore

```
.git
.gitignore
.idea
.vscode
.kiro
*.md
!README.md
tmp/
vendor/
*.test
*.out
coverage.*
docker-compose*.yml
deploy/
docs/
```

---

## Step 11: 创建 deploy 目录

参考 **deployment** steering 文件创建完整的部署配置。

```bash
mkdir -p deploy/docker-compose/configs
mkdir -p deploy/k8s/test/app
mkdir -p deploy/k8s/production/app
```

创建以下文件：

- `deploy/.gitignore` — 忽略 `secret.yaml` 等敏感文件
- `deploy/docker-compose/.env.example` — 环境变量模板
- `deploy/docker-compose/docker-compose.yaml` — Docker Compose 编排（后端 + MySQL + Redis）
- `deploy/docker-compose/docker-compose.prod.yaml` — 生产环境覆盖配置
- `deploy/docker-compose/configs/config.yaml` — 应用配置文件
- `deploy/k8s/test/app/app.yaml` — K8s 测试环境（Deployment + Service + HPA）
- `deploy/k8s/test/app/configmap.yaml` — K8s ConfigMap
- `deploy/k8s/test/app/secret.yaml.example` — K8s Secret 模板
- `deploy/k8s/test/argo.yaml` — Argo Workflows CI/CD 工作流
- `deploy/k8s/production/app/app.yaml` — K8s 生产环境
- `deploy/k8s/production/app/ingress.yaml` — Ingress 配置
- `deploy/README.md` — 部署文档

> 具体文件内容参见 `deployment` steering 文件中的模板。

---

## Step 12: 创建 Makefile

Makefile 是 Go 后端项目的统一入口，所有构建、测试、格式化、Git Hooks 等操作都通过 `make` 命令管理。

```makefile
# {app_name} Makefile

# ==================== 变量定义 ====================
APP_NAME={app_name}
VERSION=1.0.0
BUILD_DIR=bin
MAIN_PATH=./cmd/server

GOCMD=go
GOBUILD=$(GOCMD) build
GOCLEAN=$(GOCMD) clean
GOTEST=$(GOCMD) test
GOMOD=$(GOCMD) mod
GOFMT=$(GOCMD) fmt

LDFLAGS=-ldflags "-s -w -X main.Version=$(VERSION)"

.PHONY: all build clean test test-coverage deps fmt lint run dev \
        setup-hooks verify-hooks install-tools \
        docker-build docker-run quality-check release help

# ==================== 默认目标 ====================

## 完整构建流程（含 Hook 安装）
all: setup-hooks clean deps fmt lint test build

# ==================== 构建 ====================

## 构建应用
build:
	@echo "构建应用..."
	@mkdir -p $(BUILD_DIR)
	$(GOBUILD) $(LDFLAGS) -o $(BUILD_DIR)/$(APP_NAME) $(MAIN_PATH)
	@echo "✅ 构建完成: $(BUILD_DIR)/$(APP_NAME)"

## 清理构建文件
clean:
	@echo "清理构建文件..."
	$(GOCLEAN)
	@rm -rf $(BUILD_DIR)
	@echo "✅ 清理完成"

# ==================== 测试 ====================

## 运行测试
test:
	@echo "运行测试..."
	$(GOTEST) -v ./...

## 运行测试并生成覆盖率报告
test-coverage:
	@echo "运行测试并生成覆盖率报告..."
	$(GOTEST) -v -coverprofile=coverage.out ./...
	$(GOCMD) tool cover -html=coverage.out -o coverage.html
	@echo "✅ 覆盖率报告: coverage.html"

# ==================== 代码质量 ====================

## 下载并整理依赖
deps:
	@echo "下载依赖..."
	$(GOMOD) download
	$(GOMOD) tidy

## 格式化代码
fmt:
	@echo "格式化代码..."
	$(GOFMT) ./...

## 代码检查（需要安装 golangci-lint）
lint:
	@echo "运行代码检查..."
	golangci-lint run

## 代码质量完整检查
quality-check: fmt lint test
	@echo "✅ 代码质量检查完成"

# ==================== 运行 ====================

## 运行应用
run:
	@echo "运行应用..."
	$(GOCMD) run $(MAIN_PATH)/main.go

## 开发模式运行（热重载，需要 air）
dev:
	@echo "开发模式运行..."
	air

# ==================== Git Hooks ====================

## 安装 Git Hooks（新成员 clone 后首先执行）
setup-hooks:
	@mkdir -p .githooks
	@chmod +x .githooks/commit-msg
	@git config core.hooksPath .githooks
	@echo "✅ Git Hooks 已安装"

## 验证 Git Hooks 是否正常工作
verify-hooks:
	@echo "验证 commit-msg hook..."
	@echo "feat: test message" | .githooks/commit-msg /dev/stdin && echo "✅ Hook 校验通过" || echo "❌ Hook 校验失败"

# ==================== 开发工具 ====================

## 安装开发工具（含 Hook 安装）
install-tools: setup-hooks
	@echo "安装开发工具..."
	go install github.com/air-verse/air@latest
	go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
	@echo "✅ 开发工具安装完成"

# ==================== Docker ====================

## 构建 Docker 镜像
docker-build:
	@echo "构建 Docker 镜像..."
	docker build -t $(APP_NAME):$(VERSION) .
	docker tag $(APP_NAME):$(VERSION) $(APP_NAME):latest
	@echo "✅ Docker 镜像构建完成"

## 运行 Docker 容器
docker-run:
	@echo "运行 Docker 容器..."
	docker run -p 8080:8080 --name $(APP_NAME) $(APP_NAME):latest

# ==================== Docker Compose 部署 ====================

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

# ==================== 发布 ====================

## 发布准备
release: clean deps quality-check build
	@echo "✅ 发布准备完成"

# ==================== 帮助 ====================

## 显示帮助信息
help:
	@echo "$(APP_NAME) Makefile 使用说明"
	@echo ""
	@echo "可用命令:"
	@echo "  make all              完整构建流程（含 Hook 安装）"
	@echo "  make build            构建应用"
	@echo "  make clean            清理构建文件"
	@echo "  make test             运行测试"
	@echo "  make test-coverage    运行测试并生成覆盖率报告"
	@echo "  make deps             下载并整理依赖"
	@echo "  make fmt              格式化代码"
	@echo "  make lint             代码检查"
	@echo "  make quality-check    代码质量完整检查"
	@echo "  make run              运行应用"
	@echo "  make dev              开发模式运行（热重载）"
	@echo "  make setup-hooks      安装 Git Hooks（提交信息格式校验）"
	@echo "  make verify-hooks     验证 Git Hooks 是否正常工作"
	@echo "  make install-tools    安装开发工具"
	@echo "  make docker-build     构建 Docker 镜像"
	@echo "  make docker-run       运行 Docker 容器"
	@echo "  make compose-up       Docker Compose 启动（开发环境）"
	@echo "  make compose-down     Docker Compose 停止"
	@echo "  make compose-logs     Docker Compose 查看日志"
	@echo "  make compose-prod     Docker Compose 生产环境启动"
	@echo "  make release          发布准备"
	@echo "  make help             显示帮助信息"
```

---

## 完成

项目脚手架搭建完成后，提醒用户：

1. 运行 `make setup-hooks` 安装 Git Hooks
2. 运行 `make install-tools` 安装开发工具（air、golangci-lint）
3. 运行 `make deps` 下载并整理依赖
4. 确保数据库已创建
5. 运行 `make run` 启动服务（或 `make dev` 热重载开发）
6. 运行 `make quality-check` 提交前代码质量检查
7. Docker Compose 部署：`cp deploy/docker-compose/.env.example deploy/docker-compose/.env && make compose-up`
8. 运行 `make help` 查看所有可用命令
9. 构建 Docker 镜像：`make docker-build`
