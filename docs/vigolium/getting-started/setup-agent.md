# Setting Up the Agent

Vigolium 的 AI 功能（autopilot、swarm、源代码审计、查询）都通过一个进程内运行时 **olium** 运行。olium 与供应商（Provider）（Claude / OpenAI / 本地模型）通信，两个专用驱动——**audit** 和 **piolium**——在其之上运行，用于源代码审计。

本页将引导你完成每个组件的配置。选择与你设置匹配的部分；你不需要全部配置。

| 内容 | 章节 | 何时需要 |
|---|---|---|
| olium 供应商 | [Olium agent](#1-olium-agent-the-engine-everything-runs-on) | 始终需要——每个 Agent 命令都需要一个供应商。 |
| Codex（OpenAI OAuth） | [Codex](#2-codex-cheapest-with-a-chatgpt-subscription) | 你拥有 ChatGPT Plus/Pro/Team 订阅。 |
| 本地模型（Ollama 等） | [Local / OpenAI-compatible](#3-local-models-ollama-openrouter-lm-studio-) | 你想离线运行 Agent，或使用 OpenRouter / vLLM / LM Studio。 |
| Claude | [Claude](#4-claude-anthropic) | 你有 Anthropic API 密钥、Claude 订阅，或已安装 `claude` CLI（不推荐——见章节说明）。 |
| Vigolium audit | [Vigolium audit](#5-vigolium-audit-source-code-driver) | 你想要白盒源代码审计，无需额外安装。 |
| Piolium audit | [Piolium audit](#6-piolium-audit-pi-native-driver) | 你想要 piolium 的 17 阶段 Pi 原生审计（需单独安装）。 |

所有设置存放在 `~/.vigolium/vigolium-configs.yaml` 中。你可以直接编辑它，或使用 `vigolium config set <key> <value>`。

---

## 1. Olium agent — the engine everything runs on

olium 是进程内 Agent 运行时（`pkg/olium/`），支撑所有 `vigolium agent …` 子命令。配置它意味着选择一个供应商并提供凭据。

支持的供应商：

| 供应商 | 认证方式 | 默认模型 | 说明 |
|---|---|---|---|
| `openai-codex-oauth` *（默认）* | `~/.codex/auth.json`（来自 `codex login`） | `gpt-5.5` | 拥有 ChatGPT 订阅时最便宜。 |
| `anthropic-api-key` | `$ANTHROPIC_API_KEY` | `claude-opus-4-7` | 直接 Anthropic API 计费。 |
| `anthropic-oauth` | `claude setup-token` bearer | `claude-opus-4-7` | 使用你的 Claude Pro/Max 套餐。 |
| `openai-api-key` | `$OPENAI_API_KEY` | `gpt-5.5` | 直接 OpenAI API 计费。 |
| `anthropic-cli` | `$PATH` 上的 `claude` 二进制文件 | `claude-opus-4-7` | 调用 Claude Code（别名：`anthropic-claude-cli`）。 |
| `anthropic-claude-sdk-bridge` | Claude Code 订阅（无需密钥） | bridge 默认值 | 通过 Agent SDK 的 Claude Code（`vigolium-audit bridge` sidecar）。 |
| `anthropic-vertex` | GCP 服务账号 JSON | `claude-opus-4-6` | Vertex AI 上的 Claude。 |
| `google-vertex` | GCP 服务账号 JSON | `gemini-2.5-pro` | Vertex AI 上的 Gemini。 |
| `openai-compatible` | 可选 `api_key` | 无——自行选择 | Ollama、OpenRouter、LM Studio、vLLM…… |

使用以下命令验证任何设置：

```bash
vigolium ol -p 'what model are you running'
```

如果返回了模型名称，则供应商配置正确。此后，`vigolium agent autopilot`、`vigolium agent swarm` 等命令均可正常工作。

---

## 2. Codex — cheapest with a ChatGPT subscription (Recommended)

如果你已在使用 OpenAI 的 **Codex CLI**，vigolium 会复用相同的 OAuth 凭据文件。无需 API 密钥，刷新自动处理。

```bash
# 1. 安装 Codex CLI（一次性）并通过 `codex login` 登录
codex exec 'hello'             # 验证——应输出一个模型名称

# 2. 将 vigolium 绑定到它（默认值已匹配；此处仅为显式声明）
vigolium config set agent.olium.provider openai-codex-oauth
vigolium config set agent.olium.oauth_cred_path ~/.codex/auth.json
vigolium config set agent.olium.model gpt-5.5

# 3. 验证
vigolium ol -p 'what model are you running'
```

`~/.codex/auth.json` 在每次运行时读取；JWT 过期时会自动刷新，因此你无需重新登录。

---

## 3. Local models (Ollama, OpenRouter, LM Studio, …)

`openai-compatible` 供应商可与任何支持 OpenAI Chat Completions 线格式的后端通信。在 `agent.olium.custom_provider` 下进行配置。

### Ollama (local, no key)

```bash
ollama pull gemma4:latest
ollama serve   # 如果尚未运行

vigolium config set agent.olium.provider openai-compatible
vigolium config set agent.olium.custom_provider.base_url http://localhost:11434/v1
vigolium config set agent.olium.custom_provider.model_id gemma4:latest

vigolium ol -p 'what model are you running'
```

空的 `api_key` 表示不发送 `Authorization` 请求头——Ollama 需要此设置。

### OpenRouter

```bash
export OPENROUTER_API_KEY=sk-or-…

vigolium config set agent.olium.provider openai-compatible
vigolium config set agent.olium.custom_provider.base_url https://openrouter.ai/api/v1
vigolium config set agent.olium.custom_provider.model_id anthropic/claude-sonnet-4.6
vigolium config set agent.olium.custom_provider.api_key '${OPENROUTER_API_KEY}'

# 可选：OpenRouter 排名信号（在排行榜上显示你的应用）
vigolium config set agent.olium.custom_provider.extra_headers.add 'HTTP-Referer: https://your-site.example'
vigolium config set agent.olium.custom_provider.extra_headers.add 'X-Title: vigolium'
```

#### Provider routing (OpenRouter)

通过类型化的 `provider_routing` 块固定或限制上游供应商：

```bash
vigolium config set agent.olium.custom_provider.provider_routing.order.add deepseek
vigolium config set agent.olium.custom_provider.provider_routing.allow_fallbacks false
vigolium config set agent.olium.custom_provider.provider_routing.sort throughput
```

或作为 YAML：

```yaml
agent:
  olium:
    custom_provider:
      provider_routing:
        order: [deepseek]
        allow_fallbacks: false
        sort: throughput
```

支持的字段：`order`、`only`、`ignore`、`allow_fallbacks`（OpenRouter 上默认为 `true`）、`sort`（`price` / `throughput` / `latency`）、`quantizations`、`data_collection`（`allow` / `deny`）、`require_parameters`、`zdr`。字段语义请参阅 [OpenRouter 的供应商路由文档](https://openrouter.ai/docs/features/provider-routing)。

对于类型化旋钮未覆盖的字段（`max_price`、`preferred_min_throughput`、`preferred_max_latency`）或其他 openai-compatible 后端扩展，请使用 `extra_body`（仅 YAML——`config set` 不遍历任意嵌套映射）：

```yaml
agent:
  olium:
    custom_provider:
      extra_body:
        provider:
          max_price: { prompt: 0.0001, completion: 0.0002 }
        transforms: [middle-out]
```

键 `model`、`messages`、`tools`、`stream`、`stream_options` 为保留字段，在请求时会被拒绝。同时设置 `provider_routing` 和 `extra_body.provider` 也会被拒绝。

### LM Studio

```bash
vigolium config set agent.olium.provider openai-compatible
vigolium config set agent.olium.custom_provider.base_url http://localhost:1234/v1
vigolium config set agent.olium.custom_provider.model_id <model-id-from-lm-studio>
```

### Custom headers (auth, routing, observability)

某些 OpenAI-compatible 后端需要额外的请求头——非 `Bearer` 认证方案、租户/路由信号、用于成本分析的请求标记等。`extra_headers` 接受一个 curl 风格 `"Key: Value"` 条目列表，这些条目在**标准请求头之后**应用，因此必要时可以覆盖 `Authorization`。

```bash
# 先清除，再添加。每个 .add 向列表追加一个请求头
vigolium config set agent.olium.custom_provider.extra_headers.clear ""
vigolium config set agent.olium.custom_provider.extra_headers.add 'X-Custom-ID: your-cli'
vigolium config set agent.olium.custom_provider.extra_headers.add 'Authorization: Bearer ***'
```

或直接编辑 `~/.vigolium/vigolium-configs.yaml`：

```yaml
agent:
  olium:
    custom_provider:
      extra_headers:
        - "X-Custom-ID: your-cli"
        - "Authorization: Bearer ***"   # 覆盖默认的 Bearer api_key
```

说明：

- `${VAR}` 引用在配置加载时从环境变量展开，因此凭据无需检入文件。
- 重复键时**最后**一个条目生效（匹配 `http.Header.Set` 语义）。
- 格式错误的条目（无 `:`）会在 warn 级别记录并跳过——Agent 继续运行。
- 要替换整个列表，先运行 `.clear ""`，然后 `.add` 每个条目。

你也可以在不修改配置的情况下，将这些作为一次性覆盖传递：

```bash
vigolium ol \
  --provider openai-compatible \
  --base-url http://localhost:11434/v1 \
  --model gemma4:latest \
  -p 'hello'
```

> `extra_headers` 没有 CLI 标志——在 YAML 中设置一次（或通过 `config set ... .add`），它会在各次运行间保持。

> **工具调用注意事项。** OpenAI 风格的函数工具是线格式的一部分，但只有部分模型实际会发出工具调用。`gemma4`、`qwen2.5-coder`、`llama3.1-instruct` 和 `mistral-nemo` 表现良好。较小的模型通常会忽略工具定义并以文字回复——如果 Agent 从不调用工具，请更换模型。

---

## 4. Claude (Anthropic)

> **不推荐用于 olium。** Anthropic 的 Pro/Max 订阅并非设计用于官方 Claude Code 客户端之外——从 vigolium（或任何第三方 Agent）驱动同一个令牌几乎会立即触发速率限制/超额计费，而 API 密钥路径按令牌计费，费率是本页列出的任何供应商中最高的。日常 Agent 工作请优先选择 Codex（[第 2 节](#2-codex--cheapest-with-a-chatgpt-subscription-recommended)）或本地模型（[第 3 节](#3-local-models-ollama-openrouter-lm-studio-)）。以下 Claude 选项的存在是为了兼容性以及为那些已经为 API 付费的用户提供。

三个选项，按偏好顺序排列：

### 4a. Claude OAuth (Claude Pro/Max subscribers)

`claude setup-token` 生成一个与你的 Claude 订阅绑定的 OAuth bearer 令牌。无需按令牌计费。

```bash
# 1. 安装 Claude Code，然后生成令牌
claude setup-token                                 # 打印 «redacted:sk-…»…
export ANTHROPIC_API_KEY=«redacted:sk-…»<your-token> # shell rc；重启后仍有效

# 2. 将 vigolium 指向 OAuth 供应商
vigolium config set agent.olium.provider anthropic-oauth
vigolium config set agent.olium.model claude-opus-4-7

# 3. 验证
vigolium ol -p 'what model are you running'
```

`anthropic-oauth` 首先读取 `agent.olium.oauth_token`，然后回退到 `$ANTHROPIC_API_KEY`。环境变量是最省事的方式。

> **注意——在你的 Claude 账户上启用额外用量。** Pro/Max 订阅的 OAuth 令牌上限为应用内 Claude Code 配额。从 vigolium（或任何第三方客户端）驱动同一个令牌会直接命中 Messages API，在你在 Anthropic Console（设置 → 计费 → 用量限制）中开启**额外用量 / 按需付费超额**之前，会被拒绝并返回 `429 rate_limit_error`。不开启此开关，即使令牌有效，上述验证调用也会失败。

### 4b. Anthropic API key

适用于通过标准 Anthropic API 计费的用户。

```bash
export ANTHROPIC_API_KEY=«redacted:sk-…»<your-key>

vigolium config set agent.olium.provider anthropic-api-key
vigolium config set agent.olium.model claude-opus-4-7

vigolium ol -p 'what model are you running'
```

### 4c. Anthropic CLI (`claude` shell-out)

如果你希望 vigolium 委托给 `$PATH` 上的 `claude` 二进制文件（从而使用 `claude` 本身配置的任何认证方式）：

```bash
which claude   # 必须能解析

vigolium config set agent.olium.provider anthropic-cli
vigolium config set agent.olium.model claude-opus-4-7
```

此模式比 API 密钥/OAuth 路径慢（子进程开销），但在你希望跨 `claude` 和 `vigolium` 使用单一认证源时很有用。

> **关于权限的说明。** vigolium 使用 `--permission-mode bypassPermissions` 调用 `claude -p`，因此 Bash / Read / WebFetch 工具调用无需交互式批准即可执行（包装器是非交互式的——没有 TTY 供你确认提示）。这相当于运行 `claude --dangerously-skip-permissions`，且仅在该子进程持续期间生效。

### 4d. Claude Code via the Agent SDK (`anthropic-claude-sdk-bridge`)

通过 **Agent SDK** 驱动 Claude Code（而非简单的 `claude -p` CLI），通过调用 `vigolium-audit bridge` sidecar 实现。与 `anthropic-cli` 一样，它使用你已登录的 Claude Code 订阅（无需密钥），但运行过程是受控的、可重现的 SDK 调用：它始终加载 `vigolium-scanner` 技能（因此 Agent 知道 `vigolium` CLI），并且**不会**拉取你的个人 `~/.claude` 配置或项目的 `CLAUDE.md`。

```bash
claude            # 登录一次

vigolium config set agent.olium.provider anthropic-claude-sdk-bridge
vigolium config set agent.olium.model opus   # 可选：opus | sonnet | 完整 ID；省略则使用 bridge 默认值

vigolium ol -p 'what model are you running'
```

承载 bridge 的 `vigolium-audit` 二进制文件**内嵌**在 vigolium 中——无需单独安装。使用 `vigolium config set agent.olium.bridge_binary /path/to/vigolium-audit` 或每次运行时的 `--bridge-bin` 标志覆盖。如果你在 `agent.olium.llm_api_key` / `oauth_token` 中有 API 密钥或 OAuth 令牌，则会转发给 bridge；否则使用环境订阅。

**何时选择此方案而非 `anthropic-cli`：** 当需要可移植、自包含的运行，在任何机器上行为一致（CI、容器），并预配置 vigolium 扫描器技能时，选择 bridge。当你希望完整的个人 Claude Code 环境——包括你的 `CLAUDE.md`、MCP 服务器和已安装的技能——应用于当前项目目录时，选择 `anthropic-cli`。

---

## 5. Vigolium Audit — source-code driver

`vigolium agent audit` 运行白盒源代码审计。驱动（Agent、命令、技能）**内嵌在 vigolium 二进制文件中**——无需额外安装。它在底层驱动 `claude` CLI，因此你需要一个来自[第 4 节](#4-claude-anthropic)的正常工作的 Claude 设置。

```bash
# 1. 确保 `claude` 已安装并已验证身份
claude --version
claude -p 'hello'   # 验证

# 2. 运行审计
vigolium agent audit --source ~/src/your-app

# 3. 或者将其接入 autopilot/swarm，以便在设置 --source 时自动运行
vigolium config set agent.audit.enable true
vigolium config set agent.audit.mode lite          # lite | balanced | deep
vigolium agent autopilot -t https://example.com --source ~/src/your-app
```

审计模式：`lite`（3 阶段，适合 CI）、`balanced`（9 阶段，`--audit=balanced` 的默认值）、`deep`（12 阶段，完整审计）。所有模式产生的漏洞发现使用与本地扫描器输出相同的解析器/模式，并导入到 vigolium 数据库中。

漏洞发现存放在 `~/.vigolium/agent-sessions/<scan-uuid>/vigolium-results/` 下。完整参考请参阅 [`docs/agentic-scan/vigolium-audit.md`](../agentic-scan/vigolium-audit.md)。

---

## 6. Piolium audit — Pi-native driver

`vigolium agent audit --driver=piolium` 通过 **Pi coding-agent 运行时**运行一个独立的、更彻底的审计（`deep` 模式下 17 个阶段）。与 audit 不同，piolium **不是**内嵌的——你需要安装一次，vigolium 驱动 `pi` 二进制文件。

```bash
# 1. 安装 Pi 运行时
bun install -g @earendil-works/pi-coding-agent
pi --version

# 2. 安装 piolium 插件
pi install git:git@github.com:vigolium/piolium.git
pi list                                # 验证 "piolium" 已出现

# 3. 配置 pi 的默认供应商（审计子进程使用 pi 自身的认证，而非 vigolium 的）。以 Anthropic 为例：
pi login                               # 或：pi /login

# 4. 运行审计
vigolium agent audit --driver=piolium --source ~/src/your-app                            # balanced（默认）
vigolium agent audit --driver=piolium --source ~/src/your-app --mode lite               # 快速分类
vigolium agent audit --driver=piolium --source ~/src/your-app --intensity deep          # 完整 17 阶段

# 5. 如果愿意，可为此运行覆盖 pi 的供应商/模型
vigolium agent audit --driver=piolium --source ~/src/your-app \
  --pi-provider vertex-anthropic --pi-model claude-opus-4-6
```

Vigolium 在审计前会对 pi 进行一次单轮预检，以尽早捕获认证/配额错误。如果预检失败，你将看到上游错误（例如 `No API key found for google-vertex. Use /login to log into a provider`），审计不会启动。

默认情况下，vigolium 使用 pi 的按用户安装路径 `~/.pi/agent`。要改用系统级安装，请导出 `PIOLIUM_HOME=/opt/piolium`（或任何其他路径）。模式、强度预设和完整标志参考请参阅 [`docs/agentic-scan/piolium-audit.md`](../agentic-scan/piolium-audit.md)。

### audit vs piolium

| | Audit | Piolium |
|---|---|---|
| 安装 | 内嵌——零设置 | 需要 `pi` + `pi install …` |
| 驱动 | `claude` CLI | `pi --mode json -p /piolium-<mode>` |
| 模式 | lite（3）、balanced（6）、deep（11） | lite（4）、balanced（9）、deep（17）、revisit、confirm、merge、diff、longshot |
| 供应商 | `claude` 配置的任何供应商 | `pi` 配置的任何供应商（与 olium 分开） |
| 适用场景 | "我想要源代码审计，无需额外设置" | "我想要最彻底的审计" |

你也可以使用 `vigolium agent audit --driver both --source …` 并行运行两者——这会在单个父扫描下依次调度 audit 和 piolium，并进行项目级去重。

---

## 7. Verifying the full stack

在完成任意章节的设置后，按顺序运行以下命令。每个命令在缺少组件时都会快速失败并显示有用的错误信息：

```bash
# Olium：一次提示，一次供应商调用。无需数据库，无需扫描
vigolium ol -p 'hello'

# Agent 查询：引擎用于源代码审查的相同路径
vigolium agent query -p 'list every route in this repo' --source .

# Autopilot 冒烟测试（仅目标，无源代码）：
vigolium agent autopilot -t https://example.com --intensity quick --max-duration 5m

# Vigolium audit（需要已安装 claude）：
vigolium agent audit --source . --mode lite

# Piolium audit（需要已安装 pi + piolium）：
vigolium agent audit --driver=piolium --source . --mode lite
```

如果其中任何一个报错，错误信息会指向缺失的组件——通常是未设置的环境变量、错误的 `agent.olium.provider` 或缺失的二进制文件。

---

## Where to go next

- [`docs/agentic-scan/olium-agent.md`](../agentic-scan/olium-agent.md) —— olium 是什么以及它的工具做什么
- [`docs/agentic-scan/autopilot.md`](../agentic-scan/autopilot.md) —— 自主扫描
- [`docs/agentic-scan/swarm.md`](../agentic-scan/swarm.md) —— 引导式多阶段扫描
- [`docs/agentic-scan/vigolium-audit.md`](../agentic-scan/vigolium-audit.md) —— audit 参考
- [`docs/agentic-scan/piolium-audit.md`](../agentic-scan/piolium-audit.md) —— piolium 参考
- [`public/vigolium-configs.example.yaml`](../../public/vigolium-configs.example.yaml) —— 所有配置项及内联文档
