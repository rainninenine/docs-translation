文档API参考服务器模式服务器模式：通过API导入数据复制页面本指南介绍如何以服务器模式启动 Vigolium，并通过 REST API 和 CLI 将 HTTP 流量导入数据库。复制页面​启动服务器
# 使用 API 密钥启动
export VIGOLIUM_API_KEY=my-secret-key
vigolium server

# 自定义主机和端口
export VIGOLIUM_API_KEY=my-secret-key
vigolium server --host 127.0.0.1 --service-port 9002

# 启用透明 HTTP 代理以记录流量
export VIGOLIUM_API_KEY=my-secret-key
vigolium server --ingest-proxy-port 9003

# 无身份验证（仅开发环境）
vigolium server -A

默认情况下，服务器监听在 `0.0.0.0:9002`。
​CORS 设置
服务器的 CORS 行为由 `~/.vigolium/vigolium-configs.yaml` 中的 `cors_allowed_origins` 控制：
server:
  cors_allowed_origins: reflect-origin

| 值 | 行为 |
|:---|:---|
| `reflect-origin`（默认） | 回显请求中的 `Origin` 头。允许携带凭据。 |
| `*` | 允许所有来源，但不允许凭据（标准通配符）。 |
| （空字符串） | 完全禁用 CORS 中间件。 |
| `https://app.example.com, https://admin.example.com` | 逗号分隔的允许列表。允许携带凭据。 |

​项目范围限定
所有服务器操作均通过 `X-Project-UUID` 请求头限定到特定项目。如果省略，则使用默认项目。
# 导入到特定项目
curl -X POST http://localhost:9002/api/ingest-http \
  -H "Authorization: Bearer my-secret-key" \
  -H "X-Project-UUID: a1b2c3d4-..." \
  -H "Content-Type: application/json" \
  -d '{"input_mode": "url", "content": "https://example.com"}'

所有查询（漏洞发现、HTTP 记录、统计信息、扫描）返回的数据都限定在请求头中指定的项目范围内。有关完整的多租户隔离参考，请参阅 Projects。
​身份验证
所有 API 请求（`/health` 除外）都需要 Bearer 令牌：
Authorization: Bearer my-secret-key

