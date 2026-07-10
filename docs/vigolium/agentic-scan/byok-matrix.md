# BYOK 参考矩阵

BYOK（自带密钥）允许操作员为每次运行提供 LLM 凭据，这样服务器范围的 `agent.olium.*` 配置就不必保存每个租户的密钥。本页是权威映射：哪些端点接受哪些字段，哪些标志/标头表面映射到哪个供应商，以及哪些内容在何处被编辑。

有关审计驱动 BYOK 路径的端到端演练，请参阅 [`audit-byok.md`](./audit-byok.md)。

## 端点 × 字段矩阵

所有 Agent 端点都接受模式部分描述的 BYOK 字段。"Driver" 指示凭据是进入子进程（audit / piolium）还是进程内 olium 引擎。

| 端点                          | 驱动               | BYOK 请求体字段 | Bearer 标头 | 每驱动覆盖 |
| ----------------------------- | ------------------ | --------------- | ----------- | ---------- |
| `POST /api/agent/run/query`       | 进程内 olium     | 是              | 否          | 不适用     |
| `POST /api/agent/run/autopilot`   | 进程内 olium     | 是              | 否          | 不适用     |
| `POST /api/agent/run/swarm`       | 进程内 olium     | 是              | 否          | 不适用     |
| `POST /api/agent/run/audit`      | 子进程 (audit)  | 是              | 否          | 不适用     |
| `POST /api/agent/run/audit`       | 子进程 (audit + piolium) | 是     | 否          | `audit_auth`, `piolium_auth` |
| `POST /api/agent/chat/completions`| 进程内 olium     | 是              | 是（仅无身份验证模式） | 不适用     |

autopilot 端点也可以在提供源路径时启动后台 vigolium-audit；该后台审计继承与进程内操作员 Agent 相同的 BYOK 覆盖。

## 模式

```jsonc
{
  // 四个字段中必须恰好有一个非空
  "api_key":         "sk-ant-…  or  sk-…",
  "oauth_token":     "sk-ant-oat01-…",            // Claude Code OAuth，仅 claude
  "oauth_cred_file": "/var/lib/.../codex.json",   // 已弃用 — 请参阅 oauth_cred_json
  "oauth_cred_json": "{\"tokens\":{\"id_token\":…,\"access_token\":…,\"refresh_token\":…,\"account_id\":…}, \"auth_mode\":\"codex\"}",

  // 仅审计端点 — 按驱动覆盖继承的字段。每个
  // 子对象接受上述相同的四个字段。仅当
  // driver=both 时有效；单驱动运行拒绝这些字段。
  "audit_auth":     { "api_key": "..." },
  "piolium_auth":    { "oauth_cred_json": "..." }
}
```

## 供应商自动检测

进程内 olium 路径根据凭据形状选择供应商，因此调用者无需发送 `provider` 字段：

| 字段集       | 模式                | 解析的供应商          |
| ------------ | ------------------- | ---------------------- |
| `oauth_cred_json` / `oauth_cred_file` | (Codex JSON)         | `openai-codex-oauth` (`OAuthCredPath`) |
| `oauth_token`   | `sk-ant-oat01-…`       | `anthropic-oauth` (`OAuthToken`)       |
| `api_key`       | `sk-ant-*`             | `anthropic-api-key` (`LLMAPIKey`)      |
| `api_key`       | 其他任何内容          | `openai-api-key` (`LLMAPIKey`)         |

供应商切换根据密钥形状进行验证：

- `openai-api-key` 拒绝以 `sk-ant-` 开头的密钥（Anthropic 形状）。
- `anthropic-api-key` 拒绝以 `sk-ant-oat` 开头的密钥（OAuth 形状）。
- `anthropic-oauth` 拒绝以 `sk-ant-api` 开头的密钥（API 密钥形状）。
- 任何在 YAML 加载后仍包含未展开的 `${VAR}` 的字段会快速失败，并提示"在 shell 中设置环境变量"。

子进程 (audit / piolium) 路径使用显式的 `agent` 字段（`claude` / `codex`）代替，并验证 `oauth_token` 仅在 claude 端使用。

## 间接性：CLI 与 REST

| 来源           | `$ENV` 展开  | `@/path` 文件读取 |
| -------------- | ------------ | ----------------- |
| YAML（加载时） | 是 (`ExpandEnvVars`) | 否               |
| CLI 标志       | 是（仅 CLI） | 是（仅 CLI）      |
| REST 请求体字段 | **仅字面量** | **仅字面量**      |

REST 按设计保持仅字面量——代表网络调用者解析 `$ENV` 或文件系统路径会让未经身份验证的客户端探测服务器的环境或文件系统。

## 标头 BYOK（仅 chat/completions）

`POST /api/agent/chat/completions` 遵循 `Authorization: Bearer <key>` 标头——每个 OpenAI SDK 都会发送。该 bearer 被提升为 `api_key` 并通过相同的覆盖路径路由，但**仅**当：

1. 服务器以 `--no-auth` 运行。启用身份验证时，bearer 是操作员的用户令牌，而不是 BYOK 密钥。
2. 请求体尚未携带 BYOK 字段。请求体字段优先。

对于所有其他端点，BYOK 必须来自 JSON 请求体。

## 每驱动覆盖（仅 `driver=both`）

使用 `driver: "both"` 的 `POST /api/agent/run/audit` 接受 `audit_auth` 和 `piolium_auth` 子对象，这些子对象仅覆盖一个驱动的顶层 BYOK。使用此功能可以用两个租户的凭据运行单个审计——例如，审计端使用一个操作员的 Claude OAuth，piolium 端使用另一个操作员的 Codex auth.json。每个子覆盖独立暂存和清理。

