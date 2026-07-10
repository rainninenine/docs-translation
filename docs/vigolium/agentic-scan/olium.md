# 文档API参考Agentic ScanOlium Agent复制页面Vigolium 中所有 Agent 功能的进程内 AI Agent 运行时，包括交互式 TUI、无头提示，以及 Autopilot、Swarm、Query 和 vigolium-audit 使用的引擎库。

复制页面olium 是 Vigolium 中所有 Agent 功能的进程内 AI Agent 运行时。它提供两种形式：

- **面向用户的命令**：`vigolium agent olium`（别名：`vigolium olium`、`vigolium ol`），用于 TUI 中的交互式聊天或脚本化的一次性提示。
- **库**：`pkg/olium/`，Autopilot、Swarm、Query、`vigolium-audit-prep` 和 source-analysis 路径均通过其分发。没有子进程 SDK 或 ACP 后端；Vigolium 中的每个 AI 调用都通过此引擎进行。

有关与其他 Agent 子命令的高级比较，请参阅 Agent Mode。

## 什么是 Olium

一个基于回合制、使用工具的 LLM Agent，用 Go 编写。组件：

| 层 | 位于 | 职责 |
|:---|:-----|:-----|
| Engine | `pkg/olium/engine/` | 多轮循环：Provider 流 → 工具分发 → 历史追加 → 重复 |
| Provider | `pkg/olium/provider/` | LLM 后端，11 个驱动（`openai-codex-oauth`、`anthropic-api-key`、`anthropic-oauth`、`openai-api-key`、`openai-responses`、`anthropic-cli`、`anthropic-claude-sdk-bridge`、`anthropic-compatible`、`anthropic-vertex`、`google-vertex`、`openai-compatible`） |
| Tools | `pkg/olium/tool/` | 模型可调用的八个内置原语（bash、文件操作、搜索、网页获取） |
| Skills | `pkg/olium/skill/` | 从项目、用户和嵌入式作用域发现的 SKILL.md 工作流文件（agentskills.io 格式） |
| TUI | `pkg/olium/tui/` | Bubble Tea 前端（内联滚动、斜杠命令、实时工具卡片） |
| Headless | `pkg/olium/headless.go` | 用于脚本和冒烟测试的非交互式单提示运行器 |
| Autopilot | `pkg/olium/autopilot/` | 基于引擎的长时间运行自主扫描循环，具有预算、停止信号和 `report_finding` |
| Vigolium tools | `pkg/olium/vigtool/` | 扫描器感知的扩展：`run_scan`、`run_extension`、`list_sessions`、`list_findings`、`auth_session_lookup` 等 |
| Auth | `pkg/olium/auth/` | Codex OAuth 凭据加载和刷新（处理 `~/.codex/auth.json`） |

## 它的作用

每次调用运行一个多轮循环：

1. 将用户提示追加到历史记录。
2. 流式传输单个 Provider 响应（文本增量、思考增量、工具调用）。
3. 将助手回合追加到历史记录；发出带有 token 使用量的 `EventTurnDone`。
4. 如果没有工具调用 → 发出 `EventRunDone` 并退出。
5. 否则分发工具调用。如果所有调用都是只读的，引擎会并行扇出（上限 = 8）；否则严格串行运行，以便写入不会与读取竞争。无论顺序如何，工具结果都会按模型的原始顺序追加到历史记录。
6. 循环回到步骤 2，上限为 `MaxTurns`（聊天/无头默认为 32，Autopilot 为 200）。

周围行为：

- **工具结果截断/溢出**：大于 `MaxToolResultBytes`（默认 16 KiB）的结果会被截断头部和尾部，并带有省略标记。如果设置了 `SpillDir`（Autopilot 会这样做），完整载荷会溢出到 `<SpillDir>/tool-results/`，模型会获得头部摘录和磁盘上的路径，它可以通过 `read_file` 读取。
- **每个工具超时**：每个工具调用都有自己的截止时间（默认 5 分钟）。失控的 bash curl 不会挂起整个会话。
- **提示缓存**：通过 `EnablePromptCache` 选择加入。Autopilot 会启用它；Anthropic Provider 会写入 `cache_control: ephemeral` 标记，Codex OAuth Provider 会写入 `prompt_cache_key` 标头，在长时间运行中将重复前缀 token 减少约 90%。`openai-api-key`、`openai-responses`、`openai-compatible`（Ollama / OpenRouter / LM Studio / vLLM / Groq / …）、`anthropic-compatible` 和 `google-vertex` 不会发出缓存标记，因此该标志对它们静默忽略。（`anthropic-claude-sdk-bridge` 在 Claude Code Agent SDK 内部管理缓存。）
- **Skills**：加载注册表时，引擎会在构建时将 `<available_skills>` 块注入系统提示，并注册一个 `load_skill` 工具，模型可以调用该工具按需获取技能主体。

