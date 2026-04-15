# 各层详细规范与代码示例

本文档详细说明显式架构中每一层的职责、规则和代码示例。

---

## 1. Domain Layer（业务领域层）

### 定位

系统的核心，包含业务逻辑和状态。所有其他层围绕 Domain 层构建。

### 规则

- **纯净性**：只依赖 Go 标准库。**严禁**引入 `gorm`、`gin`、`sql` 等第三方库
- **结构体**：定义纯粹的 Entity 和 Value Object
  - 禁止使用：`gorm.Model`、`gorm tag`、`sql.NullString`
  - JSON tag 仅在业务序列化确实需要时使用
- **接口定义**：定义 Repository 和 Port 接口，但不实现它们
- **业务方法**：Entity 上的方法只包含业务逻辑

### 代码示例

#### Entity 定义

```go
// internal/{context}/domain/entity.go
package domain

import "time"

// User 用户业务实体（纯净，不依赖任何第三方库）
type User struct {
    ID        string
    Name      string
    Email     string
    Status    UserStatus  // Value Object
    Profile   Profile     // Value Object
    CreatedAt time.Time
    UpdatedAt time.Time
}

// Activate 激活用户（业务方法）
func (u *User) Activate() error {
    if u.Status == StatusActive {
        return ErrAlreadyActive
    }
    u.Status = StatusActive
    return nil
}

// IsActive 是否已激活
func (u *User) IsActive() bool {
    return u.Status == StatusActive
}
```

#### Value Object 定义

```go
// internal/{context}/domain/value_objects.go
package domain

// UserStatus 用户状态（Value Object）
type UserStatus string

const (
    StatusPending  UserStatus = "pending"
    StatusActive   UserStatus = "active"
    StatusDisabled UserStatus = "disabled"
)

// Profile 用户资料（Value Object，复杂嵌套结构）
type Profile struct {
    Avatar   string
    Bio      string
    Settings map[string]interface{}
}
```

#### Repository 接口

```go
// internal/{context}/domain/repository.go
package domain

import "context"

// UserRepository 仓储接口（定义在 Domain 层，实现在 Infra 层）
type UserRepository interface {
    Save(ctx context.Context, user *User) error
    FindByID(ctx context.Context, id string) (*User, error)
    FindByEmail(ctx context.Context, email string) (*User, error)
    Delete(ctx context.Context, id string) error
    List(ctx context.Context, offset, limit int) ([]*User, int64, error)
}
```

#### Port 接口（外部服务）

```go
// internal/{context}/domain/ports.go
package domain

import "context"

// EmailSenderPort 邮件发送端口（外部服务接口）
type EmailSenderPort interface {
    SendWelcomeEmail(ctx context.Context, email, name string) error
}

// CachePort 缓存端口
type CachePort interface {
    Get(ctx context.Context, key string) (string, error)
    Set(ctx context.Context, key, value string, ttl int) error
    Delete(ctx context.Context, key string) error
}
```

#### 错误定义

```go
// internal/{context}/domain/errors.go
package domain

import "errors"

var (
    ErrNotFound      = errors.New("资源未找到")
    ErrAlreadyExists = errors.New("资源已存在")
    ErrAlreadyActive = errors.New("用户已激活")
    ErrInvalidInput  = errors.New("无效的输入")
)
```

---

## 2. Application Layer（应用服务层）

### 定位

编排业务流程，作为 Domain 的入口。采用 CQRS 模式，Command（写）和 Query（读）分离。

### 规则

- 每个业务动作对应一个 Handler
- 接受 Command/Query 结构体，调用 Domain 方法
- 通过 Repository 接口持久化
- **不含 HTTP**：不处理 HTTP Request/Response
- 通过构造函数注入依赖

### 代码示例

#### Command（写操作）

```go
// internal/{context}/app/command/create_user.go
package command

import (
    "context"
    "fmt"

    "{module}/internal/{context}/domain"
)

// CreateUserCommand 创建用户命令
type CreateUserCommand struct {
    Name  string
    Email string
}

// CreateUserHandler 创建用户处理器
type CreateUserHandler struct {
    repo        domain.UserRepository
    emailSender domain.EmailSenderPort  // 可选的外部服务
}

// NewCreateUserHandler 构造函数（依赖注入）
func NewCreateUserHandler(repo domain.UserRepository, emailSender domain.EmailSenderPort) *CreateUserHandler {
    return &CreateUserHandler{
        repo:        repo,
        emailSender: emailSender,
    }
}

// Handle 处理创建用户命令
func (h *CreateUserHandler) Handle(ctx context.Context, cmd CreateUserCommand) (*domain.User, error) {
    // 1. 检查是否已存在
    existing, _ := h.repo.FindByEmail(ctx, cmd.Email)
    if existing != nil {
        return nil, fmt.Errorf("邮箱已注册: %w", domain.ErrAlreadyExists)
    }

    // 2. 创建 Domain Entity
    user := &domain.User{
        Name:   cmd.Name,
        Email:  cmd.Email,
        Status: domain.StatusPending,
    }

    // 3. 持久化
    if err := h.repo.Save(ctx, user); err != nil {
        return nil, fmt.Errorf("保存用户失败: %w", err)
    }

    // 4. 发送欢迎邮件（可选，失败不影响主流程）
    if h.emailSender != nil {
        _ = h.emailSender.SendWelcomeEmail(ctx, user.Email, user.Name)
    }

    return user, nil
}
```

