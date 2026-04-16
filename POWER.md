---
name: "explicit-architecture"
displayName: "Explicit Architecture for Go"
description: "基于显式架构（Explicit Architecture）的 Go 后端服务搭建指南，融合 Clean Architecture 和 Hexagonal Architecture 思想，使用 Gin + Gorm 技术栈，支持 MySQL/PostgreSQL/Redis。"
keywords: ["explicit-architecture", "go-backend", "clean-architecture", "hexagonal", "gin", "gorm", "ddd"]
author: "yanjixian"
---

# Explicit Architecture for Go

## Overview

这是一份基于 **Explicit Architecture（显式架构）** 的 Go 后端服务搭建规范，融合了 Clean Architecture 和 Hexagonal Architecture 的核心思想。

本规范适用于使用 Go + Gin + Gorm 技术栈搭建后端服务，支持 MySQL、PostgreSQL、Redis 等数据存储。它定义了清晰的分层边界、依赖方向和代码组织方式，确保项目在规模增长时仍然保持可维护性。

## Available Steering Files

- **scaffolding** — 新项目脚手架搭建工作流：从零开始创建符合显式架构的项目结构
- **layer-rules** — 各层详细规范与代码示例：Domain、Application、Infrastructure、UI 层的具体规则
- **go-coding-standards** — Go 语言代码规范：命名、格式化、错误处理、并发、测试、性能优化
- **api-response-format** — API 统一响应格式规范：JSON 结构、状态码、Go 实现代码

## 核心设计理念

### 三大原则

1. **依赖倒置** — 内层（Domain）绝不依赖外层（Infra/UI）。外层通过依赖注入（DI）实现内层定义的接口。
2. **显式边界** — 业务实体（Entity）与数据库模型（PO/Model）必须严格分离，禁止混用。通过显式的 Mapper 进行转换。
3. **适配器模式** — 所有外部服务（第三方 API、消息队列、缓存等）必须通过 Adapter 接入，核心业务不感知具体实现。

### 依赖方向

```
UI Layer → Application Layer → Domain Layer ← Infrastructure Layer
```

- Domain 层是核心，不依赖任何外层
- Application 层依赖 Domain 层的接口
- Infrastructure 层实现 Domain 层定义的接口
- UI 层调用 Application 层的 UseCase

## 技术栈

| 组件 | 技术选型 | 说明 |
|------|---------|------|
| 语言 | Go 1.21+ | |
| Web 框架 | Gin | HTTP 路由和中间件 |
| ORM | Gorm v2 | 数据库操作 |
| 关系数据库 | MySQL 8.0+ 或 PostgreSQL 14+ | 根据项目需求选择 |
| 缓存 | Redis | 可选，按需引入 |
| 配置管理 | Viper | YAML 配置文件 |
| 日志 | Zap 或 Slog | 结构化日志 |

## 目录结构

采用 Go 标准项目布局与 DDD 限界上下文结合：

```text
/
├── cmd/
│   └── server/                 # main.go，依赖注入初始化
├── internal/                   # 私有应用代码
│   ├── {context_name}/         # 限界上下文（如 user、order、product）
│   │   ├── domain/             # 1. 核心层：Entity, ValueObject, Repository Interface
│   │   ├── app/                # 2. 应用层：Command, Query (CQRS)
│   │   │   ├── command/        #    写操作 Handler
│   │   │   └── query/          #    读操作 Handler
│   │   ├── infra/              # 3. 基础设施层：Persistence, Adapter
│   │   │   ├── persistence/    #    数据库实现（Model, Repository Impl）
│   │   │   └── adapter/        #    外部服务适配器
│   │   └── ui/                 # 4. 接口层：HTTP Handler, DTO, Router
│   └── pkg/                    # 跨上下文共享代码（Logger, Error types, Middleware）
├── configs/                    # 配置文件（.yaml）
├── api/                        # OpenAPI/Swagger 定义
├── deploy/                     # 部署配置（K8s 等）
├── Dockerfile                  # Docker 多阶段构建
├── .dockerignore               # Docker 忽略文件
├── go.mod
└── go.sum
```

### 限界上下文命名

每个业务领域对应一个限界上下文目录：

```text
internal/
├── user/           # 用户管理
├── order/          # 订单管理
├── product/        # 商品管理
└── notification/   # 通知服务
```

## 开发检查清单

在提交代码前，请自查：

- [ ] **Domain 纯净性**：Domain 层是否引入了 `gorm`、`gin`、`sql` 等第三方库？（如果有，请删除）
- [ ] **显式转换**：是否编写了 `Model ↔ Entity` 的 Mapper 代码？（禁止直接透传 Model 到 Handler）
- [ ] **JSON 处理**：复杂嵌套结构是否使用了数据库 JSON 类型而非建立无意义的关联表？
- [ ] **接口依赖**：Application 层是否依赖 Repository 的接口（Interface）而非结构体（Struct）？
- [ ] **配置分离**：外部服务的 URL、Key 等是否放在 `configs/` 或环境变量中？
- [ ] **错误处理**：是否使用 `fmt.Errorf("xxx: %w", err)` 包装错误？
- [ ] **Context 传递**：所有跨层调用是否传递了 `context.Context`？

## Best Practices

- 每个限界上下文独立自治，上下文之间通过 Application 层的接口通信，避免直接引用其他上下文的 Domain
- Command 和 Query 分离（CQRS），写操作放 `app/command/`，读操作放 `app/query/`
- Repository 接口定义在 Domain 层，实现在 Infrastructure 层
- 使用依赖注入在 `cmd/server/main.go` 中组装所有依赖
- 统一 API 响应格式：`{"code": 200, "data": {...}, "message": "成功"}`
- 数据库复杂嵌套结构优先使用 JSON 类型列存储
- 外部服务调用必须通过 Adapter，便于 Mock 测试和替换实现

## Troubleshooting

### Domain 层被污染

**问题**：Domain 层引入了 gorm tag 或 gin 依赖
**解决**：
1. 检查 domain/ 目录下的 import，确保只有标准库
2. 将 gorm tag 移到 infra/persistence/ 的 Model 中
3. 编写 ToDomain() / FromDomain() 转换方法

### Model 和 Entity 混用

**问题**：Handler 直接返回数据库 Model 给前端
**解决**：
1. 在 UI 层定义 Response DTO
2. 在 Infrastructure 层通过 Mapper 转换 Model → Entity
3. 在 UI 层通过 DTO 转换 Entity → Response

### 循环依赖

**问题**：两个限界上下文互相引用
**解决**：
1. 提取共享接口到 `internal/pkg/`
2. 通过 Application 层的事件或接口解耦
3. 考虑是否需要合并为一个上下文
