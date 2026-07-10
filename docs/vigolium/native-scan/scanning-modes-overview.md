# Scanning Modes Overview（扫描模式概述）

Vigolium 支持多种扫描模式，取决于您可用的输入：仅 URL、源代码、AI 代理，或以上全部。本文档帮助您选择合适的模式并理解执行管道。

## Scanning Modes at a Glance（扫描模式一览）

| 模式 | 所需输入 | 命令 | 功能 |
|------|---------------|---------|--------------|
| **Lite** | URL | `vigolium scan -t URL --strategy lite` | 仅动态评估，无内容发现 |
| **Balanced** | URL | `vigolium scan -t URL` | 内容发现 + 爬取 + 动态评估 + 已知问题扫描 |
| **Deep** | URL | `vigolium scan -t URL --strategy deep` | 在 Balanced 基础上增加外部采集 |
| **Single URL** | URL | `vigolium scan-url URL` | 单次扫描单个 URL |
| **Single Request** | 原始 HTTP | `vigolium scan-request -i request.txt` | 单次扫描原始 HTTP 请求 |
| **Extension** | URL + JS/YAML 插件 | `vigolium run extension -t URL --ext script.js` | 仅运行自定义插件模块 |
| **Agent (query)** | 源代码 | `vigolium agent query --prompt-template X --source ./app` | AI 驱动的单次代码审查 |
| **Agent (swarm)** | URL ± 源代码 | `vigolium agent swarm -t URL [--source ./app]` | AI 规划模块 + 插件，本地扫描器执行，可选分类循环 |
| **Agent (autopilot)** | URL ± 源代码 | `vigolium agent autopilot -t URL [--source ./app]` | 单个长时间 LLM 会话自主驱动扫描 |
| **Agent (audit)** | 源代码 | `vigolium agent audit --source ./app` | 前台多阶段 AI 源代码审计 |

## Decision Guide（决策指南）

```
是否希望 AI 参与？
├── 否
│   ├── 快速单 URL 测试？──────────── vigolium scan-url <URL>
│   ├── 想要快速结果？──────────────── vigolium scan -t URL --strategy lite
│   ├── 标准扫描？─────────────────── vigolium scan -t URL
│   ├── 最大外部信息收集？───────────── vigolium scan -t URL --strategy deep
│   └── 仅自定义插件脚本？──────────── vigolium run extension -t URL --ext script.js
│
└── 是
    ├── 仅有源代码（无在线目标）？
    │   ├── 单次代码审查？───────────── vigolium agent query --prompt-template security-code-review --source ./app
    │   └── 多阶段 AI 审计？─────────── vigolium agent audit --source ./app
    │
    └── 有目标 URL？
        ├── AI 指导本地扫描器 ────────── vigolium agent swarm -t URL [--source ./app]
        └── AI 作为扫描器 ────────────── vigolium agent autopilot -t URL [--source ./app]
```

## Phase Execution Pipeline（阶段执行管道）

各阶段按此顺序执行。每种策略启用这些阶段的子集：

```
1. Heuristics Check     预检探测（检测 WAF、重定向、技术栈）
2. External Harvesting  查询 Wayback、CommonCrawl、AlienVault OTX、URLScan、VirusTotal
3. Spidering            基于浏览器的爬取（Chromium），SPA 支持，表单填写
4. Discovery            内容发现（暴力破解目录/文件，JS 分析）
5. DynamicAssessment    对所有已发现端点的活动 + 被动扫描器模块
6. KnownIssueScan       已知问题扫描（Nuclei 模板 + Kingfisher 机密）
7. Extension            自定义 JS/YAML 插件模块（当使用 --only extension 或 --ext 时）
```

## Strategy Comparison（策略比较）

| 阶段 | Lite | Balanced | Deep |
|-------|:----:|:--------:|:----:|
| 外部采集 | - | - | 是 |
| 内容发现 | - | 是 | 是 |
| 爬取 | - | 是 | 是 |
| 动态评估 | 是 | 是 | 是 |
| 已知问题扫描 | - | 是 | 是 |

**Balanced** 是未指定 `--strategy` 时的默认策略。

## Phase Aliases（阶段别名）

多个阶段有短别名，可与 `--only` 和 `--skip` 一起使用：

| 别名 | 规范阶段 |
|-------|-----------------|
| `deparos` | `discovery` |
| `discover` | `discovery` |
| `spitolas` | `spidering` |
| `ext` | `extension` |
| `audit` | `dynamic-assessment` |
| `dast` | `dynamic-assessment` |
| `assessment` | `dynamic-assessment` |

## Phase Control: `--only` and `--skip`（阶段控制）

这两个标志是**互斥的**。同时使用会产生错误。

### `--only <phase>` — 运行单个阶段

禁用所有其他阶段并关闭启发式检查。

