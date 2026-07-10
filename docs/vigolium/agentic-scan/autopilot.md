# Autopilot Mode

`vigolium agent autopilot` 是**单循环 Agent 扫描**：一个长时间运行的 LLM 会话决定调查什么，驱动工具（bash、文件 I/O、web 抓取、vigolium CLI 命令），在确认漏洞发现后将其报告到数据库，并在没有更多有价值的工作可做时自行停止。

这是最简单的 Agent 扫描模式——没有主/工作流水线，没有计划/分类阶段拆分，只有一个引擎运行直到它调用 `halt_scan`。

---

## 心智模型

将其视为坐在终端前的安全分析师：

- 分析师获得一个目标 URL，可选地获得源代码树和一个关注提示。
- 他们拥有 shell 访问权限、文件读取权限、Web 访问权限以及 vigolium CLI。
- 他们四处探查，跟踪线索，在确认每个漏洞时将其写入笔记本，并在没有更多值得深挖的内容时停止。

Autopilot 正是如此，只不过分析师是 LLM，笔记本是 vigolium 的 `findings` 表。

---

## 生命周期（高级）

```
                  ┌──────────────────────────────────────────────┐
 vigolium agent ─►│ 1. CLI 标志解析                              │
 autopilot …      │    强度预设 → max-cmds, timeout, …          │
                  │    --input → curl/HTTP/Burp/url 规范化       │
                  │    --source → 解析 git URL/diff/本地路径     │
                  │    --provider/model → olium.ResolveProvider  │
                  └──────────────────────────────────────────────┘
                                       │
                                       ▼
                  ┌──────────────────────────────────────────────┐
                  │ 2. 会话引导                                   │
                  │    EnsureSessionDir(~/.vigolium/agent-…/UUID)│
                  │    WriteRunPID, CleanupOrphanedProcesses     │
                  │    创建 AgenticScan 行（status=running）      │
                  │    tee stdout → {session}/runtime.log        │
                  └──────────────────────────────────────────────┘
                                       │
                                       ▼
                  ┌──────────────────────────────────────────────┐
                  │ 3. autopilot.Run (pkg/olium/autopilot)       │
                  │    构建系统提示 + 初始用户提示                │
                  │    注册工具：内置工具 + halt_scan +          │
                  │      report_finding + load_skill             │
                  │    engine.New + engine.Run(ctx, initial)     │
                  └──────────────────────────────────────────────┘
                                       │
                       ┌───────────────┴───────────────┐
                       ▼                               ▼
              ┌────────────────┐              ┌─────────────────┐
              │ provider       │  多轮交互    │ 工具注册表      │
              │ (codex /       │◄────────────►│ bash, read,     │
              │  anthropic /   │   工具调用   │ write, edit,    │
              │  openai / …)   │              │ ls, grep, glob, │
              └────────────────┘              │ web_fetch,      │
                                              │ load_skill,     │
                                              │ halt_scan,      │
                                              │ report_finding  │
                                              └─────────────────┘
                                       │
                                       ▼
                  ┌──────────────────────────────────────────────┐
                  │ 4. 停止 → 最终化                             │
                  │    halt_scan 被调用 OR ctx 完成 OR 达到最大轮次│
                  │    UPDATE AgenticScan: status, duration,     │
                  │    finding_count, error_message              │
                  │    打印摘要，移除 run.pid                    │
                  └──────────────────────────────────────────────┘
```

---

## 数据流

