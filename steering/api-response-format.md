# API 响应格式规范

本规范定义了所有 API 端点必须遵循的统一 JSON 响应格式。

---

## 统一响应结构

### 基本格式

所有 API 响应都必须遵循以下 JSON 结构：

```json
{
  "code": <HTTP状态码>,
  "data": <响应数据或null>,
  "message": "<状态消息>"
}
```

### 字段说明

- **code**: `number` — HTTP 状态码（200, 400, 500 等）
- **data**: `object | null` — 响应数据，成功时包含实际数据，错误时为 `null`
- **message**: `string` — 状态消息，成功时为"成功"，错误时为具体错误描述

## Go 实现

### 响应工具包

在 `internal/pkg/response/response.go` 中实现统一响应：

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

### Handler 中使用

```go
// ✅ 正确用法：使用统一响应
func (h *Handler) Create(c *gin.Context) {
	var req CreateRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		response.BadRequest(c, "参数错误: "+err.Error())
		return
	}

	entity, err := h.createHandler.Handle(c.Request.Context(), cmd)
	if err != nil {
		response.InternalError(c, "创建失败")
		return
	}

	response.Success(c, EntityResponse{
		ID:   entity.ID,
		Name: entity.Name,
	})
}

// ❌ 错误用法：直接返回不统一的格式
func (h *Handler) Create(c *gin.Context) {
	c.JSON(200, entity) // 没有统一格式
}
```

## 成功响应示例

### 200 成功

```json
{
  "code": 200,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "示例资源",
    "created_at": "2025-01-01 12:00:00"
  },
  "message": "成功"
}
```

### 列表响应

```json
{
  "code": 200,
  "data": {
    "items": [...],
    "total": 100,
    "page": 1,
    "page_size": 20
  },
  "message": "成功"
}
```

### 健康检查

```json
{
  "code": 200,
  "data": {
    "status": "ok",
    "service": "my-service"
  },
  "message": "成功"
}
```

## 错误响应示例

### 400 客户端错误

```json
{
  "code": 400,
  "data": null,
  "message": "参数错误: name 不能为空"
}
```

### 404 资源未找到

```json
{
  "code": 404,
  "data": null,
  "message": "资源未找到"
}
```

### 500 服务器内部错误

```json
{
  "code": 500,
  "data": null,
  "message": "服务器内部错误"
}
```

### 503 服务不可用

```json
{
  "code": 503,
  "data": null,
  "message": "服务繁忙，请稍后重试"
}
```

## 最佳实践

### DO（应该做）

- ✅ 始终使用统一的响应格式
- ✅ 提供清晰的错误消息
- ✅ 使用适当的 HTTP 状态码
- ✅ 在测试中验证响应格式
- ✅ 成功响应的 data 字段必须有值

### DON'T（不应该做）

- ❌ 不要在不同端点使用不同的响应格式
- ❌ 不要在成功响应中返回 null data
- ❌ 不要使用模糊的错误消息（如"出错了"）
- ❌ 不要忽略 HTTP 状态码的语义
- ❌ 不要在响应中包含敏感信息（密码、密钥等）
- ❌ 不要直接返回数据库 Model，必须转换为 Response DTO
