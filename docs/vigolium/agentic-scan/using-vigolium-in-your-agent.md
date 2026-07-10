# 文档API 参考Agent 扫描在 Claude Code & Codex 中使用 Vigolium Scanner Skill复制页面安装并使用 `vigolium-scanner` skill，配合 AI 编码代理进行 Web 漏洞扫描和插件开发。复制页面本指南介绍如何安装和使用 `vigolium-scanner` skill，配合 AI 编码代理（Claude Code 和 OpenAI Codex）来操作 Vigolium CLI，进行 Web 漏洞扫描、安全测试和自定义插件开发。

Skill 仓库：github.com/vigolium/skills — 或者安装内嵌在 vigolium 二进制文件中的副本，该副本始终与您安装的版本匹配。

## 快速安装

```bash
# 将内嵌 skill 安装到当前项目（推荐——始终与您的二进制文件匹配）
vigolium skills install --agent claude --scope project
```

`vigolium skills install`（自 v0.2.1 起新增）直接从二进制文件中写入 `vigolium-scanner` 包，因此它永远不会与您正在驱动的 CLI 产生偏差。使用 `--agent claude|codex|agents` 选择代理，使用 `--scope project`（当前文件夹）或 `--scope global`（主目录）选择位置。运行 `vigolium skills` 列出已打包的内容，运行 `vigolium skills get vigolium-scanner` 打印包而不安装它。

更倾向于从仓库拉取最新版本？使用外部 skills 安装器：

```bash
bunx skills add vigolium/skills --skill vigolium-scanner --agent <agent-name> --yes
```

将 `<agent-name>` 替换为您的代理（例如 `claude-code`、`codex`）。

## Skill 的功能

该 skill 教会 AI 代理如何：

- 为任何安全测试任务选择正确的 vigolium 命令
- 使用正确的语法构造正确的标志组合
- 端到端地遵循扫描工作流（数据导入 -> 扫描 -> 分类 -> 导出）
- 使用 `vigolium.*` API 编写自定义 JavaScript 插件
- 操作 AI 代理模式（查询、autopilot、swarm）
- 管理数据：浏览流量、过滤漏洞发现、导出报告、清理数据库

该 skill 使用懒加载引用：主 `SKILL.md` 保持小巧，详细文档在代理需要深层标志信息或插件编写指导时按需加载。

## Skill 结构

```
public/skills/vigolium-scanner/
├── SKILL.md                              # 主 skill（决策树、配方、标志）
└── references/
    ├── scanning-commands.md              # scan, scan-url, scan-request, run
    ├── server-and-ingestion.md           # server, ingest, traffic, traffic replay
    ├── agent-commands.md                 # agent, agent query, autopilot, swarm
    ├── data-and-management.md            # db, module, ext, config, scope, source, export
    ├── flags-reference.md                # 完整的字母顺序标志索引
    └── writing-extensions.md             # JS 扩展 API 和示例
```

## 安装

### 选项 A：`vigolium skills install`（推荐——内嵌、版本匹配）

```bash
# 默认：为 Claude 将 vigolium-scanner 包安装到当前项目
vigolium skills install

# 显式选择代理和作用域
vigolium skills install --agent codex --scope global

# 安装所有打包的 skill，或覆盖目标目录
vigolium skills install --all
vigolium skills install --dir ./.claude/skills
```

该包内嵌在 vigolium 二进制文件中，因此 `vigolium skills install` 永远不会安装过时的副本——它始终与您正在驱动的 CLI 版本匹配。`--agent` 接受 `claude`、`codex` 或 `agents`（兼容 agentskills.io 的布局）；`--scope` 为 `project`（当前文件夹）或 `global`（主目录）。首先使用 `vigolium skills`（列表）和 `vigolium skills get vigolium-scanner`（打印）检查已打包的内容。

### 选项 B：通过 npx / bunx 安装（从仓库拉取最新版本）