```
┌──────────────────────────────────────────────────────────────┐
│ 用户输入                                                     │
│   --target / --input (curl/raw/Burp/base64) / stdin 管道     │
│   --source (本地路径, git URL, diff, 最近 N 次提交)          │
│   --focus, --instruction, --intensity                        │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼ resolveInputAndTarget,
                                ResolveSourceAndDiff
┌──────────────────────────────────────────────────────────────┐
│ runAutopilotOlium                                            │
│   解析 olium provider+model                                  │
│   创建会话目录 + AgenticScan 父行                            │
│   构造 autopilot.Options{Target, SourcePath, Focus,          │
│     Instruction, Repo, ProjectUUID, ScanUUID, MaxTurns,      │
│     Out, ExtraSystemPrompt}                                  │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│ autopilot.Run                                                │
│   • LoadSkillsFor(includeUser=true) → ~/.vigolium/skills/    │
│   • 工具注册表 := 内置工具 + halt_scan + report_finding      │
│     (+ load_skill 如果加载了任何技能)                        │
│   • 系统提示 := 角色文件 + 技能 XML + 额外内容               │
│   • 初始提示 := 目标/源代码/扫描范围/关注点 + 指令           │
│   • engine.New({Provider, Tools, Skills, MaxTurns,           │
│       EnablePromptCache: true, …})                           │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│ engine.run 循环  (pkg/olium/engine)                          │
│                                                              │
│   for turn < MaxTurns:                                       │
│     ┌─ provider.Stream(System, Tools, History, CacheControl)─┐│
│     │  发出文本/思考增量 + 工具调用帧                        ││
│     └────────────────────────────────────────────────────────┘│
│     将助手消息（文本 + tool_calls）追加到历史记录             │
│     发出 EventTurnDone（包含使用令牌数）                     │
│                                                              │
│     如果没有工具调用：发出 EventRunDone，返回                │
│                                                              │
│     对于每个工具调用：                                        │
│       • 只读批次（read_file/ls/grep/glob/web_fetch）：        │
│         并行分发（最大扇出 8）                                │
│       • 其他任何操作（bash, write_file, edit_file, …）：     │
│         串行分发                                              │
│       每个工具截止时间（默认 5 分钟）叠加在父上下文上        │
│       结果在追加到历史记录前截断为 16 KiB 头部+尾部          │
│       将 RoleTool 消息追加到历史记录                          │
│       发出 EventToolExecStart / End                          │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│ 副作用                                                       │
│                                                              │
│   report_finding ──► repo.SaveFindingDirect（按哈希去重）    │
│                     计数器++；50 时软警告，200 时硬上限      │
│                                                              │
│   halt_scan      ──► HaltSignal.Set(reason)                  │
│                     引擎看到没有更多工具调用并正常退出        │
│                                                              │
│   流事件        ──► stdout（tee 到 runtime.log）用于文本，   │
│                     stderr 用于单行工具活动                   │
└──────────────────────────────────────────────────────────────┘
```

---

## Agent 实际可访问的内容

Autopilot **不**限于 vigolium CLI。引擎提供通用的 Agent 工具集；安全特定行为来自系统提示和技能，而不是工具表面本身。

| 工具             | 来源                            | 说明                                                                 |
| ---------------- | ------------------------------- | -------------------------------------------------------------------- |
| `bash`           | `pkg/olium/tool/bash.go`        | 无沙箱 shell。灾难性模式（例如 `rm -rf /`）被硬拒绝。               |
| `read_file`      | `pkg/olium/tool/files.go`       | 只读。可并行化。                                                     |
| `write_file`     | `pkg/olium/tool/files.go`       | 副作用；仅串行。                                                     |
| `edit_file`      | `pkg/olium/tool/files.go`       | 副作用；仅串行。                                                     |
| `ls`             | `pkg/olium/tool/files.go`       | 可并行化。                                                           |
| `grep`           | `pkg/olium/tool/search.go`      | 可并行化。                                                           |
| `glob`           | `pkg/olium/tool/search.go`      | 可并行化。                                                           |
| `web_fetch`      | `pkg/olium/tool/web.go`         | 可并行化。                                                           |
| `load_skill`     | `pkg/olium/skill/tool.go`       | 按名称拉取技能主体（技能在系统提示中索引）。                         |
| `halt_scan`      | `pkg/olium/autopilot/halt.go`   | 仅 Autopilot。设置 `HaltSignal`，引擎在下一轮正常退出。              |
| `report_finding` | `pkg/olium/autopilot/report_…`  | 仅 Autopilot。写入作用域为 AgenticScan UUID 的 `Finding` 行。        |

模型决定何时调用 `vigolium scan-url`、`vigolium finding` 等——这些是 **shell 命令**，不是一等工具。Agent 通过 `bash` 运行它们（如果它认为有用）。

---

## 供应商选择

`olium.ResolveProvider` 按以下顺序选择后端：

1. CLI 覆盖（`--provider`）
2. 配置文件（`vigolium-configs.yaml` 中的 `agent.olium.provider`）
3. 自动检测 → `openai-codex-oauth`