## 模式

### 交互式 TUI（默认）

```bash
vigolium olium                     # 聊天
vigolium ol                        # 别名
vigolium agent olium               # 完整路径
vigolium ol "audit this repo"      # 自动提交的第一个提示
echo "summarise" | vigolium ol     # 通过管道输入时自动检测 stdin
```

Bubble Tea 内联模式（无 alt-screen，输出在流式传输时追加到滚动缓冲区）。实时部分行，通过 chroma 进行围栏代码块高亮，以及工具运行时的一行“工具执行”卡片。斜杠选择器在 `/` 上打开：

- `/clear`：清除对话历史。
- `/skill:<name> [args]`：内联展开已加载的技能；主体被粘贴到提示中，因此模型不必花费工具调用来 `load_skill`。

模型 ID、Provider 和推理努力显示在横幅标题中。

### 一次性非交互式

传递 `-p` / `--prompt` 以非交互方式运行单个提示并流式传输到 stdout，TUI 会自动跳过。

```bash
vigolium ol -p "list every route in this repo"
```

将助手文本打印到 stdout；思考增量、工具开始/结束卡片以及每轮 `[turn done in= out= cached=]` 摘要输出到 stderr。引擎错误时以非零退出。

### 库使用（Autopilot、Swarm、Query）

`pkg/agent/olium_adapter.go` 是每个其他 Agent 功能汇集的单个分发路径：

- `runOliumPrompt(ctx, cfg, prompt, streamWriter, sourcePath)`：每次调用新建引擎。
- `runOliumOnEngine(ctx, cfg, eng, prompt, streamWriter)`：重用引擎，以便对话前缀保持热状态（由 source-analysis 用于将探索阶段分叉为 3 个并行格式调用）。
- `acquireProviderSlot(ctx, cfg)`：全局信号量（大小 = `agent.olium.max_concurrent`，默认 4），限制整个进程范围内的进行中 Provider 调用，以便 Swarm 阶段扇出不会在 tier-1 计划上触发 429 错误。
- `EffectiveCallTimeout()`：每个 Provider 调用默认 10 分钟；0 → 默认，负数 → 无超时。

## Provider

`pkg/olium/provider/` 中的 11 个驱动。Provider ID 以供应商优先，因此适用的凭据字段显而易见：

| Provider | 身份验证 | 默认/典型模型 | 凭据来源 |
|:---------|:---------|:---------------|:---------|
| `openai-codex-oauth` | OAuth 凭据文件 | `gpt-5.5` | `--oauth-cred` → `agent.olium.oauth_cred_path` → `~/.codex/auth.json`（由 `codex login` 生成） |
| `anthropic-api-key` | `x-api-key` 标头 | `claude-opus-4-7` | `--llm-api-key` → `agent.olium.llm_api_key` → `$ANTHROPIC_API_KEY` |
| `anthropic-oauth` | Bearer token（Claude Code OAuth） | `claude-opus-4-7` | `--oauth-token` → `agent.olium.oauth_token` → `$ANTHROPIC_API_KEY`（由 `claude setup-token` 生成） |
| `openai-api-key` | `x-api-key` 标头 | `gpt-5.5` | `--llm-api-key` → `agent.olium.llm_api_key` → `$OPENAI_API_KEY` |
| `openai-responses` | `x-api-key` 标头 | `gpt-5.5` | `--llm-api-key` → `agent.olium.llm_api_key` → `$OPENAI_API_KEY`；使用公共 OpenAI Responses API（`/v1/responses`）而不是 Chat Completions |
| `anthropic-cli` | （无，子进程） | `claude-opus-4-7` | `--claude-bin`（默认 `claude` 在 `$PATH` 上）；别名 `anthropic-claude-cli` |
| `anthropic-claude-sdk-bridge` | （无，订阅） | Claude Code 默认 | 通过 `vigolium-audit` 桥接 sidecar 使用您登录的订阅驱动 Claude Code；`--bridge-bin` → `agent.olium.bridge_binary` → 嵌入式 blob → `vigolium-audit` 在 `$PATH` 上 |
| `anthropic-compatible` | 可选的 Bearer | 通过 `custom_provider.model_id` | `custom_provider.base_url`（Anthropic Messages `/v1/messages` 网关或代理）、`custom_provider.api_key`、`custom_provider.model_id`、`custom_provider.extra_headers` |
| `anthropic-vertex` | GCP SA JSON | `claude-opus-4-6` | `agent.olium.oauth_cred_path`（或 `$GOOGLE_APPLICATION_CREDENTIALS`）+ `google_cloud_project` / `google_cloud_location`；将 `claude-*` 模型路由到 `publishers/anthropic` |
| `google-vertex` | GCP SA JSON | `gemini-2.5-pro` | 与 `anthropic-vertex` 相同的 GCP 凭据；将 `gemini-*` 模型路由到 `publishers/google` |
| `openai-compatible`（默认） | 可选的 Bearer | `gemma4:latest`（Ollama） | `custom_provider.base_url`（必需）、`custom_provider.api_key`（可选）、`custom_provider.model_id`、`custom_provider.extra_headers`（Ollama / OpenRouter / LM Studio / vLLM / Together / Groq / LocalAI / 自定义代理） |

