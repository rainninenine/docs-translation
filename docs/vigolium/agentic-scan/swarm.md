# Agent Swarm Mode

`vigolium agent swarm` 是 AI 引导的 Agent 扫描模式。主 Agent 读取目标的请求/响应表面（以及可选的源代码），选择合适的扫描器模块，在需要时生成自定义 JavaScript 插件，运行原生扫描器，并可选择在验证和重新扫描循环中对结果进行分类。

它介于两个极端之间：比 `autopilot` 更**有方向性**（autopilot 赋予 Agent 完整的 Claude Code 工具自由支配权），比手动调优的 `vigolium scan` 更**灵活**（模块和插件由模型而非用户选择）。

本文档涵盖管道的结构、阶段之间的数据流以及 AI / 原生边界的位置。

---

## 1. 何时使用 swarm

| 场景 | 使用 |
|---|---|
| 你有目标 URL、原始请求或 HTTP 记录，希望 Agent 选择模块并编写载荷 | `swarm` |
| 你还有应用源代码——希望从代码中推断路由 + 认证 + 插件 | `swarm --source <path>` |
| 你想要误报验证和基于模型判断的定向重新扫描 | `swarm --triage` |
| 你更愿意将 shell + 代码库交给 Agent，让它决定一切 | `autopilot` |
| 你想要一次性提示，无需扫描 | `agent query` |

Swarm 是当你希望 AI **驱动原生扫描器**而非替代它时的正确模式。

---

## 2. 管道概览

十个阶段按严格顺序执行。每个阶段要么是**原生**（确定性 Go），要么是 **AI**（通过 olium 引擎的 LLM 调用）。大多数阶段是有条件的——仅当它们的输入存在或用户选择加入时才触发。

```
┌────────────────────────────────────────────────────────────────────────────┐
│                          swarm 管道                                         │
│                                                                             │
│  [N] native-normalize ─────────────────────────────── 始终执行              │
│         │                                                                   │
│  [A] auth ─────────────────────────── 如果 --browser-auth 且 --browser      │
│         │                                                                   │
│  [A] source-analysis ─────────────── 如果 --source  (4 次调用波)            │
│  [A] code-audit ──────────────────── 如果 --code-audit 且 --source          │
│         │                                                                   │
│  [N] native-discover ─────────────── 如果 --discover                        │
│         │                                                                   │
│  [A] plan         ────────────────── 始终执行（主 Agent，可批处理）         │
│  [N] native-extension ────────────── 如果计划声明了插件                     │
│  [N] native-scan  ────────────────── 始终执行（移交给扫描器）               │
│         │                                                                   │
│  [A] triage       ────────────────── 如果 --triage                          │
│  [N] native-rescan ───────────────── 每轮，当分类请求时                     │
│         │                                                                   │
│  [N] finalize     ────────────────── 始终执行                               │
└────────────────────────────────────────────────────────────────────────────┘

[A] = AI 调用    [N] = 原生 Go
```

阶段常量：`pkg/agent/agenttypes/constants.go:52-61`。阶段排序和分发：`pkg/agent/swarm_pipeline.go`。

---

## 3. 数据流

管道由单个 `swarmPipelineState`（`pkg/agent/swarm_pipeline.go:24-44`）驱动，所有阶段都从中读取和写入。流经管道的两个主要载荷是**记录**（HTTP 请求/响应对）和**计划**（模块选择 + 插件规范）。

