# Olium Agent

`olium` 是进程内 AI Agent 运行时，为 vigolium 中的所有 Agent 功能提供支持。它以两种形式提供：

- **面向用户的命令** — `vigolium agent olium`（别名：`vigolium olium`、`vigolium ol`）— 用于在 TUI 中进行交互式聊天或脚本化的一次性提示。
- **库** — `pkg/olium/` — Autopilot、Swarm、Query、审计准备和源代码分析路径都通过它进行调度。**没有子进程 SDK 后端**；vigolium 中的每个 AI 调用都通过此引擎进行。

本文档介绍了 Olium Agent 是什么、它的功能以及如何使用。有关与其他 Agent 子命令的高级比较，请参阅 [`agent-mode.md`](agent-mode.md)。

---

## 是什么

一个基于回合、使用工具的 LLM Agent，用 Go 编写。组件：

| 层 | 位置 | 职责 |
|---|---|---|
| **引擎** | `pkg/olium/engine/` | 多轮循环：供应商流 → 工具调度 → 历史追加 → 重复 |
| **供应商** | `pkg/olium/provider/` | LLM 后端 — 五个驱动（openai-codex-oauth、anthropic-api-key、anthropic-oauth、openai-api-key、anthropic-cli） |
| **工具** | `pkg/olium/tool/` | 模型可以调用的八个内置原语（bash、文件操作、搜索、网页获取） |
| **技能** | `pkg/olium/skill/` | SKILL.md 工作流文件（agentskills.io 格式），从项目、用户和嵌入式范围中发现 |
| **TUI** | `pkg/olium/tui/` | Bubble Tea 前端（内联回滚、斜杠命令、实时工具卡片） |
| **无头模式** | `pkg/olium/headless.go` | 用于脚本和冒烟测试的非交互式单提示运行器 |
| **Autopilot** | `pkg/olium/autopilot/` | 基于引擎的长时间运行自主扫描循环，具有预算、停止信号和 `report_finding` |
| **Vigolium 工具** | `pkg/olium/vigtool/` | 扫描器感知的扩展：`run_scan`、`run_extension`、`list_sessions`、`list_findings`、`auth_session_lookup` 等 |
| **身份验证** | `pkg/olium/auth/` | Codex OAuth 凭据加载和刷新（处理 `~/.codex/auth.json`） |

入口点：

- `pkg/olium/runner.go` — `RunTUI()`、`LoadSkillsFor()`、`ResolveProvider()`、`Options`
- `pkg/olium/headless.go` — `RunHeadless()`
- `pkg/olium/autopilot/autopilot.go` — `Run()`（由 `vigolium agent autopilot` 使用）
- `pkg/agent/olium_adapter.go` — 由 query/swarm/source-analysis 使用的桥接

---

## 功能

每次调用运行一个**多轮循环**（`pkg/olium/engine/engine.go:171`）：

1. 将用户提示追加到历史记录。
2. 流式传输单个供应商响应（文本增量、思考增量、工具调用）。
3. 将助手回合追加到历史记录；发出带有 token 使用量的 `EventTurnDone`。
4. 如果没有工具调用 → 发出 `EventRunDone` 并退出。
5. 否则调度工具调用。如果**所有**调用都是只读的，引擎会并行分发它们（上限 = 8）；否则严格串行运行，以便写入不会与读取竞争。无论哪种情况，工具结果都会按模型的原始顺序追加到历史记录。
6. 循环回到步骤 2 — 受 `MaxTurns` 限制（聊天/无头模式默认 **32**，autopilot 默认 **200**）。

周围行为：

