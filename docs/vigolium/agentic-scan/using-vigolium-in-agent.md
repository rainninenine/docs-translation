# 在 Claude Code 和 Codex 中使用 Vigolium Scanner 技能

本指南说明如何在 AI 编程 Agent（Claude Code 和 OpenAI Codex）中安装和使用 `vigolium-scanner` 技能（`public/skills/vigolium-scanner/`），以操作 Vigolium CLI 进行 Web 漏洞扫描、安全测试和自定义插件编写。

---

## 目录

- [技能功能](#技能功能)
- [技能结构](#技能结构)
- [安装](#安装)
- [按类别分类的使用示例](#按类别分类的使用示例)
  - [扫描](#1-扫描)
  - [输入格式](#2-输入格式)
  - [阶段控制](#3-阶段控制)
  - [模块过滤](#4-模块过滤)
  - [服务器与数据导入](#5-服务器与数据导入)
  - [AI Agent 模式](#6-ai-agent-模式)
  - [流量与结果浏览](#7-流量与结果浏览)
  - [数据管理](#8-数据管理)
  - [导出与报告](#9-导出与报告)
  - [JavaScript 插件](#10-javascript-插件)
  - [设置与项目](#11-设置与项目)
- [自然语言示例](#自然语言示例)
- [提示与最佳实践](#提示与最佳实践)

---

## 技能功能

该技能教会 AI Agent 如何：

1. **为任何安全测试任务选择正确的 vigolium 命令**
2. **使用正确的语法构造正确的标志组合**
3. **端到端遵循扫描工作流程**（数据导入 → 扫描 → 分类 → 导出）
4. **使用 `vigolium.*` API 编写自定义 JavaScript 插件**
5. **操作 AI Agent 模式**（query、autopilot、swarm）
6. **管理数据**——浏览流量、过滤漏洞发现、导出报告、清理数据库

该技能使用懒加载引用：主 `SKILL.md` 保持精简，详细文档在 Agent 需要深度标志信息或插件编写指导时按需加载。

---

## 技能结构

```
public/skills/vigolium-scanner/
├── SKILL.md                              # 主技能（决策树、配方、标志）
└── references/
    ├── scanning-commands.md              # scan、scan-url、scan-request、run
    ├── server-and-ingestion.md           # server、ingest、traffic、traffic --replay
    ├── agent-commands.md                 # agent、agent query、autopilot、swarm
    ├── data-and-management.md            # db、module、ext、config、scope、export
    ├── flags-reference.md                # 完整的字母顺序标志索引
    └── writing-extensions.md             # JS 插件 API 和示例
```

---

## 安装

**选项 A：通过 npx / bunx 安装（推荐）**

```bash
bunx skills add vigolium/skills --skill vigolium-scanner --agent <agent-name> --yes
```

或使用 `npx`：

```bash
npx skills add vigolium/skills --skill vigolium-scanner --agent <agent-name> --yes
```

将 `<agent-name>` 替换为您的 Agent（例如 `claude-code`、`codex`）。这将从 [vigolium/skills](https://github.com/vigolium/skills) 仓库获取技能并自动注册。

**选项 B：手动克隆和复制**

```bash
git clone https://github.com/vigolium/skills.git
cd skills
```

然后将技能文件夹复制到 Agent 的配置目录：

```bash
# 对于 Claude Code
cp -R vigolium-scanner ~/.claude

# 对于其他 Agent
cp -R vigolium-scanner ~/.agents
```

安装完成后，当您提到 `scan`、`vigolium`、`agent autopilot`、`vulnerability scanner`、`openapi scan` 等关键词时，技能会**自动触发**。在 Claude Code 中，您也可以使用 `/vigolium-scanner` 显式调用它。

---

## 按类别分类的使用示例

### 1. 扫描

**对单个目标进行基本扫描：**
```
> Scan https://example.com for vulnerabilities
```
```bash
vigolium scan -t https://example.com
```

**多个目标：**
```
> Scan both https://example.com and https://api.example.com
```
```bash
vigolium scan -t https://example.com -t https://api.example.com
```

**从文件读取目标：**
```
> I have a list of URLs in targets.txt, scan all of them
```
```bash
vigolium scan -T targets.txt
```

**使用特定策略进行扫描：**
```
> Do a deep scan of https://example.com with discovery and spidering
```
```bash
vigolium scan -t https://example.com --strategy deep
```

**使用自定义方法、标头和正文扫描单个 URL：**
```
> Test the login endpoint for vulnerabilities: POST to https://api.example.com/login with JSON credentials
```
```bash
vigolium scan-url https://api.example.com/login \
  --method POST \
  --body '{"username":"admin","password":"test123"}' \
  -H "Content-Type: application/json"
```

**从文件扫描原始 HTTP 请求：**
```
> I captured a raw request in request.txt, scan it
```
```bash
vigolium scan-request -i request.txt
```

**从 stdin 扫描原始请求：**
```
> Scan this raw request from the terminal
```
```bash
echo -e "GET /api/users?id=1 HTTP/1.1\r\nHost: example.com\r\nAuthorization: Bearer ***" | vigolium scan-request
```

**使用代理进行扫描（例如 Burp Suite）：**
```
> Scan https://example.com and route traffic through Burp
```
```bash
vigolium scan -t https://example.com --proxy http://127.0.0.1:8080
```

**高速扫描，调整并发数：**
```
> Scan fast — 100 workers, 200 req/s, max 5 per host
```
```bash
vigolium scan -t https://example.com -c 100 --rate-limit 200 --max-per-host 5
```

**扫描并以 JSONL 格式输出结果：**
```
> Scan and save results as JSON lines
```
```bash
vigolium scan -t https://example.com --format jsonl -o results.jsonl
```

**扫描并生成 HTML 报告：**
```
> Scan and produce an interactive HTML report
```
```bash
vigolium scan -t https://example.com --format html -o report.html
```

**使用自定义扫描配置文件进行扫描：**
```
> Use the aggressive scanning profile
```
```bash
vigolium scan -t https://example.com --scanning-profile aggressive
```

**使用严格的来源范围进行扫描：**
```
> Only scan URLs on the exact same origin
```
```bash
vigolium scan -t https://example.com --scope-origin strict
```

---

### 2. 输入格式

**带显式基础 URL 的 OpenAPI 3.x 规范：**
```
> Scan my API using the OpenAPI spec
```
```bash
vigolium scan -I openapi -i api-spec.yaml -t https://api.example.com
```

**使用规范中定义的服务器 URL 的 OpenAPI 规范：**
```
> Use the server URLs defined in the spec itself
```
```bash
vigolium scan -I openapi -i api-spec.yaml --spec-url
```

**带身份验证标头和参数值的 OpenAPI：**
```
> Scan the spec with bearer auth and set the user_id parameter to 42
```
```bash
vigolium scan -I openapi -i spec.yaml -t https://api.example.com \
  --spec-header "Authorization: Bearer ***" \
  --spec-var "user_id=42"
```

**Swagger 2.0 规范：**
```
> Import a Swagger 2.0 spec and scan
```
```bash
vigolium scan -I swagger -i swagger.json -t https://api.example.com
```

**Burp Suite XML 导出：**
```
> I exported traffic from Burp, scan it
```
```bash
vigolium scan -I burp -i burp-export.xml -t https://example.com
```

**HAR（HTTP Archive）文件：**
```
> Scan my browser-recorded HAR file
```
```bash
vigolium scan -I har -i traffic.har
```

**cURL 命令文件：**
```
> I have a file of curl commands, scan them all
```
```bash
vigolium scan -I curl -i curl-commands.txt
```

**Postman 集合：**
```
> Import and scan my Postman collection
```
```bash
vigolium scan -I postman -i collection.json -t https://api.example.com
```

**Nuclei 模板：**
```
> Run these Nuclei templates against the target
```
```bash
vigolium scan -I nuclei -i templates/ -t https://example.com
```

**从 stdin 管道输入 URL：**
```
> Pipe a list of URLs into the scanner
```
```bash
cat urls.txt | vigolium scan -i -
```

---

### 3. 阶段控制

**仅运行内容发现：**
```
> Just run content discovery against the target
```
```bash
vigolium run discover -t https://example.com
# 或
vigolium scan -t https://example.com --only discovery
```

**仅运行爬取（无头浏览器爬取）：**
```
> Spider the target with a headless browser
```
```bash
vigolium run spidering -t https://example.com
```

**仅运行动态评估（漏洞扫描）：**
```
> Skip discovery, just run the vulnerability modules
```
```bash
vigolium run dynamic-assessment -t https://example.com
# 或
vigolium scan -t https://example.com --only dynamic-assessment
```

**仅运行已知问题扫描（Nuclei 模板 + Kingfisher 密钥）：**
```
> Run known-issue scan, only critical and high severity
```
```bash
vigolium run known-issue-scan -t https://example.com --known-issue-scan-severities critical,high
```

**仅运行外部收集（Wayback、Common Crawl、OTX）：**
```
> Gather URLs from external intelligence sources
```
```bash
vigolium run external-harvest -t https://example.com
```

**跳过特定阶段：**
```
> Scan but skip discovery and spidering
```
```bash
vigolium scan -t https://example.com --skip discovery,spidering
```

**仅运行 JavaScript 插件：**
```
> Run only my custom extension, skip built-in modules
```
```bash
vigolium scan -t https://example.com --only extension --ext ./custom-check.js
# 或
vigolium run ext -t https://example.com --ext ./custom-check.js
```

**阶段别名参考：**

| 别名 | 解析为 |
|---|---|
| `deparos`、`discover` | `discovery` |
| `spitolas` | `spidering` |
| `ext` | `extension` |

---

### 4. 模块过滤

**列出所有可用的扫描器模块：**
```
> Show me all scanner modules
```
```bash
vigolium module ls
# 或
vigolium scan -M
```

**按关键词过滤模块：**
```
> Show me all XSS-related modules
```
```bash
vigolium module ls xss
```

**仅列出活跃模块并显示详细信息：**
```
> Show active modules with descriptions
```
```bash
vigolium module ls --type active -v
```

**仅使用特定模块进行扫描：**
```
> Only run reflected XSS and error-based SQL injection modules
```
```bash
vigolium scan -t https://example.com -m xss-reflected,sqli-error
```

**按标签过滤模块（OR 逻辑）：**
```
> Scan with modules tagged 'spring' or 'injection'
```
```bash
vigolium scan -t https://example.com --module-tag spring --module-tag injection
```

**组合模块 ID 和标签：**
```
> Run sqli-error plus all XSS-tagged modules
```
```bash
vigolium scan -t https://example.com -m sqli-error --module-tag xss
```

**持久启用/禁用模块：**
```
> Disable all SQL injection modules, enable all XSS modules
```
```bash
vigolium module disable sqli
vigolium module enable xss
```

**按精确模块 ID 启用：**
```
> Enable only the reflected XSS module
```
```bash
vigolium module enable active-xss-reflected --id
```

---

### 5. 服务器与数据导入

**启动 API 服务器（默认端口 9002）：**
```
> Start the vigolium server
```
```bash
vigolium server
```

**在自定义端口启动服务器，无需身份验证：**
```
> Start the server on port 8443 with no authentication
```
```bash
vigolium server --service-port 8443 --no-auth
```

**启动服务器并启用接收时扫描（自动扫描导入的流量）：**
```
> Start the server and auto-scan every request that comes in
```
```bash
vigolium server -t https://example.com --scan-on-receive
```

**启动服务器并启用透明代理进行记录：**
```
> Start server with a recording proxy on port 8080
```
```bash
vigolium server --ingest-proxy-port 8080
```

**高并发服务器：**
```
> Start a high-throughput server with 200 workers
```
```bash
vigolium server -c 200 --mem-buffer 50000
```

**本地导入 OpenAPI 规范：**
```
> Import the spec into the database without scanning
```
```bash
vigolium ingest -t https://api.example.com -I openapi -i spec.yaml
```

**导入并自动扫描：**
```
> Import the spec and scan immediately
```
```bash
vigolium ingest -t https://api.example.com -I openapi -i spec.yaml -S
```

**导入 Burp 导出文件：**
```
> Import Burp traffic into the database
```
```bash
vigolium ingest -t https://example.com -I burp -i export.xml
```

**远程导入到运行中的服务器：**
```
> Send traffic to the vigolium server running on localhost
```
```bash
vigolium ingest -s http://localhost:9002 -I openapi -i spec.yaml
```

**导入时不获取响应：**
```
> Store request-only records, don't make network requests
```
```bash
vigolium ingest -t https://example.com -I burp -i export.xml --disable-fetch-response
```

---

### 6. AI Agent 模式

#### Agent（基于模板）

**安全代码审查：**
```
> Review my source code for security vulnerabilities
```
```bash
vigolium agent query --prompt-template security-code-review --source ./src
```

**从源代码发现端点：**
```
> Find all API endpoints in my source code
```
```bash
vigolium agent query --prompt-template endpoint-discovery --source ./src
```

**仅审查特定文件：**
```
> Review only auth.go and middleware.go for security issues
```
```bash
vigolium agent query --prompt-template security-code-review --source ./src \
  --files "src/auth.go,src/middleware.go"
```

**向模板追加额外指令：**
```
> Code review, but focus on authentication and authorization
```
```bash
vigolium agent query --prompt-template security-code-review --source ./src \
  --append "Focus specifically on authentication and authorization vulnerabilities"
```

**使用自定义提示文件：**
```
> Run the agent with my own prompt template
```
```bash
vigolium agent query --prompt-file custom-prompt.md --source ./src
```

**试运行以预览渲染后的提示：**
```
> Show me what prompt would be sent to the agent
```
```bash
vigolium agent query --prompt-template security-code-review --source ./src --dry-run
```

**将 Agent 输出保存到文件：**
```
> Save the review results to a JSON file
```
```bash
vigolium agent query --prompt-template security-code-review --source ./src \
  --output review-results.json
```

**列出可用模板：**
```
> What prompt templates are available?
```
```bash
vigolium agent --list-templates
```

**内置模板包括：**
- `security-code-review` — 综合安全审查
- `injection-sinks` — 查找注入点
- `auth-bypass` — 身份验证绕过向量
- `secret-detection` — 硬编码密钥
- `endpoint-discovery` — 从源代码发现 API 端点
- `api-input-gen` — 生成测试输入
- `curl-command-gen` — 生成 cURL 命令
- `attack-surface-mapper` — 映射攻击面
- `nextjs-security-audit` — Next.js 安全审查
- `react-xss-audit` — React XSS 审计
- `cors-csrf-review` — CORS/CSRF 配置审计

#### Agent Query（自由格式提示）

**内联提示：**
```
> Ask the agent to review code for vulnerabilities
```
```bash
vigolium agent query 'review this code for SQL injection vulnerabilities'
```

**命名提示标志：**
```
> Analyze the authentication flow
```
```bash
vigolium agent query --prompt 'analyze the authentication flow for bypass vectors'
```

**从 stdin 管道输入提示：**
```
> Pipe a prompt to the agent
```
```bash
echo "check for SSRF in the URL-fetching handler" | vigolium agent query --stdin
```

**带扩展超时：**
```
> Run a comprehensive review with extra time
```
```bash
vigolium agent query --max-duration 10m 'comprehensive security review of all handlers'
```

#### Agent Autopilot（自主扫描）

**基本自主扫描：**
```
> Let the AI autonomously scan the target
```
```bash
vigolium agent autopilot -t https://example.com
```

**带源代码上下文和关注领域：**
```
> Autonomous scan focused on auth bypass, with source code for context
```
```bash
vigolium agent autopilot -t https://api.example.com --source ./src --focus "auth bypass"
```

**自定义限制（更少的命令，更短的超时）：**
```
> Limit the agent to 50 commands and 15 minutes
```
```bash
vigolium agent autopilot -t https://example.com --max-commands 50 --timeout 15m
```

**预览系统提示（试运行）：**
```
> Show me what system prompt the autopilot agent would receive
```
```bash
vigolium agent autopilot -t https://example.com --dry-run
```

**自定义系统提示：**
```
> Use my own system prompt for autopilot
```
```bash
vigolium agent autopilot -t https://example.com --system-prompt my-system-prompt.md
```

**每次调用使用不同的供应商：**
```
> Run autopilot using the anthropic-api-key provider
```
```bash
vigolium agent autopilot -t https://example.com --provider anthropic-api-key --llm-api-key "$ANTHROPIC_API_KEY"
```

**Autopilot 工具集：**
- `bash`（灾难性模式被硬拒绝：`rm -rf /`、fork 炸弹、对块设备的 `dd`、对真实设备的 `mkfs`）、`read_file`、`write_file`、`edit_file`、`ls`、`grep`、`glob`、`web_fetch`
- 加上 autopilot 独有工具：`halt_scan`、`report_finding`、`load_skill`（当技能已加载时）和 vigtool（`run_scan`、`list_findings` 等）（当附加了数据库 Repository 时）
- 每个工具超时：5 分钟（可通过 `agent.olium.call_timeout_sec` 配置）
- 硬上限：200 个漏洞发现（50 个时发出软警告）；轮次上限来自 `--intensity`：quick 150、balanced 500（默认）、deep 1500

#### Agent Swarm（AI 引导的多阶段扫描）

**带发现功能的基本 swarm 扫描（所有阶段）：**
```
> Run the full AI swarm scan
```
```bash
vigolium agent swarm --discover -t https://example.com
```

Swarm 运行以下阶段：
1. **发现** — 本地内容发现 + 爬取（无 AI）
2. **规划** — AI 分析发现结果，生成攻击计划
3. **扫描** — 本地执行器使用 Agent 选择的模块（无 AI）
4. **分类** — AI 审查漏洞发现，确认/驳回，建议后续操作
5. **重新扫描** — 根据分类建议进行定向重新扫描（无 AI）
6. **报告** — 从数据库生成结构化输出（无 AI）

**带关注领域和源代码的 swarm 扫描：**
```
> Swarm scan focused on SQL injection, with source code
```
```bash
vigolium agent swarm --discover -t https://example.com --focus "SQL injection" --source ./src
```

**控制重新扫描迭代次数：**
```
> Allow up to 3 triage→rescan iterations
```
```bash
vigolium agent swarm --discover -t https://example.com --max-iterations 3
```

**跳过发现并从规划开始（使用现有数据库数据）：**
```
> I already have traffic in the database, start from planning
```
```bash
vigolium agent swarm --discover -t https://example.com --skip discover --start-from plan
```

**跳过分类（仅发现 → 规划 → 扫描）：**
```
> Run swarm but skip triage and rescan
```
```bash
vigolium agent swarm --discover -t https://example.com --skip triage --skip rescan
```

**使用扫描配置文件：**
```
> Run swarm with the deep scanning profile
```
```bash
vigolium agent swarm --discover -t https://example.com --profile deep
```

**预览 Agent 提示（试运行）：**
```
> Show me the prompts without executing
```
```bash
vigolium agent swarm --discover -t https://example.com --dry-run
```

**用于 Agent 上下文的特定源文件：**
```
> Only include routes.go and handlers.go as context
```
```bash
vigolium agent swarm --discover -t https://example.com --source ./src \
  --files "routes.go,handlers.go"
```

**每次调用使用不同的供应商：**
```
> Run swarm using anthropic-api-key
```
```bash
vigolium agent swarm --discover -t https://example.com --provider anthropic-api-key --llm-api-key "$ANTHROPIC_API_KEY"
```

---

### 7. 流量与结果浏览

**浏览所有存储的 HTTP 流量：**
```
> Show me the HTTP traffic in the database
```
```bash
vigolium traffic
```

**模糊搜索流量：**
```
> Show traffic related to login
```
```bash
vigolium traffic login
```

**树形视图（分层 URL 结构）：**
```
> Show traffic as a directory tree
```
```bash
vigolium traffic --tree
```

**Burp 风格彩色输出：**
```
> Show traffic in Burp Suite style
```
```bash
vigolium traffic --burp
```

**按主机、方法、状态码过滤：**
```
> Show POST and PUT requests to api.example.com that returned 200
```
```bash
vigolium traffic --host api.example.com --method POST,PUT --status 200
```

**按日期范围过滤：**
```
> Show traffic from January 2024
```
```bash
vigolium traffic --from 2024-01-01 --to 2024-01-31
```

**在请求/响应正文中搜索：**
```
> Find traffic containing "password" in the body
```
```bash
vigolium traffic --body password
```

**在标头中搜索：**
```
> Find traffic with JWT tokens in headers
```
```bash
vigolium traffic --header "Bearer"
```

**自定义列：**
```
> Show host, method, path, status, and auth columns
```
```bash
vigolium traffic --columns HOST,METHOD,PATH,STATUS,AUTH
```

**监视模式（自动刷新）：**
```
> Monitor traffic in real-time, refresh every 5 seconds
```
```bash
vigolium traffic --watch 5s
```

**查看原始 HTTP 请求/响应：**
```
> Show raw traffic for the last 5 records
```
```bash
vigolium traffic --raw --limit 5
```

**浏览漏洞发现：**
```
> Show all vulnerability findings
```
```bash
vigolium finding
```

**按严重性过滤漏洞发现：**
```
> Show only high and critical findings
```
```bash
vigolium finding --severity high,critical
```

**搜索漏洞发现：**
```
> Find SQL injection findings
```
```bash
vigolium finding --search "sql injection"
```

**实时监视漏洞发现：**
```
> Monitor findings as they come in
```
```bash
vigolium finding --watch 5s
```

**重放存储的流量（重新发送请求）：**
```
> Replay login-related requests and compare responses
```
```bash
vigolium traffic login --replay
```

**重放并替换存储的响应：**
```
> Replay requests to api.example.com and update stored responses
```
```bash
vigolium traffic --host api.example.com --replay --in-replace
```

---

### 8. 数据管理

**数据库统计信息：**
```
> Show me database stats
```
```bash
vigolium db stats
```

**带主机细分的详细统计信息：**
```
> Show detailed stats broken down by host
```
```bash
vigolium db stats --detailed
```

**特定主机的统计信息：**
```
> Stats for example.com only
```
```bash
vigolium db stats --host example.com
```

**实时更新统计信息：**
```
> Watch database stats, refresh every 10 seconds
```
```bash
vigolium db stats --watch 10s
```

**带过滤器的数据库记录列表：**
```
> Show findings table, critical and high severity
```
```bash
vigolium db ls --table findings --severity critical,high
```

**列出可用表和列：**
```
> What tables are in the database? What columns does findings have?
```
```bash
vigolium db ls --list-tables
vigolium db ls --list-columns --table findings
```

**按主机名清理记录：**
```
> Delete all records for old-target.com
```
```bash
vigolium db clean --host old-target.com --force
```

**带试运行预览的旧记录清理：**
```
> Preview what would be deleted before January 2024
```
```bash
vigolium db clean --before 2024-01-01 --dry-run
```

**仅清理漏洞发现（保留 HTTP 记录）：**
```
> Delete info-severity findings but keep the HTTP records
```
```bash
vigolium db clean --findings-only --severity info --force
```

**清理孤立漏洞发现：**
```
> Remove findings without associated HTTP records
```
```bash
vigolium db clean --orphans
```

**重置整个数据库：**
```
> Wipe the entire database and start fresh
```
```bash
vigolium db clean --force
```

**删除后回收磁盘空间：**
```
> Vacuum the database to reclaim space
```
```bash
vigolium db clean --vacuum
```

---

### 9. 导出与报告

**完整 JSONL 导出：**
```
> Export everything from the database as JSONL
```
```bash
vigolium export --format jsonl -o full-export.jsonl
```

**仅导出漏洞发现：**
```
> Export just the findings
```
```bash
vigolium export --format jsonl --only findings -o findings.jsonl
```

**导出漏洞发现和 HTTP 记录：**
```
> Export findings and associated HTTP traffic
```
```bash
vigolium export --format jsonl --only findings,http -o results.jsonl
```

**HTML 报告：**
```
> Generate an interactive HTML report
```
```bash
vigolium export --format html -o report.html
```

**精简导出（省略原始 HTTP 请求/响应字节，保留元数据）：**
```
> Export HTTP records without the raw request/response bodies
```
```bash
vigolium export --omit-response --only http -o urls.jsonl
```

**带搜索过滤器的导出：**
```
> Export only records matching example.com
```
```bash
vigolium export --search "example.com" -o filtered.jsonl
```

**数据库级 CSV 导出：**
```
> Export HTTP records as CSV
```
```bash
vigolium db export -f csv -o records.csv
```

**导出为 Markdown：**
```
> Export records as a Markdown report
```
```bash
vigolium db export -f markdown -o report.md
```

**仅导出原始请求：**
```
> Export just the raw HTTP requests
```
```bash
vigolium db export -f raw --request-only -o requests.txt
```

**按主机和日期过滤导出：**
```
> Export records for example.com from 2024 onwards
```
```bash
vigolium db export -f csv -o records.csv --host example.com --from 2024-01-01
```

**按 UUID 导出单条记录：**
```
> Export record abc12345
```
```bash
vigolium db export --uuid abc12345
```

**导出模块注册表：**
```
> Export all available scanner modules
```
```bash
vigolium export --only modules
```

---

### 10. JavaScript 插件

**安装预设示例：**
```
> Install the example extension scripts
```
```bash
vigolium ext preset
```

**查看插件 API 参考：**
```
> Show me the extension API docs
```
```bash
vigolium ext docs
vigolium ext docs --example          # 带代码示例
vigolium ext docs http               # 按命名空间过滤
```

**列出已加载的插件：**
```
> Show currently loaded extensions
```
```bash
vigolium ext ls
vigolium ext ls --type active        # 仅活跃插件
```

**内联快速测试 JS 代码：**
```
> Test a JS expression
```
```bash
vigolium ext eval 'vigolium.log.info("hello from extension")'
vigolium ext eval 'vigolium.utils.md5("password")'
```

**评估 JS 文件：**
```
> Run a JS script file
```
```bash
vigolium ext eval --ext-file script.js
```

**对目标运行自定义插件：**
```
> Run my custom scanner extension
```
```bash
vigolium run extension -t https://example.com --ext custom-check.js
# 或
vigolium run ext -t https://example.com --ext custom-check.js
```

**将插件与内置模块一起运行：**
```
> Run built-in modules plus my custom extension
```
```bash
vigolium scan -t https://example.com --ext custom-check.js
```

**仅运行插件（跳过内置模块）：**
```
> Run only my custom extensions
```
```bash
vigolium scan -t https://example.com --only extension --ext custom-check.js
```

**加载多个插件：**
```
> Run three extensions together
```
```bash
vigolium scan -t https://example.com --ext check1.js --ext check2.js --ext check3.js
```

**从目录加载所有插件：**
```
> Run all extensions in my extensions folder
```
```bash
vigolium scan -t https://example.com --ext-dir ./my-extensions/
```

**让 Agent 编写插件：**
```
> Write me a passive extension that checks for missing security headers
```

Agent 将生成如下 JS 文件：

```javascript
module.exports = {
  id: "missing-security-headers",
  name: "Missing Security Headers",
  type: "passive",
  severity: "low",
  confidence: "certain",
  scope: "response",
  tags: ["headers", "misconfiguration", "light"],
  scanTypes: ["per_request"],

  scanPerRequest: function(ctx) {
    if (!ctx.response) return null;
    var headers = ctx.response.headers;
    var missing = [];

    if (!headers["strict-transport-security"]) missing.push("HSTS");
    if (!headers["x-content-type-options"]) missing.push("X-Content-Type-Options");
    if (!headers["x-frame-options"] && !headers["content-security-policy"]) {
      missing.push("X-Frame-Options/CSP");
    }

    if (missing.length === 0) return null;

    return {
      url: ctx.request.url,
      name: "Missing Security Headers: " + missing.join(", "),
      severity: "low",
      description: "Response is missing: " + missing.join(", ")
    };
  }
};
```

**让 Agent 编写 AI 增强插件：**
```
> Write an active extension that uses AI to generate XSS payloads
```

Agent 将生成使用 `vigolium.agent.generatePayloads()` 和 `vigolium.agent.analyzeResponse()` 的 JS 文件。

**YAML 插件（简单模式匹配）：**
```
> Write a YAML extension that detects stack traces and SQL errors
```

```yaml
id: error-pattern-detector
name: Verbose Error Pattern Detector
type: passive
severity: suspect
confidence: tentative
scope: response
tags: [error, information-disclosure, light]
scanTypes: [per_request]
patterns:
  - name: "Stack Trace Detected"
    regex: "(?:at\\s+[\\w.$]+\\(|Traceback \\(most recent|Exception in thread)"
    severity: suspect
  - name: "SQL Error Message"
    regex: "(?:mysql_|pg_|sqlite_|ORA-\\d{5}|SQLSTATE\\[)"
    severity: medium
```

---

### 11. 设置与项目

**查看所有配置：**
```
> Show the current vigolium config
```
```bash
vigolium config ls
```

**查看特定配置部分：**
```
> Show scope configuration
```
```bash
vigolium config ls scope
vigolium config ls scanning_pace
vigolium config ls server
```

**设置配置值：**
```
> Set the default strategy to deep
```
```bash
vigolium config set scanning_strategy.default_strategy deep
```

**设置范围模式：**
```
> Set origin scope to strict
```
```bash
vigolium config set scope.origin.mode strict
```

**全局启用插件：**
```
> Enable extensions in dynamic-assessment
```
```bash
vigolium config set dynamic-assessment.extensions.enabled true
```

**查看范围规则：**
```
> Show current scope rules
```
```bash
vigolium scope view
vigolium scope view host
```

**查看扫描策略：**
```
> Show available strategies and their phases
```
```bash
vigolium strategy
```

**创建和管理项目：**
```
> Create a project, then switch to it
```
```bash
vigolium project create my-project
vigolium project list
vigolium project use my-project
```

**将 CLI 操作限定到项目：**
```
> Scan within a specific project
```
```bash
vigolium scan -t https://example.com --project-name my-project
```

**项目级数据库访问：**
```
> Show stats for my-project
```
```bash
VIGOLIUM_PROJECT=my-project vigolium db stats
```

---

## 自然语言示例

以下是在安装了技能的情况下，您可以向 Claude Code 或 Codex 提供的自然语言提示示例。Agent 会将其转换为正确的 vigolium 命令。

| 您说 | Agent 运行 |
|---|---|
| "Scan example.com" | `vigolium scan -t https://example.com` |
| "Deep scan with spidering" | `vigolium scan -t <url> --strategy deep` |
| "Import my Burp export and scan it" | `vigolium scan -I burp -i export.xml` |
| "Scan my OpenAPI spec with auth" | `vigolium scan -I openapi -i spec.yaml -t <url> --spec-header "Authorization: Bearer ***"` |
| "Only run XSS modules" | `vigolium scan -t <url> --module-tag xss` |
| "Review my code for security issues" | `vigolium agent query --prompt-template security-code-review --source ./src` |
| "Autonomous scan focused on injection" | `vigolium agent autopilot -t <url> --focus "injection"` |
| "Run the full AI pipeline" | `vigolium agent swarm --discover -t <url>` |
| "Show me all critical findings" | `vigolium finding --severity critical` |
| "Export results as HTML report" | `vigolium export --format html -o report.html` |
| "What traffic is in the database?" | `vigolium traffic` |
| "Write me an extension that checks for exposed .env files" | 生成 JS 插件文件 |
| "Start the server with auto-scan" | `vigolium server -t <url> --scan-on-receive` |
| "Source-aware AI scan" | `vigolium agent swarm -t <url> --source ./src` |
| "Multi-phase code audit" | `vigolium agent audit --source ./src` |
| "Clean up old scan data" | `vigolium db clean --before <date> --force` |

---

## 提示与最佳实践

1. **从 `scan -t` 开始** — 这是最常用的命令。逐步添加标志。
2. **使用策略** — `lite` 用于快速检查，`balanced` 用于大多数情况，`deep` 用于全面覆盖。当您有源代码时，请使用 Agent 模式（`swarm`、`autopilot`、`audit` 或 `query`）。
3. **阶段隔离** — 使用 `--only` 或 `vigolium run <phase>` 在单个阶段上迭代，无需重新运行整个管道。
4. **模块标签** — 按技术（`spring`、`nodejs`）或漏洞类别（`xss`、`injection`）过滤模块以减少噪音。
5. **监视模式** — 在长时间扫描期间，为 `traffic`、`finding` 或 `db stats` 添加 `--watch 5s` 以实现实时监控。
6. **Agent 试运行** — 对于 Agent 命令，始终先使用 `--dry-run` 预览提示，然后再花费 AI 令牌。
7. **优先使用 Swarm 而非 Autopilot** — 使用 `agent swarm --discover` 进行结构化扫描（成本更低、可重复）。使用 `agent autopilot` 进行探索性、创造性扫描。
8. **使用插件实现自定义逻辑** — 编写 JS 插件而非修改核心模块。它们通过 `--ext` 与内置模块一起运行。
9. **使用项目实现隔离** — 使用 `vigolium project create` 保持不同项目间的扫描数据分离。
10. **尽早导出** — 运行 `vigolium export --format html -o report.html` 以交互式报告的形式分享结果。