```
                 ┌────────────────────┐
   inputs ──────►│  normalize         │──► []*HttpRequestResponse
   (curl/HTTP/   └────────────────────┘            │
    burp/url/                                      │
    record uuid/                                   ▼
    stdin)                              ┌────────────────────┐
                                        │ source-analysis    │ 如果 --source
                                        │  (4 次调用波)      │──► +routes
                       ┌────────────────┤                    │   +session_cfg
                       │                │                    │   +extensions
                       │                └────────────────────┘
                       │                          │
                       │                          ▼
                       │                ┌────────────────────┐
                       │                │ native-discover    │ 如果 --discover
                       │                └────────────────────┘
                       │                          │
                       │                          ▼   merged []records
                       │                ┌────────────────────┐
                       │                │  plan (master)     │
                       │                │  - 选择模块        │
                       │                │  - 关注区域        │
                       │                │  - 插件规范        │
                       │                └────────────────────┘
                       │                          │
                       │                          ▼ SwarmPlan
                       │                ┌────────────────────┐
                       │                │ extension          │
                       │                │ 验证 + 持久化      │
                       │                └────────────────────┘
                       │                          │
                       │                          ▼ extensions/*.js
                       │                ┌────────────────────┐
                       └───────────────►│ native-scan        │
                            ScanFunc    │ runner.RunNative…  │
                          (回调)        └────────────────────┘
                                                  │
                                                  ▼ findings → DB
                                        ┌────────────────────┐
                                        │ triage (循环)      │ 如果 --triage
                                        │ 每个发现的判定     │
                                        │ + 重新扫描请求     │
                                        └────────────────────┘
                                                  │
                                                  ▼
                                        ┌────────────────────┐
                                        │ finalize           │
                                        │ 聚合结果           │
                                        └────────────────────┘
```

记录在管道运行过程中增长：源码分析追加发现的路由，原生发现在规划之前合并爬取/蜘蛛结果。计划一旦生成，就是原生扫描器将运行什么的唯一真相来源。

---

## 4. 阶段参考

| # | 阶段 | 类型 | 目的 | 触发条件 |
|---|---|---|---|---|
| 1 | `native-normalize` | 原生 | 将 `--input`/stdin/record-uuid 解析为 `HttpRequestResponse` | 始终执行 |
| 2 | `auth` | AI/原生 | 浏览器驱动的登录，写入认证请求头/cookie | `--browser-auth` + `--browser` |
| 3 | `source-analysis` | AI | 4 次调用波：探索 → 路由 / 会话 / 插件 | `--source` |
| 4 | `code-audit` | AI | 代码级安全审计，发现 → DB | `--code-audit`（当 `--source` 且强度为 balanced/deep 时自动） |
| 5 | `native-discover` | 原生 | 爬取/蜘蛛/JS 扫描以发现端点 | `--discover` |
| 6 | `plan` | AI | 主 Agent：选择模块、关注区域、插件规范 | 始终执行 |
| 7 | `native-extension` | 原生 | 验证生成的 JS，写入 `extensions/` | 计划声明了插件 |
| 8 | `native-scan` | 原生 | 移交给 `runner.RunNativeScan()` | 始终执行 |
| 9 | `triage` | AI | 验证发现；标记为已确认 / 误报 / 重新扫描 | `--triage` |
| 10 | `native-rescan` | 原生 | 根据分类请求进行定向重新扫描（循环） | 分类判定 = "rescan" |

步骤分发位于 `pkg/agent/swarm_pipeline.go`（每个阶段一个 `…SwarmStep` 函数）。通过 `--skip-phases` 和 `--start-from` 支持跳过/恢复，它们读取检查点文件（参见 §9）。

---

## 5. 源码感知模式：4 次调用波

当指定 `--source <path>` 时，源码分析作为一次探索调用后跟三次并行的格式化调用运行（`pkg/agent/engine.go:356-369`）。

```
                 ┌──────────────────────────────────┐
                 │ 调用 1  swarm-source-explore      │
                 │ 读取源码一次 → 记录：              │
                 │   • 路由记录                      │
                 │   • 认证记录                      │
                 └────────────────┬─────────────────┘
                                  │ 会话历史重用
                                  │（provider 缓存，不会完整重新发送）
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
   ┌────────────────────┐ ┌────────────────────┐ ┌──────────────────────┐
   │ 调用 2a            │ │ 调用 2b            │ │ 调用 3               │
   │ format-routes      │ │ format-session     │ │ source-extensions    │
   │ 记录 → JSONL       │ │ 记录 →             │ │ 记录 → JS 文件      │
   │ http_records       │ │ session_config     │ │                      │
   └────────────────────┘ └────────────────────┘ └──────────────────────┘
              │                   │                   │
              └─────────────┬─────┴───────┬───────────┘
                            ▼             ▼
                    追加到          写入
                    ps.records      auth-config.yaml
```

为什么这样拆分：探索输出很大（上限 64 KB，`engine.go:465-469`），每个主题的格式化调用只看到 48 KB 的切片（`engine.go:481-489`）。Provider 会话缓存使探索上下文在三个后续调用中重用成本低廉，而不是重新支付。