- **工具结果截断/溢出** — 大于 `MaxToolResultBytes`（默认 16 KiB）的结果会被截断头部和尾部，并带有省略标记。如果设置了 `SpillDir`（autopilot 会这样做），完整载荷会溢出到 `<SpillDir>/tool-results/`，模型会获得头部摘录和磁盘上的路径，可以通过 `read_file` 读取。
- **每个工具超时** — 每个工具调用都有自己的截止时间（默认 5 分钟）。失控的 `bash curl` 不会挂起整个会话。
- **提示缓存** — 通过 `EnablePromptCache` 选择加入。Autopilot 会启用它；Anthropic 供应商随后会在系统提示和工具列表上写入 `cache_control: ephemeral` 标记，在长时间运行中将重复前缀 token 减少约 90%。不支持缓存的供应商会忽略该标志。
- **技能** — 当加载注册表时，引擎会在构造时向系统提示中注入一个 `<available_skills>` 块（`engine.go:97`），并注册一个 `load_skill` 工具，模型可以调用该工具按需获取技能主体。

---

## 模式

### 交互式 TUI（默认）

```bash
vigolium olium                     # 聊天
vigolium ol                        # 别名
vigolium agent olium               # 完整路径
vigolium ol "audit this repo"      # 自动提交的第一个提示
echo "summarise" | vigolium ol     # 管道输入时自动检测 stdin
```

Bubble Tea 内联模式（无 alt-screen — 输出在流式传输时追加到回滚）。实时部分行、通过 chroma 进行围栏代码块高亮，以及工具运行时的一行“工具执行”卡片。斜杠选择器在输入 `/` 时打开：

- `/clear` — 清除对话历史。
- `/skill:<name> [args]` — 内联展开已加载的技能（`pkg/olium/skill/systemprompt.go:66`）；主体被粘贴到提示中，这样模型就不必花费工具调用来 `load_skill`。

模型 ID、供应商和推理努力显示在横幅标题中。

### 无头模式（一次性）

传递 `-p` / `--prompt` 以非交互方式运行单个提示并流式传输到 stdout — TUI 会自动跳过。

```bash
vigolium ol -p "list every route in this repo"
```

将助手文本打印到 **stdout**；思考增量、工具开始/结束卡片以及每回合的 `[turn done in= out= cached=]` 摘要打印到 **stderr**。引擎错误时以非零退出。

### 库使用（autopilot、swarm、query）

`pkg/agent/olium_adapter.go` 是其他所有 Agent 功能汇集的单一调度路径：

- `runOliumPrompt(ctx, cfg, prompt, streamWriter, sourcePath)` — 每次调用创建新引擎。
- `runOliumOnEngine(ctx, cfg, eng, prompt, streamWriter)` — 重用引擎，以便对话前缀保持温暖（由源代码分析使用，将探索阶段分叉为 3 个并行格式调用）。
- `acquireProviderSlot(ctx, cfg)` — 全局信号量（大小 = `agent.olium.max_concurrent`，默认 4），限制整个进程中正在进行的供应商调用，以便 swarm 阶段扇出不会在 tier-1 计划上触发 429 错误。
- `EffectiveCallTimeout()` — 每次供应商调用默认 10 分钟；0 → 默认，负数 → 无超时。

---

## 供应商

`pkg/olium/provider/` 中的五个驱动。供应商名称指明了身份验证机制，因此可以清楚哪个凭据字段适用：

| 供应商 | 身份验证 | 默认模型 | 凭据来源 |
|---|---|---|---|
| `openai-codex-oauth` *(默认)* | OAuth 凭据文件 | `gpt-5.5` | `--oauth-cred` → `agent.olium.oauth_cred_path` → `~/.codex/auth.json`（由 `codex login` 生成） |
| `anthropic-api-key` | `x-api-key` 标头 | `claude-opus-4-7` | `--llm-api-key` → `agent.olium.llm_api_key` → `$ANTHROPIC_API_KEY` |
| `anthropic-oauth` | Bearer token（Claude Code OAuth） | `claude-opus-4-7` | `--oauth-token` → `agent.olium.oauth_token` → `$ANTHROPIC_API_KEY`（由 `claude setup-token` 生成） |
| `openai-api-key` | `x-api-key` 标头 | `gpt-5.5` | `--llm-api-key` → `agent.olium.llm_api_key` → `$OPENAI_API_KEY` |
| `anthropic-cli` | （无 — 子进程） | `claude-opus-4-7` | `--claude-bin`（默认 `claude` 在 `$PATH` 上） |