#### Query（读操作）

```go
// internal/{context}/app/query/get_user.go
package query

import (
    "context"
    "fmt"

    "{module}/internal/{context}/domain"
)

// GetUserHandler 查询用户处理器
type GetUserHandler struct {
    repo domain.UserRepository
}

// NewGetUserHandler 构造函数
func NewGetUserHandler(repo domain.UserRepository) *GetUserHandler {
    return &GetUserHandler{repo: repo}
}

// Handle 根据 ID 查询用户
func (h *GetUserHandler) Handle(ctx context.Context, id string) (*domain.User, error) {
    user, err := h.repo.FindByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("查询用户失败: %w", err)
    }
    return user, nil
}
```

```go
// internal/{context}/app/query/list_users.go
package query

import (
    "context"
    "fmt"

    "{module}/internal/{context}/domain"
)

// ListUsersQuery 列表查询参数
type ListUsersQuery struct {
    Page     int
    PageSize int
}

// ListUsersHandler 列表查询处理器
type ListUsersHandler struct {
    repo domain.UserRepository
}

// NewListUsersHandler 构造函数
func NewListUsersHandler(repo domain.UserRepository) *ListUsersHandler {
    return &ListUsersHandler{repo: repo}
}

// ListUsersResult 列表查询结果
type ListUsersResult struct {
    Items []*domain.User
    Total int64
}

// Handle 处理列表查询
func (h *ListUsersHandler) Handle(ctx context.Context, q ListUsersQuery) (*ListUsersResult, error) {
    offset := (q.Page - 1) * q.PageSize
    items, total, err := h.repo.List(ctx, offset, q.PageSize)
    if err != nil {
        return nil, fmt.Errorf("查询列表失败: %w", err)
    }
    return &ListUsersResult{Items: items, Total: total}, nil
}
```

---

## 3. Infrastructure Layer（基础设施层）

### 定位

实现 Domain 定义的接口，处理数据库、外部 API、缓存等。

### A. Persistence（持久化）

#### 规则

- 定义独立的 Model（PO），包含 ORM 细节
- 复杂嵌套结构使用数据库 JSON 类型
- 必须显式编写 `ToDomain()` 和 `FromDomain()` Mapper
- Repository 实现返回 Domain Entity，不暴露 Model

#### 代码示例

```go
// internal/{context}/infra/persistence/model.go
package persistence

import (
    "encoding/json"
    "time"

    "{module}/internal/{context}/domain"
    "gorm.io/gorm"
)

// UserModel 数据库模型（包含 ORM 细节，与 Domain Entity 严格分离）
type UserModel struct {
    ID        string         `gorm:"primaryKey;type:varchar(36)"`
    Name      string         `gorm:"type:varchar(255);not null"`
    Email     string         `gorm:"type:varchar(255);uniqueIndex;not null"`
    Status    string         `gorm:"type:varchar(20);default:pending"`
    Profile   []byte         `gorm:"type:json"`  // 复杂结构用 JSON 类型
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt `gorm:"index"`
}

// TableName 表名
func (UserModel) TableName() string {
    return "users"
}

// ToDomain Model → Entity
func (m *UserModel) ToDomain() *domain.User {
    user := &domain.User{
        ID:        m.ID,
        Name:      m.Name,
        Email:     m.Email,
        Status:    domain.UserStatus(m.Status),
        CreatedAt: m.CreatedAt,
        UpdatedAt: m.UpdatedAt,
    }

    if m.Profile != nil {
        _ = json.Unmarshal(m.Profile, &user.Profile)
    }

    return user
}

// FromDomain Entity → Model
func FromDomain(entity *domain.User) *UserModel {
    model := &UserModel{
        ID:     entity.ID,
        Name:   entity.Name,
        Email:  entity.Email,
        Status: string(entity.Status),
    }

    if profileBytes, err := json.Marshal(entity.Profile); err == nil {
        model.Profile = profileBytes
    }

    return model
}
```

