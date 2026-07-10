# Server & API Architecture（服务器与 API 体系结构）

> _体系结构系列：[overview](overview.md) · [native-scan](native-scan.md) · [agentic-scan](agentic-scan.md) · [data-and-storage](data-and-storage.md) · **server-and-api**_

Vigolium 通过 `vigolium server` 作为长期运行的服务运行：一个 Fiber REST API，负责数据导入、触发本地扫描和 Agent 扫描，并提供结果查询——所有功能共享与 CLI 相同的项目级持久化层。本文档说明服务的架构和请求生命周期。有关逐条 curl 示例，请参见 [server-and-ingestion.md](../server-and-ingestion.md)；完整端点目录请参见 [api-overview.md](../api-overview.md) 和 [api-references/](../api-references/)。

---

## 1. 进程模型

```
                         vigolium server
   ┌───────────────────────────────────────────────────────────┐
   │  Fiber HTTP 服务器            :9002（默认 0.0.0.0）          │
   │    中间件：CORS → 身份验证（Bearer）→ 项目解析                │
   │                                                             │
   │  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐  │
   │  │ 数据导入     │  │ 扫描控制     │  │ Agent 运行 API    │  │
   │  │ 处理器       │  │ 处理器       │  │ handlers_agent.go │  │
   │  └──────┬──────┘  └──────┬───────┘  └─────────┬─────────┘  │
   │         └──────────┬─────┴────────────────────┘            │
   │                    ▼                                        │
   │            共享 Repository（SQLite / PostgreSQL）            │
   └───────────────────────────────────────────────────────────┘
            ▲                                  ▲
            │ 可选透明代理                      │ vigolium ingest
            │（--ingest-proxy-port :9003）      │（远程客户端）
```

服务器位于 `pkg/server/`（Fiber）。它本身不包含扫描逻辑——它封装了与 CLI 相同的 `internal/runner` 本地扫描管道和相同的 `pkg/agent` 编排器，因此无论从哪个入口启动扫描，行为都是一致的。三个流量源通过相同的 `RecordWriter` → repository 路径进行数据导入：`/api/ingest-http` 端点、可选的透明 HTTP 代理以及 `vigolium ingest` 客户端。

### 中间件链

每个请求（`/`、`/health`、`/metrics`、`/swagger/*` 除外）依次经过：

