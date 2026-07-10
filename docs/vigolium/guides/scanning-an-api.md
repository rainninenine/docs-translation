# Scanning an API

本指南介绍如何使用 Vigolium 扫描 API 端点。无论您使用的是 REST、GraphQL 还是 SOAP API，Vigolium 都能自动检测并应用适当的扫描策略。

## 1. 提供 API 规范

Vigolium 支持多种 API 规范格式。提供规范可以让 Vigolium 了解 API 的结构、端点和参数，从而实现更精确的扫描。

```bash
# OpenAPI/Swagger 规范
vigolium scan -I openapi -i api-spec.yaml -t https://api.example.com

# Postman 集合
vigolium scan -I postman -i collection.json -t https://api.example.com

# 自动检测格式
vigolium scan -i api-spec.yaml -t https://api.example.com
```

### 身份验证标头

大多数 API 需要身份验证。使用 `--spec-header` 注入标头：

```bash
# Bearer token
vigolium scan -I openapi -i spec.yaml -t https://api.example.com \
  --spec-header "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."

# 多个标头
vigolium scan -I openapi -i spec.yaml -t https://api.example.com \
  --spec-header "Authorization: Bearer eyJhbG..." \
  --spec-header "X-API-Key: abc123"
```

## 2. 无需规范的扫描

如果您没有 API 规范，Vigolium 仍然可以通过自动发现端点进行有效扫描：

```bash
# 扫描一个已知端点
vigolium scan -t https://api.example.com/api/users

# 扫描并让 Vigolium 发现更多端点
vigolium scan -t https://api.example.com --strategy deep
```

## 3. GraphQL 扫描

Vigolium 自动检测 GraphQL 端点并运行专门的 GraphQL 扫描：

```bash
# 扫描 GraphQL 端点
vigolium scan -t https://api.example.com/graphql

# 使用规范（如果可用）
vigolium scan -I openapi -i spec.yaml -t https://api.example.com/graphql
```

GraphQL 特定检查包括：
- 内省查询（Introspection query）
- 批量攻击（Batching abuse）
- 字段建议枚举（Field suggestion enumeration）
- 深度递归查询（Deep recursive queries）
- 基于别名的攻击（Alias-based attacks）

## 4. REST API 最佳实践

```bash
# 限制范围到 API 路径
vigolium scan -t https://api.example.com --scope-path "/api/*"

# 排除健康检查端点
vigolium scan -t https://api.example.com \
  --scope-exclude-path "/api/health" \
  --scope-exclude-path "/api/metrics"

# 增加 JSON API 的速率限制
vigolium scan -t https://api.example.com --rate-limit 200
```

## 5. 无状态扫描

对于快速、一次性的 API 安全检查，使用无状态模式：

```bash
vigolium scan -t https://api.example.com/api/users --stateless
```

这在 CI/CD 流水线中特别有用，详见 [CI/CD Integration](ci-cd-integration.md)。

## 延伸阅读

- [Stateless Scan](stateless-scan.md) — 快速一次性扫描
- [CI/CD Integration](ci-cd-integration.md) — 流水线集成
- [Native Scan Phases](native-scan-phases.md) — 扫描阶段详解
- [Agentic Scan](agentic-scan.md) — AI 驱动的 API 扫描