```bash
bunx skills add vigolium/skills --skill vigolium-scanner --agent <agent-name> --yes
```

或使用 npx：

```bash
npx skills add vigolium/skills --skill vigolium-scanner --agent <agent-name> --yes
```

将 `<agent-name>` 替换为您的代理（例如 `claude-code`、`codex`）。这将从 `vigolium/skills` 仓库获取 skill 并自动注册。

### 选项 C：克隆并手动复制

```bash
git clone https://github.com/vigolium/skills.git
cd skills
```

然后将 skill 文件夹复制到您的代理配置目录：

```bash
# 对于 Claude Code
cp -R vigolium-scanner ~/.claude

# 对于其他代理
cp -R vigolium-scanner ~/.agents
```

安装完成后，当您提到诸如 `scan`、`vigolium`、`agent autopilot`、`vulnerability scanner`、`openapi scan` 等关键词时，skill 会自动触发。在 Claude Code 中，您也可以使用 `/vigolium-scanner` 显式调用它。

## 按类别分类的使用示例

### 1. 扫描

对单个目标进行基本扫描：

```
> 扫描 https://example.com 的漏洞

vigolium scan -t https://example.com
```

多个目标：

```
> 同时扫描 https://example.com 和 https://api.example.com

vigolium scan -t https://example.com -t https://api.example.com
```

从文件读取目标：

```
> 我有一个 targets.txt 文件，里面是 URL 列表，扫描所有目标

vigolium scan -T targets.txt
```

使用特定策略扫描：

```
> 对 https://example.com 进行深度扫描，包含内容发现和爬取

vigolium scan -t https://example.com --strategy deep
```

使用自定义方法、标头和请求体扫描单个 URL：

```
> 测试登录端点的漏洞：向 https://api.example.com/login 发送 POST 请求，包含 JSON 凭据

vigolium scan-url https://api.example.com/login \
  --method POST \
  --body '{"username":"admin","password":"test123"}' \
  -H "Content-Type: application/json"
```

从文件扫描原始 HTTP 请求：

```
> 我在 request.txt 中捕获了一个原始请求，扫描它

vigolium scan-request -i request.txt
```

从标准输入扫描原始请求：

```
> 从终端扫描这个原始请求

echo -e "GET /api/users?id=1 HTTP/1.1\r\nHost: example.com\r\nAuthorization: Bearer tok123\r\n" | vigolium scan-request
```

通过代理扫描（例如 Burp Suite）：

```
> 扫描 https://example.com 并将流量路由到 Burp

vigolium scan -t https://example.com --proxy http://127.0.0.1:8080
```

调整并发数的高速扫描：

```
> 快速扫描——100 个工作线程，200 请求/秒，每个主机最多 5 个

vigolium scan -t https://example.com -c 100 --rate-limit 200 --max-per-host 5
```

扫描并以 JSONL 格式输出结果：

```
> 扫描并将结果保存为 JSON lines 格式

vigolium scan -t https://example.com --format jsonl -o results.jsonl
```

扫描并生成 HTML 报告：

```
> 扫描并生成交互式 HTML 报告

vigolium scan -t https://example.com --format html -o report.html
```

扫描到独立的 SQLite 文件：

```
> 运行一次性扫描，给我一个自包含的 .sqlite 文件，以后可以重新打开

vigolium scan -S -t https://example.com --format sqlite -o scan
```

`--format sqlite`（别名 `sqlite3`、`db`）需要 `-S/--stateless` + `-o`；使用 `vigolium finding -S --db scan.sqlite` 重新打开。

扫描到可浏览的文件系统树（无需数据库）：

```
> 扫描并将流量和漏洞发现作为文件输出，方便我使用 grep 搜索

vigolium scan -t https://example.com --format fs -o run
```