1. **CORS** — `server.cors_allowed_origins`：`reflect-origin`（默认，支持凭据的反射模式）、`*`（通配符，无凭据）、显式逗号分隔的白名单，或为空（禁用）。
2. **身份验证** — `Authorization: Bearer ***` 密钥解析顺序：`VIGOLIUM_API_KEY` 环境变量 > `server.auth_api_key` 配置。`vigolium server -A` 禁用身份验证（仅限开发环境）。
3. **项目解析** — `X-Project-UUID` 标头选择租户（未指定时使用默认项目）；`X-User-Email` 驱动可选的按项目访问控制（拒绝时返回 `403`）。参见 [data-and-storage.md](data-and-storage.md#1-multi-tenancy-the-project_uuid-spine)。

---

## 2. 流量数据导入

`POST /api/ingest-http` 是通用入口点。单个端点接受多种 `input_mode`，将其标准化为 `HttpRequestResponse` 项，并交给异步的 `RecordWriter`：

| `input_mode` | 载荷字段 | 来源 |
|---|---|---|
| `url` / `url_file` | `content` | 单个 URL / 换行分隔的列表 |
| `curl` | `content` 或 `content_base64` | cURL 命令字符串 |
| `burp_base64` | `http_request_base64`（+ 可选 `http_response_base64`、`url` 提示） | 原始 HTTP 请求 |
| `openapi` / `swagger` | `content` 或 `content_base64` | OpenAPI/Swagger 规范 |
| `postman_collection` | `content_base64` | Postman 集合 |

大型载荷应使用 `*_base64` 字段以避免 JSON 转义问题。`vigolium ingest` CLI 使用相同的解析器，支持两种模式：**远程**（`-s http://server` → POST 到 `/api/ingest-http`）或**本地**（省略 `--server` → 直接获取并写入数据库）。

### 透明代理

`vigolium server --ingest-proxy-port 9003` 启动一个记录型 HTTP 代理。通过它路由的明文 HTTP 流量会被捕获到数据库中；HTTPS `CONNECT` 隧道**不会**被记录。这是从任意工具（`curl -x`、`httpx -proxy`、`nuclei -proxy`）捕获流量的零侵入方式。

---

## 3. REST 接口

端点按关注点分组；每组在 [api-references/](../api-references/) 下有专门的页面。

| 分组 | 代表性端点 | 用途 |
|---|---|---|
| 元信息（无需身份验证） | `GET /`、`/health`、`/metrics`、`/swagger/*`、`/server-info` | 存活检查、Prometheus、OpenAPI UI |
| 身份验证/用户 | `POST /api/auth/login`、`GET /api/user/info` | 令牌发放、身份识别 |
| 数据导入 | `POST /api/ingest-http` | 流量输入（见 §2） |
| HTTP 记录 | `GET/DELETE /api/http-records[/:uuid]` | 查询/检查捕获的流量 |
| 漏洞发现 | `GET /api/findings`、`PATCH /api/findings/:id/status` | 查询/分类结果 |
| 扫描控制 | `POST /api/scan-url`、`/api/scan-request`、`/api/scans/run`、`/api/scans/:uuid/{stop,pause,resume}` | 触发和管理本地扫描 |
| 范围/设置 | `GET/POST /api/scope`、`/api/config` | 实时范围与设置 |
| 项目 | `GET/POST/PUT/DELETE /api/projects[/:uuid]`、`/domain-map` | 多租户隔离管理 |
| 存储 | `/api/storage/{source,results,upload-source,presign}` | 云存储捆绑包（需启用存储） |
| 数据库浏览 | `/api/db/tables/...` | 通用表检查（管理员） |
| Agent | `POST /api/agent/run/{query,autopilot,swarm,audit}`、sessions/status | AI 运行（见 §4） |

### 异步任务模式

扫描和 Agent 运行是长时间操作，因此 API 采用**启动并轮询**模式，而非请求/响应：

1. `POST /api/scans/run` 或 `/api/agent/run/*` → `202 Accepted` 并返回 UUID（如果已有活跃运行则返回 `409 Conflict`——同一时间只能有一个 Agent 运行）。
2. 轮询 `GET /api/scan/status` 或 `GET /api/agent/status/:id` 直到状态离开 `running`。
3. 获取产物：`GET /api/agent/sessions/:id/{logs,artifacts,artifacts/*}`（日志支持 SSE；产物读取支持嵌套路径和 `?max_bytes=` 上限）。

在运行端点上选择 `"stream": true` 可切换到 Server-Sent Events——`data:` 行携带 `{"type":"chunk|phase|done|error", …}`。大多数消费者应优先使用异步轮询跟踪流程，以保持会话温状态和提示缓存稳定。

---

## 4. Agent 运行 API

`pkg/server/handlers_agent.go` 通过 HTTP 镜像了 `vigolium agent` 子命令。它不会重新实现 Agent 逻辑——它调用与 [agentic-scan.md](agentic-scan.md) 中记录的相同的编排器，并带有一个有意为之的限制：

- **供应商仅由服务器配置。** CLI 的每次调用供应商标志（`--provider`、`--model`、`--oauth-token`……）*不*在 REST 模式中镜像。服务器从 `vigolium-configs.yaml` 中的 `agent.olium.*` 解析一次供应商并重复使用，以保持会话温状态和提示缓存稳定。切换供应商意味着编辑 YAML 并重新加载。
- **向后兼容的 source 字段。** 请求类型暴露 `EffectiveSourcePath()` 以同时接受 `source` 和旧的 `repo_path` JSON 键。
- **审计驱动分发。** `POST /api/agent/run/audit` 接受 `driver: "auto"|"both"|"audit"|"piolium"`；多驱动模式顺序运行，并在流式传输时使用 `driver` 字段多路复用 SSE 数据块。

---

## 相关文档

- [server-and-ingestion.md](../server-and-ingestion.md) — 启动标志、每种输入模式的 curl 示例
- [api-overview.md](../api-overview.md) · [api-references/](../api-references/) — 完整端点目录和请求/响应模式
- [agentic-scan.md](agentic-scan.md) — Agent 端点调用的编排器
- [data-and-storage.md](data-and-storage.md) — 项目范围界定和处理器共享的持久化层
