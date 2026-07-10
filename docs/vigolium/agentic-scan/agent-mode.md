# Agent Mode（Agent 模式）

Vigolium 在 `vigolium agent` 下提供了 **8 个 agent 子命令**。它们分为四个系列：

- **Agent 扫描模式** — 自主或 AI 引导的漏洞扫描：`autopilot`、`swarm`
- **源代码审计模式** — 多阶段 AI 代码审计：`audit`（Claude/Codex）、`piolium`（Pi 原生）和 `audit`（统一驱动，可依次运行两者）
- **单次/交互式** — `query`、`olium`
- **实用工具** — `session`

父命令 `vigolium agent` 本身仅支持 `--list-templates` 和 `--list-agents`。所有执行都需要子命令。

---

## 何时使用何种模式

| 你想要…… | 使用 | 原因 |
|---|---|---|
| 对代码或目标运行一次性提示（无扫描循环） | `query` | 单次、模板驱动，返回结构化漏洞发现或 HTTP 记录 |
| 将控制权交给 agent 进行完整的自主渗透测试 | `autopilot` | 一次长时间的 LLM 会话，具备完整的 Bash/读/写/Grep/Glob 能力；agent 自行选择策略 |
| 让 AI **驱动本地扫描器**对特定目标进行扫描 | `swarm` | 主/从管道：AI 规划 → 本地模块执行 → 可选的三审+重新扫描 |
| 在扫描之前或同时深度审计源代码（密钥、SAST 三审、PoC） | `audit` | 多阶段代码审计（`lite`/`balanced`/`deep`）；将漏洞发现导入数据库 |
| 通过 Pi（`pi install` 模型）运行相同的多阶段审计 | `piolium` | Pi 原生 piolium 框架；与 audit 相同的输出模式，在数据库中单独标记 |
| 在一个 AgenticScan 下对同一源代码运行统一审计（audit，失败时回退到 piolium） | `audit` | 统一驱动：`--driver=auto`（默认）先运行 audit，仅在 audit 失败时运行 piolium；`--driver=both` 无条件运行两者；每个驱动有子行 + 后处理漏洞发现去重；`--driver=piolium\|audit` 强制使用其中一个 |
| 在 TUI 中与 LLM 交互式聊天（调试、探索、临时任务） | `olium` | 实时多轮对话；非安全扫描 — 通用 agent |
| 查看过去的 agent 运行记录 | `session` | 列出之前的运行记录，显示原始输出和工件 |

---

## 模式参考

### `query` — 单次提示
- **文件：** `pkg/cli/agent.go`
- **用途：** 代码审查、端点发现、密钥检测、临时提示。
- **不适用于：** 网络扫描或多阶段编排。
- **关键标志：** `--prompt-template`、`-p/--prompt`、`--stdin`、`--source`、`--files`、`--source-label`、`--output`、`--dry-run`。

### `autopilot` — 自主 Agent 扫描
- **文件：** `pkg/cli/agent_autopilot.go`
- **用途：** 渗透测试类型的任务，希望 agent 发挥创造力并自行决定操作。
- **工作原理：** 一次长时间运行的 LLM 会话，具备完整的 CLI 工具访问权限，直到 agent 调用 `halt_scan` 或达到限制。
- **关键标志：** `-t/--target`、`--input`（curl/原始 HTTP/Burp XML/URL/stdin）、`--plan-file`、`--source`（自动运行审计）、`--focus`、`--max-commands`、`--max-duration`、`--intensity {quick|balanced|deep}`、`--audit`（`lite|balanced|deep|off`）、`--browser`、`--credentials`、`--diff`、`--last-commits`。

### `swarm` — AI 引导的多阶段扫描
- **文件：** `pkg/cli/agent_swarm.go`
- **用途：** 针对特定目标的扫描，需要结构化流程（规划 → 本地扫描 → 三审）、源代码感知的路由发现或验证循环。
- **工作原理：** 10 阶段管道 — 归一化 → 身份验证（可选）→ 源代码分析（可选）→ 代码审计（可选）→ 发现（可选）→ 规划（AI）→ 插件（可选）→ 本地扫描 → 三审（可选）→ 重新扫描（可选）。
- **关键标志：** `-t/--target`（与 `--source` 一起使用时必需）、`--input`、`--plan-file`、`--record-uuid`、`--source`、`--discover`、`--code-audit`、`--triage`、`--max-iterations`、`-m/--modules`、`--vuln-type`、`--focus`、`--audit {lite|balanced|deep|off}`、`--intensity`、`--only`/`--skip`/`--start-from`。

