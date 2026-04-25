# Git 提交规范

本指南定义了 Go 后端项目的 Git 初始化、提交信息格式校验和相关工具配置。所有操作通过 Makefile 统一管理。

---

## Git 初始化

### .gitignore

```gitignore
# 二进制文件
*.exe
*.exe~
*.dll
*.so
*.dylib
/bin/
/tmp/

# 测试覆盖
*.test
*.out
coverage.*

# 依赖（Go Modules 缓存由系统管理）
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

---

## 提交信息格式

### 格式规范

所有提交信息必须遵循 [Conventional Commits](https://www.conventionalcommits.org/) 规范：

```
<type>(<scope>): <subject>

<body>

<footer>
```

### type 类型

| 类型 | 说明 |
|------|------|
| `feat` | 新功能 |
| `fix` | 修复 bug |
| `docs` | 文档更新 |
| `style` | 代码格式化（不影响逻辑） |
| `refactor` | 重构（既不是新功能也不是修复） |
| `perf` | 性能优化 |
| `test` | 测试相关 |
| `build` | 构建系统或外部依赖变更 |
| `ci` | CI 配置变更 |
| `chore` | 其他杂项 |
| `revert` | 回滚提交 |

### 规则

- `type` 必填，全小写
- `scope` 可选，全小写，建议使用限界上下文名称（如 `user`、`order`）
- `subject` 必填，不以句号结尾
- `header`（第一行）不超过 100 个字符

### 示例

```bash
feat(user): 添加用户注册接口
fix(order): 修复订单金额计算精度问题
docs: 更新 README 部署说明
refactor(auth): 重构 JWT 中间件逻辑
chore: 升级 Go 版本到 1.22
```

---

## commit-msg Hook 脚本

Go 项目采用零依赖的 Shell 脚本方案，通过 `.githooks/commit-msg` 校验提交信息格式。

### .githooks/commit-msg

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
    echo "  docs(api): 更新接口文档"
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

---

## Makefile 管理

所有 Git 相关操作通过 Makefile 统一管理，开发者只需记住 `make` 命令即可。

### Makefile 中的 Git 相关目标

```makefile
# ==================== Git Hooks ====================

.PHONY: setup-hooks verify-hooks

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
```

### 集成到开发工作流

在 Makefile 的 `all` 和 `install-tools` 目标中集成 `setup-hooks`：

```makefile
# 默认目标（包含 Hook 安装）
all: setup-hooks clean deps fmt lint test build

# 安装开发工具（包含 Hook 安装）
install-tools: setup-hooks
	@echo "安装开发工具..."
	go install github.com/cosmtrek/air@latest
	go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
	@echo "✅ 开发工具安装完成"
```

### 完整的 help 输出应包含

```
  setup-hooks          安装 Git Hooks（提交信息格式校验）
  verify-hooks         验证 Git Hooks 是否正常工作
```

---

## 开发者工作流

```bash
# 1. clone 项目后首先执行
make setup-hooks

# 2. 日常开发
make dev                    # 启动开发服务器
make fmt                    # 格式化代码
make lint                   # 代码检查
make test                   # 运行测试

# 3. 提交代码（Hook 自动校验提交信息）
git add .
git commit -m "feat(user): 添加用户注册接口"

# 4. 提交前完整检查
make quality-check          # fmt + lint + test

# 5. 发布准备
make release                # clean + deps + quality-check + build
```
