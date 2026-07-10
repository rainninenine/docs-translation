# 文档 API 参考 Agentic Scan Agent 模式工作原理 复制页面

Vigolium Agent 运行时的体系结构、进程内 olium 引擎、提示编排、Swarm/Autopilot 管道以及供应商模型。复制页面

Vigolium 的 Agent 模式在本地扫描器之上运行 AI 驱动的安全扫描。本文档解释了体系结构、各个组件以及典型 Agent 运行的流程。

近期变更：之前基于子进程的 SDK 和 ACP 后端（claudesdk、codexsdk、opencode SDK、ACP 桥接）已被移除。所有 AI 调度现在通过一个名为 olium（`pkg/olium/`）的进程内 Go 运行时进行。统一的供应商接口、统一的对话状态、统一的超时和重试逻辑处理位置。

## 1. 子命令表面

`vigolium agent` 是一个父命令，仅包含信息性标志（`--list-templates`、`--list-agents`）。实际工作由子命令完成。

| 子命令 | 用途 |
|:-------|:-----|
| `query` | 单次提示（模板或内联）。代码审查、秘密搜索、端点发现。 |
| `autopilot` | Agent 扫描：具有完整工具访问权限的自主操作器。 |
| `swarm` | Agent 扫描：10 阶段引导管道（计划 → 插件 → 扫描 → 分类）。 |
| `audit` | 统一源代码审计调度器（驱动嵌入式 vigolium-audit 工具、piolium 或两者）。 |
| `piolium` | 直接 piolium 工具驱动程序（需要 Pi 原生安装）。 |
| `olium` | 交互式 olium TUI 或单次非交互式提示。 |
| `session` | 列出或检查过去的 Agent 运行（会话列表/详情视图）。 |

`query` 是唯一不编排扫描的模式，它是一个带有可选源代码上下文的单次提示。`autopilot` 和 `swarm` 是两种 Agent 扫描模式。

## 2. 体系结构层

```
┌─────────────────────────────────────────────────────────────────┐
│  CLI                          pkg/cli/agent_*.go                │
│  query · autopilot · swarm · audit · piolium · olium · session  │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│  编排器                      pkg/agent/                        │
│                                                                 │
│   SwarmRunner          swarm.go + swarm_pipeline.go             │
│     normalize → auth → source-analysis → code-audit →           │
│     discovery → plan → extension → scan → triage → finalize     │
│                                                                 │
│   AutopilotPipelineRunner    autopilot_pipeline.go              │
│     audit (可选, 前台) → 自主操作器                              │
│                                                                 │
│   Audit 驱动调度器    audit_drivers.go + audit_chain.go         │
│     driver=auto|both|audit|piolium · 每个驱动子行               │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│  引擎                        pkg/agent/engine.go               │
│  Preflight → buildPrompt → enrichContext → run on olium →       │
│  parse → ingest (findings / http_records / plans / triage)     │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│  Olium 运行时                pkg/olium/{engine,tool,skill}     │
│  多轮 Agent 循环 · 工具注册表 · 事件流                          │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│  供应商                      pkg/olium/provider/               │
│  anthropic (API/OAuth/CLI/SDK-bridge/compatible/Vertex) ·       │
│  openai (API/Responses/codex-OAuth) ·                           │
│  google-vertex · openai-compatible (HTTP)                       │
└─────────────────────────────────────────────────────────────────┘
```

### 2.1 引擎 (`pkg/agent/engine.go`)

引擎是编排器与 olium 运行时之间的接缝。其职责包括：

- **Preflight**：验证供应商/模型选择。
- **构建提示**：加载模板，解析前置元数据，使用 `TemplateData` 渲染。
- **丰富上下文**：通过一个线程安全的 LRU 缓存（生命周期为一次 swarm/autopilot 运行）从数据库拉取上下文（之前的漏洞发现、已发现的端点、高风险端点、模块列表、扫描统计）。
- **调度**：调用 olium 引擎。
- **重试**：对瞬时错误进行指数退避（默认 2 次重试，2-30 秒退避加抖动）。
- **解析**：模式感知的 JSON 提取，容忍围栏、散文和类型强制转换（string ↔ int, object ↔ string body）。
- **导入**：将解析后的漏洞发现/HTTP 记录写入数据库仓库。

关键入口点：

