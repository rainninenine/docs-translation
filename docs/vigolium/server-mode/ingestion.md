# Ingestion

Vigolium 支持多种方式将 HTTP 流量导入其数据库。无论您是从文件导入、从代理捕获，还是通过 API 发送，Vigolium 都能处理。

## 通过 CLI 导入

### 从文件导入

```bash
# 从 Burp Suite XML 导入
vigolium ingest -I burp -i export.xml

# 从 OpenAPI/Swagger 规范导入
vigolium ingest -I openapi -i spec.yaml

# 从 Postman 集合导入
vigolium ingest -I postman -i collection.json

# 从 URL 列表导入
vigolium ingest -I url -i urls.txt

# 从原始 HTTP 请求导入
vigolium ingest -I raw -i request.txt
```

### 从标准输入导入

```bash
# 通过管道传输 curl 命令
echo 'curl -H "Authorization: Bearer token" https://api.example.com/users' | vigolium ingest -I curl

# 通过管道传输 URL
cat urls.txt | vigolium ingest -I url
```

### 导入时扫描

```bash
# 导入并在导入的流量上立即运行扫描
vigolium scan -I openapi -i spec.yaml -t https://api.example.com

# 导入 Burp 导出文件并扫描
vigolium scan -I burp -i export.xml -t https://example.com
```

## 通过服务器 API 导入

当 Vigolium 作为服务器运行时，您可以通过 REST API 导入流量：

```bash
# 导入单个 URL
curl -X POST http://localhost:9002/api/ingest-http \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "input_mode": "url",
    "content": "https://example.com/api/users"
  }'

# 导入 curl 命令
curl -X POST http://localhost:9002/api/ingest-http \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "input_mode": "curl",
    "content": "curl -H \"Authorization: Bearer token\" https://api.example.com/users"
  }'

# 导入 Burp 请求（Base64 编码）
curl -X POST http://localhost:9002/api/ingest-http \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "input_mode": "burp_base64",
    "http_request_base64": "R0VUIC9hcGkvdXNlcnMgSFRUUC8xLjENCkhvc3Q6IGV4YW1wbGUuY29t"
  }'
```

### 输入模式

| `input_mode` | 载荷字段 | 来源 |
|---|---|---|
| `url` / `url_file` | `content` | 单个 URL / 换行分隔的列表 |
| `curl` | `content` 或 `content_base64` | curl 命令字符串 |
| `burp_base64` | `http_request_base64`（+ 可选的 `http_response_base64`、`url` 提示） | 原始 HTTP 请求 |
| `openapi` / `swagger` | `content` 或 `content_base64` | OpenAPI/Swagger 规范 |
| `postman_collection` | `content_base64` | Postman 集合 |

## 通过透明代理导入

Vigolium 服务器可以运行一个透明 HTTP 代理来捕获流量：

```bash
# 启动带代理的服务器
vigolium server --ingest-proxy-port 9003

# 通过代理发送流量
curl -x http://localhost:9003 http://example.com/api/users
```

详见 [Proxy](proxy.md)。

## 通过 `vigolium ingest` CLI 客户端导入

`vigolium ingest` 命令有两种模式：

### 远程模式

连接到正在运行的 Vigolium 服务器并将数据 POST 到 `/api/ingest-http`：

```bash
vigolium ingest -I openapi -i spec.yaml -s http://localhost:9002
```

### 本地模式

省略 `-s` 标志直接写入本地数据库：

```bash
vigolium ingest -I openapi -i spec.yaml
```

## 数据导入后

导入后，您可以：

```bash
# 查看导入的流量
vigolium traffic

# 在导入的流量上运行扫描
vigolium scan --only dynamic-assessment

# 导出结果
vigolium export --format html -o report.html
```

## 延伸阅读

- [Proxy](proxy.md) — 透明代理详情
- [Running the Server](running-the-server.md) — 服务器启动和配置
- [Server & API Architecture](../architecture/server-and-api.md) — 服务器架构