```bash
# 仅运行内容发现
vigolium scan -t https://example.com --only discovery

# 仅运行动态评估（跳过所有内容发现）
vigolium scan -t https://example.com --only dynamic-assessment
# 仅运行自定义插件（跳过内置模块）
vigolium scan -t https://example.com --only extension
# 或使用别名：
vigolium scan -t https://example.com --only ext
```

有效值：`ingestion`、`discovery`（`deparos`、`discover`）、`spidering`（`spitolas`）、`external-harvest`、`dynamic-assessment`（`dast`、`audit`、`assessment`）、`known-issue-scan`、`extension`（`ext`）

### `--skip <phase>` — 跳过特定阶段

禁用指定的阶段，同时保持策略启用的其他所有阶段。

```bash
# 在 balanced 扫描中跳过爬取
vigolium scan -t https://example.com --skip spidering

# 同时跳过内容发现和已知问题扫描
vigolium scan -t https://example.com --skip discovery --skip known-issue-scan
```

有效值：`discovery`（`deparos`、`discover`）、`external-harvest`、`spidering`（`spitolas`）、`dynamic-assessment`（`dast`、`audit`、`assessment`）、`known-issue-scan`、`extension`（`ext`）

### `vigolium run <phase>` 快捷方式

`vigolium run <phase>` 是 `vigolium scan --only <phase>` 的直接别名：

```bash
# 以下命令等效：
vigolium run discovery -t https://example.com
vigolium scan -t https://example.com --only discovery

# 仅运行插件模块：
vigolium run extension -t https://example.com --ext my-scanner.js
# 等效于：
vigolium scan -t https://example.com --only extension --ext my-scanner.js
```

## Scanning Profiles（扫描配置文件）

**扫描策略（scanning strategy）** 仅切换阶段的开启/关闭。**扫描配置文件（scanning profile）** 更进一步——它将策略、速度、范围、内容发现、爬取和模块配置打包到单个 YAML 文件中，选择后覆盖主配置。

### 使用配置文件

```bash
# 使用内置的标准配置文件
vigolium scan -t https://example.com --scanning-profile standard

# 按名称使用自定义配置文件（从 profiles_dir 解析）
vigolium scan -t https://example.com --scanning-profile api-pentest

# 按路径使用配置文件
vigolium scan -t https://example.com --scanning-profile ~/profiles/custom.yaml

# 显示策略、阶段、强度、agent 模式和可用配置文件
vigolium strategy
```

### 创建自定义配置文件

在 `~/.vigolium/profiles/` 中创建 YAML 文件。第一行可以包含 `# description:` 注释，该注释会显示在 `vigolium strategy` 中。

配置文件可以覆盖以下任意配置部分的组合（省略的部分保留其主配置值）：

```yaml
# description: 快速 API 聚焦扫描，最小化内容发现
scanning_strategy:
  default_strategy: lite

scanning_pace:
  concurrency: 100
  rate_limit: 200

discovery:
  mode: files_only

known_issue_scan:
  enrich_targets: false         # 仅主机级别（更快）

dynamic-assessment:
  max_findings_per_module: 10   # 限制嘈杂模块
  enabled_modules:
    active_modules:
      - sqli-error-based
      - xss-reflected-brutelogic
    passive_modules:
      - all

scope:
  path:
    include:
      - "/api/*"
```

可覆盖的配置部分：`scanning_strategy`、`scanning_pace`、`discovery`、`spidering`、`known_issue_scan`、`dynamic-assessment`、`external_harvester`、`mutation_strategy`、`scope`。

### 配置文件设置

在 `vigolium-configs.yaml` 中设置默认配置文件或更改配置文件目录：

```yaml
scanning_strategy:
  scanning_profile: ""                    # 空 = 无配置文件，使用 default_strategy
  profiles_dir: ~/.vigolium/profiles/     # 配置文件 YAML 文件的目录
```

### 覆盖优先级

配置文件位于 CLI 标志和主配置文件之间：

1. CLI 标志（`--strategy`、`-c`、`--discover-max-time` 等）
2. `--scanning-profile` / `scanning_strategy.scanning_profile`
3. 主配置文件（`vigolium-configs.yaml`）
4. 内置默认值

## Detailed Guides（详细指南）

- [Strategies](strategies.md) — 按策略的阶段详解和调优
- [Authentication](authentication.md) — 多会话扫描、IDOR/BOLA
- [How a scan works](../architecture/native-scan.md) — 端到端管道架构
- [Phase reference](phases/) — 各阶段深入详解（内容发现、爬取、动态评估、插件、已知问题扫描）
- [Agent mode](../agentic-scan/agent-mode.md) — AI 驱动的扫描模式
- [Writing extensions](../customization/writing-extensions.md) — 自定义 JS/YAML 插件模块
