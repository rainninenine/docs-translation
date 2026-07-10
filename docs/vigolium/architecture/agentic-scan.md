# Agentic Scan Architecture — Agent 模式工作原理

> _体系结构系列：[overview](overview.md) · [native-scan](native-scan.md) · **agentic-scan** · [data-and-storage](data-and-storage.md) · [server-and-api](server-and-api.md)_

Vigolium 的 Agent 模式在本地扫描器之上运行 AI 驱动的安全扫描。本文档解释其体系结构、各个组件以及典型 Agent 运行的流程。

> **近期变更：** 之前基于子进程的 SDK 后端（`claudesdk`、`codexsdk`）已被移除。所有 AI 调度现在通过一个名为 **olium**（`pkg/olium/`）的进程内 Go 运行时进行。统一的供应商接口、统一的对话状态、统一处理超时和重试的位置。

---

## 1. Subcommand surface

`vigolium agent` 是一个父命令，仅包含信息性标志（`--list-templates`、`--list-agents`）。实际工作由子命令完成。

| 子命令 | 用途 | 关键文件 |
|--------|------|----------|
| `query` | 单次提示（模板或内联）。代码审查、秘密搜索、端点发现。 | `pkg/cli/agent.go` |
| `autopilot` | Agent 扫描：拥有完整工具访问权限的自主操作员。 | `pkg/cli/agent_autopilot.go` |
| `swarm` | Agent 扫描：10 阶段引导管道（规划 → 插件 → 扫描 → 分类）。 | `pkg/cli/agent_swarm.go` |
| `audit` | 前台 vigolium-audit（多阶段 AI 源代码审计）。 | `pkg/cli/agent_audit.go` |
| `olium` | 交互式 olium TUI 或无头提示。 | `pkg/cli/agent_olium.go` |
| `session` | 列出或检查过去的 Agent 运行（会话列表/详情视图）。 | `pkg/cli/agent_session.go` |

`query` 是唯一**不**编排扫描的模式——它是带有可选源代码上下文的单次提示。`autopilot` 和 `swarm` 是两种 **Agent 扫描**模式。

---

## 2. Architecture layers

```
┌─────────────────────────────────────────────────────────────────┐
│  CLI                          pkg/cli/agent_*.go                │
│  query · autopilot · swarm · audit · olium · session           │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│  编排器                       pkg/agent/                        │
│                                                                 │
│   SwarmRunner          swarm.go + swarm_pipeline.go             │
│     normalize → auth → source-analysis → code-audit →           │
│     discovery → plan → extension → scan → triage → finalize     │
│                                                                 │
│   AutopilotPipelineRunner    autopilot_pipeline.go              │
│     audit（可选，前台）→ 自主操作员                              │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│  引擎                         pkg/agent/engine.go               │
│  Preflight → buildPrompt → enrichContext → run on olium →       │
│  parse → ingest（findings / http_records / plans / triage）      │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│  Olium 运行时                 pkg/olium/{engine,tool,skill}     │
│  多轮 Agent 循环 · 工具注册表 · 事件流                           │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│  供应商                       pkg/olium/provider/               │
│  anthropic · openai · codex（OAuth）· claudecode（CLI）          │
└─────────────────────────────────────────────────────────────────┘
```

### 2.1 Engine（`pkg/agent/engine.go`）

引擎是编排器与 olium 运行时之间的接缝。其职责为：

1. **Preflight**——验证供应商/模型选择（`Preflight(agentName)`）。
2. **构建提示**——加载模板、解析前置元数据、使用 `TemplateData` 渲染。
3. **丰富上下文**——通过线程安全的 LRU 缓存（`agentprompt.ContextCache`，生命周期为一个 swarm/autopilot 运行）从数据库拉取上下文（之前的漏洞发现、已发现的端点、高风险端点、模块列表、扫描统计）。
4. **调度**——在 olium 引擎实例上调用 `runOliumOnEngineWithThinking()`。
5. **重试**——对瞬时错误进行指数退避（`retry.go`，默认 2 次重试，2-30 秒退避）并加入抖动。
6. **解析**——模式感知的 JSON 提取，容忍分隔符、散文描述和类型强制转换（string ↔ int、object ↔ string body）。
7. **数据导入**——将解析后的漏洞发现/HTTP 记录写入数据库仓库。