如果没有 `--provider` 标志且没有 YAML 覆盖，Vigolium 默认为 `openai-compatible`，使用 `gemma4:latest` 针对本地 Ollama 端点，因此新初始化的配置无需任何云凭据即可开箱即用。`anthropic-oauth` Provider 还会在系统提示前添加 Claude Code 序言，并添加 `oauth-2025-04-20` beta 标头，以便在与 `anthropic-api-key` 相同的端点上被接受。

Codex 身份验证会自行刷新：它解析 JWT，检查过期时间（60 秒偏差），并使用存储的刷新令牌向 `/oauth/token` 发送 POST 请求，重写 `~/.codex/auth.json`（模式 0o600）。

注意：REST API 会回退到 `vigolium-configs.yaml` 中的 `agent.olium.*`（这会在请求之间保持热会话和提示缓存稳定），但每个 Agent 运行端点也接受每个请求的 BYOK 凭据（`api_key`、`oauth_token`、`oauth_cred_file`、`oauth_cred_json`）。审计分发器还接受 `audit_auth` / `piolium_auth` 用于每个驱动的覆盖。

## 工具

内置工具注册表，按此顺序注册八个工具：

| 名称 | 只读？ | 功能 |
|:-----|:-------|:-----|
| `bash` | 否 | `bash -lc <cmd>`，对灾难性模式（`rm -rf /`、对块设备的 `dd`、fork 炸弹、对真实设备的 `mkfs`）进行硬拒绝。默认超时 = 引擎 `ToolTimeout`（5 分钟）。 |
| `read_file` | 是 | 读取文件，带行号前缀。参数：`path`、`offset`、`limit`（默认 2000）。 |
| `write_file` | 否 | 创建或覆盖文件。 |
| `edit_file` | 否 | 对文件进行查找替换编辑。 |
| `ls` | 是 | 列出目录。 |
| `grep` | 是 | 正则表达式搜索，可用时使用 ripgrep，否则使用原生 Go 正则表达式。参数：`pattern`、`path`、`glob`、`max_matches`（200）、`ignore_case`。 |
| `glob` | 是 | Glob 模式 → 路径。 |
| `web_fetch` | 是 | 获取 URL。两种模式：`http`（默认，快速）和 `browser`（委托给 `agent-browser` 处理 SPA / JS 密集型页面）。参数：`url`、`method`、`headers`、`body`、`max_bytes`、`mode`、`wait_selector`、`wait_ms`。 |

`IsReadOnly()` 标志是引擎用来决定是否并行扇出一轮工具调用的依据。`bash` 无需批准提示即可运行（yolo 模式），只有灾难性模式防护可防止灾难。

### Autopilot 添加更多

当引擎在 `vigolium agent autopilot` 下运行时，注册表还会获得：

- `halt_scan`：模型驱动的退出。设置停止信号；运行循环在当前轮次后退出。
- `report_finding`：将漏洞发现持久化到数据库（标题、严重性、描述、修复、CWE、证据、置信度、状态）。50 次调用时软警告，200 次硬上限。
- `load_skill`：按名称获取技能主体（当技能注册表非空时注册）。
- Vigtool：`run_scan`、`run_extension`、`list_sessions`、`get_session`、`list_findings`、`list_auth_sessions`、`auth_session_lookup`（当 `Repo` 非 nil 时注册）。