`--format fs` 在 `-o` 基础目录下写入两个同级目录——`run-traffic/` 和 `run-findings/`——因此您可以使用简单的 `ls`/`grep`/`jq` 进行调查。每个 `<host>/<id>.req` 是原始的可重放请求（去掉开头的 `@target` 行），`<id>.resp.headers`/`<id>.resp.body` 保存 gzip 解码后的响应，`-findings/` 下的 `<id>.md` 是与其 `.req` 交叉引用的漏洞发现。从每个目录中的 `index.json` 开始——使用 `jq` 将每个 id 映射到其 url/status/severity。可与 `-S` 一起使用或不使用；遵循 `--omit-response`。有关完整布局，请参阅输出和报告。

使用自定义扫描配置文件扫描：

```
> 使用激进扫描配置文件

vigolium scan -t https://example.com --scanning-profile aggressive
```

使用严格来源范围扫描：

```
> 仅扫描完全同源的 URL

vigolium scan -t https://example.com --scope-origin strict
```

### 2. 输入格式

OpenAPI 3.x 规范，带显式基础 URL：

```
> 使用 OpenAPI 规范扫描我的 API

vigolium scan -I openapi -i api-spec.yaml -t https://api.example.com
```

使用规范中的服务器 URL 的 OpenAPI 规范：

```
> 使用规范本身定义的服务器 URL

vigolium scan -I openapi -i api-spec.yaml --spec-url
```

带认证标头和参数值的 OpenAPI：

```
> 使用 bearer 认证扫描规范，并将 user_id 参数设置为 42

vigolium scan -I openapi -i spec.yaml -t https://api.example.com \
  --spec-header "Authorization: Bearer eyJ..." \
  --spec-var "user_id=42"
```

Swagger 2.0 规范：

```
> 导入 Swagger 2.0 规范并扫描

vigolium scan -I swagger -i swagger.json -t https://api.example.com
```

Burp Suite XML 导出：

```
> 我从 Burp 导出了流量，扫描它

vigolium scan -I burp -i burp-export.xml -t https://example.com
```

HAR（HTTP 存档）文件：

```
> 扫描我的浏览器录制的 HAR 文件

vigolium scan -I har -i traffic.har
```

cURL 命令文件：

```
> 我有一个包含 curl 命令的文件，全部扫描

vigolium scan -I curl -i curl-commands.txt
```

Postman 集合：

```
> 导入并扫描我的 Postman 集合

vigolium scan -I postman -i collection.json -t https://api.example.com
```

Nuclei 模板：

```
> 针对目标运行这些 Nuclei 模板

vigolium scan -I nuclei -i templates/ -t https://example.com
```

从标准输入管道传输 URL：

```
> 将 URL 列表通过管道传输到扫描器

cat urls.txt | vigolium scan -i -
```

### 3. 阶段控制

仅运行内容发现（内容枚举）：

```
> 仅对目标运行内容发现

vigolium run discover -t https://example.com
# 或
vigolium scan -t https://example.com --only discovery
```

仅运行爬取（无头浏览器爬取）：

```
> 使用无头浏览器爬取目标

vigolium run spidering -t https://example.com
```

仅运行审计（漏洞扫描）：

```
> 跳过发现，仅运行漏洞模块

vigolium run audit -t https://example.com
# 或
vigolium scan -t https://example.com --only audit
```

仅运行 Nuclei 已知问题扫描（仅严重/高）：

```
> 运行基于 Nuclei 的已知问题检查，仅严重和高严重性

vigolium run known-issue-scan -t https://example.com --known-issue-scan-severities critical,high
```

运行白盒源代码审计（SAST）：

```
> 对我的 Go 应用源代码运行安全审计

vigolium agent audit --source /path/to/app
```

仅运行外部收集（Wayback、Common Crawl、OTX）：

```
> 从外部情报源收集 URL

vigolium run external-harvest -t https://example.com
```

跳过特定阶段：

```
> 扫描但跳过发现和爬取

vigolium scan -t https://example.com --skip discovery,spidering
```

仅运行 JavaScript 插件：

```
> 仅运行我的自定义插件，跳过内置模块

vigolium scan -t https://example.com --only extension --ext ./custom-check.js
# 或
vigolium run ext -t https://example.com --ext ./custom-check.js
```

