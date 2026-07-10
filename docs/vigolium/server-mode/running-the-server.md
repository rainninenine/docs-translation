# Running the Server

Vigolium 可以作为一个长时间运行的服务运行，通过 REST API 暴露其所有功能。这对于团队设置、自动化工作流以及将 Vigolium 集成到更大的安全基础设施中非常理想。

## 快速开始

```bash
# 使用默认设置启动服务器
vigolium server

# 指定端口和绑定地址
vigolium server --port 9002 --host 0.0.0.0

# 使用 API 密钥进行身份验证
vigolium server --api-key "your-secret-key"
```

## 启动标志

| 标志 | 默认值 | 描述 |
|---|---|---|
| `--port` | `9002` | HTTP 服务器端口 |
| `--host` | `0.0.0.0` | 绑定地址 |
| `--api-key` | `""` | API 密钥（覆盖配置） |
| `-A` / `--no-auth` | `false` | 禁用身份验证（仅开发环境） |
| `--ingest-proxy-port` | `0` | 透明代理端口（0 = 禁用） |
| `--scan-on-receive` | `false` | 接收流量时自动运行扫描 |
| `--config` | `vigolium-configs.yaml` | 配置文件路径 |

## 服务器配置

```yaml
# vigolium-configs.yaml
server:
  host: "0.0.0.0"
  port: 9002
  auth_api_key: "your-secret-key"      # 覆盖 VIGOLIUM_API_KEY 环境变量
  cors_allowed_origins: "*"            # CORS 设置
  ingest_proxy_port: 0                 # 透明代理端口（0 = 禁用）
  scan_on_receive: false               # 接收流量时自动扫描

agent:
  olium:
    provider: openai-codex-oauth       # AI agent 供应商
    model: gpt-5.5                     # AI 模型
```

## 端点概览

| 组 | 代表端点 | 用途 |
|---|---|---|
| 元数据（无需认证） | `GET /`、`/health`、`/metrics`、`/swagger/*` | 健康检查、Prometheus、OpenAPI UI |
| 认证/用户 | `POST /api/auth/login`、`GET /api/user/info` | 令牌签发、身份识别 |
| 数据导入 | `POST /api/ingest-http` | 流量导入 |
| HTTP 记录 | `GET/DELETE /api/http-records[/:uuid]` | 查询/检查捕获的流量 |
| 漏洞发现 | `GET /api/findings`、`PATCH /api/findings/:id/status` | 查询/分类结果 |
| 扫描控制 | `POST /api/scan-url`、`/api/scans/run` | 触发和管理扫描 |
| 范围/配置 | `GET/POST /api/scope`、`/api/config` | 实时范围和配置 |
| 项目 | `GET/POST/PUT/DELETE /api/projects[/:uuid]` | 多租户管理 |
| Agent | `POST /api/agent/run/{query,autopilot,swarm,audit}` | AI 驱动的扫描 |

## 异步作业模式

扫描和 agent 运行是长时间运行的操作，因此 API 采用**启动并轮询**模式：

```bash
# 1. 启动扫描
curl -X POST http://localhost:9002/api/scans/run \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"target": "https://example.com"}'
# 响应: 202 Accepted { "uuid": "abc-123", "status": "running" }

# 2. 轮询状态
curl http://localhost:9002/api/scan/status \
  -H "Authorization: Bearer $API_KEY"
# 响应: { "status": "completed", "findings_count": 5 }

# 3. 获取结果
curl http://localhost:9002/api/findings \
  -H "Authorization: Bearer $API_KEY"
```

## 身份验证

服务器支持 Bearer token 身份验证：

```bash
# 通过环境变量设置 API 密钥
export VIGOLIUM_API_KEY="your-secret-key"
vigolium server

# 或通过配置
vigolium server --api-key "your-secret-key"

# 在请求中使用
curl -H "Authorization: Bearer your-secret-key" http://localhost:9002/api/findings
```

对于开发环境，使用 `-A` 禁用身份验证。

## 多租户隔离

服务器通过 `X-Project-UUID` 标头支持多租户隔离：

```bash
# 创建项目
curl -X POST http://localhost:9002/api/projects \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "my-project"}'

# 在项目范围内操作
curl -H "X-Project-UUID: <project-uuid>" \
  -H "Authorization: Bearer $API_KEY" \
  http://localhost:9002/api/findings
```

## 健康检查和指标

```bash
# 健康检查
curl http://localhost:9002/health

# Prometheus 指标
curl http://localhost:9002/metrics

# Swagger UI
open http://localhost:9002/swagger/
```

## 延伸阅读

- [Server & API Architecture](../architecture/server-and-api.md) — 服务器架构详情
- [Ingestion](ingestion.md) — 流量导入
- [Proxy](proxy.md) — 透明代理
- [API Overview](../api-overview.md) — 完整端点目录