关键入口点：

- `Engine.Run(ctx, opts)`——单次提示执行；创建一个全新的 olium 引擎。
- `Engine.RunOnOliumEngine(ctx, opts, eng)`——在**共享引擎实例**上运行，保留对话前缀以实现跨阶段的提示缓存命中。
- `Engine.RunSourceAnalysisParallel(ctx, cfg)`——扇出式源代码分析（单次探索调用 → 同一引擎上的并行格式化/插件子调用）。

全局信号量（`oliumProviderSem`）限制并发供应商调用数；其值来自 `Settings.Agent.Olium.MaxConcurrent`（默认为 4）。

### 2.2 Olium 运行时（`pkg/olium/`）

原生的进程内替代方案，取代了旧的 SDK 池。

- **`pkg/olium/engine`**——`Engine.Run(ctx, prompt) <-chan Event` 返回事件流：`EventTextDelta`、`EventThinkingDelta`、`EventToolCall`、`EventTurnDone`（含 token 用量）、`EventError`。对话状态（系统提示、工具定义、先前轮次）保存在引擎上，并在各阶段共享同一引擎时复用。
- **`pkg/olium/tool`**——内置工具注册表（bash、文件操作、grep、fetch 等），通过 `tool.NewRegistry()` + `tool.RegisterBuiltins()` 注册。Autopilot 暴露全部工具集；swarm 按阶段使用较小的子集。
- **`pkg/olium/skill`**——可选的技能文件（Markdown SKILL.md 包），用于增强系统提示；从嵌入资源和 `~/.vigolium/skills/` 加载。
- **`pkg/olium/provider`**——供应商调度：

  | 供应商 | 认证来源 |
  |--------|----------|
  | `openai-codex-oauth` | `oauth_cred_path`（来自 `codex login` 的 JSON） |
  | `anthropic-api-key` | `llm_api_key` 或 `$ANTHROPIC_API_KEY` |
  | `anthropic-oauth` | `oauth_token`（来自 `claude setup-token`）；回退至 `$ANTHROPIC_API_KEY` |
  | `openai-api-key` | `llm_api_key` 或 `$OPENAI_API_KEY` |
  | `anthropic-cli` | 调用外部 `claude` 二进制文件 |
  | `anthropic-vertex` | `oauth_cred_path`（GCP 服务账号 JSON，或 `$GOOGLE_APPLICATION_CREDENTIALS`）+ `google_cloud_project` / `google_cloud_location` |
  | `google-vertex` | 相同的 GCP 凭据；路由 `gemini-*` 模型 |
  | `openai-compatible` | `custom_provider.base_url`（必填）、`custom_provider.api_key`（可选）、`custom_provider.model_id`、`custom_provider.extra_headers` |

在 `vigolium-configs.yaml` 的 `agent.olium` 中配置。每次调用的超时默认为 10 分钟（`call_timeout_sec`）。

`openai-compatible` 供应商使用 OpenAI Chat Completions 线格式，适用于任何支持该格式的后端——Ollama、OpenRouter、LM Studio、vLLM、Together、Groq、LocalAI、自定义代理。将 `base_url` 设置为端点（`/v1` 根路径或完整 `/v1/chat/completions` URL——两者均可）；对于无需认证的本地服务器，`api_key` 留空。`extra_headers` 在标准头部之后应用，因此可覆盖 `Authorization` 以用于使用非 Bearer 方案的后端。