阶段别名参考：

| 别名 | 解析为 |
|:-----|:-------|
| `discover` | `discovery` |
| `spider` | `spidering` |
| `dynamic-assessment` | `audit` |
| `ext` | `extension` |

### 4. 模块过滤

列出所有可用的扫描器模块：

```
> 显示所有扫描器模块

vigolium module ls
# 或
vigolium scan -M
```

按关键词过滤模块：

```
> 显示所有与 XSS 相关的模块

vigolium module ls xss
```

仅列出活动模块并显示详细描述：

```
> 显示活动模块及其描述

vigolium module ls --type active -v
```

仅使用特定模块扫描：

```
> 仅运行反射型 XSS 和基于错误的 SQL 注入模块

vigolium scan -t https://example.com -m xss-reflected,sqli-error
```

按标签过滤模块（OR 逻辑）：

```
> 使用标记为 'spring' 或 'injection' 的模块扫描

vigolium scan -t https://example.com --module-tag spring --module-tag injection
```

组合模块 ID 和标签：

```
> 运行 sqli-error 以及所有标记为 XSS 的模块

vigolium scan -t https://example.com -m sqli-error --module-tag xss
```

持久启用/禁用模块：

```
> 禁用所有 SQL 注入模块，启用所有 XSS 模块

vigolium module disable sqli
vigolium module enable xss
```

按精确模块 ID 启用：

```
> 仅启用反射型 XSS 模块

vigolium module enable active-xss-reflected --id
```

### 5. 服务器与数据导入

启动 API 服务器（默认端口 9002）：

```
> 启动 vigolium 服务器

vigolium server
```

在自定义端口上启动服务器，无需认证：

```
> 在端口 8443 上启动服务器，无需认证

vigolium server --service-port 8443 --no-auth
```

启动服务器并启用接收时扫描（自动扫描导入的流量）：

```
> 启动服务器，并自动扫描每个传入的请求

vigolium server -t https://example.com --scan-on-receive
```

启动服务器并启用透明代理用于录制：

```
> 启动服务器，在端口 8080 上启用录制代理

vigolium server --ingest-proxy-port 8080
```

将导入的流量实时镜像到文件：

```
> 运行服务器，并将其导入的所有内容镜像到 ./mirror，以便我可以实时 grep

vigolium server --mirror-fs ./mirror
```

每个保存的 HTTP 记录和漏洞发现都会在进入数据库时写入 `./mirror/traffic/<host>/…` 和 `./mirror/findings/<host>/…`——布局与 `--format fs` 相同，只是索引是仅追加的 `index.jsonl`（每行一个对象——可实时 `tail`/`grep`），并且每个主机的 id 在重启后继续。将您的代理指向 `./mirror`，以便在流量流入时以文件形式读取导入的 Burp/代理流量。

高并发服务器：

```
> 启动高吞吐量服务器，200 个工作线程

vigolium server -c 200 --mem-buffer 50000
```

本地导入 OpenAPI 规范：

```
> 将规范导入数据库而不扫描

vigolium ingest -t https://api.example.com -I openapi -i spec.yaml
```

导入并自动扫描：

```
> 导入规范并立即扫描

vigolium ingest -t https://api.example.com -I openapi -i spec.yaml -S
```

导入 Burp 导出：

```
> 将 Burp 流量导入数据库

vigolium ingest -t https://example.com -I burp -i export.xml
```

远程导入到正在运行的服务器：

```
> 将流量发送到在 localhost 上运行的 vigolium 服务器

vigolium ingest -s http://localhost:9002 -I openapi -i spec.yaml
```

导入时不获取响应：

```
> 仅存储请求记录，不发起网络请求

vigolium ingest -t https://example.com -I burp -i export.xml --disable-fetch-response
```

### 6. AI 代理模式

#### Agent（基于模板）

安全代码审查：