- `Engine.Run(ctx, opts)`：单次提示执行；创建一个新的 olium 引擎。
- `Engine.RunOnOliumEngine(ctx, opts, eng)`：在共享引擎实例上运行，保留对话前缀以实现跨阶段的提示缓存命中。
- `Engine.RunSourceAnalysisParallel(ctx, cfg)`：扇出源代码分析（单个 explore 调用 → 在同一引擎上并行 format/extension 子调用）。

一个全局信号量限制正在进行的供应商调用数量；该值来自 `agent.olium.max_concurrent`（默认 4）。

### 2.2 Olium 运行时 (`pkg/olium/`)

原生、进程内替代旧的子进程池。

- `pkg/olium/engine`：`Engine.Run(ctx, prompt) <-chan Event` 返回一个事件流：`EventTextDelta`、`EventThinkingDelta`、`EventToolCall`、`EventTurnDone`（带 token 使用量）、`EventError`。对话状态（系统提示、工具定义、先前轮次）存在于引擎上，并在阶段共享引擎时跨调用重用。
- `pkg/olium/tool`：内置工具注册表（bash、文件操作、grep、fetch 等）。Autopilot 暴露完整集合；swarm 每个阶段使用较小的集合。
- `pkg/olium/skill`：可选的技能文件（Markdown `SKILL.md` 包），用于增强系统提示；从嵌入资源和 `~/.vigolium/skills/` 加载。
- `pkg/olium/provider`：供应商调度（十一个驱动）：

| 供应商 | 认证来源 |
|:-------|:---------|
| `openai-codex-oauth` | `oauth_cred_path`（来自 `codex login` 的 JSON） |
| `anthropic-api-key` | `llm_api_key` 或 `$ANTHROPIC_API_KEY` |
| `anthropic-oauth` | `oauth_token`（来自 `claude setup-token`）；回退到 `$ANTHROPIC_API_KEY` |
| `openai-api-key` | `llm_api_key` 或 `$OPENAI_API_KEY` |
| `openai-responses` | `llm_api_key` 或 `$OPENAI_API_KEY`（公共 OpenAI Responses API，`/v1/responses`） |
| `anthropic-cli` | shell 调用本地 `claude` 二进制文件（别名：`anthropic-claude-cli`） |
| `anthropic-claude-sdk-bridge` | 通过 vigolium-audit 桥接 sidecar 登录的 Claude Code 订阅；`bridge_binary` / `--bridge-bin`（嵌入 blob，然后 PATH） |
| `anthropic-compatible` | `custom_provider.base_url`（Anthropic Messages `/v1/messages`）、`custom_provider.api_key`、`custom_provider.model_id`、`custom_provider.extra_headers` |
| `anthropic-vertex` | `oauth_cred_path`（GCP SA JSON 或 `$GOOGLE_APPLICATION_CREDENTIALS`）+ `google_cloud_project` / `google_cloud_location` |
| `google-vertex` | 相同的 GCP 凭据；路由 `gemini-*` 模型 |
| `openai-compatible` | `custom_provider.base_url`（必需）、`custom_provider.api_key`（可选）、`custom_provider.model_id`、`custom_provider.extra_headers`（Ollama / OpenRouter / LM Studio / vLLM / Groq / Together / LocalAI / 自定义代理） |

未配置任何内容时的默认供应商是 `openai-compatible`，模型为 `gemma4:latest`（本地 Ollama 端点）。在 `vigolium-configs.yaml` 的 `agent.olium` 下配置。每次调用的截止时间默认为 10 分钟（`call_timeout_sec`）。

### 2.3 提示模板

带有 YAML 前置元数据的 Markdown 文件，按以下顺序加载：

1. `agent.templates_dir`（配置目录）
2. `~/.vigolium/prompts/`
3. 嵌入的（`public/presets/prompts/` 编译到二进制文件中）

前置元数据声明 Agent 预期产生的输出模式：

| 模式 | 使用场景 | 解析为 |
|:-----|:---------|:-------|
| `findings` | 代码审查、分类、审计 | `[]AgentFinding` → 数据库 |
| `http_records` | 端点发现 | `[]AgentHTTPRecord` → 数据库 |
| `source_analysis` | swarm 源代码分析阶段 | `SourceAnalysisResult` |
| `attack_plan` / `swarm_plan` | swarm 计划 + 插件阶段 | `SwarmPlan` |
| `triage_result` | swarm 分类阶段 | `TriageResult` |