> **工具调用注意事项。** OpenAI 风格的函数工具受线格式支持，但并非所有模型都支持。效果良好的本地指令微调模型：`qwen2.5-coder`、`llama3.1` instruct、`mistral-nemo`。较小的模型通常会静默忽略工具定义并以散文形式回复。如果 Agent 从未调用工具，这很可能是原因——请切换到经过工具训练的模型。

### 2.3 Prompt templates

带 YAML 前置元数据的 Markdown 文件，按以下顺序加载：

1. `agent.templates_dir`（配置目录）
2. `~/.vigolium/prompts/`
3. 嵌入资源（`public/presets/prompts/`，编译进二进制文件）

前置元数据声明 Agent 预期生成的**输出模式**：

| 模式 | 使用者 | 解析目标 |
|------|--------|----------|
| `findings` | 代码审查、分类、审计 | `[]AgentFinding` → 数据库 |
| `http_records` | 端点发现 | `[]AgentHTTPRecord` → 数据库 |
| `source_analysis` | swarm 源代码分析阶段 | `SourceAnalysisResult` |
| `attack_plan` / `swarm_plan` | swarm 规划 + 插件阶段 | `SwarmPlan` |
| `triage_result` | swarm 分类阶段 | `TriageResult` |

模板使用 `TemplateData`（`pkg/agent/agenttypes/types.go`）渲染，其中包含：源代码片段、目录树、目标 URL、主机名、之前的漏洞发现（数据库）、已发现的端点（数据库）、模块列表/标签、扫描统计以及一个自由格式的 `Extra` 映射，用于编排器注入的提示。

---

## 3. Swarm pipeline

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
   │ auth（可选）         │  基于浏览器的登录（--browser-auth + --browser）
   └──────────┬──────────┘
              ▼
   ┌─────────────────────┐
   │ source-analysis（AI）│  如果 --source：并行探索 + 格式化
   │                     │  输出路由、会话配置、源代码插件
   └──────────┬──────────┘
              ▼
   ┌─────────────────────┐
   │ code-audit（AI）     │  可选的代码级审计（--code-audit）
   └──────────┬──────────┘
              ▼
   ┌─────────────────────┐
   │ native-discover     │  可选的爬取/爬行（--discover 或 deep）
   └──────────┬──────────┘
              ▼
   ┌─────────────────────┐
   │ plan（AI）           │  主 Agent 选择模块 + 插件
   │                     │  大批量输入时分批处理（每批 5 条记录）
   └──────────┬──────────┘
              ▼
   ┌─────────────────────┐
   │ native-extension    │  编译/验证 JS 插件（Sobek）
   │                     │  语法错误的 LLM 修复（最多 5 个并行）
   └──────────┬──────────┘
              ▼
   ┌─────────────────────┐
   │ native-scan         │  ScanFunc：本地模块 + 自定义插件
   └──────────┬──────────┘
              ▼
   ┌─────────────────────┐
   │ triage（AI）─────────┼──→ 重新扫描？循环至 MaxIterations
   └──────────┬──────────┘     （使用定向模块重新运行 native-scan）
              ▼
   ┌─────────────────────┐
   │ finalize            │  汇总结果、token 用量、数据库更新
   └─────────────────────┘