### 计划文件（`--plan-file`）
`autopilot` 和 `swarm` 都接受 `--plan-file <path>`：一个单一文件，混合了自由文本指导和原始 HTTP 请求 — 正是您会粘贴到终端的内容。无需前置元数据或模式。

- **第一个 HTTP 请求行之前**的散文成为指令。
- 请求区域（从第一个请求行到文件末尾的所有内容）在恰好为 `---` 的行上分割成独立的请求块。也识别围栏代码块 ` ```http ` / ` ```request `。
- `autopilot` 是单种子：第一个请求是实时种子；任何额外的块作为带标签的上下文折叠到指令中。
- `swarm` 是多种子：每个请求块作为独立的种子输入提供。
- 没有请求行的文件被视为纯指令（然后提供 `--target`/`--source`）。
- `--plan-file` 同时拥有指令和种子输入，因此**不能**与 `--input`、`--instruction` 或 `--instruction-file` 组合使用（autopilot 也拒绝 `--record-uuid`）。

```
以下是订单 ID 0254685 和 0254774 — 重点关注 IDOR。

GET /order/details?orderId=0254809 HTTP/2
Host: ginandjuice.shop
Cookie: session=...
```
运行方式：`vigolium agent autopilot --plan-file ginandjuice-plan.md`（或 `swarm`）。

### `audit` — AI 安全源代码审计（Claude/Codex）
- **文件：** `pkg/cli/agent_audit.go`
- **用途：** 独立的深度代码审计，或作为 `autopilot`/`swarm` 的源代码感知伴侣。
- **模式：** `lite`（3 阶段）、`balanced`/`scan`（9）、`deep`（12）、`revisit`、`confirm`、`merge`、`diff`、`longshot`、`refresh`、`reinvest`、`status`、`mock`。`--modes a,b,c` 通过 audit 的原生 `--modes` 链式运行多个模式（一个子进程；在第一个非完成模式处停止；`--max-cost` 是总上限；后续模式自动继承前一个模式的 `--from-audit`）。`--intensity deep` 解析为链 `deep,confirm`；`quick`→`lite` 和 `balanced`→`balanced` 保持单模式。
- **关键标志：** `--mode`、`--modes a,b,c`、`--list-modes`（打印 audit 的模式图并退出）、`--source <path|git-url>`、`--provider <olium-provider>`（驱动 agent **并**转发该供应商的 BYOK 身份验证：`anthropic-*` → claude，`openai-*` → codex）、`--agent {claude|codex}`（纯 agent 选择器 — 覆盖 `--provider` 隐含的 agent，同时保留其解析的身份验证；无效则拒绝）、`--no-stream`。
- **持久化 agent：** 在 `vigolium-configs.yaml` 中设置 `agent.audit.default_agent: {claude|codex}` 以固定审计 agent，无需更改 `agent.olium.provider`（供应商仍提供 BYOK 身份验证）。它是纯 agent 选择器，语义与 `--agent` 相同；空值继承供应商派生的 agent。优先级（最高优先）：每次运行的 `--agent` > `--provider` > `agent.audit.default_agent` > `agent.olium.provider` 派生 > claude。
- **详情：** [`docs/agentic-scan/vigolium-audit.md`](vigolium-audit.md)。

### `piolium` 驱动 — AI 安全源代码审计（Pi 原生）
- **调用方式：** `vigolium agent audit --driver=piolium` — 没有独立的 `agent piolium` 子命令。共享驱动辅助函数：`pkg/cli/agent_piolium.go`。
- **用途：** 与 `audit` 相同的多阶段审计，但通过 Pi 运行时 + 用户安装的 piolium 插件运行。当主机已配置 Pi 用于开发工作时很有用。
- **模式：** `lite`（4）、`balanced`（9）、`deep`（17）、`revisit`（9）、`confirm`（7）、`merge`（7）、`diff`（1）、`longshot`（3）、`status`、`smoke`。
- **关键标志：** `--mode`、`--intensity {quick|balanced|deep}`、`--source <path|git-url>`、`--commit-depth`、`--no-stream`、`--upload-results`，以及用于 piolium 会话标志的 `--plm-*` 透传参数。
- **详情：** [`docs/agentic-scan/piolium-audit.md`](piolium-audit.md)。