发现的会话配置可能格式错误；引擎会将无效条目通过 LLM 往返修复（`swarm_pipeline.go:293-317`），然后再水化为认证请求头（`swarm_pipeline.go:429-479`）并持久化到 `auth-config.yaml`。

---

## 6. 主 Agent 和批处理

`plan` 阶段是两个子调用：

1. **Plan Agent**（`swarm.go:737-826`）——分析记录，返回包含模块标签/ID、关注区域和插件规范的 `SwarmPlan`。Markdown 节输出，在解析或临时错误时最多重试 `MaxMasterRetries`（默认 3）次。
2. **Extension Agent**（有条件）——仅当计划声明了插件时触发。生成 JS 扫描器代码。如果此调用失败，步骤 1 的计划仍然有效（优雅降级）。

当 `len(records) > MasterBatchSize`（默认 5；`agenttypes/constants.go:300`）时，规划扇出：

```
   records ─► 分区为 MasterBatchSize 大小的批次
                   │
                   ├─► batch 1 ┐
                   ├─► batch 2 │ 并行，最多 BatchConcurrency（默认 3）
                   ├─► batch 3 │ 通过 errgroup 的 goroutine
                   └─► batch N ┘
                                  ▼
                          plan_1, plan_2, … plan_N
                                  ▼
                         最后合并一次
                          • 模块标签/ID：集合并集
                          • 关注区域：去重
                          • 插件：按文件名合并；
                            不同代码的冲突 → 重命名
                          • 来源：每个批次贡献了什么
                                  ▼
                              SwarmPlan
```

实现：`pkg/agent/swarm.go:1374-1611`。第一个批次错误会取消其余批次；仅当调用者选择继续时才尝试部分成功合并。

当记录被过滤用于提示时，所有端点的紧凑摘要表会被追加，以便 Agent 即使只有前 N 个有完整的请求头/请求体，仍然能看到完整的表面（`swarm.go:602-627`）。

---

## 7. 分类和重新扫描循环

分类**默认关闭**。传递 `--triage` 以启用（`pkg/cli/agent_swarm.go:369-372`）。

```
   for round in 1..MaxIterations:
       ┌──────────────────────────────────┐
       │  从 DB 查询发现                   │  按严重性 / 模块过滤
       │  (从 last_finding_id 恢复)        │
       └─────────────┬────────────────────┘
                     ▼
       ┌──────────────────────────────────┐
       │  分类 Agent (AI)                  │  每个发现：
       │                                   │   confirmed | false-positive | rescan
       └─────────────┬────────────────────┘
                     ▼
            判定 == rescan?
              │              │
              │ 是            │ 完成 / 无需重新扫描
              ▼              ▼
       ┌──────────────┐    break
       │ native-rescan│
       │ OnlyPhase =  │
       │ dynamic-     │
       │ assessment   │
       └──────┬───────┘
              ▼
       检查点轮次，继续
```

`MaxIterations` 默认为 1（`quick`）、3（`balanced`）、5（`deep`）。分类每轮以 25 个发现为一批进行处理。实现在 `swarm.go:1623-1692` 中。

重新扫描在 `ScanRequest` 上设置 `IsRescan=true`，这强制 `OnlyPhase = "dynamic-assessment"` 和 `SkipIngestion = true`，因此只有目标模块执行（`agent_swarm.go:732-750`）。

---

## 8. 原生扫描器移交

Swarm 运行器本身不调用模块——它通过 CLI 安装在 `SwarmConfig` 上的回调移交：

| 回调 | 设置时机 | 功能 |
|---|---|---|
| `ScanFunc` | 始终 | 使用计划中的模块/插件运行 `runner.RunNativeScan()` |
| `DiscoverFunc` | `--discover` | 爬取/蜘蛛/JS 扫描；在规划前合并发现的记录 |
| `SourceAnalysisCallback` | `--source` | 从会话配置写入 `auth-config.yaml` |

`ScanFunc` 在 `pkg/cli/agent_swarm.go:714-776` 中构建：

