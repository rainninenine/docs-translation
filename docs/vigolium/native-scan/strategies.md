# Blackbox Scanning（黑箱扫描）

黑箱扫描从外部测试 Web 应用程序，无需访问源代码。Vigolium 发送精心构造的 HTTP 请求并分析响应以发现漏洞。

## Quick Start（快速开始）

```bash
# 扫描单个 URL
vigolium scan -t https://example.com

# 扫描多个目标
vigolium scan -t https://api.example.com -t https://admin.example.com

# 从文件扫描目标
vigolium scan -T targets.txt
```

## Strategies（策略）

使用 `--strategy` 控制 Vigolium 在动态测试之前执行多少信息收集。

### Lite — 快速，最小化发现

仅对提供的目标运行动态评估阶段。无爬取，无内容发现。

```bash
vigolium scan -t https://example.com --strategy lite
```

最佳用途：快速检查、CI 管道、已知端点。

### Balanced — 默认

运行内容发现、浏览器爬取、已知问题扫描分析和动态评估。

```bash
vigolium scan -t https://example.com
# 等效于：
vigolium scan -t https://example.com --strategy balanced
```

最佳用途：通用扫描，覆盖面良好。

### Deep — 最大信息收集

在 Balanced 基础上增加外部情报采集（Wayback Machine、CommonCrawl 等）。

```bash
vigolium scan -t https://example.com --strategy deep
```

最佳用途：全面评估，希望发现被遗忘的端点和历史路径。

## Phase-by-Phase Walkthrough（分阶段详解）

### Input Formats（输入格式）

Vigolium 通过 `-I` / `--input-mode` 接受多种格式的目标：

| 格式 | 标志 | 示例 |
|--------|------|---------|
| URL 列表（默认） | `-I urls` | `vigolium scan -T urls.txt` |
| OpenAPI / Swagger | `-I openapi` | `vigolium scan -i api-spec.yaml -I openapi` |
| Postman Collection | `-I postman` | `vigolium scan -i collection.json -I postman` |
| Burp XML 导出 | `-I burpxml` | `vigolium scan -i burp-export.xml -I burpxml` |
| 原始 HTTP 请求 | `-I burpraw` | `vigolium scan -i request.txt -I burpraw` |
| cURL 命令 | `-I curl` | `vigolium scan -i curls.sh -I curl` |
| HAR 文件 | `-I har` | `vigolium scan -i traffic.har -I har` |
| Nuclei 输出 | `-I nuclei` | `vigolium scan -i nuclei.json -I nuclei` |

```bash
# 扫描 OpenAPI 规范（自动使用规范中的服务器 URL）
vigolium scan -i openapi.yaml -I openapi

# 针对特定目标扫描 OpenAPI 规范
vigolium scan -i openapi.yaml -I openapi -t https://staging.example.com

# 扫描 Burp Suite 导出
vigolium scan -i burp-session.xml -I burpxml

# 管道输入 curl 命令
cat curls.sh | vigolium scan -I curl

# 带自定义标头和参数值的 OpenAPI
vigolium scan -i api.yaml -I openapi \
  --spec-header "Authorization: Bearer ***" \
  --spec-var "user_id=42"
```

### External Harvesting（外部采集）

查询外部数据源以获取历史 URL 和端点。通过 `--strategy deep` 或 `--external-harvest` 启用。

```bash
# 显式启用
vigolium scan -t https://example.com --external-harvest

# 或通过 deep 策略
vigolium scan -t https://example.com --strategy deep
```

数据源：Wayback Machine、CommonCrawl、AlienVault OTX、URLScan、VirusTotal。URLScan 和 VirusTotal 的 API 密钥可在 `vigolium-configs.yaml` 的 `external_harvester.sources` 下配置。

### Content Discovery（内容发现）

使用 deparos 引擎进行暴力破解目录和文件发现。通过 `--strategy balanced`/`deep` 或 `--discover` 启用。

```bash
# 带时间限制运行内容发现
vigolium scan -t https://example.com --discover --discover-max-time 30m

# 仅运行内容发现阶段
vigolium scan -t https://example.com --only discovery
```

内容发现引擎使用递归暴力破解（默认深度 5）、观察到的文件名变体、JS 分析和大小写敏感自动检测。