模板针对 `TemplateData` 进行渲染，其中包含：源代码片段、目录树、目标 URL、主机名、之前的漏洞发现（数据库）、已发现的端点（数据库）、模块列表/标签、扫描统计信息以及一个自由格式的 `Extra` 映射，用于编排器注入的提示。

## 3. Swarm 管道

`vigolium agent swarm --target ... [--source ...]` 运行一个状态机管道。每个步骤实现 `swarmPhaseStep.Run(ctx, *swarmPipelineState)`。

```
        ┌────────────────────────────────────────────────────┐
        │                  agent swarm                       │
        └────────────────────────────────────────────────────┘
                                 │
                                 ▼
   ┌─────────────────────┐
   │ native-normalize    │  解析 curl/原始 HTTP/Burp/URL → 记录
   └──────────┬──────────┘
              ▼
   ┌─────────────────────┐
   │ auth (可选)         │  基于浏览器的登录 (--browser-auth + --browser)
   └──────────┬──────────┘
              ▼
   ┌─────────────────────┐
   │ source-analysis (AI)│  如果 --source: 并行 explore + format
   │                     │  输出 routes, session-config, source extensions
   └──────────┬──────────┘
              ▼
   ┌─────────────────────┐
   │ code-audit (AI)     │  可选的代码级审计 (--code-audit)
   └──────────┬──────────┘
              ▼
   ┌─────────────────────┐
   │ native-discover     │  可选的爬取/爬行 (--discover 或 deep)
   └──────────┬──────────┘
              ▼
   ┌─────────────────────┐
   │ plan (AI)           │  主 Agent 选择模块 + 插件
   │                     │  对于大型输入集分批处理 (5 条记录/批)
   └──────────┬──────────┘
              ▼
   ┌─────────────────────┐
   │ native-extension    │  编译/验证 JS 插件 (Sobek)
   │                     │  LLM 修复语法错误 (最多 5 个并行)
   └──────────┬──────────┘
              ▼
   ┌─────────────────────┐
   │ native-scan         │  ScanFunc: 原生模块 + 自定义插件
   └──────────┬──────────┘
              ▼
   ┌─────────────────────┐
   │ triage (AI) ────────┼──→ 重新扫描? 循环直到 MaxIterations
   └──────────┬──────────┘     (使用目标模块重新运行 native-scan)
              ▼
   ┌─────────────────────┐
   │ finalize            │  聚合结果、token 使用量、数据库更新
   └─────────────────────┘
```

前缀为 `native-` 的阶段是纯 Go 实现（无 LLM）。管道由以下条件控制：

- `--only` / `--skip` / `--start-from` 标志（通过 `NormalizeSwarmPhase` 提供旧版别名）
- 强度预设（`SwarmPresets[Quick|Balanced|Deep]`）
- `cfg.SourcePath`：空源跳过 `source-analysis` 和 `code-audit`
- `cfg.Discover`、`cfg.CodeAudit`、`cfg.Triage` 开关
- 检查点恢复，`--resume <session-dir>` 跳过已完成的阶段

当提供源代码时，一个并行的 `vigolium-audit` 子进程可以在后台运行（`cfg.Audit != ""`），贡献源代码审计漏洞发现而不阻塞 swarm。Swarm 直接使用嵌入的 `vigolium-audit` 工具；多驱动 `agent audit` 命令在此基础上添加了 `piolium` 支持。

### 计划与插件阶段

主 Agent 接收输入记录（分块为 `MasterBatchSize`，默认 5）并返回一个 `SwarmPlan`：

```
SwarmPlan {
    ModuleTags / ModuleIDs        // 要启用的原生模块
    Extensions []GeneratedExtension // 自定义 JS 插件（完整源代码）
    QuickChecks []QuickCheck       // 简写 → 由 extensions/quickcheck_gen.go 展开为 JS
    FocusAreas / Snippets / Hints  // 自由格式指导
}
```

插件阶段通过 Sobek 引擎编译每个 JS 插件。语法错误触发 LLM 修复过程（最多 5 个并行）。幸存的插件写入 `<session>/extensions/`，文件名经过清理。

### 分类循环

原生扫描后，如果存在漏洞发现且启用了分类，分类 Agent 接收一个夹具（按详细级别截断，15 个完整详情 / 40 个表格含前 10 个等）并输出：