```go
opts.Modules     = ResolveModulesFromPlan(req.ModuleTags, req.ModuleIDs)
opts.AuthConfigs = []string{generatedAuthConfigYAML} // 来自源码分析
opts.AuthConfigBestEffort = true                     // 容忍部分 AI 输出
if req.IsRescan {
    opts.OnlyPhase     = "dynamic-assessment"
    opts.SkipIngestion = true
}
runner := runner.New(opts)
return runner.RunNativeScan()
```

这是 AI / 原生边界：以上所有内容都是 AI 形状的（提示、计划、JS 代码、判定），此调用以下的内容是标准执行器（`pkg/core/executor.go`）运行已注册的模块。

---

## 9. 会话产物和检查点

Swarm 运行创建一个会话目录（默认 `~/.vigolium/agent-sessions/<run-id>/`）：

```
<sessionDir>/
├── swarm-plan.json            ← 来自主 Agent 的合并 SwarmPlan
├── swarm-checkpoint.json      ← 阶段进度、计划、分类轮次
├── source-analysis-prompt.md
├── source-analysis-output.md
├── source-analysis-sections.json
├── source-extensions.json
├── auth-config.yaml           ← 从 session_config 水化
├── code-audit-{prompt,output}.md
├── master-{prompt,output}.md
├── auth-{prompt,output}.md
├── extensions/*.js            ← 已验证、已持久化
├── triage/triage-round-N-{prompt,output}.md
└── runtime.log                ← tee'd LLM 流；使用 `vigolium log` 重放
```

`swarm-checkpoint.json` 在每个阶段后被重写。结合 `--start-from`，它允许部分运行恢复而无需为早期阶段重新付费（`agent_swarm.go:430-436`）。

---

## 10. CLI 速查表

```bash
# 最小：仅目标——模型从探测响应中选择模块
vigolium agent swarm --target https://app.example.com

# 源码感知：4 次调用源码分析 + 代码审计
vigolium agent swarm --target https://app.example.com --source ./repo

# 带分类和重新扫描（验证发现，重试）
vigolium agent swarm --target https://… --triage --max-iterations 3

# 使用预设
vigolium agent swarm --target https://… --intensity deep

# 从现有会话的特定阶段开始
vigolium agent swarm --resume <run-id> --start-from plan

# 渲染提示而不调用 LLM
vigolium agent swarm --target https://… --dry-run --show-prompt
```

重要标志（`pkg/cli/agent_swarm.go:24-71`）：

| 标志 | 效果 |
|---|---|
| `--target, -t` | 动态阶段的目标 URL |
| `--input` | 原始输入（curl、HTTP、Burp XML、base64、URL）——自动检测 |
| `--source` | 源代码路径；启用源码分析 + 代码审计 |
| `--code-audit` | 强制代码审计（使用 `--source` 且强度为 balanced/deep 时自动开启） |
| `--discover` | 在规划前运行原生发现 |
| `--triage` | 启用验证和重新扫描循环 |
| `--max-iterations` | 分类轮次（预设驱动默认值） |
| `--master-batch-size` | 每个主 Agent 批次的记录数（默认 5） |
| `--batch-concurrency` | 并行主批次（默认 3） |
| `--intensity` | `quick` / `balanced` / `deep` 预设捆绑 |
| `--skip-phases`, `--start-from` | 阶段控制 / 恢复 |
| `--source-analysis-only` | 在源码分析后停止 |

强度预设表：`pkg/agent/agenttypes/constants.go:292-338`。

---

## 11. 代码位置

| 关注点 | 文件 |
|---|---|
| CLI 标志 + 回调接线 | `pkg/cli/agent_swarm.go` |
| 阶段分发 + 状态 | `pkg/agent/swarm_pipeline.go` |
| 主 Agent、批处理、分类 | `pkg/agent/swarm.go` |
| 4 次调用源码分析 | `pkg/agent/engine.go`（`RunSourceAnalysisParallel`） |
| 阶段常量 + 预设 | `pkg/agent/agenttypes/constants.go` |
| 原生扫描器入口 | `pkg/core/executor.go`、`internal/runner/runner.go` |

有关更广泛的体系结构（olium 运行时、providers、通用引擎），请参见 [`architecture/agentic-scan.md`](../architecture/agentic-scan.md)。