### Browser Spidering（浏览器爬取）

基于 Chromium 的爬取，处理 SPA、JavaScript 渲染和表单交互。通过 `--strategy balanced`/`deep` 或 `--spider` 启用。

```bash
# 带时间限制爬取
vigolium scan -t https://example.com --spider --spider-max-time 15m

# 多个浏览器实例以加快爬取速度
vigolium scan -t https://example.com --spider -b 3

# 非无头模式（可见浏览器）
vigolium scan -t https://example.com --spider --headless=false

# 禁用表单填写
vigolium scan -t https://example.com --spider --no-forms
```

爬取标志：
- `-b` / `--browsers` — 浏览器实例数量（默认：1）
- `-E` / `--browser-engine` — `chromium`、`ungoogled` 或 `fingerprint`（默认：`chromium`）
- `--headless` — 无头模式（默认：true）
- `--no-cdp` — 禁用 CDP 事件监听器检测
- `--no-forms` — 禁用自动表单填写
- `--spider-max-time` — 最大持续时间（默认：30m）

### Known Issue Scan（已知问题扫描）

对已发现的主机和响应体运行 Nuclei 模板和 Kingfisher 机密扫描。通过 `--strategy balanced`/`deep` 或策略启用。

默认情况下，已知问题扫描会使用先前阶段（内容发现、爬取）发现的路径前缀丰富其目标列表。这增加了覆盖面——Nuclei 模板针对单个路径前缀（例如 `https://example.com/api/v1/`）运行，而不仅仅是主机根路径。禁用此功能可获得更快但粒度更低的扫描：

```yaml
# vigolium-configs.yaml
known_issue_scan:
  enrich_targets: true    # 默认：true（使用发现的路径作为目标）
                          # false：仅使用主机级别 URL（更快）
```

```bash
# 按标签过滤 Nuclei 模板
vigolium scan -t https://example.com --known-issue-scan-tags cve,misconfig

# 按严重级别过滤
vigolium scan -t https://example.com --known-issue-scan-severities critical,high

# 自定义模板目录
vigolium scan -t https://example.com --known-issue-scan-templates-dir ~/nuclei-templates/
```

### DynamicAssessment（动态评估）

核心扫描阶段。对所有已发现的 HTTP 记录运行活动和被动模块。在所有策略中均启用。CLI 别名：`audit`、`dast`、`assessment`。

使用反馈循环（最多 3 轮）：每轮之后，检查是否有新发现的记录，如有则重新扫描。

OAST（带外应用安全测试）在配置时注入盲回调载荷：

```bash
# 使用固定的 OAST URL
vigolium scan -t https://example.com --oast-url https://your-oast.example.com/callback
```

## Performance Tuning（性能调优）

### CLI Speed Flags（CLI 速度标志）

| 标志 | 默认值 | 描述 |
|------|---------|-------------|
| `-c` / `--concurrency` | 50 | 并发扫描工作线程数 |
| `--max-per-host` | 2 | 每主机最大并发请求数 |
| `-r` / `--rate-limit` | 100 | 每秒最大请求提交数 |
| `--max-host-error` | 30 | 连续 N 次错误后跳过主机 |
| `--max-findings-per-module` | 15 | 每个模块达到此数量后抑制发现（0 = 无限制） |
| `--timeout` | 15s | 每请求 HTTP 超时 |
| `--retries` | 1 | 失败请求的重试次数 |
| `--scanning-max-duration` | 未设置 | 覆盖全局最大扫描持续时间 |
| `--discover-max-time` | 1h | 内容发现的最大持续时间 |
| `--spider-max-time` | 30m | 爬取的最大持续时间 |

```bash
# 高并发快速扫描
vigolium scan -t https://example.com -c 100 -r 200 --max-per-host 5

# 低速率温和扫描
vigolium scan -t https://example.com -c 10 -r 20 --max-per-host 1

# 限时深度扫描
vigolium scan -t https://example.com --strategy deep \
  --scanning-max-duration 1h \
  --discover-max-time 20m \
  --spider-max-time 10m
```

### Scanning Pace（Config File）（扫描速度（配置文件））

`vigolium-configs.yaml` 中的 `scanning_pace` 部分提供集中式速度控制。公共值作为基线被所有阶段继承；每个阶段的子部分覆盖特定值。