支持的供应商 ID 及其使用的凭据：

| 供应商           | 默认模型            | 凭据                                                |
| ------------------ | ------------------- | --------------------------------------------------------- |
| `openai-codex-oauth`      | `gpt-5.5`           | OAuth 凭据文件（`--oauth-cred` / `agent.olium.oauth_cred_path`） |
| `anthropic-api-key`| `claude-opus-4-7`   | `--llm-api-key` / `$ANTHROPIC_API_KEY`                    |
| `anthropic-oauth`     | `claude-opus-4-7`   | 来自 `claude setup-token` 的 Bearer 令牌（`--oauth-token` / `$ANTHROPIC_API_KEY`） |
| `openai-api-key`   | `gpt-5.5`           | `--llm-api-key` / `$OPENAI_API_KEY`                       |
| `anthropic-cli`  | `claude-opus-4-7`   | `$PATH` 上的 `claude` 二进制文件                            |

引擎上设置了 `EnablePromptCache: true`——支持它的供应商（Anthropic、Claude OAuth）在轮次之间缓存系统提示和工具列表，这在长时间 autopilot 运行中占据了大部分前缀。

---

## 强度预设

`--intensity` 捆绑多个设置；显式标志始终覆盖。

| 预设     | `MaxCommands` | `Timeout` | `AuditMode` | `Browser` |
| ---------- | ------------- | --------- | ------------ | --------- |
| `quick`    | 30            | 1h        | `lite`       | 关闭       |
| `balanced`（默认） | 100  | 6h        | `balanced`   | 关闭       |
| `deep`     | 300           | 12h       | `deep`       | 开启        |

`MaxCommands` 成为引擎的 `MaxTurns` 上限（每轮一个 LLM→工具循环）。达到上限时，运行以错误事件结束——模型未能干净停止。

---

## 停止条件

Autopilot 以四种方式之一退出：

1. **自然停止**——模型调用 `halt_scan`。当前轮次允许完成；引擎在下一轮看到没有更多工具调用并发出 `EventRunDone`。`Result.Halted=true`，`HaltReason` 已填充。
2. **静默停止**——模型完成一轮且没有工具调用和 `halt_scan`。视为自然停止。`Result.Halted=false`，`HaltReason="(natural stop — engine max turns or no more tool calls)"`。
3. **达到最大轮次**——轮次数达到 `MaxCommands`。引擎发出 `EventError`；autopilot 返回非 nil 错误。
4. **上下文取消**——超时或 SIGINT/SIGTERM。引擎拆卸取消正在进行的工具；autopilot 返回包装的 `context.DeadlineExceeded` / `context.Canceled` 错误。

一个单独的**漏洞发现速率限制**位于 `report_finding` 内部：
- 50 个漏洞发现时软警告（仍保存）
- 200 个时硬上限（拒绝并返回 `IsError` 结果，提示模型转向 `halt_scan`）

---

## 漏洞发现持久化

每次成功的 `report_finding` 调用通过 `repo.SaveFindingDirect` 写入一行。Autopilot 标记的关键字段：

- `ProjectUUID`、`ScanUUID`、`AgenticScanUUID`——传播项目/扫描范围，以便 `vigolium finding` 和 `vigolium agent sessions` 可以回连。
- `ModuleID = "olium-autopilot"`、`ModuleType = "ai-agent"`、`FindingSource = "autopilot"`——区分 Agent 产生的漏洞发现与扫描器模块的漏洞发现。
- `FindingHash`——对（标题、严重性、source_file、url、描述指纹）的 SHA-256 哈希，或者如果模型提供了显式的 `dedup_key` 则使用该键。数据库的 `ON CONFLICT` 处理程序会消除重复项。

由于 autopilot 在确认漏洞发现时立即写入，部分结果在崩溃或超时时仍然存在——父 `AgenticScan` 行仅更新为 `status=failed` 并附带错误消息，但在此之前保存的所有内容都保留在数据库中。

---

## 会话工件

每次运行，autopilot 在配置的 `agent.sessions_dir`（默认 `~/.vigolium/agent-sessions/`）下创建一个 UUID 命名的目录：

```
~/.vigolium/agent-sessions/{run-uuid}/
  ├── run.pid       # pgid + 开始时间；退出时清除
  └── runtime.log   # stdout 的 tee（助手文本流）
```