选择逻辑位于 `pkg/olium/runner.go:resolveProvider()` 和 `pkg/olium/select.go`。如果没有 `--provider` 标志且没有 YAML 覆盖，vigolium 会自动检测为 **`openai-codex-oauth`**（`select.go:78`）。`anthropic-oauth` 供应商还会在系统提示前添加一个 Claude Code 序言，并添加 `oauth-2025-04-20` beta 标头，以便在与 `anthropic-api-key` 相同的端点上被接受。

Codex 身份验证会自我刷新：`pkg/olium/auth/codex.go` 解析 JWT，检查过期时间（60 秒偏差），并使用存储的刷新令牌向 `/oauth/token` 发送 POST 请求，重写 `~/.codex/auth.json`（`0o600`）。

> **注意：** REST API **不**镜像这些每次调用的标志。服务器从 `vigolium-configs.yaml` 中的 `agent.olium.*` 解析一次供应商，并在请求之间重用，以便热缓存保持稳定。要在服务器端切换供应商，请编辑 YAML 并重新加载。

---

## 工具

内置工具注册表（`pkg/olium/tool/builtin.go:6` — 八个工具，按此顺序注册）：

| 名称 | 只读？ | 功能 |
|---|---|---|
| `bash` | 否 | `bash -lc <cmd>`，对灾难性模式（`rm -rf /`、`dd` 到块设备、fork 炸弹、`mkfs` 针对真实设备）进行硬拒绝。默认超时 = 引擎 `ToolTimeout`（5 分钟）。 |
| `read_file` | 是 | 读取文件，带行号前缀。参数：`path`、`offset`、`limit`（默认 2000）。 |
| `write_file` | 否 | 创建或覆盖文件。 |
| `edit_file` | 否 | 对文件进行查找替换编辑。 |
| `ls` | 是 | 列出目录。 |
| `grep` | 是 | 正则表达式搜索 — 可用时使用 ripgrep，否则使用原生 Go 正则表达式。参数：`pattern`、`path`、`glob`、`max_matches`（200）、`ignore_case`。 |
| `glob` | 是 | Glob 模式 → 路径。 |
| `web_fetch` | 是 | 获取 URL。两种模式：`http`（默认，快速）和 `browser`（委托给 `agent-browser` 处理 SPA / JS 密集型页面）。参数：`url`、`method`、`headers`、`body`、`max_bytes`、`mode`、`wait_selector`、`wait_ms`。 |

`IsReadOnly()` 标志是引擎用来决定是否并行分发一个回合的工具调用的依据。`bash` 运行**无需批准提示**（yolo 模式）— 只有灾难性模式防护可以防止灾难。`RegisterBuiltins` 上的 `ApprovalFn` 参数已连接但今天未使用；它用于将来插件安装的工具。

### Autopilot 添加更多

当引擎在 `vigolium agent autopilot` 下运行时，注册表还会获得：

- `halt_scan` — 模型驱动的退出。设置停止信号；运行循环在当前回合后退出。
- `report_finding` — 将漏洞发现持久化到数据库（标题、严重性、描述、修复建议、CWE、证据、置信度、状态）。50 次调用时软警告，200 次硬上限。
- `load_skill` — 按名称获取技能主体（只要技能注册表非空就注册）。
- **Vigtool** — `run_scan`、`run_extension`、`list_sessions`、`get_session`、`list_findings`、`list_auth_sessions`、`auth_session_lookup`（当 `Repo` 非 nil 时注册）。

---

## 技能