```yaml
scanning_pace:
  # 公共默认值（所有阶段继承）
  concurrency: 50
  rate_limit: 100
  max_per_host: 10

  # 每阶段覆盖（0 = 从公共继承）
  discovery:
    concurrency: 30
    max_duration: 1h

  known_issue_scan:
    concurrency: 50
    rate_limit: 100
    max_duration: 30m

  external_harvester:
    max_duration: 5m

  dynamic-assessment:
    concurrency: 50
```

优先级（从高到低）：CLI 标志 > 扫描配置文件 > 每阶段覆盖 > 公共值 > 内置默认值。

## Output Formats（输出格式）

| 格式 | 标志 | 描述 |
|--------|------|-------------|
| 控制台 | `--format console`（默认） | 彩色人类可读表格输出 |
| JSONL | `--format jsonl` 或 `--json` / `-j` | 每行一个 JSON 对象表示一个发现 |
| HTML | `--format html -o report.html` | 交互式 ag-grid HTML 报告 |

```bash
# JSON 输出
vigolium scan -t https://example.com --json

# HTML 报告
vigolium scan -t https://example.com --format html -o scan-report.html

# JSON 输出到文件
vigolium scan -t https://example.com -j -o findings.jsonl
```

## Lightweight Scan Commands（轻量级扫描命令）

用于对单个 URL 或原始请求进行快速、有针对性的扫描。

### `scan-url` — 单 URL

```bash
# 基本 GET 扫描
vigolium scan-url https://example.com/api/users

# 带请求体和标头的 POST
vigolium scan-url https://example.com/api/login \
  --method POST \
  --body '{"username":"admin","password":"test"}' \
  -H "Content-Type: application/json"

# 跳过被动模块
vigolium scan-url https://example.com/api/data --no-passive

# JSON 输出
vigolium scan-url https://example.com/api/users --json

# 带内容发现
vigolium scan-url https://example.com --discover
```

### `scan-request` — 原始 HTTP 请求

```bash
# 从标准输入
echo -e "GET /api/users HTTP/1.1\r\nHost: example.com\r\n" | vigolium scan-request

# 从文件
vigolium scan-request -i request.txt

# 带目标覆盖
vigolium scan-request -i request.txt --target https://staging.example.com
```

当这些命令使用阶段标志（`--discover`、`--spider`、`--external-harvest`、`--known-issue-scan`）时，它们会委托给完整的 Runner 管道（需要数据库）。

## Module Selection（模块选择）

```bash
# 列出所有可用模块
vigolium scan -t https://example.com --list-modules
# 或：
vigolium module ls

# 仅运行特定模块
vigolium scan -t https://example.com -m xss-reflected,sqli-error

# 运行单个模块
vigolium scan-url https://example.com/search?q=test -m xss-reflected
```

### 按标签过滤

模块使用分类标签进行标记（例如 `spring`、`rails`、`django`、`xss`、`injection`、`light`）。使用 `--module-tag` 仅运行匹配特定标签的模块：

```bash
# 运行所有 Spring 相关模块
vigolium scan -t https://example.com --module-tag spring

# 运行所有 Rails 和 Django 模块
vigolium scan -t https://example.com --module-tag rails --module-tag django

# 与 -m 结合使用（结果合并为并集）
vigolium scan -t https://example.com -m xss-reflected --module-tag spring
```

标签使用 OR 逻辑匹配——模块如果匹配*任意*指定标签则运行。当同时提供 `-m` 和 `--module-tag` 时，结果合并（并集）。

## Custom Extensions（自定义插件）

加载 JavaScript 或 YAML 插件模块，可与内置模块一起使用或替代内置模块。详见 [Extension Scanning](extension-scan.md)。

```bash
# 在内置模块基础上添加插件
vigolium scan -t https://example.com --ext ./my-scanner.js

# 从目录加载插件
vigolium scan -t https://example.com --ext-dir ~/my-extensions/

# 仅运行插件模块（跳过内置模块）
vigolium run extension -t https://example.com --ext ./my-scanner.js

# 列出已加载的插件
vigolium ext ls
```

## Heuristics（启发式检查）