### `audit` — 统一驱动（audit + piolium）
- **文件：** `pkg/cli/agent_audit.go`
- **用途：** 在一个 AgenticScan 下使用一个或两个审计框架对同一源代码树进行评估。默认 `--driver=auto` 先运行 audit，仅在 audit 失败时回退到 piolium（干净的 audit 运行完成审计，piolium 从不启动 — 因此缺少 piolium 运行时保持静默）。`--driver=both` 无条件先运行 audit 再运行 piolium。每个驱动有会话子目录（`{session}/audit/`、`{session}/piolium/`），每个驱动有子 AgenticScan 行（位于一个父行下），以及驱动退出后的项目范围漏洞发现去重。
- **模式：** 任何对参与驱动有效的模式都被接受。`--modes a,b,c` 链式运行多个模式，在第一个非完成模式处停止。audit 原生运行链（一个子进程，一行，总成本）；piolium 通过同一源代码树中的顺序 `pi` 运行进行链式处理，合并为一个聚合子行。对于 `--driver=auto|both`，驱动无法运行的模式在该驱动的阶段被跳过（例如 `--modes deep,refresh,confirm` 在 audit 上运行全部三个，在 piolium 上运行 `deep,confirm`，因为 `refresh` 仅 audit 可用）；两个驱动都未知的模式是硬错误。`--intensity deep` 解析为链 `deep,confirm`（`quick`→`lite`、`balanced`→`balanced` 保持单模式）。
- **关键标志：** `--driver {auto|both|audit|piolium}`（默认 `auto`）、`--mode`、`--modes a,b,c`、`--list-modes`（打印模式图并退出）、`--intensity`、`--source`、`--commit-depth`、`--no-stream`、`--no-dedup`、`--upload-results`、`--provider <olium-provider>` 和 `--agent {claude|codex}`（均为 audit 阶段专用 — `--agent` 覆盖供应商隐含的 agent 而不更改身份验证，并在 `--driver=piolium` 时发出警告而非错误），以及用于 piolium 驱动的 `--pi-*` 和 `--plm-*` 标志。
- **BYOK 身份验证：** `--api-key` / `--oauth-token` / `--oauth-cred-file` 接受字面值、`$ENV` 或 `@path`，并应用于运行的任何驱动。详情： [`docs/agentic-scan/audit-byok.md`](audit-byok.md)。
- **REST 等效：** `POST /api/agent/run/audit`，带有 `driver: "auto"|"both"|"audit"|"piolium"`（默认 `"auto"`）。

### `olium` — 交互式 TUI 聊天
- **别名：** 顶层 `vigolium olium` / `vigolium ol`
- **文件：** `pkg/cli/agent_olium.go`
- **用途：** 交互式调试、探索或一次性无头提示。供应商无关。
- **不适用于：** 编排扫描 — 没有扫描阶段。
- **关键标志：** `--provider`、`--model`、`--llm-api-key`、`--oauth-cred`/`--oauth-token`、`--system`、`--headless`、`-p/--prompt`、`--stdin`。

### `session` — Agent 运行历史
- **别名：** `sessions`、`sess`
- **文件：** `pkg/cli/agent_session.go`
- **用途：** 审计之前的运行记录、调试失败的扫描。
- **关键标志：** `--mode {query|autopilot|pipeline|swarm}`、`-n/--limit`、`-o/--offset`、`--tail`、`--full`。

---

## 在 `autopilot` 和 `swarm` 之间选择

两者都是 Agent 扫描模式。区别在于：

- **`autopilot`** — agent **就是**扫描器。它打开 shell、读取文件、运行工具并决定一切。最适合目标模糊或需要创造性探索性测试的场景。
- **`swarm`** — agent **指挥**本地扫描器。它规划、选择模块、生成 JS 插件，而确定性的 Go 管道执行繁重的流量任务。最适合需要结构化、可重复结果以及可选验证循环的场景。

如果您有**源代码**和**目标 URL**，两者都可以使用；`swarm --source --target ... --code-audit --triage` 提供最结构化的输出，而 `autopilot --source ...` 给予 agent 更多自由。

---

## 横切关注点

- **会话目录：** `~/.vigolium/agent-sessions/`（可通过 `vigolium-configs.yaml` 中的 `agent.sessions_dir` 覆盖）。
- **提示模板：** `~/.vigolium/prompts/` 或嵌入在 `public/presets/prompts/` 下。
- **输出模式：** `findings`、`http_records`、`attack_plan`、`triage_result`、`source_analysis`。

### 技能（Skills）

