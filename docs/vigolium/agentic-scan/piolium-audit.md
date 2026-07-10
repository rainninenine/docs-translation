# Piolium Audit（Piolium 审计）

Piolium 是 Vigolium 的 **Pi 原生多阶段白盒安全审计框架**。它通过 `vigolium agent audit --driver=piolium`（piolium 审计驱动——没有独立的 `agent piolium` 子命令）运行，通过 `pi --mode json -p "/piolium-<mode>"` 驱动用户安装的 Pi 插件。漏洞发现结果会自动导入 Vigolium 数据库，与本地扫描器的结果并存。

> **寻找统一驱动？** `vigolium agent audit` 在同一个 AgenticScan 下依次运行 piolium 和 [audit](vigolium-audit.md)，每个驱动有独立的子行，并在通过后执行漏洞发现去重。使用 `--driver=piolium`（或 `--driver=audit`）强制使用单个驱动。本页描述 piolium 独立运行模式；统一形态请参见 audit 子命令的 `--help`。

Piolium 与 [`vigolium-audit`](vigolium-audit.md) 共享磁盘模式（`audit-state.json`、漏洞发现 markdown、前置元数据约定）——解析器、导入器和报告工具在两者间共享。区别在于运行时（Pi 而非 Claude Code / Codex）、文件夹名称（`piolium/` 而非 `audit/`）、环境变量前缀（`PIOLIUM_*` 而非 `ARCHON_*`）以及数据库标记（`mode=piolium` 而非 `audit`）。

## 目录