```go
// internal/{context}/infra/persistence/repository.go
package persistence

import (
    "context"
    "fmt"

    "{module}/internal/{context}/domain"
    "github.com/google/uuid"
    "gorm.io/gorm"
)

// UserRepositoryImpl 仓储实现
type UserRepositoryImpl struct {
    db *gorm.DB
}

// NewUserRepository 构造函数（返回接口类型）
func NewUserRepository(db *gorm.DB) domain.UserRepository {
    return &UserRepositoryImpl{db: db}
}

// Save 保存实体
func (r *UserRepositoryImpl) Save(ctx context.Context, entity *domain.User) error {
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
func (r *UserRepositoryImpl) FindByID(ctx context.Context, id string) (*domain.User, error) {
    var model UserModel
    if err := r.db.WithContext(ctx).Where("id = ?", id).First(&model).Error; err != nil {
        return nil, fmt.Errorf("查询失败: %w", err)
    }
    return model.ToDomain(), nil
}

// FindByEmail 根据邮箱查找
func (r *UserRepositoryImpl) FindByEmail(ctx context.Context, email string) (*domain.User, error) {
    var model UserModel
    if err := r.db.WithContext(ctx).Where("email = ?", email).First(&model).Error; err != nil {
        return nil, fmt.Errorf("查询失败: %w", err)
    }
    return model.ToDomain(), nil
}

// Delete 删除实体
func (r *UserRepositoryImpl) Delete(ctx context.Context, id string) error {
    if err := r.db.WithContext(ctx).Where("id = ?", id).Delete(&UserModel{}).Error; err != nil {
        return fmt.Errorf("删除失败: %w", err)
    }
    return nil
}

// List 分页查询
func (r *UserRepositoryImpl) List(ctx context.Context, offset, limit int) ([]*domain.User, int64, error) {
    var models []UserModel
    var total int64

    db := r.db.WithContext(ctx).Model(&UserModel{})

    if err := db.Count(&total).Error; err != nil {
        return nil, 0, fmt.Errorf("统计总数失败: %w", err)
    }

    if err := db.Offset(offset).Limit(limit).Order("created_at DESC").Find(&models).Error; err != nil {
        return nil, 0, fmt.Errorf("查询列表失败: %w", err)
    }

    users := make([]*domain.User, 0, len(models))
    for _, m := range models {
        users = append(users, m.ToDomain())
    }

    return users, total, nil
}
```

### B. Adapter（外部适配器）

#### 规则

- 隔离外部服务变更，只修改 Adapter，不修改 Domain
- 实现 Domain 层定义的 Port 接口

#### 代码示例

```go
// internal/{context}/infra/adapter/email_sender.go
package adapter

import (
    "context"
    "fmt"
    "net/http"
)

// EmailSenderAdapter 邮件发送适配器
type EmailSenderAdapter struct {
    client  *http.Client
    baseURL string
    apiKey  string
}

// NewEmailSenderAdapter 构造函数
func NewEmailSenderAdapter(baseURL, apiKey string) *EmailSenderAdapter {
    return &EmailSenderAdapter{
        client:  &http.Client{},
        baseURL: baseURL,
        apiKey:  apiKey,
    }
}

// SendWelcomeEmail 发送欢迎邮件（实现 domain.EmailSenderPort）
func (a *EmailSenderAdapter) SendWelcomeEmail(ctx context.Context, email, name string) error {
    // 调用外部邮件服务 API
    // ...
    return nil
}
```

```go
// internal/{context}/infra/adapter/redis_cache.go
package adapter

import (
    "context"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

// RedisCacheAdapter Redis 缓存适配器
type RedisCacheAdapter struct {
    client *redis.Client
}

// NewRedisCacheAdapter 构造函数
func NewRedisCacheAdapter(client *redis.Client) *RedisCacheAdapter {
    return &RedisCacheAdapter{client: client}
}

// Get 获取缓存（实现 domain.CachePort）
func (a *RedisCacheAdapter) Get(ctx context.Context, key string) (string, error) {
    val, err := a.client.Get(ctx, key).Result()
    if err != nil {
        return "", fmt.Errorf("获取缓存失败: %w", err)
    }
    return val, nil
}

// Set 设置缓存
func (a *RedisCacheAdapter) Set(ctx context.Context, key, value string, ttl int) error {
    if err := a.client.Set(ctx, key, value, time.Duration(ttl)*time.Second).Err(); err != nil {
        return fmt.Errorf("设置缓存失败: %w", err)
    }
    return nil
}

// Delete 删除缓存
func (a *RedisCacheAdapter) Delete(ctx context.Context, key string) error {
    if err := a.client.Del(ctx, key).Err(); err != nil {
        return fmt.Errorf("删除缓存失败: %w", err)
    }
    return nil
}
```

---

## 4. UI Layer（用户接口层）

### 定位

处理 HTTP 请求，解析参数，格式化响应。

### 规则

- 使用 Gin 框架
- 定义 Request 和 Response DTO，负责 JSON Tag
- 调用 Application 层的 Handler，**严禁**直接调用 Repository 或写业务逻辑
- 使用统一响应格式（参见 api-response-format steering）

