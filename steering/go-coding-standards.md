# Go 语言代码规范

本规范定义了项目中 Go 代码的编写标准，确保代码风格一致、可读性强。

---

## 核心原则

- **简洁性**：代码简洁明了，避免过度复杂
- **可读性**：优先考虑代码可读性
- **一致性**：保持一致的编码风格
- **错误处理**：明确处理所有错误情况

## 1. 命名规范

### 1.1 包名

```go
// ✅ 好的包名
package user
package http
package auth

// ❌ 不好的包名
package userUtils
package users
package util
```

### 1.2 变量和函数

```go
// ✅ 好的命名
var userID int64
var isActive bool
func CreateUser(name string) *User { }
func (u *User) Name() string { }        // 获取器，不需要 GetName

// ❌ 不好的命名
var user_id int64        // 不要使用下划线
var UserID int64         // 局部变量不要大写开头
func (u *User) GetName() string { }     // 不需要 Get 前缀
```

### 1.3 结构体和接口

```go
// ✅ 好的命名
type User struct {
    ID       int64
    Name     string
    IsActive bool
}

type UserRepository interface {
    Create(*User) error
    GetByID(int64) (*User, error)
}

// ❌ 不好的命名
type userInfo struct { }      // 导出类型应该大写开头
type UserInterface interface { } // 不要使用 Interface 后缀
```

## 2. 代码格式化

- 使用 `gofmt` 和 `goimports` 格式化代码
- 每行不超过 100 个字符
- 导入分组：标准库 → 第三方库 → 本地包

```go
import (
    // 标准库
    "context"
    "fmt"
    "time"

    // 第三方库
    "github.com/gin-gonic/gin"

    // 本地包
    "myproject/internal/user"
)
```

## 3. 错误处理

```go
// ✅ 正确的错误处理：使用 %w 包装错误
func CreateUser(name, email string) (*User, error) {
    if err := validateEmail(email); err != nil {
        return nil, fmt.Errorf("无效的邮箱地址: %w", err)
    }

    user := &User{Name: name, Email: email}
    if err := db.Create(user).Error; err != nil {
        return nil, fmt.Errorf("创建用户失败: %w", err)
    }

    return user, nil
}

// ❌ 错误的错误处理：忽略错误
func CreateUser(name, email string) *User {
    user := &User{Name: name, Email: email}
    db.Create(user)  // 忽略了错误
    return user
}
```

## 4. 函数设计

```go
// ✅ 使用选项模式处理可选参数
type UserOption func(*User)

func WithAge(age int) UserOption {
    return func(u *User) { u.Age = age }
}

func NewUser(name, email string, opts ...UserOption) *User {
    user := &User{Name: name, Email: email}
    for _, opt := range opts {
        opt(user)
    }
    return user
}

// 使用
user := NewUser("张三", "test@example.com", WithAge(25))
```

## 5. 接口设计

```go
// ✅ 小而专注的接口
type Reader interface {
    Read([]byte) (int, error)
}

type Writer interface {
    Write([]byte) (int, error)
}

// 在使用方定义接口（符合显式架构的依赖倒置原则）
type UserService struct {
    repo UserRepository
}

type UserRepository interface {
    Create(*User) error
    GetByID(int64) (*User, error)
}
```

## 6. 并发编程

```go
// ✅ 正确使用 goroutine：传递 context，处理错误
func ProcessUsers(ctx context.Context, users []*User) error {
    var wg sync.WaitGroup
    errCh := make(chan error, len(users))

    for _, user := range users {
        wg.Add(1)
        go func(u *User) {
            defer wg.Done()
            select {
            case <-ctx.Done():
                errCh <- ctx.Err()
                return
            default:
            }
            if err := processUser(u); err != nil {
                errCh <- err
            }
        }(user)
    }

    go func() {
        wg.Wait()
        close(errCh)
    }()

    for err := range errCh {
        if err != nil {
            return err
        }
    }
    return nil
}
```

## 7. 测试规范

```go
// 使用表驱动测试
func TestCreateUser(t *testing.T) {
    tests := []struct {
        name    string
        input   CreateUserRequest
        want    *User
        wantErr bool
    }{
        {
            name: "有效用户创建",
            input: CreateUserRequest{
                Name:  "张三",
                Email: "test@example.com",
            },
            want: &User{
                Name:  "张三",
                Email: "test@example.com",
            },
            wantErr: false,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := CreateUser(tt.input)
            if tt.wantErr {
                require.Error(t, err)
                return
            }
            require.NoError(t, err)
            assert.Equal(t, tt.want.Name, got.Name)
        })
    }
}
```

## 8. 性能优化

```go
// ✅ 预分配 slice 容量
func ProcessUsers(users []*User) []*ProcessedUser {
    result := make([]*ProcessedUser, 0, len(users))
    for _, user := range users {
        result = append(result, processUser(user))
    }
    return result
}

// ✅ 使用 strings.Builder 拼接字符串
func BuildMessage(users []*User) string {
    var builder strings.Builder
    builder.Grow(len(users) * 50)

    for i, user := range users {
        builder.WriteString(strconv.Itoa(i + 1))
        builder.WriteString(". ")
        builder.WriteString(user.Name)
        builder.WriteString("\n")
    }
    return builder.String()
}
```

## 9. 工具和检查清单

### 必备工具

```bash
go fmt ./...           # 格式化
goimports -w .         # 管理导入
go vet ./...           # 代码检查
golangci-lint run      # 静态分析
go test ./...          # 测试
go test -race ./...    # 竞态检测
```

### 提交前检查清单

- [ ] 代码已使用 `gofmt` 格式化
- [ ] 通过 `go vet` 和 `golangci-lint` 检查
- [ ] 所有测试通过
- [ ] 公开函数有中文文档注释
- [ ] 错误处理完整，使用 `%w` 包装
- [ ] 没有 goroutine 泄漏
- [ ] 导入分组正确（标准库 → 第三方 → 本地）