预检检查在扫描前检测 WAF、重定向和技术栈。通过 `--heuristics-check` 控制：

```bash
# 跳过启发式检查以加快启动速度
vigolium scan -t https://example.com --skip-heuristics

# 高级启发式检查
vigolium scan -t https://example.com --heuristics-check advanced
```

使用 `--only` 时，启发式检查会自动禁用。

## OAST（Out-of-Band Testing）（带外测试）

OAST 检测应用程序触发带外交互（DNS/HTTP）而非在响应中反射载荷的盲漏洞。Vigolium 使用 [interactsh](https://github.com/projectdiscovery/interactsh) 服务器进行回调跟踪。

OAST **默认启用**。OAST 探测模块将回调 URL 注入插入点，并在扫描期间和扫描后监控交互。

```bash
# 使用固定的 OAST 回调 URL（绕过 interactsh 自动生成）
vigolium scan -t https://example.com --oast-url https://your-oast.example.com/callback
```

`vigolium-configs.yaml` 中的配置：

```yaml
oast:
  enabled: true                     # 启用/禁用 OAST（默认：true）
  server_url: "oast.pro"            # Interactsh 服务器 URL
  token: ""                         # 私有 interactsh 服务器的认证令牌
  poll_interval: 5                  # 交互轮询间隔（秒）
  grace_period: 10                  # 扫描后等待延迟回调的时间（秒）

  oast_url: ""                      # 固定回调 URL（绕过 interactsh）

  enabled_blind_xss: false          # 启用盲 XSS 载荷注入
  blind_xss_src: ""                 # 盲 XSS 载荷的 JavaScript src URL
```

## Mutation Strategy（变异策略）

变异策略控制 Vigolium 如何为参数模糊测试生成载荷。值感知变异分析原始参数值，按语义类型分类，并生成类型适当的变异。

```yaml
mutation_strategy:
  default_modes:
    - append                        # 载荷应用方式："append"、"replace"、"insert"

  value_aware:
    enabled: true                   # 启用值感知变异（默认：true）
    max_per_intent: 5               # 每个参数每个意图的最大变异数
    default_intents:
      - neighbor                    # 相似值（例如 id=5 → id=4、id=6）
      - boundary                    # 边界情况（例如 0、-1、MAX_INT、空值）
      - escalation                  # 权限提升（例如 role=user → role=admin）

    enum_mappings:                  # 自定义枚举提升映射
      role: ["admin", "superadmin", "root"]
      status: ["active", "approved", "verified"]

    param_synonyms:                 # 自定义参数名同义词
      user_id: ["uid", "userId", "account_id"]
```

识别的值类型包括：整数、UUID、电子邮件、JWT、布尔值、路径、顺序 ID 等 15 种以上。每种类型都有专门的相邻值、边界和权限提升变异。

## Project Scoping（项目范围）

使用 `--project-uuid`（带 UUID）或 `--project-name`（带名称）将所有扫描数据限定到特定项目，实现多租户隔离：

```bash
# 在项目内扫描
vigolium scan -t https://example.com --project-uuid a1b2c3d4-...

# 或为您的会话设置环境变量
eval $(vigolium project use a1b2c3d4-...)
vigolium scan -t https://example.com
```

详见 [Projects](../projects.md) 完整的多租户参考。

## Common Scenarios（常见场景）

```bash
# 从 OpenAPI 规范进行 API 扫描
vigolium scan -i api-spec.yaml -I openapi \
  --spec-header "Authorization: Bearer ***"

# 带 HTML 报告的完整深度扫描
vigolium scan -t https://example.com --strategy deep \
  --format html -o report.html

# 带 JSON 输出的快速 CI 检查
vigolium scan -t https://staging.example.com --strategy lite -j

# 重新扫描现有数据库记录
vigolium scan

# 通过代理扫描（Burp、ZAP）
vigolium scan -t https://example.com --proxy http://127.0.0.1:8080

# 使用自定义扫描 ID 进行分组
vigolium scan -t https://example.com --scan-uuid "sprint-42"

# 带流量转储的详细输出
vigolium scan -t https://example.com -v --dump-traffic

# 在项目内仅扫描 Spring 相关模块
vigolium scan -t https://example.com --module-tag spring --project-uuid a1b2c3d4-...
```