单驱动运行（`driver=audit` / `driver=piolium`）拒绝 `audit_auth` / `piolium_auth`，返回 400——请改为传递顶层字段。

## 生命周期和磁盘暂存

| 字段            | 运行时所在位置                                | 清理 |
| --------------- | --------------------------------------------- | ---- |
| `api_key`        | 供应商结构体字段（包装在格式化安全的 `secret` 中） | 引擎拆卸 |
| `oauth_token`    | 供应商结构体字段（包装在格式化安全的 `secret` 中） | 引擎拆卸 |
| `oauth_cred_file` | argv 标志 (audit) / 暂存文件 (piolium) | 每次运行清理 |
| `oauth_cred_json` | 位于 `<sessions_dir>/byok-creds/byok-<uuid>.json` 的 0600 文件 | 每次运行清理 |

piolium codex BYOK 还会交换 `<pi-agent-dir>/auth.json` 及其锁文件 (`.vigolium-auth.lock`)。Audit-ts 对 `.audit-auth.lock` 执行相同操作。两个锁文件都包含 JSON 面包屑（`run`、`pid`、`started_at`），以便操作员可以归因于过时的锁。

如果 vigolium 在运行中途被 SIGKILL，下一次审计启动时会运行清理：

- Go 端 (`SweepStalePioliumAuth`) — 如果持有者 PID 已死，则恢复匹配的 `auth.json.vigolium-bak-<uuid>`。
- TS 端 (`sweepStaleAuthBackups`) — 对 `.audit-backup-<uuid>` 执行相同操作。

操作员每个清理条目会看到一条日志行。

## 编辑

编辑堆栈在五个层运行——所有层都必须遗漏才能泄露秘密：

| 层                          | 擦除内容                                                                 |
| --------------------------- | ------------------------------------------------------------------------ |
| `DebugRequestMiddleware`       | JSON 请求体字段（`api_key`、`oauth_token`、`oauth_cred_file`、`oauth_cred_json`、`llm_api_key`、`password`、`secret`、环境形状的键）；敏感标头（`Authorization`、`Cookie`、`X-API-Key`、`X-Anthropic-Key`、`X-OpenAI-Key`、`Proxy-Authorization`） |
| `pkg/agent/audit_redact.go`    | 跟随 `--api-key` / `--oauth-token` / `--oauth-cred-file` 的 argv 令牌；按名称的环境变量（`ANTHROPIC_API_KEY`、`CLAUDE_CODE_OAUTH_TOKEN`、`OPENAI_API_KEY` 等） |
| `pkg/olium/provider/debug.go`  | `VIGOLIUM_OLIUM_DEBUG=1` stderr 转储 — 正则擦除 `sk-ant-…`、`sk-…`、`Bearer …`、`AIza…`、`gh[pousr]_…` |
| `pkg/olium/provider/secret`    | 对 `Secret` 包装的密钥使用 `%v`、`%+v`、`%#v`、`%s`、`%q` 会打印 `<redacted>` |
| `internal/config/flatconfig.go`| `vigolium config ls` 掩码其键匹配敏感后缀（`api_key`、`bot_token`、`webhook_url`、`password`）或包含敏感词（`key`、`token`、`secret`、`authorization`）的条目；还掩码 `*.webhook.*.url` 路径以捕获令牌化的钩子 URL |

Webhook URL 还会通过 `redactURL` 在 `*url.Error` 链中被擦除，这样令牌化的 Slack/Discord/Teams URL 不会通过包装的网络错误泄露。

## 配额

重型 Agent 运行（autopilot、swarm、audit）由两个信号量控制：

- `agent_heavy_max`（默认 5）——集群范围。
- `agent_heavy_per_project`（默认 2）——每个项目。设置为负数以禁用；设置为 0 则回退到默认值。

超过每个项目上限的请求会快速失败，返回 429（`项目 X 已有 N 个重型 Agent 运行正在进行`）。通过每个项目门但阻塞在集群门的请求在返回自己的 429 之前使用配置的 `agent_queue_timeout`。

轻型运行（query、chat/completions）共享 `agent_light_max`（默认 10），没有每个项目上限——它们成本低且寿命短。

## 示例

内联 Codex JSON 用于一次审计运行：

```bash
curl -X POST $HOST/api/agent/run/audit \
  -H 'X-Project-UUID: 11111111-1111-1111-1111-111111111111' \
  -H 'Authorization: Bearer $SERVER_TOKEN' \
  -d @- <<'EOF'
{
  "source":  "git@github.com:acme/widget.git",
  "agent":   "codex",
  "oauth_cred_json": "{\"auth_mode\":\"codex\",\"tokens\":{\"id_token\":\"...\",\"access_token\":\"...\",\"refresh_token\":\"...\",\"account_id\":\"...\"}}"
}
EOF
```

双租户 `driver=both`：

```json
{
  "source": "/repo",
  "driver": "both",
  "audit_auth":  { "oauth_token":    "sk-ant-oat01-tenant-A-..." },
  "piolium_auth": { "oauth_cred_json": "{\"tokens\":{...tenant B...}}" }
}
```

无身份验证模式下的 OpenAI-SDK 风格聊天补全：

```bash
curl -X POST $HOST/api/agent/chat/completions \
  -H 'Authorization: Bearer sk-ant-api03-...' \
  -d '{"model":"any","messages":[{"role":"user","content":"hi"}]}'
```