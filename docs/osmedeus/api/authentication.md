---
title: "身份验证"
description: "API 访问的 JWT 和 API 密钥身份验证"
---

# 身份验证

大多数 API 端点需要 JWT 身份验证。首先通过登录端点获取令牌，然后在后续请求中包含该令牌。

## 登录

**POST** `/osm/api/login`

通过身份验证获取 JWT 令牌。

### 请求

```bash
curl -X POST http://localhost:8002/osm/api/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "osmedeus",
    "password": "your-password"
  }'
```

### 请求体

| 字段 | 类型 | 必填 | 描述 |
|-------|------|----------|-------------|
| `username` | string | 是 | 服务器设置中配置的用户名 |
| `password` | string | 是 | 用户密码 |

### 响应（200 OK）

```json
{
  "token": "eyJhbG...c123..."
}
```

### 错误响应

**400 Bad Request** - 无效的请求体：
```json
{
  "error": true,
  "message": "Invalid request body"
}
```

**401 Unauthorized** - 无效的凭据：
```json
{
  "error": true,
  "message": "Invalid credentials"
}
```

## 登出

**POST** `/osm/api/logout`

清除会话 Cookie 以登出。

### 请求

```bash
curl -X POST http://localhost:8002/osm/api/logout
```

### 响应（200 OK）

```json
{
  "message": "Logged out successfully"
}
```

---

## 令牌详情

- **算法**：HS256（HMAC-SHA256）
- **过期时间**：可通过设置中的 `server.jwt.expiration_minutes` 配置（默认：1440 分钟 / 1 天）
- **声明**：包含 `username`、`exp`（过期时间）和 `iat`（签发时间）

## 使用令牌

在后续请求中使用 `Authorization: Bearer ***` 头包含令牌：

```bash
# 将令牌存储在环境变量中
export TOKEN="eyJhbG...VCJ9..."

# 在 API 请求中使用
curl http://localhost:8002/osm/api/workflows \
  -H "Authorization: Bearer ***"
```

### 身份验证错误

**401 Unauthorized** - 缺少头信息：
```json
{
  "error": true,
  "message": "Missing authorization header"
}
```

**401 Unauthorized** - 格式无效：
```json
{
  "error": true,
  "message": "Invalid authorization header format"
}
```

**401 Unauthorized** - 令牌过期或无效：
```json
{
  "error": true,
  "message": "Invalid or expired token"
}
```

## API 密钥身份验证

作为 JWT 令牌的替代方案，您可以通过 `x-osm-api-key` 头使用静态 API 密钥进行身份验证。这对于脚本、CI/CD 管道或难以管理 JWT 令牌刷新的集成场景非常有用。

### 配置

API 密钥身份验证在 `~/osmedeus-base/osm-settings.yaml` 中配置：

```yaml
server:
  # 启用 API 密钥身份验证（默认：true）
  enabled_auth_api: true
  # 用于 x-osm-api-key 头身份验证的 API 密钥
  # 首次运行时自动生成随机的 32 字符密钥
  auth_api_key: "your-api-key-here"
```

### 使用 API 密钥

在请求中使用 `x-osm-api-key` 头包含 API 密钥：

```bash
# 将 API 密钥存储在环境变量中
export OSM_API_KEY="your-api-key-here"

# 在 API 请求中使用
curl http://localhost:8002/osm/api/workflows \
  -H "x-osm-api-key: ***"
```

### 错误响应

**401 Unauthorized** - API 密钥无效或缺失：
```json
{
  "error": true,
  "message": "Invalid or missing API key"
}
```

### 注意事项

- 启用后，API 密钥身份验证优先于 JWT
- 首次启动服务器时自动生成随机的 32 字符 API 密钥
- API 密钥以明文形式存储在设置文件中；请确保适当的文件权限
- 空值、仅空白字符的值或占位符值（`null`、`undefined`、`nil`）将被拒绝

## 禁用身份验证

可以通过使用 `--no-auth` 标志启动服务器来禁用身份验证：

```bash
osmedeus server --no-auth
```

禁用后，所有 API 端点无需令牌即可访问。