运行 UUID 与 `AgenticScan.uuid` 行匹配，因此 `vigolium agent sessions` 和 `vigolium log <uuid>` 都可以工作，无需额外管道。启动时会清理超过 48 小时的陈旧目录；孤儿 PID 文件（来自 SIGKILL 的运行）通过 `agent.CleanupOrphanedProcesses` 清除。

---

## 源代码感知模式

当设置 `--source` 时，三件事发生变化：

1. **源代码解析**——`agent.ResolveSourceAndDiff` 接受本地路径、git URL（克隆到临时目录）、`--diff PR-url|ref...ref|HEAD~N` 和 `--last-commits N`。Agent 获得本地路径和（可选）更改文件列表。
2. **初始提示模式提示**——提示在黑箱（"探测实时目标"）、白箱（"浏览源代码树"）或灰箱混合（"阅读代码以发现风险，然后探测"）之间切换。
3. **技能范围**——`LoadSkillsFor(includeUser=true)` 在内置技能集之上添加 `~/.vigolium/skills/`，因此像 `audit-auth` 和 `triage-finding` 这样的扫描特定技能可通过 `load_skill` 使用。

在 olium 支持的 autopilot 中没有单独的代码审计预阶段——单个 Agent 循环处理代码阅读和动态探测。

---

## 多应用扇出

当位置提示解析为**多个**应用时（`vigolium agent autopilot "scan source at ~/src/A, ~/src/B"`），包级别的 autopilot 标志被快照并重新应用于每个应用，然后为每个应用顺序调用 `runAutopilotOlium`。每个应用获得自己的会话目录、AgenticScan 行和供应商会话。

单应用提示直接使用解析后的标志重新进入 `runAgentAutopilot`——与标志驱动的调用相同的代码路径。

---

## REST API

```
POST /api/agent/run/autopilot   # 异步启动，返回运行 UUID
GET  /api/agent/status/list     # 列出活动/最近的运行
GET  /api/agent/status/:id      # 轮询单个运行
```

HTTP 请求体镜像 CLI 标志。`EffectiveSourcePath()` 接受 `source` 或遗留的 `repo_path` 字段。处理程序以与 CLI 相同的方式解析供应商/源代码，然后在 goroutine 上进入 `autopilot.Run`；运行 UUID 立即返回。

---

## 关键代码引用

| 关注点                           | 文件                                                |
| --------------------------------- | --------------------------------------------------- |
| CLI 命令 + 标志                   | `pkg/cli/agent_autopilot.go`                        |
| Olium 支持的入口                  | `pkg/cli/agent_autopilot_olium.go`                  |
| 运行编排（Options/Run）           | `pkg/olium/autopilot/autopilot.go`                  |
| 系统/初始提示构建器               | `pkg/olium/autopilot/prompt.go`                     |
| `halt_scan` 工具 + 信号           | `pkg/olium/autopilot/halt.go`                       |
| `report_finding` 工具 + 去重      | `pkg/olium/autopilot/report_finding.go`             |
| 引擎多轮循环                      | `pkg/olium/engine/engine.go`                        |
| 引擎事件类型                      | `pkg/olium/engine/event.go`                         |
| 内置工具注册表                    | `pkg/olium/tool/builtin.go`                         |
| 供应商解析                        | `pkg/olium/select.go`, `pkg/olium/runner.go`        |
| 技能加载                          | `pkg/olium/skill/`                                  |
| 强度预设                          | `pkg/agent/agenttypes/constants.go`                 |
| 系统提示角色（覆盖）              | `~/.vigolium/prompts/olium-system.md`               |
| 系统提示角色（内嵌）              | `public/presets/prompts/autopilot/olium-system.md`  |

---

## TL;DR

Autopilot 是一个单一的 LLM Agent，受 `MaxCommands` 轮次和 `Timeout` 挂钟时间的约束。它拥有 shell、文件和 Web；vigolium 特定行为完全编码在系统提示、两个自定义工具（`halt_scan`、`report_finding`）以及 `~/.vigolium/skills/` 下的可选技能中。漏洞发现在运行过程中写入；运行在模型自行停止、达到轮次上限或被取消时结束。