```

前缀为 `native-` 的阶段是纯 Go 实现（无 LLM）。管道由以下条件控制：

- `--only` / `--skip` / `--start-from` 标志（通过 `NormalizeSwarmPhase` 提供遗留别名）
- 强度预设（`agenttypes/constants.go` 中的 `SwarmPresets[Quick|Balanced|Deep]`）
- `cfg.SourcePath`——空源代码跳过 source-analysis 和 code-audit
- `cfg.Discover`、`cfg.CodeAudit`、`cfg.Triage` 开关
- 检查点恢复——`--resume <session-dir>` 跳过已完成阶段

当提供源代码时，一个并行的 **vigolium-audit** 子进程可在后台运行（`cfg.Audit != ""`），贡献源代码审计漏洞发现而不阻塞 swarm。

### Plan & extension phases

主 Agent 接收输入记录（按 `MasterBatchSize` 分块，默认为 5）并返回一个 `SwarmPlan`：

```go
SwarmPlan {
    ModuleTags / ModuleIDs        // 要启用的本地模块
    Extensions []GeneratedExtension // 自定义 JS 插件（完整源代码）
    QuickChecks []QuickCheck       // 简写 → 由 extensions/quickcheck_gen.go 展开为 JS
    FocusAreas / Snippets / Hints  // 自由格式指导
}
```

插件阶段通过 Sobek 引擎编译每个 JS 插件。语法错误触发 `RepairExtensionsWithLLM()`，该函数扇出修复调用（最多 5 个并行）。幸存的插件写入 `<session>/extensions/`，使用经过清理的文件名；`extensionRenames` 映射保留原始→重命名的映射关系，用于下游结果归属。

### Triage loop

本地扫描完成后，如果存在漏洞发现且分类已启用，分类 Agent 接收一个 fixture（按详细层级截断——15 条完整详情 / 40 条含前十的表格等）并输出：

```go
TriageResult {
    Confirmed []Finding
    FalsePositive []Finding
    FollowUpScans []FollowUpScan  // 可选的重新扫描（模块 + URL）
}
```

如果 `FollowUpScans` 非空且重新扫描已启用，管道将循环回 `native-scan`，使用定向模块。循环上限为 `MaxIterations`（默认为 3）；当所有漏洞发现具有"确定"置信度时提前退出。

---

## 4. Autopilot pipeline

`vigolium agent autopilot --target ... [--source ...]` 更简单——没有规划/插件阶段。Agent 自行决定运行什么。

```
   ┌─────────────────────────────────────────────┐
   │ audit（可选，前台）                           │
   │   如果 --source：运行 vigolium-audit，冻结     │
   │   漏洞发现至 vigolium-results/ 目录           │
   └─────────────────┬───────────────────────────┘
                     ▼
   ┌─────────────────────────────────────────────┐
   │ 上下文准备                                    │
   │   - 加载冻结的审计漏洞发现                     │
   │   - prepareAutopilotAuth → AuthHeaders       │
   │   - buildAutopilotContextBundle              │
   │     （路由、认证流程、浏览器决策）              │
   │   - buildAutopilotPlan                       │
   │     （预算、任务、停止条件）                    │
   └─────────────────┬───────────────────────────┘
                     ▼
   ┌─────────────────────────────────────────────┐
   │ 自主操作员（AI）                              │
   │   olium 引擎，拥有完整工具访问权限：            │
   │     Bash, Read, Grep, Glob, Edit, Write,    │
   │     vigolium scan-url, vigolium findings,   │
   │     vigolium traffic 等                      │
   │   受 MaxCommands + Timeout 限制              │
   └─────────────────┬───────────────────────────┘
                     ▼
   ┌─────────────────────────────────────────────┐
   │ 验证                                         │
   │   verifyAutopilotArtifacts → 已确认的         │
   │   漏洞发现；如有警告则设置 degraded 标志       │
   └─────────────────────────────────────────────┘
