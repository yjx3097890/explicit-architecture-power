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

## Step 2: 创建项目结构

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

## Step 3: 初始化 Go Module

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

## Step 4: 创建配置文件

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

## Step 5: 创建共享基础代码

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

## Step 6: 创建第一个限界上下文

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

## Step 7: 创建 main.go

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

## Step 8: 自动迁移（可选）

在 main.go 中添加自动迁移：

```go
// 自动迁移表结构
if err := db.AutoMigrate(&persistence.{EntityName}Model{}); err != nil {
    log.Fatalf("数据库迁移失败: %v", err)
}
```

---

## Step 9: 创建 Dockerfile

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

## 完成

项目脚手架搭建完成后，提醒用户：

1. 运行 `go mod tidy` 整理依赖
2. 确保数据库已创建
3. 运行 `go run cmd/server/main.go` 启动服务
4. 使用 `curl` 或 Postman 测试 API
5. 构建 Docker 镜像：`docker build -t {project-name} .`
6. 运行容器：`docker run -p 8080:8080 {project-name}`