```
TriageResult {
    Confirmed []Finding
    FalsePositive []Finding
    FollowUpScans []FollowUpScan  // 可选的重新扫描（模块 + URL）
}
```

如果 `FollowUpScans` 非空且启用了重新扫描，管道循环回 `native-scan`，使用目标模块。循环受 `MaxIterations` 限制（默认 3）；当所有漏洞发现具有“确定”置信度时提前退出。

## 4. Autopilot 管道

`vigolium agent autopilot --target ... [--source ...]` 更简单，没有计划/插件阶段。Agent 自行决定运行什么。

```
   ┌─────────────────────────────────────────────┐
   │ vigolium-audit (可选, 前台)                  │
   │   如果 --source: 运行 vigolium-audit, 冻结   │
   │   漏洞发现到 vigolium-audit/ 目录            │
   └─────────────────┬───────────────────────────┘
                     ▼
   ┌─────────────────────────────────────────────┐
   │ 上下文准备                                   │
   │   - 加载冻结的 vigolium-audit 漏洞发现       │
   │   - prepareAutopilotAuth → AuthHeaders      │
   │   - buildAutopilotContextBundle             │
   │     (路由、认证流程、浏览器决策)             │
   │   - buildAutopilotPlan                      │
   │     (预算、任务、停止条件)                   │
   └─────────────────┬───────────────────────────┘
                     ▼
   ┌─────────────────────────────────────────────┐
   │ 自主操作器 (AI)                              │
   │   olium 引擎，具有完整工具访问权限:          │
   │     Bash, Read, Grep, Glob, Edit, Write,    │
   │     vigolium scan-url, vigolium findings,   │
   │     vigolium traffic, 等。                   │
   │   受 MaxCommands + Timeout 限制              │
   └─────────────────┬───────────────────────────┘
                     ▼
   ┌─────────────────────────────────────────────┐
   │ 验证                                         │
   │   verifyAutopilotArtifacts → 确认的          │
   │   漏洞发现；如果有警告则设置 degraded 标志   │
   └─────────────────────────────────────────────┘
```

努力级别（low/medium）从 `vigolium-audit` 模式中选择：balanced/deep `vigolium-audit` → medium 努力，否则 low。操作器流捕获到 `<session>/output.md`。

### 强度预设 (autopilot)

| 强度 | 最大命令数 | 超时 | Vigolium-audit 模式 | 浏览器 |
|:-----|:-----------|:-----|:--------------------|:-------|
| quick | 150 | 1h | lite | on |
| balanced | 500 | 6h | balanced | on |
| deep | 1500 | 12h | deep | on |

## 5. 会话目录

每次 swarm 和 autopilot 运行都会在 `agent.sessions_dir`（默认 `~/.vigolium/agent-sessions/<run-uuid>/`）下写入一个会话目录。布局：

```
<session>/
├── checkpoint.json         # swarm: 已完成的阶段、记录统计、最后一轮分类
├── swarm-plan.json         # 序列化的 SwarmPlan
├── session-config.json     # 认证会话定义（登录流程、token 规则）
├── extensions/             # 编译的 JS 插件（清理后的文件名）
├── vigolium-audit/         # vigolium-audit 子进程输出（audit-state.json + 漏洞发现）
├── piolium/                # piolium 子进程输出（当审计调度器运行 piolium 时）
├── master-output.md        # 发送给主 Agent 的渲染提示（调试）
├── source-analysis-output.md
├── code-audit-output.md
├── output.md               # autopilot: Agent 流 / 最终记录
├── runtime.log             # 原始运行时日志镜像（用于 `vigolium agent log <uuid>`）
├── audit-stream.jsonl      # `agent audit` 流式 Agent 事件（NDJSON）
├── inputs.json             # 规范化的输入记录
└── skills/                 # 复制的嵌入技能（vigolium-scanner, agent-browser）
```

`EnsureSessionDir(baseDir, agenticScanUUID)` 在 `pkg/agent/pipeline_types.go` 中是规范的创建函数。

## 6. 设置

所有 Agent 设置位于 `vigolium-configs.yaml` 的 `agent` 下：