API 密钥解析顺序：`VIGOLIUM_API_KEY` 环境变量 > 配置文件中的 `server.auth_api_key`。
​API 端点
| 方法 | 路径 | 描述 |
|:-----|:-----|:-----|
| GET | / | 应用信息（无需身份验证） |
| GET | /health | 健康检查（无需身份验证） |
| GET | /metrics | Prometheus 指标（无需身份验证） |
| GET | /swagger/* | Swagger UI 和 OpenAPI 规范（无需身份验证） |
| GET | /server-info | 服务器状态、队列深度、记录/漏洞发现计数 |
| GET | /api/modules | 列出可用的扫描器模块 |
| GET | /api/http-records | 查询存储的 HTTP 记录 |
| GET | /api/findings | 查询扫描漏洞发现 |
| POST | /api/ingest-http | 将 HTTP 流量导入数据库 |
| GET | /api/stats | 聚合的扫描统计信息 |
| GET | /api/scope | 查看扫描范围配置 |
| POST | /api/scope | 更新扫描范围配置 |
| GET | /api/config | 查看服务器配置 |
| POST | /api/config | 更新服务器配置 |
| POST | /api/scan | 触发后台扫描 |
| GET | /api/scan/status | 检查扫描状态 |
| DELETE | /api/scan | 取消正在运行的扫描 |
| GET | /api/source-repos | 列出源代码仓库 |
| POST | /api/source-repos | 创建源代码仓库 |
| GET | /api/source-repos/:id | 获取源代码仓库 |
| PUT | /api/source-repos/:id | 更新源代码仓库 |
| DELETE | /api/source-repos/:id | 删除源代码仓库 |
| POST | /api/agent/run/query | 单次 Agent 提示执行 |
| POST | /api/agent/run/autopilot | 自主 AI 驱动扫描会话 |
| POST | /api/agent/run/swarm | AI 引导的多阶段漏洞扫描 |
| POST | /api/agent/run/audit | 统一白盒审计（audit 和/或 piolium），驱动模式：auto\|both\|audit\|piolium |
| POST | /api/agent/chat/completions | 兼容 OpenAI 的聊天补全（同步） |
| GET | /api/agent/status/list | 列出 Agent 运行 |
| GET | /api/agent/status/:id | 获取 Agent 运行状态（完成后包含完整结果） |
| GET | /api/agent/sessions | 分页的会话历史 |
| GET | /api/agent/sessions/:id | 包含调试字段的完整会话详情 |
| GET | /api/agent/sessions/:id/logs | 读取或追踪 runtime.log（支持 SSE） |
| GET | /api/agent/sessions/:id/artifacts | 列出会话工件文件 |
| GET | /api/agent/sessions/:id/artifacts/{name} | 读取特定工件 |

​通过 API 导入数据
`/api/ingest-http` 端点支持多种输入模式。所有请求均使用 POST 方法，请求体为 JSON。
​导入单个 URL
curl -X POST http://localhost:9002/api/ingest-http \
  -H "Authorization: Bearer my-secret-key" \
  -H "Content-Type: application/json" \
  -d '{
    "input_mode": "url",
    "content": "https://example.com/api/users?id=1"
  }'

​导入多个 URL（url_file 模式）
传入一个以换行符分隔的 URL 列表。以 `#` 开头的行被视为注释。
curl -X POST http://localhost:9002/api/ingest-http \
  -H "Authorization: Bearer my-secret-key" \
  -H "Content-Type: application/json" \
  -d '{
    "input_mode": "url_file",
    "content": "https://example.com/api/users?id=1\nhttps://example.com/api/posts?page=2\nhttps://example.com/login"
  }'

​导入 curl 命令
curl -X POST http://localhost:9002/api/ingest-http \
  -H "Authorization: Bearer my-secret-key" \
  -H "Content-Type: application/json" \
  -d '{
    "input_mode": "curl",
    "content": "curl -X POST https://example.com/api/login -H \"Content-Type: application/json\" -d \"{\\\"username\\\":\\\"admin\\\",\\\"password\\\":\\\"test\\\"}\""
  }'

使用 `content_base64` 避免 JSON 转义问题：
# 对 curl 命令进行 base64 编码
ENCODED=$(echo -n 'curl -X POST https://example.com/api/login -H "Content-Type: application/json" -d "{\"username\":\"admin\",\"password\":\"test\"}"' | base64)

curl -X POST http://localhost:9002/api/ingest-http \
  -H "Authorization: Bearer my-secret-key" \
  -H "Content-Type: application/json" \
  -d "{
    \"input_mode\": \"curl\",
    \"content_base64\": \"$ENCODED\"
  }"

​导入原始 HTTP 请求（Burp 风格）
发送 base64 编码的原始 HTTP 请求，可选附带响应：
# 对原始请求进行 base64 编码
RAW_REQ=$(printf 'GET /api/users?id=1 HTTP/1.1\r\nHost: example.com\r\nCookie: session=abc123\r\n\r\n' | base64)

curl -X POST http://localhost:9002/api/ingest-http \
  -H "Authorization: Bearer my-secret-key" \
  -H "Content-Type: application/json" \
  -d "{
    \"input_mode\": \"burp_base64\",
    \"http_request_base64\": \"$RAW_REQ\"
  }"

同时包含请求和响应：
RAW_REQ=$(printf 'POST /api/login HTTP/1.1\r\nHost: example.com\r\nContent-Type: application/json\r\n\r\n{"username":"admin","password":"test"}' | base64)
RAW_RESP=$(printf 'HTTP/1.1 200 OK\r\nContent-Type: application/json\r\n\r\n{"token":"eyJhbGciOiJIUzI1NiJ9..."}' | base64)

curl -X POST http://localhost:9002/api/ingest-http \
  -H "Authorization: Bearer my-secret-key" \
  -H "Content-Type: application/json" \
  -d "{
    \"input_mode\": \"burp_base64\",
    \"http_request_base64\": \"$RAW_REQ\",
    \"http_response_base64\": \"$RAW_RESP\"
  }"

​导入带有 URL 提示的原始 HTTP 请求
原始 HTTP 请求不包含协议（https 与 http），并且 `Host` 头可能与公共主机名不匹配（例如在负载均衡器后面）。使用 `url` 字段提供正确的协议和主机：
RAW_REQ=$(printf 'POST /api/login HTTP/1.1\r\nHost: internal-lb\r\nContent-Type: application/json\r\n\r\n{"user":"admin"}' | base64)

curl -X POST http://localhost:9002/api/ingest-http \
  -H "Authorization: Bearer my-secret-key" \
  -H "Content-Type: application/json" \
  -d "{
    \"input_mode\": \"burp_base64\",
    \"url\": \"https://app.example.com\",
    \"http_request_base64\": \"$RAW_REQ\"
  }"

​导入 OpenAPI / Swagger 规范
curl -X POST http://localhost:9002/api/ingest-http \
  -H "Authorization: Bearer my-secret-key" \
  -H "Content-Type: application/json" \
  -d '{
    "input_mode": "openapi",
    "content": "{\"openapi\":\"3.0.0\",\"info\":{\"title\":\"Example\",\"version\":\"1.0\"},\"servers\":[{\"url\":\"https://api.example.com\"}],\"paths\":{\"/users\":{\"get\":{\"summary\":\"List users\"}},\"/users/{id}\":{\"get\":{\"summary\":\"Get user\",\"parameters\":[{\"name\":\"id\",\"in\":\"path\",\"required\":true,\"schema\":{\"type\":\"integer\"}}]}}}}"
  }'

对于较大的规范，使用 base64：
SPEC=$(base64 < openapi.yaml)

curl -X POST http://localhost:9002/api/ingest-http \
  -H "Authorization: Bearer my-secret-key" \
  -H "Content-Type: application/json" \
  -d "{
    \"input_mode\": \"openapi\",
    \"content_base64\": \"$SPEC\"
  }"

​导入 Postman 集合
COLLECTION=$(base64 < collection.json)

curl -X POST http://localhost:9002/api/ingest-http \
  -H "Authorization: Bearer my-secret-key" \
  -H "Content-Type: application/json" \
  -d "{
    \"input_mode\": \"postman_collection\",
    \"content_base64\": \"$COLLECTION\"
  }"

​通过 CLI 导入数据
`vigolium ingest` 命令支持远程（服务器）和本地（直接写入数据库）两种模式。
​远程导入（向运行中的服务器）
export VIGOLIUM_API_KEY=my-secret-key

# 从标准输入管道传入 URL
cat urls.txt | vigolium ingest -s http://localhost:9002

# 从文件导入
vigolium ingest -s http://localhost:9002 --input targets.txt

# OpenAPI 规范，附带基础 URL
vigolium ingest -s http://localhost:9002 \
  --input api.yaml -I openapi -t https://api.example.com

# 控制提交速率
vigolium ingest -s http://localhost:9002 \
  --input urls.txt --concurrency 20 -r 200

​本地导入（直接写入数据库）
当省略 `--server` 时，请求将被获取并直接存储到本地数据库中：
# 导入 URL（获取每个 URL 并存储请求和响应）
cat urls.txt | vigolium ingest

# 从 OpenAPI 规范导入
vigolium ingest --input api.yaml -I openapi -t https://api.example.com

# 使用自定义扫描 ID 进行标记
vigolium ingest --input urls.txt --scan-id recon-2026-02

# 使用特定的数据库文件
vigolium ingest --input urls.txt --db ./project.db

# 导入到特定项目
vigolium ingest --input urls.txt --project-id a1b2c3d4-...

​通过透明代理导入
使用代理端口启动服务器，以被动记录 HTTP 流量：
export VIGOLIUM_API_KEY=my-secret-key
vigolium server --ingest-proxy-port 9003

然后将您的工具路由到该代理：
# 通过代理使用 curl
curl -x http://localhost:9003 https://example.com/api/users

# 通过代理使用 httpx
echo "https://example.com" | httpx -proxy http://localhost:9003

# 通过代理使用 nuclei
nuclei -u https://example.com -proxy http://localhost:9003

所有经过代理的 HTTP 流量都会自动记录到数据库中。HTTPS CONNECT 隧道将被透传，不会记录。
​查询已导入的数据
​列出 HTTP 记录
# 所有记录（分页，默认 limit=50）
curl -s http://localhost:9002/api/http-records \
  -H "Authorization: Bearer my-secret-key" | jq .

# 按域名过滤
curl -s "http://localhost:9002/api/http-records?domain=example.com" \
  -H "Authorization: Bearer my-secret-key" | jq .

# 按状态码和方法过滤
curl -s "http://localhost:9002/api/http-records?status_code=200,302&method=GET,POST" \
  -H "Authorization: Bearer my-secret-key" | jq .

# 在 URL 和请求头中搜索
curl -s "http://localhost:9002/api/http-records?search=admin&limit=10" \
  -H "Authorization: Bearer my-secret-key" | jq .

# 分页
curl -s "http://localhost:9002/api/http-records?limit=20&offset=40" \
  -H "Authorization: Bearer my-secret-key" | jq .

​列出漏洞发现
# 所有漏洞发现
curl -s http://localhost:9002/api/findings \
  -H "Authorization: Bearer my-secret-key" | jq .

# 按严重性过滤
curl -s "http://localhost:9002/api/findings?severity=high,critical" \
  -H "Authorization: Bearer my-secret-key" | jq .

# 按模块过滤
curl -s "http://localhost:9002/api/findings?module_name=xss-reflected" \
  -H "Authorization: Bearer my-secret-key" | jq .

# 按域名过滤
curl -s "http://localhost:9002/api/findings?domain=example.com" \
  -H "Authorization: Bearer my-secret-key" | jq .

​服务器信息
curl -s http://localhost:9002/server-info \
  -H "Authorization: Bearer my-secret-key" | jq .

响应：
{
  "version": "0.1.0",
  "uptime": "2h15m30s",
  "service_addr": "0.0.0.0:9002",
  "proxy_addr": "0.0.0.0:9003",
  "db_driver": "sqlite",
  "queue_depth": 0,
  "total_records": 1542,
  "total_findings": 23
}

​通过 API 管理扫描
导入 HTTP 记录后，通过 API 触发漏洞扫描。
​触发扫描
curl -s -X POST http://localhost:9002/api/scan \
  -H "Authorization: Bearer my-secret-key" \
  -H "Content-Type: application/json" \
  -d '{}' | jq .

使用特定模块强制重新扫描：
curl -s -X POST http://localhost:9002/api/scan \
  -H "Authorization: Bearer my-secret-key" \
  -H "Content-Type: application/json" \
  -d '{
    "force": true,
    "enable_modules": ["xss-scanner", "sqli-error-based"]
  }' | jq .

成功时返回 202 Accepted，如果扫描已在运行则返回 409 Conflict。
​检查扫描状态
curl -s http://localhost:9002/api/scan/status \
  -H "Authorization: Bearer my-secret-key" | jq .

​取消正在运行的扫描
curl -s -X DELETE http://localhost:9002/api/scan \
  -H "Authorization: Bearer my-secret-key" | jq .

有关完整的请求/响应详情，请参阅 API Reference。
​通过 API 运行 AI Agent
Agent API 提供四种运行模式，与 `vigolium agent` CLI 子命令（query、autopilot、swarm、audit）对应。并发由 `server.agent_heavy_max` 和 `server.agent_light_max` 控制。
​Query，单次 Agent 运行
curl -s -X POST http://localhost:9002/api/agent/run/query \
  -H "Authorization: Bearer my-secret-key" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt_template": "security-code-review",
    "source": "/home/user/src/my-app"
  }' | jq .

至少需要 `prompt_template`、`prompt_file` 或 `prompt` 其中之一。成功时返回 202 Accepted。设置 `"stream": true` 可获取实时 SSE 输出。旧的 `repo_path` JSON 字段仍可作为 `source` 的别名被接受。
​Autopilot，自主扫描
curl -s -X POST http://localhost:9002/api/agent/run/autopilot \
  -H "Authorization: Bearer my-secret-key" \
  -H "Content-Type: application/json" \
  -d '{
    "target": "https://example.com",
    "focus": "API injection",
    "intensity": "balanced",
    "stream": true
  }'

​Swarm，AI 引导的多阶段扫描
curl -s -X POST http://localhost:9002/api/agent/run/swarm \
  -H "Authorization: Bearer my-secret-key" \
  -H "Content-Type: application/json" \
  -d '{
    "input": "https://example.com",
    "discover": true,
    "triage": true,
    "stream": true
  }'

SSE 事件为 `data:` 行，包含 JSON 载荷：`{"type":"chunk","text":"..."}` 用于实时输出，`{"type":"phase","phase":"..."}` 用于 swarm 阶段转换，`{"type":"done","result":{...}}` 表示完成，或 `{"type":"error","error":"..."}` 表示失败。

供应商覆盖仅限 CLI。服务器从 `vigolium-configs.yaml` 中的 `agent.olium.*` 解析一次 olium 供应商。要切换服务器端工作负载的供应商，请编辑 YAML 并重新加载，没有按请求的供应商字段。

​列出所有 Agent 运行
curl -s http://localhost:9002/api/agent/status/list \
  -H "Authorization: Bearer my-secret-key" | jq .

​检查 Agent 运行状态
curl -s http://localhost:9002/api/agent/status/agt-550e8400... \
  -H "Authorization: Bearer my-secret-key" | jq .

运行完成后，响应中包含 `result` 字段，包含完整的 Agent 输出（原始文本、漏洞发现、HTTP 记录）。
有关完整的 Agent 文档（autopilot、swarm、audit、piolium、query、olium），请参阅 Agent Mode；有关请求/响应详情，请参阅 API Reference。
​输入模式参考
| 模式 | 字段 | 描述 |
|:-----|:-----|:-----|
| `url` | `content` | 单个 URL |
| `url_file` | `content` | 以换行符分隔的 URL 列表 |
| `curl` | `content` 或 `content_base64` | curl 命令字符串 |
| `burp_base64` | `http_request_base64` | Base64 编码的原始 HTTP 请求 |
| `openapi` / `swagger` | `content` 或 `content_base64` | OpenAPI/Swagger 规范（JSON 或 YAML） |
| `postman_collection` | `content` 或 `content_base64` | Postman 集合（JSON） |

对于 `burp_base64` 模式，您还可以包含 `http_response_base64` 以将响应与请求一起存储。
对于接受大型载荷的模式，建议使用 `content_base64` 以避免 JSON 转义问题。上一页运行服务器将 Vigolium 作为持久化 REST API 服务器运行，用于流量导入、扫描触发和 Agent 运行。下一页⌘IwebsitetwittergithubdiscordxlinkedinPowered by本文档基于 Mintlify 构建和托管，一个开发者文档平台本页内容启动服务器CORS 设置项目范围限定身份验证API 端点通过 API 导入数据导入单个 URL导入多个 URL（url_file 模式）导入 curl 命令导入原始 HTTP 请求（Burp 风格）导入带有 URL 提示的原始 HTTP 请求导入 OpenAPI / Swagger 规范导入 Postman 集合通过 CLI 导入数据远程导入（向运行中的服务器）本地导入（直接写入数据库）通过透明代理导入查询已导入的数据列出 HTTP 记录列出漏洞发现服务器信息通过 API 管理扫描触发扫描检查扫描状态取消正在运行的扫描通过 API 运行 AI AgentQuery，单次 Agent 运行Autopilot，自主扫描Swarm，AI 引导的多阶段扫描列出所有 Agent 运行检查 Agent 运行状态输入模式参考