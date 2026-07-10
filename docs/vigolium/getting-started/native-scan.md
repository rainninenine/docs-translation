# Native Scan & Stateless Scanning

**本地扫描（Native Scan）** 是 Vigolium 的确定性、基于 Go 的扫描流水线——快速、模块化且无需 AI。本页是使用 CLI 运行本地扫描的实践指南，重点介绍**无状态扫描（Stateless Scanning）**：通过单条命令获取结果，无需管理持久化数据库。

适用于 CI/CD 流水线、脚本编写、AI Agent 集成以及快速临时检查。概念性深入内容请参阅 [Scanning Modes Overview](../native-scan/scanning-modes-overview.md)；扩展方案集请参阅 [Stateless Scanning](../guides/stateless-scan.md)。

## Stateless at a glance

| 命令 | 持久化到数据库？ | 阶段 | 适用场景 |
|---------|-----------------|--------|------------|
| `scan-url` | 否 | 无——直接运行模块 | 单个 URL，快速 |
| `scan-request` | 否 | 无——直接运行模块 | 原始 HTTP 请求 / curl |
| `scan --stateless` | 否（临时数据库，使用后丢弃） | 完整流水线 | 一次性完整扫描 |
| `scan` | 是（`~/.vigolium/...sqlite`） | 完整流水线 | 持久化项目 |

`scan-url` 和 `scan-request` 从不接触数据库。`scan --stateless` 创建一个临时 SQLite 数据库，运行所有请求的阶段，导出结果，并在退出时删除数据库。

> 使用 `--stateless` 时请传递 `-o/--output`（配合 `--format`）——否则结果会随临时数据库一起丢弃。Vigolium 会在你忘记时打印警告。`--stateless` 和 `--db` 互斥。

## Scan a single URL — `scan-url`

```bash
# 最简单的扫描
vigolium scan-url https://example.com/api/users?id=1

# 用于脚本的 JSON 输出
vigolium scan-url -j https://example.com/api/users?id=1
```

带请求体和请求头的 POST：

```bash
vigolium scan-url \
  --method POST \
  --body '{"user":"admin","pass":"secret"}' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer ***' \
  https://example.com/api/login
```

限定模块范围，跳过不需要的工作：

```bash
# 仅注入类模块（按 ID/名称模糊匹配）
vigolium scan-url -m sqli -m xss "https://example.com/search?q=test"

# 按标签筛选
vigolium scan-url --module-tag injection https://example.com/api/data

# 跳过被动分析和注入点模糊测试，获得最快结果
vigolium scan-url --no-passive --no-insertion-points https://example.com/api/data
```

在扫描*之前*运行发现/爬虫阶段（这些操作将 `scan-url` 提升为完整流水线，需要数据库——传递 `--db`）：

```bash
vigolium scan-url --discover --db /tmp/scan.sqlite https://example.com
```

## Scan a raw HTTP request — `scan-request`

```bash
# 从包含原始 HTTP 请求的文件
vigolium scan-request -i request.txt

# 从标准输入
printf 'GET /api/users?id=1 HTTP/1.1\r\nHost: example.com\r\n\r\n' \
  | vigolium scan-request

# 从 curl 命令（自动检测）
echo "curl -X POST -d 'user=admin' https://example.com/login" \
  | vigolium scan-request
```

当请求文件仅包含路径时，覆盖主机：

```bash
vigolium scan-request -i request.txt --target https://staging.example.com
```

## Piping from stdin

`scan-url` 和 `scan-request` 都会自动检测标准输入格式——纯 URL、curl 命令或原始 HTTP 请求：

```bash
# 纯 URL
echo 'https://example.com/search?q=test' | vigolium scan-url

# Curl 命令
echo "curl -H 'Content-Type: application/json' -d '{\"id\":1}' https://example.com/api" \
  | vigolium scan-url

# 原始 HTTP 请求
printf 'POST /api/login HTTP/1.1\r\nHost: example.com\r\nContent-Type: application/x-www-form-urlencoded\r\n\r\nuser=admin&pass=secret' \
  | vigolium scan-request

# 扫描剪贴板内容（macOS）
pbpaste | vigolium scan-url -j
```

## Full stateless pipeline — `scan --stateless`

运行发现、爬虫和动态评估，不留持久状态。`--stateless` 同时适用于 `scan` 和 `run`：

```bash
# 完整流水线，JSONL 输出，不留痕迹
vigolium scan --stateless -t https://example.com --format jsonl -o results

# 添加内容发现，同时写入 JSONL 和 HTML
vigolium scan --stateless -t https://example.com \
  --discover --format jsonl,html -o scan-output

# 单个阶段，无状态运行
vigolium run dynamic-assessment --stateless -t https://example.com \
  --format jsonl -o results
```

来自文件的多个目标——每个目标获得独立的临时数据库，输出文件名按主机添加后缀，防止结果覆盖：