技能是指令性的攻击向量操作手册（例如 `idor-blast-radius`、`xss-browser-confirm`、`sqli-to-data-exfil`），agent 通过 `load_skill` 工具按需加载。内置技能嵌入在二进制文件中，在 `vigolium init` 时物化到 `~/.vigolium/skills/`（可通过 `VIGOLIUM_SKILLS_DIR` 覆盖目录）；操作员的编辑会保留，`vigolium init --force` 会重置它们。将您自己的 `SKILL.md` 文件（带有前置元数据 `name`、`description`、可选的 `tags`）放入该目录、`.agents/skills/` 或 `.claude/skills/` — 范围越近，名称冲突时优先级越高。

**按模式加载：**
- **olium TUI** — 加载所有技能；启动时打印计数行；使用 `/skill:<name>` 内联调用。
- **autopilot** — 一次尽力而为的预检调用选择与目标匹配的技能；operator agent 按需加载主体。任何失败时回退到完整集合。
- **swarm** — **规划**阶段看到完整菜单并发出 `RECOMMENDED_SKILLS`；**三审**阶段加载该选择加上始终启用的集合，并获取读取+重放工具子集（`query_records`、`inspect_record`、`replay_request`、`attack_kit`；`browser_probe` 始终是内置的），使技能可操作。

**选择覆盖**（autopilot + swarm）：`--skill <name>`（可重复）和 `--skill-tag <tag>` 强制包含并绕过规划器判断；`--no-skill-filter` 加载完整集合。始终启用的技能（默认 `triage-finding`、`write-jsext`）可通过 `agent.olium.always_on_skills` 配置。
- **引擎：** 每个 agent 运行都通过进程内 olium 运行时（`pkg/olium/`）分发。供应商选择位于 `vigolium-configs.yaml` 的 `agent.olium.provider` 中 — 参见 [`docs/architecture/agentic-scan.md`](../architecture/agentic-scan.md) 获取供应商列表。
- **源代码标志：** `--source` 是所有模式的标准源代码标志；旧的 `--repo`/`--repo-url` 标志已被移除。

### REST API

服务器暴露相同的三种模式以及状态/工件接口，使控制器无需 CLI 即可启动和跟踪运行。

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| POST | `/api/agent/run/query` | 一次性提示执行。 |
| POST | `/api/agent/run/autopilot` | 启动 autopilot 扫描。 |
| POST | `/api/agent/run/swarm` | 启动 swarm 扫描。 |
| GET | `/api/agent/status/list` | 列出活动和历史运行（数据库 + 内存合并）。 |
| GET | `/api/agent/status/:id` | 单个运行的状态。 |
| GET | `/api/agent/sessions` | 分页会话历史（比 `/status/list` 更丰富）。 |
| GET | `/api/agent/sessions/:id` | 完整的会话详情，包括原始输出、计划、子运行。 |
| GET | `/api/agent/sessions/:id/logs` | 读取或跟踪 `runtime.log`（当 `Accept: text/event-stream` 时使用 SSE）。 |
| GET | `/api/agent/sessions/:id/artifacts` | 列出 session_dir 内的文件（递归，上限 500 条）。 |
| GET | `/api/agent/sessions/:id/artifacts/{name}` | 读取一个文件（`output.md`、`plan.json` 等）。通配符支持嵌套（`vigolium-results/state.json`）。可选的 `?max_bytes=N` 上限（默认 10 MiB，硬上限 100 MiB）。 |

运行端点返回 `202 Accepted`，带有 `{agentic_scan_uuid, status: "running"}`，并在后台执行。预期的工作流程是：

1. POST 到某个 `run/*` 端点，捕获 `agentic_scan_uuid`。
2. 轮询 `GET /api/agent/status/:id` 直到 `status` 离开 `running`。
3. 通过 `/api/agent/sessions/:id/logs`、`/artifacts` 或 `/artifacts/{name}` 获取会话工件以获取原始输出（`output.md`、`plan.json`、`audit-stream.jsonl`、生成的插件等）。

运行端点上的 `stream: true` 选择使用 Server-Sent Events 而不是异步响应 — 大多数消费者应坚持使用异步流程并按需跟踪日志。

#### 供应商覆盖仅限 CLI / 服务器配置

CLI 暴露每次调用的供应商标志（`--provider`、`--model`、`--oauth-cred`、`--oauth-token`、`--llm-api-key`、`--system`）。这些**不会**镜像到 REST 请求模式中。服务器从 `vigolium-configs.yaml` 中的 `agent.olium.*` 解析供应商一次，并在请求之间重用，这保持了热会话和提示缓存的稳定性。要在服务器端工作负载上切换供应商，编辑 YAML 并重新加载 — 没有每次请求的覆盖字段。