## Skills

Skills 是带有 YAML 前置元数据的 Markdown 工作流文件，遵循 `agentskills.io` 约定，因此为 Claude Code 或 pi 编写的文件可以在 olium 中逐字使用。格式：

```yaml
---
name: triage-finding
description: Walk a candidate finding from suspicious response → root cause → PoC.
license: optional
allowed-tools: optional list
---

# Body
Instructional prose the model reads after calling load_skill.
```

`name` 必须匹配 `[a-z0-9-]+`（≤64 个字符）；`description` ≤1024 个字符。

### 发现

技能注册表遍历四个作用域，按名称首先找到的获胜：

- **项目**：工作目录及每个祖先目录中的 `.agent/skills/` 和 `.claude/skills/`，最近优先。
- **用户**：`~/.vigolium/skills/`（仅当 `IncludeUserSkills=true` 时）。
- **嵌入式**：通过 `go:embed` 在二进制文件中以 `public/presets/skills/` 形式提供。

接受两种磁盘布局：`<root>/<name>/SKILL.md`（目录技能，`agentskills.io` 标准）或 `<root>/<name>.md`（单文件简写；前置元数据名称必须与文件名主干匹配）。

通用聊天（`vigolium agent olium`、无头）仅加载作用域 1 和 3。Autopilot 和 Swarm 加载所有三个，以便 `~/.vigolium/skills/` 中的安全特定工作流不会污染随意聊天。

### 使用

引擎将 `<available_skills>` 块写入系统提示，列出每个技能的名称 + 描述 + 位置。模型通过 `load_skill` 工具按需获取主体，渐进式披露，因此未使用的技能不会消耗 token。

在 TUI 中，键入 `/skill:<name> [args]` 可直接将技能主体内联展开到您的提示中，无需工具调用。

## CLI 标志

```bash
--provider          openai-codex-oauth | anthropic-api-key | anthropic-oauth | openai-api-key | openai-responses | anthropic-cli | anthropic-claude-sdk-bridge | anthropic-compatible | anthropic-vertex | google-vertex | openai-compatible
--model             provider-specific (empty = provider default)
--oauth-cred        OAuth credential file (openai-codex-oauth; default ~/.codex/auth.json). Also the GCP SA JSON for the Vertex providers.
--oauth-token       Claude Code OAuth bearer token (anthropic-oauth)
--llm-api-key       API key for anthropic-api-key / openai-api-key / openai-responses
--claude-bin        Path to the `claude` binary (anthropic-cli)
--bridge-bin        Path to the vigolium-audit binary hosting the SDK bridge (anthropic-claude-sdk-bridge; empty = embedded blob, then PATH)
--base-url          Override custom_provider.base_url (openai-compatible / anthropic-compatible)
--system            Override the built-in system prompt
-p, --prompt        Initial prompt (alternative to a positional arg). Forces non-interactive mode.
--stdin             Force reading the prompt from stdin
```

初始提示的优先级：位置参数 → `-p/--prompt` → stdin（通过管道输入时自动检测，或使用 `--stdin` 强制）。值流为 CLI → YAML → env：每个 CLI 标志都会回退到其 `agent.olium.*` YAML 字段，该字段又会回退到文档化的默认值或环境变量。

## 设置

完整的 `agent.olium` 块：