```bash
vigolium scan --stateless -T targets.txt --format jsonl -o results
# -> results-example.com.jsonl, results-test.example.com.jsonl, ...
```

## Stateless scans from other input sources

```bash
# OpenAPI / Swagger 规范
vigolium scan --stateless -i api.yaml -I openapi \
  -t https://api.example.com --format jsonl -o results

# Postman 集合
vigolium scan --stateless -i collection.json -I postman \
  -t https://api.example.com --format jsonl -o results

# Burp Suite XML 导出
vigolium scan --stateless -i export.xml -I burpxml --format jsonl -o results

# HAR 捕获
vigolium scan --stateless -i traffic.har -I har --format jsonl -o results

# Nuclei JSONL
vigolium scan --stateless -i nuclei.jsonl -I nuclei --format jsonl -o results
```

## Tuning a scan

```bash
# 速度调节——默认值：-c 50, -r 100 req/s, --max-per-host 50, --timeout 15s
vigolium scan --stateless -t https://example.com \
  -c 100 -r 200 --max-per-host 10 --timeout 30s \
  --format jsonl -o results

# 策略预设——在深度和速度之间权衡
vigolium scan --stateless -t https://example.com --strategy lite -o r --format jsonl
vigolium scan --stateless -t https://example.com --strategy deep -o r --format jsonl

# 通过代理路由所有流量
vigolium scan --stateless -t https://example.com --proxy http://127.0.0.1:8080 \
  --format jsonl -o results

# 限制范围的解释宽度
vigolium scan --stateless -t https://example.com --scope-origin strict \
  --format jsonl -o results

# 在漏洞发现中包含完整 HTTP 响应体（仅 scan / run）
vigolium scan --stateless -t https://example.com --include-response \
  --format jsonl -o results
```

## Authenticated stateless scans

传递内联会话或会话文件——两者在无状态模式下均有效：

```bash
# 内联会话：name:Header:value
vigolium scan --stateless -t https://example.com \
  --auth "admin:Cookie:session_id=abc123" \
  --format jsonl -o results

# 会话/认证配置文件（YAML 或 JSON）
vigolium scan --stateless -t https://example.com \
  --auth-file ./admin-session.yaml \
  --format jsonl -o results

# 静态请求头通常足以用于令牌认证
vigolium scan-url -H 'Authorization: Bearer ***' \
  https://example.com/api/me
```

有关登录流程、令牌提取和多会话 IDOR/BOLA 测试，请参阅 [Authenticated Scanning](../native-scan/authentication.md)。

## CI/CD integration

`--ci-output-format` 强制输出干净的 JSONL，不含横幅或颜色代码，非常适合在流水线中解析：

```bash
vigolium scan --stateless -t https://example.com \
  --ci-output-format -o findings
```

一个简单的门控，当发现任何漏洞时使构建失败：

```bash
vigolium scan --stateless -t "$TARGET" --ci-output-format -o findings
test ! -s findings.jsonl || { echo "Vulnerabilities found"; exit 1; }
```

完整的流水线示例请参阅 [CI/CD Integration](../guides/ci-cd-integration.md)。

## Output formats recap

| `--format` | 输出 | 说明 |
|------------|--------|-------|
| `console` | 终端（默认） | 彩色，人类可读 |
| `jsonl` | `<o>.jsonl` | 每行一个 JSON 对象；`-j` 为简写 |
| `html` | `<o>.html` | 交互式 ag-grid 报告；需要 `-o` |
| `console,jsonl,html` | 以上全部 | 逗号分隔以组合 |

对于无状态运行，`-o` 是**基础**路径——Vigolium 会为每种格式追加正确的扩展名，并在销毁临时数据库之前从其中生成所有请求的格式。

## Cheat sheet

```bash
# 快速检查单个端点
vigolium scan-url https://example.com/api/users?id=1

# 一次性完整扫描，JSON 输出，无持久化
vigolium scan --stateless -t https://example.com --discover --format jsonl -o findings

# 扫描剪贴板中的 curl 命令
pbpaste | vigolium scan-url -j

# API 规范 → HTML 报告，不留痕迹
vigolium scan --stateless -i openapi.yaml -I openapi \
  -t https://api.example.com --format html -o report

# CI 门控
vigolium scan --stateless -t "$TARGET" --ci-output-format -o findings
```

## Next steps

- [Stateless Scanning](../guides/stateless-scan.md) —— 扩展方案集
- [Scanning Strategies](../native-scan/strategies.md) —— 策略、配置文件、速率
- [Native Scan: How It Works](../architecture/native-scan.md) —— 流水线内部机制
- [Scanner Modules Reference](../native-scan/modules-reference.md) —— 所有模块
- [Output & Reporting](../output-and-reporting.md) —— 格式和报告的深入介绍