### 代码示例

```go
// internal/{context}/ui/dto.go
package ui

// CreateUserRequest 创建用户请求 DTO
type CreateUserRequest struct {
    Name  string `json:"name" binding:"required"`
    Email string `json:"email" binding:"required,email"`
}

// UserResponse 用户响应 DTO
type UserResponse struct {
    ID        string `json:"id"`
    Name      string `json:"name"`
    Email     string `json:"email"`
    Status    string `json:"status"`
    CreatedAt string `json:"created_at"`
}

// ListUsersRequest 列表请求 DTO
type ListUsersRequest struct {
    Page     int `form:"page,default=1" binding:"min=1"`
    PageSize int `form:"page_size,default=20" binding:"min=1,max=100"`
}

// ListUsersResponse 列表响应 DTO
type ListUsersResponse struct {
    Items    []UserResponse `json:"items"`
    Total    int64          `json:"total"`
    Page     int            `json:"page"`
    PageSize int            `json:"page_size"`
}
```

```go
// internal/{context}/ui/handler.go
package ui

import (
    "github.com/gin-gonic/gin"

    "{module}/internal/{context}/app/command"
    "{module}/internal/{context}/app/query"
    "{module}/internal/pkg/response"
)

// Handler HTTP 请求处理器
type Handler struct {
    createHandler *command.CreateUserHandler
    getHandler    *query.GetUserHandler
    listHandler   *query.ListUsersHandler
}

// NewHandler 构造函数
func NewHandler(
    createHandler *command.CreateUserHandler,
    getHandler *query.GetUserHandler,
    listHandler *query.ListUsersHandler,
) *Handler {
    return &Handler{
        createHandler: createHandler,
        getHandler:    getHandler,
        listHandler:   listHandler,
    }
}

// RegisterRoutes 注册路由
func (h *Handler) RegisterRoutes(r *gin.RouterGroup) {
    group := r.Group("/users")
    {
        group.POST("", h.Create)
        group.GET("/:id", h.GetByID)
        group.GET("", h.List)
    }
}

// Create 创建用户
func (h *Handler) Create(c *gin.Context) {
    var req CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        response.BadRequest(c, "参数错误: "+err.Error())
        return
    }

    cmd := command.CreateUserCommand{
        Name:  req.Name,
        Email: req.Email,
    }

    entity, err := h.createHandler.Handle(c.Request.Context(), cmd)
    if err != nil {
        response.InternalError(c, "创建失败")
        return
    }

    response.Success(c, UserResponse{
        ID:        entity.ID,
        Name:      entity.Name,
        Email:     entity.Email,
        Status:    string(entity.Status),
        CreatedAt: entity.CreatedAt.Format("2006-01-02 15:04:05"),
    })
}

// GetByID 根据 ID 查询
func (h *Handler) GetByID(c *gin.Context) {
    id := c.Param("id")

    entity, err := h.getHandler.Handle(c.Request.Context(), id)
    if err != nil {
        response.NotFound(c, "用户未找到")
        return
    }

    response.Success(c, UserResponse{
        ID:        entity.ID,
        Name:      entity.Name,
        Email:     entity.Email,
        Status:    string(entity.Status),
        CreatedAt: entity.CreatedAt.Format("2006-01-02 15:04:05"),
    })
}

// List 列表查询
func (h *Handler) List(c *gin.Context) {
    var req ListUsersRequest
    if err := c.ShouldBindQuery(&req); err != nil {
        response.BadRequest(c, "参数错误: "+err.Error())
        return
    }

    q := query.ListUsersQuery{
        Page:     req.Page,
        PageSize: req.PageSize,
    }

    result, err := h.listHandler.Handle(c.Request.Context(), q)
    if err != nil {
        response.InternalError(c, "查询失败")
        return
    }

    items := make([]UserResponse, 0, len(result.Items))
    for _, entity := range result.Items {
        items = append(items, UserResponse{
            ID:        entity.ID,
            Name:      entity.Name,
            Email:     entity.Email,
            Status:    string(entity.Status),
            CreatedAt: entity.CreatedAt.Format("2006-01-02 15:04:05"),
        })
    }

    response.Success(c, ListUsersResponse{
        Items:    items,
        Total:    result.Total,
        Page:     req.Page,
        PageSize: req.PageSize,
    })
}
```

---

## 层间依赖总结

| 层 | 可以依赖 | 不可以依赖 |
|---|---------|-----------|
| Domain | Go 标准库 | gorm, gin, sql, 任何第三方库 |
| Application | Domain | gin, gorm, HTTP 相关 |
| Infrastructure | Domain, gorm, 第三方库 | Application, UI |
| UI | Application, gin, response pkg | Domain（直接）, Infrastructure |