- [为何使用及何时使用](#为何使用及何时使用)
- [快速开始](#快速开始)
- [前提条件](#前提条件)
- [工作原理](#工作原理)
- [审计模式](#审计模式)
- [CLI](#cli)
- [REST API](#rest-api)
- [管道内审计（autopilot / swarm）](#管道内审计-autopilot--swarm)
- [流式输出](#流式输出)
- [成本追踪](#成本追踪)
- [设置](#设置)
- [会话产物](#会话产物)
- [漏洞发现格式](#漏洞发现格式)
- [漏洞发现导入](#漏洞发现导入)
- [对比：piolium vs audit](#对比piolium-vs-audit)
- [与本地扫描的对比](#与本地扫描的对比)
- [端到端测试](#端到端测试)

---

## 为何使用及何时使用

Piolium 和 [vigolium-audit](vigolium-audit.md) 共享相同的磁盘模式、漏洞发现格式和报告工具——它们的区别在于**哪个模型系列驱动审计**。根据您实际拥有的凭据选择：

- **使用 [vigolium-audit](vigolium-audit.md)**：当您可以访问 **Claude Opus** 系列（Claude Code、Vertex Anthropic、Bedrock Anthropic）时。Audit 的提示编排针对 Opus 进行了调优，在该系列上始终产生最高质量的漏洞发现。
- **使用 piolium**：当您的可用供应商是 **OpenAI（GPT/Codex）**、**Google（Gemini）** 或任何其他非 Claude 模型时。Pi 的适配器层抽象了供应商，piolium 的阶段提示旨在从非 Opus 模型中提取可靠的审计结果，同时不牺牲对抗性辩论/冷验证等质量控制手段。

### 何时使用

在以下情况下使用 `vigolium agent audit --driver=piolium`（piolium）：

- 您使用 **OpenAI 密钥**（`gpt-5.5`、Codex）运行，并希望获得与 Claude Opus 审计运行相当的质量。
- 您使用 **Gemini** 或其他 Vertex/Bedrock 托管的非 Anthropic 模型。
- 您需要一个模型无关的框架，以便在不更改审计管道的情况下切换供应商（`--pi-provider` / `--pi-model`）。
- 您需要 `longshot` 模式（逐个文件的全面搜索）——这是 piolium 独有的，audit 中不可用。

在以下情况下使用 [vigolium-audit](vigolium-audit.md)：

- 您拥有 Claude Opus 访问权限，并希望获得最佳的审计保真度。
- 您希望将 swarm/autopilot 审计锁定到 Claude/Codex，无论本地安装了什么——传递 `--audit=<mode>` 以退出自动选择。

两个框架都可以在 `vigolium agent autopilot` 和 `vigolium agent swarm` 期间作为管道内审计运行。当您不传递 `--audit` 或 `--piolium` 时，CLI 会自动选择：如果 `pi` + piolium 插件在本地安装，则运行 piolium，否则回退到 audit。两者可互操作：来自任一框架的漏洞发现都进入同一个 `findings` 表，标记其来源，并可以一起报告。

---

## 快速开始

```bash
# 对当前目录执行默认均衡审计
vigolium agent audit --driver=piolium --source .

# 对代码仓库执行快速分类（lite，4 个阶段）
vigolium agent audit --driver=piolium --source ./backend --mode lite

# 对远程仓库执行深度审计（17 个阶段），完整克隆历史
vigolium agent audit --driver=piolium --source git@github.com:org/repo.git --mode deep

# 确认现有审计的漏洞发现对目标仍然有效
vigolium agent audit --driver=piolium --source ./backend --mode confirm

# 全面搜索模式，逐文件扫描，按文件设置预算
vigolium agent audit --driver=piolium --source ./backend --mode longshot --plm-longshot-limit 200

# 使用强度预设重新运行（预设 → 模式 + 提交深度）
vigolium agent audit --driver=piolium --source ./backend --intensity deep
```

`vigolium agent audit --driver=piolium` 需要 `--source`——它审计源代码，而非网络流量。

---

## 前提条件

Piolium 作为 Pi 插件运行，因此主机上必须安装两个二进制文件：

1. **Pi 运行时**（[@earendil-works/pi-coding-agent](https://www.npmjs.com/package/@earendil-works/pi-coding-agent)）：

   ```bash
   bun install -g @earendil-works/pi-coding-agent
   pi --version
   ```

2. **Piolium 插件**（[github.com/vigolium/piolium](https://github.com/vigolium/piolium)）：

   ```bash
   pi install git:git@github.com:vigolium/piolium.git
   pi list   # 验证 "piolium" 出现
   ```

Vigolium **不会**自动安装 piolium。在任何子命令工作之前，`vigolium agent audit --driver=piolium` 按以下顺序解析活动的 piolium 安装，然后验证对应的 `settings.json` 在 `packages` 下列出了 piolium。如果未找到，则中止并给出可操作的安装提示——vigolium 从不写入用户设置或拉取 node_modules。

| 顺序 | 来源 | 使用的路径 | 运行时效果 |
|---|---|---|---|
| 1 | `$PIOLIUM_HOME` 环境变量（设置时） | `$PIOLIUM_HOME/agent` | Vigolium 将 `PI_CODING_AGENT_DIR=$PIOLIUM_HOME/agent` 注入 `pi` 子进程，使 Pi 从系统树加载 piolium。 |
| 2 | 自动探测 `/opt/piolium`（当 `agent/` 存在时） | `/opt/piolium/agent` | 与（1）相同的注入——当 piolium 部署在规范路径时，操作员无需记住导出环境变量。 |
| 3 | 按用户回退 | `~/.pi/agent` | 无环境注入；使用 Pi 自己的默认值。 |

Vigolium **不会**自动探测 `~/.piolium/`，即使那是 piolium 自己的独立启动器默认路径（`bin/piolium.mjs`）。`~/.piolium/` 下的按用户安装是操作员的独立 piolium 树；静默地将其与 vigolium 共享会导致审计状态跨上下文混合。要针对按用户安装驱动 vigolium，请显式设置 `export PIOLIUM_HOME=$HOME/.piolium`。

Vigolium 的每次扫描会话输出始终位于 vigolium 会话目录（`<vigolium-session>/pi-session/`），无论 `PIOLIUM_HOME` 如何解析——安装根目录管理 piolium 的 *agent 配置*，而非转录存储。

使用 `--debug` 运行以确认哪个路径胜出——审计日志行会输出 `piolium_agent_dir`（当（3）生效时为空），并在（1）或（2）下将 `PI_CODING_AGENT_DIR=...` 前置到渲染的命令行。

可选，用于更丰富的扫描（当这些不在 `$PATH` 上时，piolium 回退到 grep）：

- `trufflehog`、`gitleaks` — 密钥扫描
- `codeql`、`semgrep` — 静态分析

---

## 工作原理

当 `vigolium agent audit --driver=piolium` 运行时：

1. Vigolium 验证 `pi` 在 `$PATH` 上且 piolium 已在 `~/.pi/agent/settings.json` 中注册。
2. 在 `~/.vigolium/agent-sessions/<scan-uuid>/` 创建会话目录。
3. 解析 `--source`（本地路径、git URL、`gs://` 归档或本地归档），解析后的树成为 `pi` 子进程的工作目录。
4. 打印审计横幅（模式、来源、会话目录），使用户在任何网络活动之前看到运行上下文。
5. **预检：** 除非传递了 `--no-preflight`，否则 Vigolium 对配置的供应商/模型执行一次 `pi --mode json -p "..."` 往返。CLI 在成功时打印 `· Pi preflight check... ok provider=… model=… in 2.3s`，或在失败时中止并显示捕获的上游错误（例如 `No API key found for google-vertex. Use /login to log into a provider`）。这会在审计子进程生成之前捕获身份验证/配额问题。
6. 创建一个 `mode=piolium` 的子 `AgenticScan` 行。
7. Vigolium 生成 `pi --mode json -p "/piolium-<mode>" [--plm-* …]`，设置 `cmd.Dir = <resolved-source>` 并导出 `PIOLIUM_*` 环境变量。
8. Pi 加载 piolium 插件，将其输出写入 `<source>/piolium/`。
9. Vigolium 跟踪 Pi 的 `--mode json` 流：
   - **stdout JSONL** 由 `pkg/piolium/pistream` 解码为彩色活动信息流。
   - **原始行**持久化到 `<sessionDir>/audit-stream.jsonl`，供重放使用（`vigolium log <uuid>`）。
10. 每 30 秒，`<source>/piolium/audit-state.json` 和 `<source>/piolium/findings-draft/` 同步到 `<sessionDir>/piolium-audit/`，以便在运行过程中可见进度。
11. 当 `pi` 退出时，完整的 `<source>/piolium/` 树被复制到 `<sessionDir>/piolium-audit/`，漏洞发现被解析并导入数据库，源端的 `<source>/piolium/` 目录被删除。
12. 成本根据 Pi 的每个工作目录会话转录计算，并存储在 `AgenticScan` 行上。

```
+-----------------------------------------------------------+
|                  vigolium agent audit --driver=piolium                      |
|                                                            |
|  +------------------+    +-----------------------------+  |
|  |   前台            |    |  子进程（`pi --mode        |  |
|  |   Vigolium        |    |  json -p /piolium-<mode>`） |  |
|  |                   |    |                             |  |
|  |  解析来源          |    |  Pi 加载 piolium 插件       |  |
|  |  生成 pi          |--->|  阶段编排器运行              |  |
|  |  解码 JSONL       |<---|  生成子 Agent               |  |
|  |  每 30 秒同步     |<---|  写入 <source>/piolium/     |  |
|  |  解析漏洞发现     |<---|  退出                       |  |
|  +-------+-----------+    +-------------+---------------+  |
|          |                                                  |
|          v                                                  |
|  +-----------------------------------------------------+  |
|  |                     数据库                            |  |
|  |  findings（来源：scanner modules + piolium）          |  |
|  |  agentic_scans（mode=piolium, parent_uuid=...）      |  |
|  +-----------------------------------------------------+  |
+-----------------------------------------------------------+
```

---

## 审计模式

Piolium 通过 `--mode` 暴露 8 种审计模式。强度阶梯之外的审计模式（`merge`、`diff`、`confirm`、`revisit`、`longshot`）需要显式的 `--mode`。

| 模式 | 阶段数 | 阶段 ID | 用途 |
|---|---|---|---|
| `lite` | 4 | Q0–Q3 | 快速侦察、密钥扫描、快速 SAST |
| `balanced` | 9 | L1–L7（含 L6b/L6c） | 默认审计路径，含 PoC 和报告 |
| `deep` | 17 | P1–P17 | 完整审计，包括对抗性辩论、冷验证、变体搜索 |
| `revisit` | 9 | R5–R11c | 对现有审计进行反锚定的二次遍历 |
| `confirm` | 7 | V1–V7 | 确认现有漏洞发现仍然有效，可选附带测试 |
| `merge` | 7 | M1–M7 | 合并和去重先前运行的结果树 |
| `diff` | 1 | D1 | 扫描自审计提交以来变更的文件 |
| `longshot` | 3 | — | 逐个文件的全面漏洞搜索 |

操作员命令（`/piolium-status`、`/piolium-smoke`、`/piolium-export`、`/piolium-learn`）不通过 `vigolium agent audit --driver=piolium` 暴露——直接使用 `pi -p /piolium-<cmd>` 调用。它们不产生 vigolium 导入的漏洞发现，因此通过审计管道（会话同步、数据库标记、去重）路由它们只会增加噪音。

阶段 ID 空间有意与 `vigolium-audit`（Q*、L*、P*、V*、R*、M*）可互换，因此相同的解析器、漏洞发现格式和报告工具均可适用。

有关每个阶段的完整语义，请参见 piolium 的 [`docs/phase-reference.md`](https://github.com/vigolium/piolium/blob/main/docs/phase-reference.md) 和 [`docs/output-structure.md`](https://github.com/vigolium/piolium/blob/main/docs/output-structure.md)。

### 强度预设

`--intensity` 将审计模式和克隆深度捆绑到一个标志中，与 autopilot/swarm/audit 的强度模型匹配：

| 强度 | 模式 | 提交深度 | 使用场景 |
|---|---|---|---|
| `quick` | `lite` | 1（浅克隆） | CI/CD、例行分类 |
| `balanced` | `balanced` | 1（浅克隆） | 默认；PoC + 报告 |
| `deep` | `deep` | 0（完整历史） | 发布前、合规性、提交考古 |

显式的 `--mode` 和 `--commit-depth` 始终覆盖预设。

---

## CLI

```
vigolium agent audit --driver=piolium \
  [--mode {lite,balanced,deep,revisit,confirm,merge,diff,longshot}] \
  [--intensity {quick,balanced,deep}] \
  --source <path|git-url|gs://...|archive> \
  [--commit-depth N] \
  [--no-stream] \
  [--upload-results] \
  [--plm-* passthroughs]
```

### 标志参考

| 标志 | 描述 |
|---|---|
| `--source` | 必需。本地目录、git URL、`gs://<project>/<key>` 归档或本地 `.zip`/`.tar.gz`/`.tar.bz2`/`.tar.xz`。 |
| `--mode` | 审计模式。覆盖 `--intensity`。 |
| `--intensity` | 预设。`quick`/`balanced`/`deep`。默认为 `balanced`。 |
| `--commit-depth` | git URL 的 `git clone --depth`。`0` = 完整历史。覆盖 `--intensity`。 |
| `--no-stream` | 不在控制台输出。流仍持久化到 `runtime.log`，因此 `vigolium log <uuid>` 仍可工作。 |
| `--upload-results` | 完成后将会话捆绑包上传到云存储（需要存储配置）。 |
| `--pi-provider` | 覆盖本次运行的 pi `defaultProvider`（例如 `vertex-anthropic`、`google-vertex`）。通过 `pi --provider <name>` 传递。 |
| `--pi-model` | 覆盖本次运行的 pi `defaultModel`（例如 `claude-opus-4-6`、`gemini-3.1-pro`）。通过 `pi --model <id>` 传递。 |
| `--no-preflight` | 跳过审计前的 pi 往返（身份验证 + 模型可用性检查）。 |
| `--preflight-timeout` | 预检调用的超时时间（默认 30 秒）。 |
| `--api-key` / `--oauth-token` / `--oauth-cred-file` | 每次运行的 BYOK 身份验证覆盖。参见 [Audit BYOK](audit-byok.md)。对于 piolium，这些成为 `pi` 子进程的环境变量（或者对于 codex 凭据文件，临时暂存 `<pi-agent-dir>/auth.json`）。 |

### `--plm-*` 透传标志

这些标志一对一映射到 piolium 自己的 `--plm-*` 会话标志。空值或零值会被丢弃——当您不覆盖时，应用 piolium 的默认值。

| 标志 | Piolium 作用域 | Piolium 默认值 |
|---|---|---|
| `--plm-scan-limit N` | 限制提交历史扫描到 N 个提交 | 500 |
| `--plm-scan-since <expr>` | git `--since` 窗口（例如 `"60 days ago"`） | `"60 days ago"` |
| `--plm-phase-retries N` | 每阶段重试次数 | 2 |
| `--plm-command-retries N` | 每条命令重试次数 | 3 |
| `--plm-longshot-limit N` | longshot 模式下搜索的最大文件数 | 1000 |
| `--plm-longshot-timeout MS` | longshot 中每个文件的终止计时器 | 21600000（6 小时） |
| `--plm-longshot-langs <list>` | 逗号分隔的语言白名单 | 自动检测 |

### 示例

```bash
# 对远程 git URL 执行均衡审计，浅克隆
vigolium agent audit --driver=piolium \
  --source git@github.com:vigolium/piolium.git \
  --intensity balanced

# 深度审计，限制提交历史扫描范围
vigolium agent audit --driver=piolium \
  --source ./backend \
  --mode deep \
  --plm-scan-limit 250 \
  --plm-scan-since "90 days ago"

# 仅对 Python 和 Go 文件执行 longshot 搜索
vigolium agent audit --driver=piolium \
  --source ./mono-repo \
  --mode longshot \
  --plm-longshot-langs python,go \
  --plm-longshot-limit 200

# 固定扫描 UUID 用于跨节点同步
vigolium --scan-uuid 019aa... agent audit --source ./backend

# 为本次运行覆盖 pi 的供应商/模型
vigolium agent audit --driver=piolium --source ./backend \
  --pi-provider vertex-anthropic \
  --pi-model claude-opus-4-6

vigolium agent audit --driver=piolium --source ./backend \
  --pi-provider google-vertex \
  --pi-model gemini-3.1-pro
```

### 同时运行 piolium 和 audit

`--driver=piolium` 仅运行 piolium——不运行 audit，不进行框架回退。要在同一个 AgenticScan 下对同一个源代码树使用两个框架进行评分（每个驱动有独立的子行，并在通过后执行项目级漏洞发现去重），请使用其他驱动模式：

```bash
# 默认——依次运行 audit 然后 piolium
vigolium agent audit --source ./backend

# 仅 piolium——不运行 audit，不回退
vigolium agent audit --driver piolium --source ./backend --mode lite

# 仅 audit（等同于 `vigolium agent audit`）
vigolium agent audit --driver audit --source ./backend
```

当 `--driver=both` 时，模式必须在共享集合中（`lite`/`balanced`/`deep`/`revisit`/`confirm`/`merge`）。驱动特定的模式（piolium 的 `longshot`、audit 的 `mock`）需要 `--driver=piolium` 或 `--driver=audit`。

---

## REST API

`POST /api/agent/run/audit` 是**统一驱动端点**——它根据 `driver` 字段分发 audit 和/或 piolium。要仅运行 piolium，传递 `driver: "piolium"`。要仅运行 audit，使用 `driver: "audit"`（或直接访问 `POST /api/agent/run/audit`）。默认的 `driver: "both"` 在同一个 AgenticScan 下依次运行 audit 然后 piolium，每个驱动有独立的子行，并在通过后执行漏洞发现去重。生命周期和 `AgenticScan` 行结构与 [`/api/agent/run/audit`](vigolium-audit.md) 相同——现有的 `/agent/status/:id`、`/agent/sessions/:id/logs` 和 `/agent/sessions/:id/artifacts` 端点在两个框架上统一工作。

服务端处理器**不**运行 CLI 的预检往返（`pi --mode json` 在请求验证后立即启动）。身份验证/配额失败会出现在 SSE/runtime.log 流中。

### 请求体

| 字段 | 类型 | 描述 |
|---|---|---|
| `source` | string | **必需。** 本地路径、git URL、`gs://<bucket>/<key>` 归档（服务端自动下载+解压）或本地 `.zip`/`.tar.gz`/`.tar.bz2`/`.tar.xz`。使用 `driver: "both"` 时，source 解析一次，两个驱动重复使用。 |
| `target` | string | 可选的目标 URL，存储在运行行上用于交叉引用扫描。 |
| `intensity` | string | `quick` / `balanced`（默认）/ `deep`。捆绑 `mode` + `timeout` + `commit_depth`（quick → `lite`/1h/depth 1，balanced → `balanced`/6h/depth 1，deep → `deep`/12h/depth 0）。显式的 `mode`/`timeout`/`commit_depth` 按字段覆盖预设。 |
| `mode` | string | 审计模式覆盖。使用 `driver: "both"` 时，必须在共享集合中：`lite` / `balanced` / `deep` / `revisit` / `confirm` / `merge`。单驱动模式添加 `diff` 以及驱动特定值（piolium 的 `longshot`、audit 的 `status`/`mock`）。 |
| `timeout` | string | Go 持续时间（例如 `"2h"`）。覆盖强度预设。在 `driver: "both"` 下按驱动应用，因此 audit 挂起不会消耗 piolium 的预算。 |
| `diff` | string | 关注变更的代码：PR URL、git 引用范围或 `HEAD~N`。 |
| `last_commits` | int | 关注最近 N 个提交（`diff HEAD~N` 的简写）。 |
| `commit_depth` | int | git URL 的 `git clone --depth`。`0` = 完整历史。覆盖强度。 |
| `files` | string[] | 要关注的特定文件。 |
| `stream` | bool | 启用 Agent 信息流的 SSE 流式传输。使用 `driver: "both"` 时，事件带有 `driver` 字段标记，并由 `driver_start`/`driver_end` 标记包围。 |
| `upload_results` | bool | 完成后将会话捆绑包上传到云存储。使用 `driver: "both"` 时，如果任一驱动失败则跳过。 |
| `project_uuid` | string | 用于数据范围界定的项目 UUID。回退到 `X-Project-UUID` 标头。 |
| `scan_uuid` | string | 可选的扫描 UUID。 |
| `driver` | string | `"piolium"` / `"audit"` / `"both"`（默认）。使用 `"both"` 时，模式必须在共享集合中。 |
| `no_dedup` | bool | 跳过 `driver: "both"` 完成后运行的项目级漏洞发现去重。单驱动运行忽略此字段（它们已经在 INSERT 时通过 `finding_hash` 去重）。 |
| `agent` | string | audit 参与时的审计平台：`claude`（默认）/ `codex`。当 `driver: "piolium"` 时忽略。 |
| `pi_provider` | string | 转发为 `pi --provider <name>`。当 `driver: "audit"` 时忽略。 |
| `pi_model` | string | 转发为 `pi --model <id>`。 |
| `plm_scan_limit` | int | `--plm-scan-limit` 透传。 |
| `plm_scan_since` | string | `--plm-scan-since` 透传。 |
| `plm_phase_retries` | int | `--plm-phase-retries` 透传。 |
| `plm_command_retries` | int | `--plm-command-retries` 透传。 |
| `plm_longshot_limit` | int | `--plm-longshot-limit` 透传。 |
| `plm_longshot_timeout` | int | `--plm-longshot-timeout` 透传（毫秒）。 |
| `plm_longshot_langs` | string | `--plm-longshot-langs` 透传（逗号分隔）。 |

所有 `pi_*` 和 `plm_*` 字段都是可选的，仅在填充时发出其标志，与 CLI 行为一致。

> **`gs://` 需要存储配置。** 服务器通过 `pkg/storage` 使用配置的 GCS 凭据解析 `gs://<bucket>/<key>`（参见 `vigolium-configs.yaml` `storage.gcs.*`）。下载内容会放入临时目录，无论结果如何，运行后都会删除。如果 URI 可访问但不是可识别的归档格式，返回 `400`；如果下载本身失败，返回 `500`。

### 示例

```bash
# 默认——driver=both 且 intensity=balanced（先 audit 后 piolium，
# 两者退出后执行项目级漏洞发现去重）
curl -s -X POST http://localhost:9002/api/agent/run/audit \
  -H "Content-Type: application/json" \
  -d '{
    "source": "/home/user/src/my-app",
    "intensity": "balanced"
  }' | jq .

# 使用强度预设的快速分类——driver=both, mode=lite,
# commit_depth=1（服务端从预设解析）
curl -s -X POST http://localhost:9002/api/agent/run/audit \
  -H "Content-Type: application/json" \
  -d '{
    "source": "/home/user/src/my-app",
    "intensity": "quick",
    "driver": "both"
  }' | jq .

# 深度审计，完整 git 历史（intensity=deep → commit_depth=0）
# 固定为仅 piolium，并覆盖供应商
curl -s -X POST http://localhost:9002/api/agent/run/audit \
  -H "Content-Type: application/json" \
  -d '{
    "source": "git@github.com:org/repo.git",
    "intensity": "deep",
    "driver": "piolium",
    "pi_provider": "vertex-anthropic",
    "pi_model": "claude-opus-4-6"
  }' | jq .

# 来源来自 Google Cloud Storage——服务端下载+解压
# 归档一次，两个驱动重复使用解析后的树（无重复克隆）
curl -s -X POST http://localhost:9002/api/agent/run/audit \
  -H "Content-Type: application/json" \
  -d '{
    "source": "gs://vigolium-uploads/acme/backend-2026-05-02.tar.gz",
    "intensity": "balanced",
    "project_uuid": "11111111-2222-3333-4444-555555555555"
  }' | jq .

# gs:// + 仅 driver=audit（即使安装了 piolium 也跳过）
curl -s -X POST http://localhost:9002/api/agent/run/audit \
  -H "Content-Type: application/json" \
  -d '{
    "source": "gs://vigolium-uploads/acme/backend-2026-05-02.zip",
    "driver": "audit",
    "agent": "claude",
    "intensity": "balanced"
  }' | jq .

# 显式模式覆盖强度——longshot 是 piolium 独有的，因此
# driver 必须设置为 piolium（driver=both 会返回 400）
curl -s -X POST http://localhost:9002/api/agent/run/audit \
  -H "Content-Type: application/json" \
  -d '{
    "source": "/home/user/src/mono-repo",
    "driver": "piolium",
    "mode": "longshot",
    "plm_longshot_langs": "python,go",
    "plm_longshot_limit": 200
  }' | jq .

# 跳过通过后去重（仅 driver=both——单驱动运行
# 已在 INSERT 时通过 finding_hash 去重）
curl -s -X POST http://localhost:9002/api/agent/run/audit \
  -H "Content-Type: application/json" \
  -d '{
    "source": "/home/user/src/my-app",
    "intensity": "balanced",
    "no_dedup": true
  }' | jq .
```

### 响应

端点返回 `202 Accepted` 并带有**父**运行 UUID；通过 `/api/agent/sessions/<uuid>/logs`（支持 SSE）跟踪进度，或轮询 `/api/agent/status/<uuid>` 获取阶段计数器。使用 `driver: "both"` 时，子行通过 `/api/agent/sessions/<uuid>`（`child_runs[]`）在父行下暴露。

```json
{
  "agentic_scan_uuid": "b1e100e5-1131-4b41-995d-f0991534ac14",
  "status": "running",
  "message": "audit (driver=both) started"
}
```

单驱动运行（`driver: "piolium"` 或 `"audit"`）返回 `"message": "audit run started"`。

在请求中传递 `"stream": true` 以保持连接打开并接收 `chunk` / `error` / `done` 事件的 SSE 流，而不是 202 响应。

### 错误

| 状态码 | 原因 |
|---|---|
| 400 | 缺少 `source`；无效的 `mode`/`intensity`/`driver`；在 `driver: "both"` 下使用驱动特定模式（`longshot`/`mock`）；或 audit 参与时无效的 audit `agent`（`claude`/`codex`）。 |
| 429 | 重型 Agent 信号量已满——等待正在进行的审计完成后重试。 |
| 503 | **单个请求的驱动**运行时不可用：`driver: "piolium"` 在 `pi` 不在 `PATH` 上或 piolium 插件未在 `~/.pi/agent/settings.json` 中注册时返回 503；`driver: "audit"` 在配置的平台二进制文件（`claude`/`codex`）不在 `PATH` 上时返回 503。**`driver: "both"` 不会因缺少运行时返回 503**——缺少 `pi` 或平台二进制文件会成为服务器警告日志，可用的驱动仍然运行，失败的驱动在子运行上显示为按驱动错误（父行以 `completed_with_errors` 结束）。仅当两个驱动都不可用时请求才失败，这仍然产生 `202 Accepted`，随后是一个命名了两个驱动的 `completed_with_errors` 父行。 |

### 组合驱动 SSE 事件

当 `driver: "both"` 且 `stream: true` 时，audit 然后 piolium 的输出被多路复用到单个 SSE 流中。每个事件包含一个 `driver` 字段（`"audit"` 或 `"piolium"`）；`driver_start` 和 `driver_end` 事件包围每个驱动的时段，以便客户端可以渲染每个驱动的进度。`driver_end` 上的 `error` 字段携带按驱动的失败信息（如果有），最终 `done` 事件仅在两个驱动都无错误完成时触发。

```bash
# 运行两个驱动，多路复用 SSE
curl -N -s -X POST http://localhost:9002/api/agent/run/audit \
  -H "Content-Type: application/json" \
  -d '{
    "source": "/home/user/src/my-app",
    "mode": "balanced",
    "driver": "both",
    "stream": true
  }'
```

---

## 管道内审计（autopilot / swarm）

`vigolium agent autopilot` 和 `vigolium agent swarm` 可以将 piolium 作为管道内审计运行，与操作员 Agent 并行运行（并将漏洞发现输入其中）。历史上默认使用 audit 的同一 CLI 现在在 piolium 本地可用时自动选择它。

### 自动选择

当设置了 `--source` 且**既未**传递 `--audit` **也未**传递 `--piolium` 时：

- 如果 `pi` 在 `$PATH` 上**且** piolium 插件已在 `~/.pi/agent/settings.json` 中注册 → **piolium** 运行（模式 = 强度预设选择的任何模式，默认为 `lite`）。
- 否则 → **audit** 运行（现有默认值）。

这保留了在没有 Pi 的机器上的先前行为，同时在用户安装 piolium 后自动启用它。

### 显式覆盖

| 标志组合 | 结果 |
|---|---|
| `--piolium`（显式，任意模式） | Piolium 以该模式运行；audit 关闭。 |
| `--audit`（显式，任意模式） | Audit 以该模式运行；piolium 关闭。 |
| `--audit=off --piolium=off` | 无审计。 |

每次扫描只运行一个框架——没有"同时运行两者"的模式。如果您想在同一个源上比较两个框架，请运行两次扫描。

### REST API 等价物

`POST /api/agent/run/autopilot` 和 `POST /api/agent/run/swarm` 都接受一个新的 `piolium` 字段，完全镜像 CLI 标志。相同的自动选择规则在服务端应用：省略 `audit` 和 `piolium` 两者时，如果**服务器进程**安装了 pi+piolium 则触发 piolium，否则触发 audit。

```bash
# Autopilot——显式 piolium 审计
curl -s -X POST http://localhost:9002/api/agent/run/autopilot \
  -H "Content-Type: application/json" \
  -d '{
    "target": "https://example.com",
    "source": "/home/user/src/my-app",
    "piolium": "balanced"
  }' | jq .

# Swarm——在定向扫描期间选择加入 piolium 审计
curl -s -X POST http://localhost:9002/api/agent/run/swarm \
  -H "Content-Type: application/json" \
  -d '{
    "input": "https://example.com/api/users?id=1",
    "source": "/home/user/src/my-app",
    "discover": true,
    "piolium": "lite"
  }' | jq .

# Autopilot——让自动选择决定；此主机上有 pi+piolium 意味着 piolium 胜出
curl -s -X POST http://localhost:9002/api/agent/run/autopilot \
  -H "Content-Type: application/json" \
  -d '{
    "target": "https://example.com",
    "source": "/home/user/src/my-app"
  }' | jq .
```

### 漏洞发现流程

无论哪个框架运行：

- 审计的 `audit-state.json` 和 `findings/` 树同步到 `<sessionDir>/<harness.SessionSubdir>/`（`vigolium-results/` 或 `piolium-audit/`）。
- Autopilot 管道在启动操作员 Agent 之前阻塞等待审计完成，然后将漏洞发现折叠到操作员的冻结上下文包中（与 audit 使用的流程相同）。
- 漏洞发现进入数据库，标记为 `finding_source = "audit"` 或 `"piolium"`。在查询中混合使用：
  ```bash
  vigolium finding list --source piolium,audit
  ```

### 注意事项

- **Pi 特定的参数**（`--pi-provider`、`--pi-model`、`--plm-*`）**不**在 autopilot/swarm 接口上暴露。如果您需要这些，直接运行 `vigolium agent audit --driver=piolium`（或 `POST /api/agent/run/audit`）。
- 自动选择是主机本地的。如果您的 CLI 机器安装了 pi 但**服务器**运行 autopilot 没有（或反之），自动选择决定在请求处理器运行的位置做出。

---

## 流式输出

Pi 的 `--mode json` 标志向 stdout 输出换行分隔的 `AgentSessionEvent` 对象。Vigolium 的 `pkg/piolium/pistream` 解码器消费它们并渲染紧凑的彩色活动信息流。

### 渲染的事件类型

| 事件 | 渲染 |
|---|---|
| `session` | 一行标题，包含会话 ID 和工作目录 |
| `agent_start` / `agent_end` | 运行生命周期标记；`agent_end` 包含经过的持续时间 |
| `message_end`（助手） | 最终助手文本，以及身份验证/配额失败时的错误消息 |
| `tool_execution_start` | `→ tool_name (args)` |
| `tool_execution_end` | `← tool_name result`（错误时显示 `✗`） |
| `auto_retry_start` / `auto_retry_end` | 带退避和原因的重试尝试 |
| `compaction_start` / `compaction_end` | 上下文压缩周期 |

`turn_start`、`turn_end`、`message_start`、`message_update`、`tool_execution_update`、`queue_update` 和 `session_info_changed` 被有意抑制——它们要么与 `*_end` 事件冗余，要么仅用于 UI 状态。

原始 JSONL 会镜像到 `<sessionDir>/audit-stream.jsonl`，无论 `--no-stream` 如何设置，因此 `vigolium log <uuid>` 可以在事后重放精确的事件流。

---

## 成本追踪

Pi 预先对每个助手轮次进行定价，并将结果写入其会话转录。Vigolium 的 `pkg/piolium/picost` 在运行后提取这些数据。

### 成本提取方式

1. 在 `pi` 退出后，`picost.BuildSummary` 定位 `~/.pi/agent/sessions/<encoded-cwd>/`，其中 `<encoded-cwd>` 是 `--<path>--`，`/` 替换为 `-`。

   示例：`/Users/alice/Desktop/repo` → `--Users-alice-Desktop-repo--`

2. 扫描每个 `<timestamp>_<sessionid>.jsonl` 转录，其 `session` 头部时间戳在 `[startedAt - 30s, startedAt + 24h]` 范围内。
3. 每个助手 `message` 事件携带：
   ```json
   "usage": {
     "input": 4254, "output": 14, "cacheRead": 0, "cacheWrite": 0, "totalTokens": 4268,
     "cost": { "input": 0.02127, "output": 0.00042, ..., "total": 0.02169 }
   }
   ```
   令牌数和 `cost.total` 在所有归属的转录中求和。
4. 唯一助手轮次是 401 错误（身份验证失败）的转录被丢弃——它们成本为零，且会扭曲模型归属。

### 输出

CLI 摘要追加：

```
ℹ Cost: ~$0.02 (model gpt-5.5)
```

多会话运行（深度模式、longshot）注释为 `(model gpt-5.5, 23 sessions)`。

`AgenticScan` 数据库行填充以下字段：

| 字段 | 来源 |
|---|---|
| `total_input_tokens` | `usage.input` 之和 |
| `total_output_tokens` | `usage.output` 之和 |
| `estimated_cost_usd` | `usage.cost.total` 之和 |
| `token_usage` | 完整 `picost.Summary` JSON（含按会话细分） |

与 `claudecost` 和 `codexcost` 不同，`picost` 没有本地定价表——Pi 在消息落地到转录时已经根据活跃供应商的费率计算了价格。

---

## 设置

### YAML 配置

Piolium 重用现有的 `agent.sessions_dir` 和存储设置。`vigolium-configs.yaml` 中没有 piolium 特定的配置块——该框架随 Pi 一起提供，而非 Vigolium。

```yaml
agent:
  sessions_dir: ~/.vigolium/agent-sessions/   # 会话产物根目录
```

Pi 端的配置（供应商、模型、身份验证）位于 `~/.pi/agent/settings.json`，通过 `pi` 本身设置：

```bash
pi config set defaultProvider openai-codex
pi config set defaultModel gpt-5.5
```

piolium 插件还从会话中读取自己的 `--plm-*` 标志，Vigolium 从 `vigolium agent audit --driver=piolium` 上匹配的 `--plm-*` 标志逐字传递。

### 导出到 `pi` 的环境变量

Vigolium 在启动前复制 piolium 期望的环境变量契约：

| 变量 | 来源 |
|---|---|
| `PIOLIUM_REPOSITORY` | 解析的仓库标识（git 远程 URL 规范化为 `owner/repo`，否则为目录基名） |
| `PIOLIUM_GIT_AVAILABLE` | 当 `<source>` 是 git 工作树时为 `true` |
| `PIOLIUM_SESSION_UUID` | 本次运行的 Vigolium `AgenticScan` UUID |
| `PIOLIUM_COMMIT_SCAN_LIMIT` | 当 `--plm-scan-limit > 0` 时设置 |
| `PIOLIUM_COMMIT_SCAN_SINCE` | 当 `--plm-scan-since` 非空时设置 |

---

## 会话产物

### 源目录（临时，导入后删除）

```
<source_path>/
└── piolium/
    ├── audit-state.json               # 阶段进度（snake_case 键，与 audit 兼容）
    ├── attack-surface/                # 侦察、知识库、SAST、探测摘要
    │   ├── lite-recon.md
    │   ├── knowledge-base-report.md
    │   ├── sast-merged.sarif
    │   ├── source-sink-flows-all-severities.md
    │   ├── public-routes-authz-matrix.md
    │   └── ...
    ├── findings-draft/                # 候选漏洞发现（运行过程中提交）
    │   ├── q2-001-sql-injection-user-lookup.md
    │   └── ...
    ├── findings/                      # 最终合并的漏洞发现
    │   ├── p10-001-direct-git-url-ref-…/
    │   │   ├── draft.md               # 前置元数据
    │   │   ├── report.md              # 精炼的结构化分析
    │   │   ├── poc.{sh,ts,py,…}       # 可执行的验证证明
    │   │   └── evidence/              # 支持文件
    │   └── ...
    ├── final-audit-report.md          # 顶级报告
    ├── confirmation-report.md         # confirm 模式输出
    ├── chamber-workspace/             # 对抗性辩论转录（深度模式）
    ├── codeql-artifacts/              # CodeQL 数据库、SARIF
    ├── semgrep-rules/
    └── tmp/                           # 每个子 Agent 的运行转录（清理后）
```

### Vigolium 会话目录（持久化）

```
~/.vigolium/agent-sessions/<uuid>/
├── piolium-audit/                     # 从 <source>/piolium/ 同步
│   ├── audit-state.json
│   ├── attack-surface/
│   ├── findings/
│   ├── final-audit-report.md
│   └── ...
├── audit-stream.jsonl                 # 原始 Pi --mode json 事件
├── piolium-audit-output.md            # 捕获的 stdout（流为空时的回退）
└── runtime.log                        # 可通过 `vigolium log <uuid>` 重放的信息流
```

---

## 漏洞发现格式

Piolium 的漏洞发现格式与 audit 相同——相同的 YAML 前置元数据、相同的正文约定、相同的冷验证覆盖模式。唯一的命名区别是**目录布局**：piolium 在其提升的漏洞发现上保留源阶段 ID（`p10-001-<slug>/`），而不是重新编号为严重性字母格式（audit 中的 `C1-<slug>/`）。Vigolium 的解析器处理两种格式。

### 前置元数据（小写 YAML，piolium 风格）

```markdown
---
id: p10-001
phase: P10
source-draft: p4-001
slug: direct-git-url-ref-reaches-simple-git-clone
severity: high
verdict: VALID
debate: piolium/chamber-workspace/c01-git-and-lock-command/debate.md
---
PoC-Status: executed
Protocol: local
Auth-Required: no

# Direct git URL/ref reaches vulnerable simple-git clone boundary

## Summary
…

## Location
- Source: `src/add.ts:895-942`
- Sink:   `src/git.ts:25-62`

## Impact
…

## Evidence
…
```

前置元数据解析器在字段名称上不区分大小写：audit 的 `Phase`/`Severity-Final` 和 piolium 的 `phase`/`severity` 都可以工作。Piolium 的单个 `severity` 字段映射到 audit 的 `SeverityFinal` 槽位，因此现有的"优先使用最终值而非原始值"解析仍然适用。

完整模式请参见 [vigolium-audit 的漏洞发现格式部分](vigolium-audit.md#finding-format)——两个框架共享它。

---

## 漏洞发现导入

审计完成后，漏洞发现会自动解析并存储在 Vigolium 数据库中。

### 数据库字段

| Piolium 字段 | 数据库字段 | 示例 |
|---|---|---|
| 漏洞发现 ID | `module_id` | `piolium:p10-001` |
| 标题（来自 H1 或 slug） | `module_name` | Direct git URL reaches simple-git clone |
| Slug | `module_short` | `direct-git-url-ref-reaches-…` |
| 严重性（最终） | `severity` | `high`（标准化） |
| 判定 | `confidence` | `firm`（`CONFIRMED`/`VALID`）或 `tentative` |
| CWE | `cwe_id` | `CWE-918` |
| 完整正文 | `description` | 带证据的 Markdown |
| 第一个位置 | `source_file` | `src/add.ts` |
| 所有位置 | `matched_at` | `src/add.ts:895-942` |
| 元数据 | `tags` | `["piolium", "phase-P10", "valid", "poc-executed", "CWE-918"]` |

所有漏洞发现都存储为：

- `finding_source`：`piolium`
- `module_type`：`whitebox`
- `finding_hash`：`MD5(auditID + moduleID + findingID)` 用于去重

关联的 `AgenticScan` 行携带：

- `mode`：`piolium`
- `agent_name`：`piolium`
- `protocol`：`pi-sdk`
- `input_type`：`piolium`

### 查询 piolium 漏洞发现

```bash
# 通过 CLI
vigolium finding list --source piolium

# 通过 API
GET /api/findings?source=piolium

# 与 audit 合并
vigolium finding list --source piolium,audit
```

### 手动导入

来自外部运行的 piolium 输出通过与 audit 相同的路径导入——解析器检测 `findings/p<phase>-<seq>-<slug>/` 目录布局并正确标记行：

```bash
vigolium import /path/to/piolium-output/
```

文件夹必须包含 `audit-state.json` 以及 `findings/` 或 `findings-draft/`。

---

## 对比：piolium vs audit

| 方面 | `vigolium agent audit` | `vigolium agent audit --driver=piolium`（piolium） |
|---|---|---|
| **运行时** | Claude Code 或 Codex | Pi（`pi` CLI + piolium 插件） |
| **安装模型** | 嵌入式框架，运行时自动解压 | 用户通过 `pi install` 安装 piolium |
| **斜杠命令** | `/vigolium-audit:audit:<mode>` | `/piolium-<mode>` |
| **输出文件夹** | `<source>/vigolium-results/` → `<sessionDir>/vigolium-results/` | `<source>/piolium/` → `<sessionDir>/piolium-audit/` |
| **环境变量前缀** | `ARCHON_*` | `PIOLIUM_*` |
| **会话标志前缀** | 无 | `--plm-*`（在 `pi` argv 上透传） |
| **模式** | `lite`/`balanced`/`deep`/`revisit`/`confirm`/`merge`/`diff`/`status`/`mock` | `lite`/`balanced`/`deep`/`revisit`/`confirm`/`merge`/`diff`/`longshot` |
| **成本来源** | `claudecost`（audit-stream.jsonl）、`codexcost`（~/.codex/sessions） | `picost`（~/.pi/agent/sessions） |
| **数据库标记** | `mode=audit`、`module_id=audit:…` | `mode=piolium`、`module_id=piolium:…` |
| **audit-state.json** | 相同模式（snake_case 键，可互换） | 相同模式 |
| **漏洞发现格式** | 相同（YAML 前置元数据 + 冷验证覆盖） | 相同 |
| **提升的漏洞发现布局** | `findings/C1-<slug>/`（严重性前缀） | `findings/p<phase>-<seq>-<slug>/`（阶段前缀） |
| **扫描期间后台运行** | 是——`swarm`/`autopilot` 上的 `--audit` 标志 | 否——仅前台驱动 |

两者有意设计为可互操作。您可以运行 piolium 审计并将输出导入到具有 audit 标记的漏洞发现的项目中；解析器、漏洞发现去重和报告工具都统一适用。

---

## 与本地扫描的对比

| 方面 | Vigolium 本地扫描（Swarm/Autopilot） | Piolium |
|---|---|---|
| **关注点** | 网络漏洞（注入、XSS、SSRF 等） | 源代码漏洞（逻辑缺陷、身份验证漏洞、规范违规） |
| **方法** | 带载荷的实时 HTTP 扫描 | 静态分析 + AI 推理 + 对抗性验证 |
| **误报处理** | AI 分类阶段 | 多层：对抗性辩论室 + 冷验证（深度模式） |
| **漏洞发现丰富度** | 标准严重性/置信度 | 对抗性判定、冷验证覆盖、CWE、PoC 状态 |
| **速度** | 数分钟到数小时 | 数分钟（lite，约 1 小时）到数小时（deep，约 6 小时） |
| **需要** | 目标 URL | 源代码路径 |
| **运行方式** | 前台（主管道） | 前台子命令 |

两种方法互为补充。网络扫描发现 HTTP 响应中显现的漏洞；piolium 发现需要理解代码语义、业务逻辑和规范合规性的漏洞。同时运行两者——对已部署实例执行 `vigolium agent swarm -t https://…`，然后对源代码执行 `vigolium agent audit --driver=piolium --source ./repo`——提供最全面的评估。

---

## 端到端测试

Piolium 审计路径有一个可选的端到端测试，位于 [`test/e2e/piolium_audit_e2e_test.go`](../../test/e2e/piolium_audit_e2e_test.go)，它对一个微小的合成 Python 测试夹具运行 `pi --mode json -p "/piolium-lite"`，并断言会话产物、数据库行和 JSONL 流都已生成。

```bash
# 默认（使用 pi 配置的 defaultProvider/defaultModel）
make test-e2e-piolium

# 覆盖本次运行的供应商/模型
VIGOLIUM_E2E_PI_PROVIDER=vertex-anthropic \
VIGOLIUM_E2E_PI_MODEL=claude-opus-4-6 \
  make test-e2e-piolium

VIGOLIUM_E2E_PI_PROVIDER=google-vertex \
VIGOLIUM_E2E_PI_MODEL=gemini-3.1-pro \
  make test-e2e-piolium

# 保留会话目录以供检查
VIGOLIUM_E2E_PI_KEEP=1 \
  make test-e2e-piolium

# 自定义超时（默认为 5 分钟）
VIGOLIUM_E2E_PI_TIMEOUT=10m make test-e2e-piolium
```

使用的环境变量：

| 变量 | 默认值 | 效果 |
|---|---|---|
| `VIGOLIUM_E2E_PI_PROVIDER` | （pi 的 `defaultProvider`） | 为测试运行设置 `--pi-provider` |
| `VIGOLIUM_E2E_PI_MODEL` | （pi 的 `defaultModel`） | 为测试运行设置 `--pi-model` |
| `VIGOLIUM_E2E_PI_TIMEOUT` | `5m` | 审计硬超时（Go 持续时间语法） |
| `VIGOLIUM_E2E_PI_KEEP` | 未设置 | 当为 `1` 时，测试退出后保留会话目录在磁盘上 |

当 `pi` 不在 `$PATH` 上或 piolium 未在 `~/.pi/agent/settings.json` 中注册时，测试会自动跳过。还有一个不会跳过的伴随子测试 `TestE2EPioliumAudit_PreArgsRoundtrip`，它证明 argv 构建器为每个供应商生成精确的 `pi --provider X --model Y --mode json -p /piolium-<mode>` 形状——用于在 CI 中捕获标志管道回归问题，无需实时凭据。