```
> 审查我的源代码是否存在安全漏洞

vigolium agent query --prompt-template security-code-review --source ./src
```

从源代码发现端点：

```
> 在我的源代码中查找所有 API 端点

vigolium agent query --prompt-template endpoint-discovery --source ./src
```

仅审查特定文件：

```
> 仅审查 auth.go 和 middleware.go 的安全问题

vigolium agent query --prompt-template security-code-review --source ./src \
  --files "src/auth.go,src/middleware.go"
```

向模板附加额外指令：

```
> 代码审查，但重点关注身份验证和授权

vigolium agent query --prompt-template security-code-review --source ./src \
  --append "Focus specifically on authentication and authorization vulnerabilities"
```

使用自定义提示文件：

```
> 使用我自己的提示模板运行代理

vigolium agent query --prompt-file custom-prompt.md --source ./src
```

选择特定的 olium 供应商：

```
> 使用 Anthropic 进行代码审查

vigolium agent query --provider anthropic-api-key --prompt-template security-code-review --source ./src
```

试运行以预览渲染后的提示：

```
> 显示将发送给代理的提示内容

vigolium agent query --prompt-template security-code-review --source ./src --dry-run
```

将代理输出保存到文件：

```
> 将审查结果保存到 JSON 文件

vigolium agent query --prompt-template security-code-review --source ./src \
  --output review-results.json
```

列出可用的模板和供应商：

```
> 哪些提示模板和 olium 供应商可用？

vigolium agent --list-templates
vigolium agent --list-agents
```

内置模板包括：

- `security-code-review`：全面安全审查
- `injection-sinks`：查找注入点
- `auth-bypass`：认证绕过向量
- `secret-detection`：硬编码密钥
- `endpoint-discovery`：从源代码发现 API 端点
- `api-input-gen`：生成测试输入
- `curl-command-gen`：生成 cURL 命令
- `attack-surface-mapper`：映射攻击面
- `nextjs-security-audit`：Next.js 安全审查
- `react-xss-audit`：React XSS 审计
- `cors-csrf-review`：CORS/CSRF 配置审计

#### Agent Query（自由格式提示）

内联提示：

```
> 让代理审查代码中的漏洞

vigolium agent query 'review this code for SQL injection vulnerabilities'
```

命名提示标志：

```
> 分析认证流程

vigolium agent query --prompt 'analyze the authentication flow for bypass vectors'
```

从标准输入管道传输提示：

```
> 将提示通过管道传输给代理

echo "check for SSRF in the URL-fetching handler" | vigolium agent query --stdin
```

使用特定供应商的自定义提示文件：

```
> 通过 Anthropic 运行自定义提示文件

vigolium agent query --provider anthropic-api-key --prompt-file custom-prompt.md
```

选择不同的模型：

```
> 在前沿模型上运行审查

vigolium agent query --provider anthropic-api-key --model claude-opus-4-7 \
  'comprehensive security review of all handlers'
```

#### Agent Autopilot（自主扫描）

基本自主扫描：

```
> 让 AI 自主扫描目标

vigolium agent autopilot -t https://example.com
```

带源代码上下文和关注领域：

```
> 自主扫描，重点关注认证绕过，并附带源代码上下文

vigolium agent autopilot -t https://api.example.com --source ./src --focus "auth bypass"
```

自定义限制（较低强度，较短运行时间）：

```
> 运行快速扫描，上限 15 分钟

vigolium agent autopilot -t https://example.com --intensity quick --max-duration 15m
```

预览系统提示（试运行）：

```
> 显示 autopilot 代理将接收的系统提示

vigolium agent autopilot -t https://example.com --dry-run
```

自定义系统提示：

```
> 使用我自己的系统提示进行 autopilot

vigolium agent autopilot -t https://example.com --system my-system-prompt.md
```

使用不同的 olium 供应商：

```
> 通过 Google Vertex 运行 autopilot

vigolium agent autopilot -t https://example.com --provid
```