```yaml
agent:
  olium:
    provider: openai-compatible    # openai-codex-oauth | anthropic-api-key | anthropic-oauth | openai-api-key | openai-responses | anthropic-cli | anthropic-claude-sdk-bridge | anthropic-compatible | anthropic-vertex | google-vertex | openai-compatible
    model: gemma4:latest           # empty = provider default
    oauth_cred_path: ~/.codex/auth.json  # also GCP SA JSON for anthropic-vertex / google-vertex
    bridge_binary: ""              # anthropic-claude-sdk-bridge; empty = embedded blob, then PATH
    oauth_token: ""                # anthropic-oauth; supports ${ENV_VAR}; falls back to $ANTHROPIC_API_KEY
    llm_api_key: ""                # supports ${ENV_VAR}; falls back to $ANTHROPIC_API_KEY / $OPENAI_API_KEY
    google_cloud_project: ""       # Vertex providers; $GOOGLE_CLOUD_PROJECT wins
    google_cloud_location: ""      # Vertex providers; default us-central1
    reasoning_effort: medium       # minimal|low|medium|high|xhigh (codex)
    system_prompt: ""              # empty = built-in olium prompt
    custom_provider:               # used when provider == openai-compatible or anthropic-compatible
      base_url: http://localhost:11434/v1   # full chat-completions URL also works
      model_id: gemma4:latest               # fallback for olium.model
      api_key: ""                           # optional
      extra_headers: []                     # curl-style "Key: Value" entries
    max_tokens: 1000000
    temperature: 0.0
    max_turns: 32                  # short non-autopilot uses; autopilot uses its own cap (DefaultAutopilotMaxTurns=200)
    cache_size: 1024               # LRU; 0 disables
    max_concurrent: 4              # global cap on simultaneous provider calls; 0 = unbounded
    call_timeout_sec: 600          # per-call deadline; negative = no timeout (parent ctx only)
```

值得了解的相邻配置块：

- `agent.sessions_dir`：每次运行的会话目录所在位置。默认 `~/.vigolium/agent-sessions/`。
- `agent.browser`：切换 `agent-browser` 集成（`web_fetch` 在 `mode: browser` 中 shell 出的二进制文件）。
- `agent.audit`：控制可选的 `vigolium-audit` / `piolium` 准备步骤，Autopilot/Swarm 可以在 olium 循环之前堆叠该步骤。

## 会话和磁盘状态

每个 Agent 运行都会在 `agent.sessions_dir`（默认 `~/.vigolium/agent-sessions/<run-uuid>/`）下获得一个会话目录。裸 `vigolium agent olium` 聊天不会写入会话，只有 Autopilot/Swarm/Query 会具体化一个。

在会话目录中，您可能会找到：

- `runtime.log`：每轮事件日志（文本增量、工具开始/结束、轮次完成摘要）。
- `tool-results/<tool>-<call-id>.txt`：溢出的过大工具输出（当引擎的 `SpillDir` 已设置时）。
- `session-config.json`：运行元数据（项目/扫描 UUID、选项）。
- `swarm-plan.json`、`master-output.md`、`audit-stream.jsonl`、`checkpoint.json`，由包装 olium 的更高级模式（Swarm、`vigolium-audit`、Autopilot）生成。

使用 `vigolium agent session list` / `--full` / `--tail` 浏览过去的运行。

## 流事件

引擎发出统一的 `Event` 通道，与 Provider 无关：

| Event | 携带内容 |
|:------|:---------|
| `EventTextDelta` | Delta，助手文本增量 |
| `EventThinkingDelta` | Delta，推理内容（Anthropic thinking，codex reasoning） |
| `EventToolCallStart` | ToolName，ToolArgs，模型决定调用工具 |
| `EventToolExecStart` / `EventToolExecProgress` / `EventToolExecEnd` | 工具调用生命周期，ToolResult，ToolIsErr |
| `EventTurnDone` | StopReason，Usage（input / output / cache-read / cache-write tokens） |
| `EventRunDone` | 最终使用量 |
| `EventError` | Err，Provider 失败，ctx 取消，超过最大轮次 |

`EventTurnDone` 上的 token 计数由每个更高级的调用者累积（Autopilot 用于预算执行，适配器用于 `agenttypes.TokenUsage`，Swarm 用于成本报告）。

## 何时使用什么

| 您想要… | 使用 |
|:---------|:-----|
| 交互式聊天/调试/探索 | `vigolium ol` |
| 从脚本运行一个提示并解析 stdout | `vigolium ol -p "..."` |
| 将方向盘交给 Agent 进行自主渗透测试 | `vigolium agent autopilot`（底层使用 olium，带有预算 + `report_finding`） |
| AI 指导原生扫描器（计划 → 模块 → 分类） | `vigolium agent swarm` |
| 一次性模板驱动提示，带结构化输出 | `vigolium agent query` |

Olium 本身是通用聊天/开发界面以及每个其他模式重用的引擎，它本身不是安全扫描。

## 另请参阅

- Agent Mode，完整的 Agent 子命令映射。
- Autopilot，基于 olium 引擎构建的自主扫描模式。
- Swarm，AI 引导的多阶段扫描，驱动原生扫描器。
- How It Works，Provider 列表和高级分发故事。