```yaml
agent:
  default_agent: olium
  templates_dir: ~/.vigolium/prompts
  sessions_dir: ~/.vigolium/agent-sessions
  context_limits:
    max_findings: 50
    max_endpoints: 100
    max_high_risk: 20
    min_risk_score: 50
  olium:
    provider: openai-compatible    # openai-codex-oauth | anthropic-api-key | openai-api-key | openai-responses | anthropic-oauth | anthropic-cli | anthropic-claude-sdk-bridge | anthropic-compatible | anthropic-vertex | google-vertex | openai-compatible
    model: gemma4:latest           # 供应商默认值（如果为空）
    oauth_cred_path: ~/.codex/auth.json
    llm_api_key: ${ANTHROPIC_API_KEY}
    reasoning_effort: medium
    max_tokens: 1000000
    max_turns: 32                  # 短的非 autopilot 使用；autopilot 使用自己的上限（DefaultAutopilotMaxTurns=200）
    max_concurrent: 4              # 全局限制正在进行的供应商调用
    call_timeout_sec: 600          # 每次调用的截止时间；-1 = 无超时
    custom_provider:               # 仅当 provider == openai-compatible 时使用
      base_url: http://localhost:11434/v1   # Ollama; OpenRouter / LM Studio / vLLM 也可用
      model_id: gemma4:latest
      api_key: ""
  audit:
    enable: false
    mode: lite
  browser:
    enable: true
    binary_path: agent-browser
```

CLI 标志（`--provider`、`--model`、`--oauth-cred`、`--oauth-token`、`--llm-api-key`、`--bridge-bin`）在运行时覆盖配置。REST API 也接受按请求的 BYOK 凭据。

## 7. 代码位置

| 内容 | 位置 |
|:-----|:-----|
| 子命令接线 | `pkg/cli/agent*.go` |
| Swarm 编排器 | `pkg/agent/swarm.go`, `swarm_pipeline.go` |
| Autopilot 编排器 | `pkg/agent/autopilot_pipeline.go` |
| Audit 驱动调度器 | `pkg/agent/audit_drivers.go`, `audit_chain.go` |
| Vigolium-audit / piolium 运行器 | `pkg/agent/audit_agent.go`, `pkg/piolium/` |
| 引擎（提示 → 调度） | `pkg/agent/engine.go` |
| 提示模板 / 渲染 | `pkg/agent/prompt/`, `public/presets/prompts/` |
| 输出解析器（JSON 容忍） | `pkg/agent/parsing/` |
| Olium 运行时 | `pkg/olium/engine`, `tool`, `skill` |
| Olium 供应商 | `pkg/olium/provider/` |
| 阶段常量 & 预设 | `pkg/agent/agenttypes/constants.go` |
| 核心类型 | `pkg/agent/agenttypes/types.go` |
| 公共别名 | `pkg/agent/aliases.go` |
| 配置模式 | `internal/config/agent.go` |

## 8. 快速心智模型

- **引擎**将提示模板 + 数据库上下文转换为结构化结果。一次 LLM 调用。
- **编排器**序列化多次引擎调用以及原生步骤（发现、扫描），检查点状态，并写入会话目录。
- **Olium** 是 Agent 运行时，它持有对话状态并调度到供应商。一个 olium 引擎可以廉价地服务多次引擎调用（提示缓存命中）。
- **阶段**是可恢复工作的单位。`--only`、`--skip`、`--start-from` 和 `--resume` 操作阶段名称。
- **强度**是一个单一旋钮，用于填充一组开关（命令数、超时、vigolium-audit 模式、发现/审计/分类标志、浏览器/认证）。

---

**上一页** Agent 模式  
Vigolium 在 `vigolium agent` 下提供了七个 Agent 子命令，涵盖自主扫描、AI 引导管道、统一源代码审计驱动、单次提示和交互式 TUI。

**下一页** ⌘I

---

网站 | Twitter | GitHub | Discord | X | LinkedIn

---

本文档使用 Mintlify 构建和托管，一个开发者文档平台。

**本页内容**

1. 子命令表面
2. 体系结构层
   2.1 引擎 (`pkg/agent/engine.go`)
   2.2 Olium 运行时 (`pkg/olium/`)
   2.3 提示模板
3. Swarm 管道
   计划与插件阶段
   分类循环
4. Autopilot 管道
   强度预设 (autopilot)
5. 会话目录
6. 设置
7. 代码位置
8. 快速心智模型