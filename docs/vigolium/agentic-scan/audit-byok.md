# Audit BYOK（自带密钥）

`vigolium agent audit` 和 `POST /api/agent/run/audit` 接受每次运行的凭据，这样单个 Vigolium 安装可以使用每个操作员自己的 Anthropic / OpenAI 密钥驱动审计，而无需将其硬编码到 `agent.olium.*` 配置中。覆盖适用于请求上实际运行的任何驱动 — **audit** 和 **piolium** 都使用相同的标志/字段，并在内部路由到正确的位置。

## 目录

- [何时使用](#何时使用)
- [快速入门](#快速入门)
- [工作原理](#工作原理)
- [CLI](#cli)
  - [标志参考](#标志参考)
  - [间接引用：`$ENV` 和 `@path`](#间接引用env-和-path)
  - [示例](#示例)
- [REST API](#rest-api)
  - [请求字段](#请求字段)
  - [示例](#rest-示例)
- [供应商选择](#供应商选择)
- [驱动特定行为](#驱动特定行为)
  - [Audit](#audit)
  - [Piolium](#piolium)
- [验证规则](#验证规则)
- [日志记录与脱敏](#日志记录与脱敏)
- [操作注意事项](#操作注意事项)

---

## 何时使用

- Vigolium 服务器由多个操作员共享，您希望每次运行的费用计入正确的账户。
- CI 管道提供短期的 OAuth 令牌，不应在作业结束后继续存在。
- 您想用与 `agent.olium.provider` 固定的不同供应商测试审计运行，而无需编辑配置。
- 您正在审计客户代码，合同要求凭据绝不能以标准 `agent.olium.llm_api_key` 字段形式存放在磁盘上。

如果以上都不适用，请将新标志留空 — 审计驱动从 `agent.olium.*` 继承凭据，与 BYOK 出现之前完全相同。

---

## 快速入门

```bash
# Anthropic API 密钥，claude 端（默认 agent）
vigolium agent audit --source ./src --intensity balanced \
  --api-key '$ANTHROPIC_API_KEY'

# Claude OAuth 令牌（由 `claude setup-token` 生成）
vigolium agent audit --source ./src --intensity balanced \
  --oauth-token '$CLAUDE_CODE_OAUTH_TOKEN'

# Codex 使用一次性凭据文件（~/.codex/auth.json 格式）
vigolium agent audit --source ./src --intensity balanced \
  --provider openai-codex-oauth \
  --oauth-cred-file ./codex-auth.json
```

```bash
# 通过 REST 的相同调用（字面值 — REST 不支持 $ENV / @path）
curl -s -X POST http://localhost:9002/api/agent/run/audit \
  -H "Content-Type: application/json" \
  -d '{
    "source": "/home/user/src/my-app",
    "intensity": "balanced",
    "api_key": "«redacted:sk-…»"
  }' | jq .
```

---

## 工作原理

单个 `AuthOverride` 包（`api_key` **或** `oauth_token` **或** `oauth_cred_file`，加上解析后的 `agent` 为 `claude`/`codex`）流经审计分发器，并由每个驱动以不同方式消费：

| 步骤 | 发生了什么 |
|---|---|
| 1. 入口点 | CLI 标志或 REST 主体字段填充覆盖。CLI 在此点解析 `$ENV` / `@path` 间接引用 — REST 不解析。 |
| 2. Agent 解析 | `claude` 与 `codex` 的选择，优先级从高到低：`--agent`（CLI）/ `agent` 字段（REST）> `--provider`（CLI）> `agent.audit.default_agent` 配置 > `agent.olium.provider` 派生 > claude。这仅选择 agent；身份验证是覆盖/供应商派生的包。 |
| 3. 验证 | 集中规则：三个凭据字段中至多一个；`--oauth-token` 需要 claude 端。解析后的 agent 也会与身份验证进行校验，因此选择器驱动的切换（例如 `default_agent: codex`）不能静默地将 codex 与仅 claude 的令牌配对。失败在任何子进程工作之前停止请求。 |
| 4a. Audit 路径 | 解析后的覆盖**替换** olium 派生的审计身份验证，并作为 `--api-key` / `--oauth-token` / `--oauth-cred-file` 标志传递给 `audit`。 |
| 4b. Piolium 路径 | 覆盖成为 `pi` 子进程的环境变量（`ANTHROPIC_API_KEY`、`OPENAI_API_KEY`、`CLAUDE_CODE_OAUTH_TOKEN`）— `pi` 本身没有等效的 CLI 标志。对于 codex + `--oauth-cred-file`，Vigolium 在运行期间将文件暂存到 `<pi-agent-dir>/auth.json`，并执行备份和恢复。 |
| 5. 清理 | 进程退出时，暂存的 `auth.json` 被移除，任何预先存在的文件被恢复。清理也会在 `cmd.Start()` 失败时触发。 |

用于生成子进程的实际 `cmd.Env` 始终携带真实值；覆盖仅在出现在日志或打印的命令行中时被脱敏（参见[日志记录与脱敏](#日志记录与脱敏)）。

### Agent 选择 vs. 身份验证（`--agent` / `agent.audit.default_agent`）

**身份验证**（转发哪个凭据）和 **agent**（claude vs codex）是独立解析的。BYOK 仅设置身份验证；它从不更改 agent。审计阶段的 agent 按以下优先级解析：`--agent` 标志 > `--provider` 标志 > `agent.audit.default_agent` 配置 > `agent.olium.provider` 派生 > claude。`--agent` 和 `default_agent` 是**纯 agent 选择器** — 它们切换 agent 同时保留解析后的身份验证，与文档化的 `--agent` 契约完全相同。因此，无论是否设置 `default_agent`，您的 BYOK 凭据都会被相同地转发。

因为选择器和身份验证是独立的，您可以固定一个同时 BYOK 另一个：例如，使用 `default_agent: codex`，`--api-key sk-ant-…` 会将 Anthropic 密钥转发给 codex（不匹配）。Vigolium 捕获它能证明不可能的组合 — 仅 claude 的 `--oauth-token`（或 `anthropic-oauth` 供应商）解析为 codex agent — 并在任何子进程启动之前以 `audit agent/auth mismatch: …` 拒绝（REST 返回 `400`）。API 密钥在此层是 agent 无关的，因此错误供应商的密钥由 `vigolium-audit` 本身发现。

---

## CLI

### 标志参考

| 标志 | 描述 |
|---|---|
| `--api-key <value\|$ENV\|@path>` | 本次运行的 API 密钥。路由到 `ANTHROPIC_API_KEY`（claude）或 `OPENAI_API_KEY`（codex）。 |
| `--oauth-token <value\|$ENV\|@path>` | Anthropic OAuth 承载令牌。仅 Claude — 由 `claude setup-token` 生成。 |
| `--oauth-cred-file <path\|$ENV>` | OAuth 凭据文件（codex `~/.codex/auth.json` 格式，或任何从 JSON 文件读取的供应商）。对于 piolium 运行，暂存到 `<pi-agent-dir>/auth.json`。 |

所有三个标志都是可选的、互斥的，并适用于**调用中实际运行的任何驱动** — 没有单独的 `--driver=audit --api-key …` 形式。

### 间接引用：`$ENV` 和 `@path`

CLI 标志值接受三种形式：

| 形式 | 含义 |
|---|---|
| `sk-ant-…` | 字面值。在 shell 历史和 `ps` 中可见。 |
| `$ANTHROPIC_API_KEY` | 从指定的环境变量读取。如果变量未设置或为空则报错。 |
| `@./key.txt` | 从指定路径的文件读取。尾部空白和换行符被去除。如果文件缺失或为空则报错。 |

> 间接引用**仅限 CLI**。REST 端点将请求字段视为字面值 — 在服务器端从网络提供的字符串解析 `$ENV` 会让任何调用者探测服务器的进程环境。

### 示例

```bash
# 来自环境变量的 Anthropic API 密钥 — 推荐形式：避免密钥泄露到 ps / shell 历史中。
# 间接引用在审计子进程生成之前发生，因此解析后的值永远不会出现在 argv 上。
vigolium agent audit --source ./src --intensity balanced \
  --api-key '$ANTHROPIC_API_KEY'

# 来自 mode-0600 文件的 Anthropic API 密钥
echo -n '«redacted:sk-…»' > ~/.vigolium/keys/anthropic.txt
chmod 600 ~/.vigolium/keys/anthropic.txt
vigolium agent audit --source ./src \
  --api-key @~/.vigolium/keys/anthropic.txt

# Claude OAuth 令牌（仅本次运行一次性使用；不触及 vigolium-configs.yaml 中的 agent.olium.oauth_token）
vigolium agent audit --source ./src --mode deep \
  --oauth-token '$CLAUDE_CODE_OAUTH_TOKEN'

# Codex 运行 — 将 agent 固定为 codex 并提供凭据文件。
# 对于 piolium，vigolium 将文件暂存到 <pi-agent-dir>/auth.json，
# 备份任何现有文件，并在运行退出后恢复。
vigolium agent audit --source ./src --driver piolium \
  --provider openai-codex-oauth \
  --oauth-cred-file ./codex-auth.json

# Codex 运行 — driver=both 使用相同的凭据文件运行 audit 然后 piolium
vigolium agent audit --source ./src --driver both \
  --provider openai-codex-oauth \
  --oauth-cred-file ./codex-auth.json

# CI / 管道：来自运行器设置的环境变量的短期 API 密钥，
# 限制为最后 50 次提交的差异。
vigolium agent audit --source . --driver audit \
  --diff HEAD~50 \
  --api-key '$CI_PROVIDED_ANTHROPIC_KEY'
```

---

## REST API

`POST /api/agent/run/audit` 接受三个可选字段。统一驱动分发器将它们应用于实际运行的任何驱动。

### 请求字段

| 字段 | 类型 | 描述 |
|---|---|---|
| `api_key` | string | 本次运行的 API 密钥。视为字面值 — 无 `$ENV` / `@path` 解析。 |
| `oauth_token` | string | Anthropic OAuth 承载令牌。仅 Claude 端。 |
| `oauth_cred_file` | string | **服务器文件系统上**的凭据文件路径（codex `auth.json` 格式）。服务器在暂存时读取该文件 — 相对路径相对于服务器的工作目录解析。 |

`agent`（已属于请求体的一部分）在 audit 参与时选择 claude vs codex，并在设置 `api_key` / `oauth_token` 时决定 piolium 获得哪种环境变量风格。如果 `agent` 为空，服务器回退到 `agent.olium.provider`。

> **`oauth_cred_file` 从服务器的文件系统读取**，而非调用者的。预先暂存文件或通过您自己的机制在上传后再启动审计。未来版本可能接受内联 JSON；目前保持为路径，以便 codex 供应商可以像本地安装一样对其进行 mmap。

### REST 示例

```bash
# Anthropic API 密钥作为字面值 — REST 不支持 $ENV / @path
curl -s -X POST http://localhost:9002/api/agent/run/audit \
  -H "Content-Type: application/json" \
  -d '{
    "source": "/home/user/src/my-app",
    "intensity": "balanced",
    "api_key": "«redacted:sk-…»"
  }' | jq .

# Claude OAuth 令牌 — 仅 claude 端（验证器在任何子进程启动前拒绝 oauth_token 与 agent: "codex" 的组合）
curl -s -X POST http://localhost:9002/api/agent/run/audit \
  -H "Content-Type: application/json" \
  -d '{
    "source": "/home/user/src/my-app",
    "mode": "deep",
    "agent": "claude",
    "oauth_token": "«redacted:sk-…»..."
  }' | jq .

# Codex 使用服务器端凭据文件。driver=piolium 将其暂存到 <pi-agent-dir>/auth.json，带备份和恢复
curl -s -X POST http://localhost:9002/api/agent/run/audit \
  -H "Content-Type: application/json" \
  -d '{
    "source": "/home/user/src/my-app",
    "driver": "piolium",
    "agent": "codex",
    "oauth_cred_file": "/etc/vigolium/secrets/codex-auth.json"
  }' | jq .

# driver=both — 相同的覆盖分发给 audit（--api-key 标志）和 piolium（pi 子进程上的 ANTHROPIC_API_KEY 环境变量）
curl -s -X POST http://localhost:9002/api/agent/run/audit \
  -H "Content-Type: application/json" \
  -d '{
    "source": "git@github.com:org/repo.git",
    "intensity": "balanced",
    "driver": "both",
    "agent": "claude",
    "api_key": "«redacted:sk-…»"
  }' | jq .

# 按项目密钥，带 project_uuid 范围
curl -s -X POST http://localhost:9002/api/agent/run/audit \
  -H "Content-Type: application/json" \
  -d '{
    "source": "/home/user/src/my-app",
    "intensity": "balanced",
    "project_uuid": "11111111-2222-3333-4444-555555555555",
    "api_key": "«redacted:sk-…»"
  }' | jq .
```

---

## 供应商选择

凭据**类型**（`api_key` vs `oauth_token` vs `oauth_cred_file`）本身并不决定运行是走 claude 还是 codex。这来自现有的供应商选择字段：

| 选择器 | CLI | REST |
|---|---|---|
| 每次运行覆盖 | `--provider` | `agent` |
| 服务器回退 | `vigolium-configs.yaml` 中的 `agent.olium.provider` | 相同 |
| 默认 | `claude`（匹配 audit 自身的默认值） | 相同 |

| 供应商覆盖 → 解析后的 agent | 示例 |
|---|---|
| `anthropic-*` | `claude` |
| `google-*`（`google-vertex` 等） | `claude` |
| `openai-*` | `codex` |
| 空 | 继承 `agent.olium.provider`，然后 claude 默认 |

相同的解析驱动 audit 的 `--agent` 标志以及 piolium 获得哪个环境变量（`ANTHROPIC_API_KEY` vs `OPENAI_API_KEY`）。

---

## 驱动特定行为

### Audit

- 覆盖被**整体**折叠到 audit 的调用中，替换解析器从 `agent.olium.*` 派生的任何身份验证（来自不同身份验证流程的陈旧配置值不能意外地交叉连接到覆盖驱动的运行中）。
- audit-ts 直接接收 `--api-key` / `--oauth-token` / `--oauth-cred-file`。标志值在 audit 进程的生命周期内在实时 argv 中可见 — 参见[操作注意事项](#操作注意事项)。
- 无需暂存或环境变量注入。`--oauth-cred-file` 上的凭据文件路径由 audit 自身读取，如同任何本地调用一样。

### Piolium

`pi` 不接受身份验证标志。Vigolium 将覆盖转换为 pi 子进程上的环境变量注入：

| 覆盖 | 解析后的 agent | 注入到 `pi` 的环境变量 |
|---|---|---|
| `api_key` | claude | `ANTHROPIC_API_KEY=…` |
| `api_key` | codex | `OPENAI_API_KEY=…` |
| `oauth_token` | claude | `CLAUDE_CODE_OAUTH_TOKEN=…` |
| `oauth_cred_file` | codex | （文件暂存到 `<pi-agent-dir>/auth.json`；无环境变量） |

对于 codex `oauth_cred_file`，Vigolium：

1. 获取 `<pi-agent-dir>/.vigolium-auth.lock` 的排他锁（在争用时返回清晰的错误，而不是让两个 BYOK 运行交错交换）。
2. 如果 `<pi-agent-dir>/auth.json` 存在，将其重命名为 `auth.json.vigolium-bak-<run-uuid>`。
3. 将提供的凭据文件复制到 `<pi-agent-dir>/auth.json`（模式 `0600`）。
4. 运行审计。`pi` 的 codex 供应商从 `PI_CODING_AGENT_DIR` 读取 `auth.json`，与独立安装完全相同。
5. 移除暂存的文件，恢复备份（如果有），并释放锁 — 即使在 `cmd.Start()` 失败或子进程崩溃时也是如此。

`<pi-agent-dir>` 是 Vigolium 作为 `PI_CODING_AGENT_DIR` 传递给 `pi` 的目录。使用 `$PIOLIUM_HOME=/opt/piolium`（文档化的系统布局）时为 `/opt/piolium/agent`。使用每用户固定值（`$PIOLIUM_HOME=~/.piolium`）时为 `~/.piolium/agent`。用户默认的 `~/.piolium/` **不会**被自动探测 — 参见 [piolium-audit.md](piolium-audit.md) 了解解析链。

### Driver = both

`vigolium agent audit --driver both` 在一个父 AgenticScan 下先运行 audit 再运行 piolium。相同的 `AuthOverride` 应用于两者 — audit 作为标志，piolium 作为环境变量/暂存文件 — 因此单个 `--api-key` 覆盖整个运行。

---

## 验证规则

CLI 和 REST 都强制执行：

| 规则 | 错误 |
|---|---|
| `api_key` / `oauth_token` / `oauth_cred_file` 中至多一个可被设置。 | `auth override: at most one of api-key / oauth-token / oauth-cred-file may be set` |
| `oauth_token` 需要 claude 端。 | `auth override: --oauth-token is only valid for the claude agent (got "codex"); codex uses --oauth-cred-file` |
| CLI：`$VAR` 必须已设置且非空。 | `<flag>: $VAR is unset or empty` |
| CLI：`@path` 必须存在且在去除空白后非空。 | `<flag>: read <path>: …` / `<flag>: <path> is empty` |

验证在**任何子进程启动之前**运行。CLI 错误返回非零退出码和单行消息；REST 错误返回 `400 Bad Request`，消息在 JSON 体中。

---

## 日志记录与脱敏

通过 BYOK 提供的值是敏感的。Vigolium 在日志和打印的命令行中对它们进行脱敏，而传递给子进程的实时 `cmd.Env` 和 argv 始终携带真实值。

| 表面 | 脱敏 |
|---|---|
| `injected_env` zap 调试日志 | `ANTHROPIC_API_KEY`、`CLAUDE_CODE_OAUTH_TOKEN`、`OPENAI_API_KEY`、`OPENAI_OAUTH_CRED_PATH`、`GOOGLE_APPLICATION_CREDENTIALS` 的值被替换为 `<redacted>`。 |
| 打印的命令行（`starting background audit cmd=…`） | `--api-key`、`--oauth-token`、`--oauth-cred-file` 之后的值被替换为 `<redacted>`。 |
| 会话包（`runtime.log`、`audit-stream.jsonl`） | 继承相同的脱敏，因为它们从上述流中分流。**audit-ts 自身的日志不会被 Vigolium 脱敏** — 如果 audit 将密钥值打印到其 stdout，它将原样出现在 `runtime.log` 中。 |
| AgenticScan 数据库行 | 身份验证字段不会被持久化。 |

如果您在日志中搜索凭据但没有找到，那是脱敏器在工作。字面字符串 `<redacted>` 是可搜索的，因此您可以确认脱敏在给定运行中已触发。

---

## 操作注意事项

- **进程列表泄露（CLI audit）。** `--api-key sk-…` 在 audit 子进程的生命周期内会出现在 `ps` 中。通过 `--api-key '$ANTHROPIC_API_KEY'` 或 `@/secure/path` 缓解，这样字面值不在 argv 上。audit 目前不接受基于管道的传递。
- **REST 没有 `$ENV` / `@path` 间接引用。** 针对网络提供的字符串解析任一者会让调用者探测服务器的进程环境或文件系统。在发出请求之前使用密钥管理器或预先暂存文件（对于 `oauth_cred_file`）。
- **并发的 codex+piolium BYOK。** `<pi-agent-dir>/.vigolium-auth.lock` 处的锁序列化共享同一 pi-agent-dir 的运行。第二个运行会以清晰的消息报错，而不是覆盖第一个运行的 `auth.json`。如果 vigolium 进程在运行中途崩溃，锁和备份文件会保留 — 手动移除锁并在重新运行前检查 `auth.json.vigolium-bak-<run-uuid>`。
- **audit 自身的日志表面。** audit-ts 是一个独立的二进制文件，可能在其 NDJSON `result` 事件或 stderr 上的错误消息中暴露身份验证值。Vigolium 仅脱敏其拥有的流；如果您将 `runtime.log` 传出机器，请将其视为可能包含密钥，直到您的环境证明相反。
- **覆盖不会持久化到磁盘。** 覆盖永远不会进入 `vigolium-configs.yaml`、AgenticScan 数据库行或会话配置快照。稍后重新运行相同的审计需要重新提供 BYOK 标志/字段。
