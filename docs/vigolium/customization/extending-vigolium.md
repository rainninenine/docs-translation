# Customizing & Extending Vigolium

Vigolium 专为可扩展性而设计。无论您需要添加新的漏洞检查、重塑扫描行为，还是集成 AI 驱动的分析，都有多个扩展点——每个扩展点都有不同的权衡。

本指南涵盖所有自定义机制，解释何时使用每种机制，并帮助您为用例选择正确的方法。

---

## 目录

- [扩展点一览](#extension-points-at-a-glance)
- [1. JavaScript 插件](#1-javascript-extensions)
- [2. YAML 插件](#2-yaml-extensions)
- [3. Go 模块](#3-go-modules)
- [4. 自定义提示模板](#4-custom-prompt-templates)
- [5. 扫描配置文件](#5-scanning-profiles)
- [6. 扫描范围规则](#6-scope-rules)
- [7. 前置钩子和后置钩子](#7-pre-hooks-and-post-hooks)
- [8. Agent 后端](#8-agent-backends)
- [9. 设置覆盖](#9-configuration-overrides)
- [决策矩阵](#decision-matrix)

---

## 扩展点一览

| 扩展点 | 语言 | 需重新编译？ | 最佳用途 |
|---|---|---|---|
| [JavaScript 插件](#1-javascript-extensions) | JavaScript | 否 | 自定义主动/被动检查，具有完整 API 访问权限 |
| [YAML 插件](#2-yaml-extensions) | YAML | 否 | 声明式模式匹配，简单的载荷/匹配器规则 |
| [Go 模块](#3-go-modules) | Go | 是 | 高性能检查，与内部深度集成 |
| [提示模板](#4-custom-prompt-templates) | Markdown + Go 模板 | 否 | AI 驱动的代码审查、端点发现、自定义分析 |
| [扫描配置文件](#5-scanning-profiles) | YAML | 否 | 可复用的扫描预设（速度、模块、阶段） |
| [扫描范围规则](#6-scope-rules) | YAML | 否 | 目标过滤（主机、路径、状态码、内容类型） |
| [前置/后置钩子](#7-pre-hooks-and-post-hooks) | JS 或 YAML | 否 | 请求变异、漏洞发现抑制、严重性升级 |
| [Agent 后端](#8-agent-backends) | 任意（子进程） | 否 | 接入新的 AI 模型或自定义 CLI 工具 |
| [设置覆盖](#9-configuration-overrides) | YAML | 否 | 调整并发、速率限制、数据库、通知 |

---

## 1. JavaScript 插件

JavaScript 插件是在不重新编译 Vigolium 的情况下添加自定义扫描逻辑最灵活的方式。它们在嵌入式 JS 引擎（Grafana Sobek）中运行，并可访问完整的 `vigolium.*` API——HTTP 请求、数据库查询、解析工具、AI 集成等。

### 您可以构建的内容

- **主动模块** — 向插入点（参数、标头、Cookie、路径）发送载荷并分析响应以发现漏洞。
- **被动模块** — 分析捕获的 HTTP 流量而不生成新请求。
- **前置钩子** — 在请求到达扫描模块之前对其进行变异（注入认证标头、跳过路径）。
- **后置钩子** — 在检测后过滤、标记或升级漏洞发现。

### 最小示例（主动模块）

```javascript
module.exports = {
  id: "reflected-param-scanner",
  name: "Reflected Parameter Scanner",
  type: "active",
  severity: "medium",
  confidence: "firm",
  scanTypes: ["per_insertion_point"],

  scanPerInsertionPoint: function(ctx, insertion) {
    var canary = "VGNM" + vigolium.utils.randomString(8);
    var resp = vigolium.http.send(insertion.buildRequest(canary));

    if (resp && resp.body.indexOf(canary) !== -1) {
      return [{
        matched: canary,
        url: ctx.request.url,
        name: "Reflected parameter: " + insertion.name,
        severity: "medium",
        request: insertion.buildRequest(canary),
        response: resp.raw
      }];
    }
    return null;
  }
};
```

### 可用 API

| 命名空间 | 用途 | 关键方法 |
|---|---|---|
| `vigolium.http` | 发送 HTTP 请求 | `get`, `post`, `request`, `send` |
| `vigolium.scan` | 扫描控制 | `listModules`, `isInScope`, `createFinding`, `startNewScan` |
| `vigolium.db` | 数据库访问 | `records.query`, `findings.query`, `compareResponses` |
| `vigolium.parse` | HTTP 解析 | `url`, `request`, `response`, `headers`, `json` |
| `vigolium.utils` | 编码、哈希、I/O | `base64Encode`, `sha256`, `readFile`, `exec`, `detectAnomaly` |
| `vigolium.ingest` | 导入流量 | `url`, `curl`, `raw`, `openapi`, `postman` |
| `vigolium.source` | 源代码访问 | `list`, `readFile`, `listFiles`, `searchFiles` |
| `vigolium.agent` | AI 集成 | `ask`, `generatePayloads`, `analyzeResponse`, `confirmFinding` |
| `vigolium.config` | 插件变量 | 只读访问 `extensions.variables` |

完整 TypeScript 定义：[`pkg/jsext/vigolium.d.ts`](../../pkg/jsext/vigolium.d.ts)

### 设置

```yaml
# vigolium-configs.yaml
dynamic-assessment:
  extensions:
    enabled: true
    extension_dir: ~/.vigolium/extensions/
    variables:
      auth_token: "Bearer eyJ..."
```

将 `.js` 文件放入 `~/.vigolium/extensions/`，并使用 `vigolium extensions ls` 验证。

### 优点

- **无需重新编译** — 放入文件即可扫描。
- **完整 API 访问** — HTTP、数据库、AI、源代码、解析和系统工具。
- **AI 增强扫描** — 使用 `vigolium.agent.generatePayloads()` 和 `vigolium.agent.analyzeResponse()` 进行 LLM 驱动的检测。
- **快速迭代** — 编辑、保存、重新扫描。
- **沙盒执行** — 文件 I/O 限制在 `sandbox_dir` 内，`exec()` 受配置控制。

### 缺点

- **比 Go 慢** — 解释型 JS 引擎每次调用增加开销。
- **无 Go 标准库** — 仅限于 `vigolium.*` API，无法任意导入。
- **每个 VM 单线程** — 每个插件实例在自己的 VM 中运行（通过池化实现线程安全，但单个插件内无并行）。
- **调试有限** — 无逐步调试器，`vigolium.log.*` 是主要工具。

### 何时使用

- 需要自定义漏洞检查且不想重新编译。
- 需要 AI 增强的载荷生成或响应分析。
- 检查逻辑中需要数据库或源代码访问。
- 构建组织特定的检查（例如自定义标头验证、业务逻辑缺陷）。

查看完整指南：[编写插件](writing-extensions.md)

---

## 2. YAML 插件

YAML 插件（`.vgm.yaml`）是 JavaScript 的声明式替代方案。它们非常适合简单的载荷和匹配器规则，无需程序化控制流。

### 最小示例（主动模块）

```yaml
id: error-pattern-detector
name: Error Pattern Detector
type: active
severity: low
confidence: firm
scan_types: [per_request]

payloads:
  - "'"
  - "\" OR 1=1--"

matchers:
  - type: body
    regex: "(?i)(SQL syntax|mysql_fetch|ORA-\\d{5}|SQLSTATE\\[)"
  - type: body
    regex: "(?i)(Traceback \\(most recent call last\\)|at \\w+\\.java:\\d+)"

matchers_condition: or

finding:
  name: "Error-Based Information Leak"
  description: "Application returns verbose error messages that reveal implementation details"
  severity: low
```

### YAML 钩子示例（前置钩子）

```yaml
id: auth-header-injector
name: Auth Header Injector
type: pre_hook

add_headers:
  Authorization: "Bearer ${AUTH_TOKEN}"
  X-Request-ID: "vgm-{{random}}"

skip_when:
  url_contains: ["/health", "/metrics"]
```

### YAML 钩子示例（后置钩子）

```yaml
id: suppress-low-on-static
name: Suppress Low Findings on Static Assets
type: post_hook

drop_when:
  severity: [low, info]
  url_contains: ["/static/", "/assets/", ".css", ".js"]

escalate:
  when_url_contains: ["/admin", "/api/v1/auth"]
  bump_severity: true
  tag: sensitive_endpoint
```

### 支持的功能

| 功能 | 描述 |
|---|---|
| `payloads` | 每个插入点或请求注入的字符串列表 |
| `matchers` | 正文正则/包含、标头检查、状态码或内联 JS |
| `matchers_condition` | `or`（任意匹配器）或 `and`（所有匹配器） |
| `add_headers` | 前置钩子：要注入的标头 |
| `skip_when` | 前置钩子：跳过处理的条件 |
| `drop_when` | 后置钩子：丢弃漏洞发现的条件 |
| `escalate` | 后置钩子：提升严重性、添加标签 |
| `script` | 内联 JS 逃生口，用于复杂逻辑 |

### 优点

- **零编码** — 纯声明式 YAML。
- **编写快速** — 载荷列表 + 匹配器正则通常就足够了。
- **易于审计** — 非技术团队成员可以审查规则。
- **相同管道集成** — 与 JS 插件一起加载，生命周期相同。

### 缺点

- **逻辑有限** — 没有条件、循环或超出匹配器提供的状态。
- **无 API 访问** — 无法进行数据库查询、HTTP 后续操作或 AI 调用（除非使用 `script` 逃生口，这实际上是 JS）。
- **无多步检查** — 无法链式请求或跨步骤比较响应。
- **插入控制较粗** — 载荷注入直接，但无法根据上下文动态生成载荷。

### 何时使用

- 简单的基于签名的检测（错误字符串、标头模式、状态码）。
- 快速的前置钩子规则（添加认证标头、跳过静态资源）。
- 后置钩子过滤（抑制静态路径上的低严重性漏洞发现）。
- 非开发人员需要贡献扫描规则时。

查看完整指南：[编写插件](writing-extensions.md)

---

## 3. Go 模块

Go 模块是最高性能的扩展点。它们直接编译到扫描器二进制文件中，完全访问 Go 内部（HTTP 客户端、去重、OAST、变异引擎），并在工作池中并发运行。

### 模块接口

所有模块实现基础 `Module` 接口：

```go
type Module interface {
    ID() string
    Name() string
    Description() string
    ShortDescription() string
    ConfirmationCriteria() string
    Severity() severity.Severity
    Confidence() severity.Confidence
    ScanScopes() ScanScope
    CanProcess(ctx *httpmsg.HttpRequestResponse) bool
}
```

**主动模块** 添加三个扫描方法（仅实现与声明的 `ScanScopes()` 匹配的方法）：

```go
type ActiveModule interface {
    Module
    AllowedInsertionPointTypes() InsertionPointTypeSet
    ScanPerInsertionPoint(ctx, ip, httpClient, scanCtx) ([]*ResultEvent, error)
    ScanPerRequest(ctx, httpClient, scanCtx) ([]*ResultEvent, error)
    ScanPerHost(ctx, httpClient, scanCtx) ([]*ResultEvent, error)
}
```

**被动模块** 分析流量而不发送新请求：

```go
type PassiveModule interface {
    Module
    Scope() PassiveScanScope
    ScanPerRequest(ctx, scanCtx) ([]*ResultEvent, error)
    ScanPerHost(ctx, scanCtx) ([]*ResultEvent, error)
}
```

### 创建模块

1. 在 `pkg/modules/active/` 或 `pkg/modules/passive/` 下创建包。
2. 嵌入 `modkit.BaseActiveModule` 或 `modkit.BasePassiveModule` 以获取默认值。
3. 为声明的范围实现扫描方法。
4. 在 `pkg/modules/default_registry.go` 中注册。

```go
package my_check

import (
    "github.com/vigolium/vigolium/pkg/modules/modkit"
    "github.com/vigolium/vigolium/pkg/output"
    // ...
)

type Module struct {
    modkit.BaseActiveModule
}

func New() *Module {
    return &Module{
        BaseActiveModule: modkit.NewBaseActiveModule(
            "my-check",
            "My Custom Check",
            "Detailed description of what this checks",
            "One-line summary",
            "How the vulnerability is confirmed",
            severity.High,
            severity.Firm,
            modkit.ScanScopeInsertionPoint,
            modkit.AllParamTypes,
        ),
    }
}

func (m *Module) ScanPerInsertionPoint(
    ctx *httpmsg.HttpRequestResponse,
    ip httpmsg.InsertionPoint,
    httpClient *http.Requester,
    scanCtx *modkit.ScanContext,
) ([]*output.ResultEvent, error) {
    // Your scanning logic here
    return nil, nil
}
```

### 扫描范围选项

| 范围 | 调用方式 | 典型用途 |
|---|---|---|
| `ScanScopeInsertionPoint` | 每个参数一次（URL 参数、正文参数、标头、Cookie、JSON 键、路径段） | 注入漏洞（XSS、SQLi、SSTI、命令注入） |
| `ScanScopeRequest` | 每个唯一请求一次 | 请求级检查（缺少标头、认证绕过、方法操纵） |
| `ScanScopeHost` | 每个唯一主机一次 | 主机级检查（TLS 配置、服务器指纹识别、路径发现） |

范围是位掩码——可以组合（例如 `ScanScopeRequest | ScanScopeHost`）。

### 优点

- **最大性能** — 编译的 Go，无解释器开销，在并发工作池中运行。
- **完整内部访问** — 带有中间件的 HTTP 客户端、去重管理器、OAST 回调、变异引擎、基线缓存。
- **类型安全** — 编译时检查、IDE 支持、Go 测试生态系统。
- **一流集成** — 与内置模块相同的生命周期，自动管道连接。

### 缺点

- **需要重新编译** — 每次更改都需要 `make build`。
- **需要 Go 知识** — 必须理解 Go、模块接口和内部类型。
- **迭代较慢** — 编译-测试周期比即插即用的 JS/YAML 更重。
- **耦合更紧** — 内部 API 的更改可能需要模块更新。

### 何时使用

- 性能关键的检查，在每个请求或插入点上运行。
- 需要与 Vigolium 内部深度集成的检查（OAST、变异引擎、去重）。
- 为核心扫描器做贡献或构建永久模块。
- 需要复杂多步逻辑且完全支持并发的检查。

查看完整指南：[开发模块](../development/developing-modules.md)

---

## 4. 自定义提示模板

提示模板驱动 Vigolium 的 agent 模式。它们是带有 YAML 前置元数据的 Markdown 文件，定义了 AI agent 应分析的内容以及如何报告结果。模板支持 Go 模板语法，并自动从数据库、模块注册表和源代码中丰富上下文。

### 模板格式

```markdown
---
id: my-custom-review
name: My Custom Review
description: What this template does
output_schema: findings    # 或: http_records
variables:
  - SourceCode
  - Language
  - PreviousFindings
---

You are a security engineer. Analyze the following code for {{.Language}} vulnerabilities.

{{if .PreviousFindings}}
Previous findings to verify:
{{.PreviousFindings}}
{{end}}

Source code:
```
{{.SourceCode}}
```

Respond with JSON: {"findings": [...]}
```

### 可用模板变量

| 变量 | 来源 | 描述 |
|---|---|---|
| `SourceCode` | 从 `--source`/`--files` 收集 | 拼接的源代码 |
| `Language` | 自动检测 | 主要语言（Go、Python、JS 等） |
| `Framework` | `--framework` 标志 | 框架提示 |
| `FilePath` | 收集 | 主要文件路径 |
| `RepoPath` | `--source` 标志 | 仓库根路径 |
| `TargetURL` | `--target` 标志 | 目标 URL |
| `Hostname` | 从目标派生 | 用于数据库查找的主机名 |
| `Endpoints` | `--endpoints` 标志 | 预发现的端点 |
| `PreviousFindings` | 数据库（JSON） | 先前的漏洞发现，用于上下文 |
| `DiscoveredEndpoints` | 数据库（JSON） | 来自数据库的 HTTP 记录 |
| `ModuleList` | 模块注册表（JSON） | 可用的扫描器模块 |
| `ScanStats` | 数据库（JSON） | 聚合扫描统计信息 |
| `AvailableCommands` | 硬编码引用 | agent 可以调用的 CLI 命令 |
| `Extra` | `--extra` 标志 | 自定义键值对 |

只有前置元数据 `variables` 数组中列出的变量才会触发数据库查询，从而保持提示快速。

### 输出模式

**`findings`** — 用于代码审查、漏洞检测：
```json
{
  "findings": [{
    "title": "SQL Injection in login handler",
    "severity": "critical",
    "confidence": "certain",
    "file": "auth/login.go",
    "line": 42,
    "snippet": "db.Query(\"SELECT * FROM users WHERE id=\" + userID)",
    "cwe": "CWE-89",
    "tags": ["sqli"]
  }]
}
```

**`http_records`** — 用于端点发现、API 输入生成：
```json
{
  "http_records": [{
    "method": "POST",
    "url": "https://api.example.com/users",
    "headers": {"Content-Type": "application/json"},
    "body": "{\"name\": \"test\"}",
    "notes": "Create user endpoint"
  }]
}
```

### 预设模板

Vigolium 附带 15 个内置模板：

| 模板 | 模式 | 用途 |
|---|---|---|
| `security-code-review` | findings | 通用 OWASP 重点代码审查 |
| `injection-sinks` | findings | 识别注入点（SQLi、cmd、SSRF） |
| `auth-bypass` | findings | 认证/授权绕过模式 |
| `secret-detection` | findings | 硬编码密钥和凭据 |
| `endpoint-discovery` | http_records | 从源代码提取 API 路由 |
| `api-input-gen` | http_records | 从端点生成 HTTP 请求 |
| `curl-command-gen` | http_records | 为所有路由生成 curl 命令 |
| `interactive-scan` | findings | Autopilot：分析 + 运行扫描 |
| `targeted-retest` | findings | Autopilot：验证先前的漏洞发现 |
| `attack-surface-mapper` | http_records | 发现并交叉引用 API |
| `nextjs-security-audit` | findings | Next.js 特定安全审查 |
| `react-xss-audit` | findings | React XSS 模式分析 |
| `auth-session-review` | findings | 认证和会话管理 |
| `cors-csrf-review` | findings | CORS/CSRF 配置审查 |
| `build-config-audit` | findings | 构建/部署配置安全 |

### 设置

将自定义模板放在 `~/.vigolium/prompts/` 中，或在配置中设置 `agent.templates_dir`。用户模板按 ID 覆盖内置模板。

```bash
# 使用自定义模板
vigolium agent query --prompt-template my-custom-review --source /path/to/source

# 干运行以查看渲染后的提示
vigolium agent query --prompt-template my-custom-review --source /path/to/source --dry-run
```

### 优点

- **无需代码** — 带有模板语法的 Markdown 文件。
- **上下文感知** — 自动丰富数据库漏洞发现、端点、扫描统计信息。
- **多供应商** — 进程内 olium 引擎在 openai-codex-oauth、anthropic-api-key、anthropic-oauth、openai-api-key 和 anthropic-cli 之间切换，无需更改提示。
- **迭代优化** — autopilot 和管道模式将先前的漏洞发现传回以进行验证。
- **两种输出模式** — 为代码审查生成漏洞发现，或为端点发现生成 HTTP 记录。

### 缺点

- **AI 依赖** — 需要配置 agent 后端和 API 访问。
- **非确定性** — LLM 输出在不同运行之间变化；误报需要调整。
- **延迟** — agent 调用比模式匹配慢（每次运行几秒到几分钟）。
- **令牌成本** — 大型代码库每次分析消耗大量令牌。

### 何时使用

- 需要语义理解的代码级安全审查（不仅仅是模式匹配）。
- 从源代码生成 HTTP 测试输入（路由提取、API 模糊测试种子）。
- 框架特定审计（Next.js、React、Django、Spring），模板可以嵌入领域知识。
- 迭代分析，agent 在多次传递中优化漏洞发现。

---

## 5. 扫描配置文件

配置文件是叠加在主配置之上的 YAML 文件。它们将扫描策略、速度、阶段设置和模块选择捆绑到可复用的预设中。

### 格式

配置文件是 `vigolium-configs.yaml` 的子集。只有非 nil 值会覆盖基础配置。

```yaml
# ~/.vigolium/profiles/aggressive.yaml
# description: Fast aggressive scan for CI/CD pipelines

scanning_strategy:
  default_strategy: deep

scanning_pace:
  concurrency: 100
  rate_limit: 200
  max_per_host: 20
  dynamic-assessment:
    concurrency: 100
    max_duration: 60m

discovery:
  mode: files_and_dirs
  recursion:
    enabled: true
    max_depth: 8

spidering:
  max_depth: 0
  headless: true
  strategy: aggressive

dynamic-assessment:
  enabled_modules:
    active_modules:
      - all
    passive_modules:
      - all
```

### 用法

```bash
# 使用命名配置文件
vigolium scan --target https://example.com --scanning-profile aggressive

# 使用配置文件路径
vigolium scan --target https://example.com --scanning-profile /path/to/profile.yaml
```

配置文件按名称从 `~/.vigolium/profiles/` 或 `public/presets/profiles/` 解析。

### 优点

- **可复用预设** — 定义一次，跨目标和团队使用。
- **可组合** — 叠加在基础配置之上；仅覆盖所需内容。
- **无需代码** — 纯 YAML。
- **团队友好** — 在版本控制中共享配置文件以实现一致的扫描策略。

### 缺点

- **仅配置** — 无法添加新的扫描逻辑，只能调整现有设置。
- **无每目标逻辑** — 同一配置文件适用于扫描中的所有目标。
- **验证有限** — 字段名称中的拼写错误会被静默忽略。

### 何时使用

- 在不同上下文中运行不同扫描强度（CI/CD 与完整审计与快速检查）。
- 希望在整个团队中强制执行一致的扫描设置。
- 需要切换阶段（例如跳过发现，仅运行被动模块）。

---

## 6. 扫描范围规则

扫描范围规则控制扫描的内容。它们在主机、路径、状态码、内容类型和正文级别进行过滤。您可以在配置中、通过 CLI 标志或从 JS 插件以编程方式定义它们。

### 配置

```yaml
# vigolium-configs.yaml
scope:
  applied_on_ingest: false     # 在导入时或扫描时强制执行

  host:
    include: ["*.example.com", "api.example.com"]
    exclude: ["staging.example.com"]

  path:
    include: ["/api/*"]
    exclude: ["/api/health", "/api/metrics", "/static/*"]

  status_code:
    include: ["2xx", "3xx"]    # 精确、通配符（2xx）、范围（400-499）
    exclude: ["404"]

  request_content_type:
    include: ["application/json*", "application/x-www-form-urlencoded*"]

  response_content_type:
    include: ["text/html*", "application/json*"]
    exclude: ["image/*", "font/*"]

  ignore_static_file: true     # 自动跳过 .jpg、.png、.css 等
```

### 从插件设置范围

```javascript
// 以编程方式检查范围
if (vigolium.scan.isInScope("api.example.com", "/users")) {
  // 继续
}

// 读取当前范围
var scope = vigolium.scan.getScope();

// 在运行时修改范围
vigolium.scan.setScope({
  host: { include: ["*.example.com"], exclude: [] },
  path: { include: ["/api/*"], exclude: [] }
});
```

### 优点

- **精确目标定位** — 仅扫描相关内容，跳过噪音。
- **多种过滤类型** — 主机通配符、路径模式、状态码、内容类型、正文字符串。
- **运行时可调整** — 插件可以在扫描期间修改范围。
- **安全网** — 防止意外扫描范围外的系统。

### 缺点

- **仅配置复杂性** — 复杂的范围规则可能难以调试。
- **无请求级条件** — 无法按请求标头值或认证状态进行范围设置（使用前置钩子实现）。

### 何时使用

- 将扫描限制到特定子域或 API 路径。
- 排除健康检查、静态资源或第三方端点。
- 具有定义范围边界的漏洞赏金计划。
- 按响应特征（状态码、内容类型）进行过滤。

---

## 7. 前置钩子和后置钩子

钩子包裹扫描管道。前置钩子在模块处理请求之前转换请求。后置钩子在检测后过滤或修改漏洞发现。

### 前置钩子用例

| 用例 | 实现 |
|---|---|
| 注入认证标头 | YAML `add_headers` 或返回 `{headers: {...}}` 的 JS |
| 跳过静态资源 | YAML `skip_when.url_contains` 或返回 `null` 的 JS |
| 添加关联 ID | 为每个请求生成唯一 ID 的 JS |
| 重写 URL | 在转发前修改 `ctx.request` 的 JS |

### 后置钩子用例

| 用例 | 实现 |
|---|---|
| 抑制静态路径上的低严重性漏洞发现 | YAML `drop_when` |
| 在管理端点上升级漏洞发现 | YAML `escalate.when_url_contains` |
| 按业务单元标记漏洞发现 | 基于 URL 模式添加元数据的 JS |
| AI 驱动的误报过滤 | 调用 `vigolium.agent.confirmFinding()` 的 JS |

### JS 前置钩子示例

```javascript
module.exports = {
  id: "inject-session",
  name: "Session Injector",
  type: "pre_hook",

  execute: function(request) {
    return {
      headers: {
        "Cookie": "session=" + vigolium.config.session_token,
        "X-Correlation-ID": vigolium.utils.randomString(16)
      }
    };
  }
};
```

### JS 后置钩子示例（AI 误报过滤器）

```javascript
module.exports = {
  id: "ai-fp-filter",
  name: "AI False Positive Filter",
  type: "post_hook",

  execute: function(result) {
    if (typeof vigolium.agent === "undefined") return result;

    var check = vigolium.agent.confirmFinding({
      name: result.info.name,
      request: result.request,
      response: result.response,
      matched: result.matched
    });

    if (!check.confirmed && check.confidence !== "low") {
      vigolium.log.info("Suppressed FP: " + result.info.name);
      return null; // 丢弃漏洞发现
    }
    return result;
  }
};
```

### 优点

- **管道集成** — 在每个请求/漏洞发现上自动运行。
- **可组合** — 多个钩子顺序链式执行。
- **支持 JS 和 YAML** — 简单规则用 YAML，复杂逻辑用 JS。
- **非侵入性** — 不修改模块代码。