```

努力级别（`low`/`medium`）从审计模式选取：`balanced`/`deep` 审计 → `medium` 努力，否则为 `low`。操作员流被捕获到 `<session>/output.md`。

### Intensity presets (autopilot)

| 强度 | MaxCommands | Timeout | 审计模式 | 浏览器 |
|------|-------------|---------|----------|--------|
| quick | 30 | 1h | lite | 关闭 |
| balanced | 100 | 6h | balanced | 关闭 |
| deep | 300 | 12h | deep | 开启 |

---

## 5. Session directories

每次 swarm 和 autopilot 运行都会在 `agent.sessions_dir`（默认为 `~/.vigolium/agent-sessions/<run-uuid>/`）下写入一个会话目录。布局：

```
<session>/
├── checkpoint.json         # swarm：已完成阶段、记录统计、最后一轮分类
├── plan.json               # 序列化的 SwarmPlan
├── session-config.json     # 认证会话定义（登录流程、token 规则）
├── extensions/             # 编译后的 JS 插件（经过清理的文件名）
├── vigolium-results/         # 审计子进程输出（audit-state.json + 漏洞发现）
├── master-prompt.md        # 发送给主 Agent 的渲染提示（调试用）
├── source-analysis-prompt.md
├── output.md               # autopilot：Agent 流 / 最终转录
├── output.txt              # query：原始 Agent 输出
├── inputs.json             # 归一化后的输入记录
└── skills/                 # 复制的嵌入技能（vigolium-scanner, agent-browser）
```

`pipeline_types.go` 中的 `EnsureSessionDir(baseDir, agenticScanUUID)` 是规范的创建函数。

---

## 6. Configuration

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
    provider: openai-codex-oauth          # 或 anthropic-api-key | openai-api-key | anthropic-oauth | anthropic-cli | anthropic-vertex | google-vertex | openai-compatible
    model: gpt-5.5                 # 留空则使用供应商默认
    oauth_cred_path: ~/.codex/auth.json
    llm_api_key: ${ANTHROPIC_API_KEY}
    reasoning_effort: medium
    max_tokens: 1000000
    max_turns: 32
    max_concurrent: 4              # 并发供应商调用的全局上限
    call_timeout_sec: 600          # 每次调用的超时；-1 = 无超时
    custom_provider:               # 仅在 provider == openai-compatible 时使用
      base_url: http://localhost:11434/v1    # Ollama 默认；OpenRouter / LM Studio / vLLM 也可用
      model_id: gemma4:latest                # olium.model 和 --model 的回退
      api_key: ""                            # 可选；空字符串 = 无 Authorization 头部
      extra_headers:                         # 可选；在标准头部之后应用
        # X-Provider: custom
  audit:
    # …
  browser:
    # 可选的 agent-browser 集成
```

CLI 标志（`--provider`、`--model`、`--oauth-token`、`--llm-api-key`、`--base-url`）在运行时覆盖配置。

---

## 7. Where things live

| 组件 | 位置 |
|------|------|
| 子命令接线 | `pkg/cli/agent*.go` |
| Swarm 编排器 | `pkg/agent/swarm.go`、`swarm_pipeline.go` |
| Autopilot 编排器 | `pkg/agent/autopilot_pipeline.go` |
| Vigolium Audit 运行器 | `pkg/agent/audit_agent.go` |
| 引擎（提示 → 调度） | `pkg/agent/engine.go` |
| 提示模板 / 渲染 | `pkg/agent/prompt/`、`public/presets/prompts/` |
| 输出解析器（JSON 容忍） | `pkg/agent/parsing/` |
| Olium 运行时 | `pkg/olium/engine`、`tool`、`skill` |
| Olium 供应商 | `pkg/olium/provider/` |
| 阶段常量与预设 | `pkg/agent/agenttypes/constants.go` |
| 核心类型 | `pkg/agent/agenttypes/types.go` |
| 公共别名 | `pkg/agent/aliases.go` |
| 配置模式 | `internal/config/agent.go` |

---

## 8. Quick mental model

- **引擎**将提示模板 + 数据库上下文转化为结构化结果。一次 LLM 调用。
- **编排器**将多次引擎调用与本地步骤（发现、扫描）串联，对状态进行检查点保存，并写入会话目录。
- **Olium** 是 Agent 运行时——它持有对话状态并调度到供应商。一个 olium 引擎可以低成本地服务于多次引擎调用（提示缓存命中）。
- **阶段**是可恢复工作的单位。`--only`、`--skip`、`--start-from` 和 `--resume` 作用于阶段名称。
- **强度**是一个单一旋钮，用于填充一组开关（命令数、超时、审计模式、发现/审计/分类标志、浏览器/认证）。