技能是带有 YAML 前置元数据的 Markdown 工作流文件，遵循 [agentskills.io](https://agentskills.io) 约定，因此为 Claude Code 或 pi 编写的文件可以在 olium 中直接使用。格式：

```markdown
---
name: triage-finding
description: Walk a candidate finding from suspicious response → root cause → PoC.
license: optional
allowed-tools: optional list
---

# Body
模型在调用 load_skill 后阅读的指导性散文。
```

`name` 必须匹配 `[a-z0-9-]+`（≤64 个字符）；`description` ≤1024 个字符。

### 发现

`Load()`（`pkg/olium/skill/registry.go:77`）遍历四个范围 — 按名称最先找到的获胜：

1. **项目** — 工作目录及其所有祖先目录中的 `.agents/skills/` 和 `.claude/skills/`，最近优先。
2. **用户** — `~/.vigolium/skills/`（仅当 `IncludeUserSkills=true` 时）。
3. **嵌入式** — 通过 `go:embed` 内置于二进制文件中的 `public/presets/skills/`。

接受两种磁盘布局：`<root>/<name>/SKILL.md`（目录技能，agentskills.io 标准）或 `<root>/<name>.md`（单文件简写；前置元数据 `name` 必须与文件名主干匹配）。

通用聊天（`vigolium agent olium`、无头模式）仅加载范围 1 和 3。Autopilot 和 swarm 加载所有三个，以便 `~/.vigolium/skills/` 中的安全特定工作流不会污染随意聊天（`pkg/olium/runner.go:25`）。

### 使用

引擎将 `<available_skills>` 块写入系统提示，列出每个技能的名称 + 描述 + 位置（`skill/systemprompt.go:14`）。模型通过 `load_skill` 工具按需获取主体 — 渐进式披露，因此未使用的技能不会消耗 token。

在 TUI 中，输入 `/skill:<name> [args]` 将技能主体直接内联展开到提示中，无需工具调用。

---

## CLI 标志

```text
--provider          openai-codex-oauth | anthropic-api-key | anthropic-oauth | openai-api-key | anthropic-cli
--model             供应商特定（空 = 供应商默认）
--oauth-cred        OAuth 凭据文件（openai-codex-oauth；默认 ~/.codex/auth.json）
--oauth-token       Claude Code OAuth Bearer token（anthropic-oauth）
--llm-api-key       anthropic-api-key / openai-api-key 的 API 密钥
--claude-bin        `claude` 二进制文件的路径（anthropic-cli）
--system            覆盖内置系统提示
--headless          一次性非交互 — 打印到 stdout，退出
-p, --prompt        初始提示（替代位置参数）
--stdin             强制从 stdin 读取提示
```

初始提示的优先级：位置参数 → `-p/--prompt` → stdin（管道输入时自动检测，或使用 `--stdin` 强制）。值按 CLI → YAML → 环境变量的顺序流动：每个 CLI 标志都回退到其 `agent.olium.*` YAML 字段，该字段又回退到文档化的默认值或环境变量。请参阅 `pkg/cli/agent_olium.go`。

---

## 设置（`vigolium-configs.yaml`）

完整的 `agent.olium` 块（在 `internal/config/agent.go:38` 中定义）：

```yaml
agent:
  olium:
    provider: openai-codex-oauth          # openai-codex-oauth | anthropic-api-key | anthropic-oauth | openai-api-key | anthropic-cli
    model: gpt-5.5                 # 空 = 供应商默认
    oauth_cred_path: ~/.codex/auth.json
    oauth_token: ""                # anthropic-oauth；支持 ${ENV_VAR}；回退到 $ANTHROPIC_API_KEY
    llm_api_key: ""                # 支持 ${ENV_VAR}；回退到 $ANTHROPIC_API_KEY / $OPENAI_API_KEY
    reasoning_effort: medium       # minimal|low|medium|high|xhigh (codex)
    system_prompt: ""              # 空 = 内置 olium 提示
    max_tokens: 1000000
    temperature: 0.0
    max_turns: 32
    cache_size: 1024               # LRU；0 禁用
    max_concurrent: 4              # 全局同时供应商调用上限；0 = 无限制
    call_timeout_sec: 600          # 每次调用截止时间；负数 = 无超时（仅父上下文）
```

`DefaultOliumConfig()`（agent.go:291）提供这些确切的默认值，因此 `vigolium config ls olium` 显示每个旋钮，无需用户 YAML。

值得了解的相邻配置块：

- `agent.sessions_dir` — 每次运行会话目录的位置。默认 `~/.vigolium/agent-sessions/`。
- `agent.browser` — 切换 `agent-browser` 集成（`web_fetch` 在 `mode: browser` 时调用的二进制文件）。
- `agent.audit` — 控制可选的 vigolium-audit 准备步骤，autopilot/swarm 可以在 olium 循环之前堆叠该步骤。

---

## 会话和磁盘状态

每次 Agent 运行都会在 `agent.sessions_dir`（默认 `~/.vigolium/agent-sessions/<run-uuid>/`）下获得一个会话目录。单纯的 `vigolium agent olium` 聊天不会写入会话 — 只有 autopilot/swarm/query 会具体化一个。

在会话目录中，您可能会找到：

- `runtime.log` — 每回合事件日志（文本增量、工具开始/结束、回合完成摘要）。
- `tool-results/<tool>-<call-id>.txt` — 溢出的超大工具输出（当引擎的 `SpillDir` 设置时）。
- `session-config.json` — 运行元数据（项目/扫描 UUID、选项）。
- `swarm-plan.json`、`master-output.md`、`audit-stream.jsonl`、`checkpoint.json` — 由包装 olium 的更高级模式（swarm、audit、autopilot）生成。

使用 `vigolium agent session list` / `--full` / `--tail` 浏览过去的运行。

---

## 流事件

引擎发出统一的 `Event` 通道（`pkg/olium/engine/event.go`），与供应商无关：

| 事件 | 携带内容 |
|---|---|
| `EventTextDelta` | `Delta` — 助手文本增量 |
| `EventThinkingDelta` | `Delta` — 推理内容（Anthropic 思考、codex 推理） |
| `EventToolCallStart` | `ToolName`、`ToolArgs` — 模型决定调用工具 |
| `EventToolExecStart` / `EventToolExecProgress` / `EventToolExecEnd` | 工具调用生命周期、`ToolResult`、`ToolIsErr` |
| `EventTurnDone` | `StopReason`、`Usage`（输入/输出/缓存读取/缓存写入 token） |
| `EventRunDone` | 终端使用量 |
| `EventError` | `Err` — 供应商故障、上下文取消、超过最大回合数 |

`EventTurnDone` 上的 token 计数由每个更高级的调用者累积（autopilot 用于预算执行，适配器用于 `agenttypes.TokenUsage`，swarm 用于成本报告）。

---

## 何时使用什么

| 您想要... | 使用 |
|---|---|
| 交互式聊天/调试/探索 | `vigolium ol` |
| 从脚本运行一个提示并解析 stdout | `vigolium ol -p "..."` |
| 将方向盘交给 Agent 进行自主渗透测试 | `vigolium agent autopilot`（底层使用 olium，带有预算 + report_finding） |
| AI 指导本地扫描器（计划 → 模块 → 分类） | `vigolium agent swarm` |
| 一次性模板驱动提示，带有结构化输出 | `vigolium agent query` |

Olium 本身是**通用**聊天/开发界面，也是其他所有模式重用的引擎 — 它本身不是安全扫描。

---

## 另请参阅

- [`agent-mode.md`](agent-mode.md) — 完整的 Agent 子命令映射。
- [`autopilot.md`](autopilot.md) — 基于 olium 引擎的自主扫描模式。
- [`swarm.md`](swarm.md) — AI 引导的多阶段扫描，驱动本地扫描器。
- [`architecture/agentic-scan.md`](../architecture/agentic-scan.md) — 供应商列表和高级调度故事。