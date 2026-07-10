# Audit-Audit

Vigolium Audit 是 Vigolium 内置的**多阶段白盒安全审计引擎**。在 Swarm 中，它作为后台进程与主扫描并行运行；在 Autopilot 中，它首先运行，然后将其输出准备为稳定的操作上下文，再启动自主 Agent。漏洞发现会自动导入 Vigolium 数据库，与本地扫描结果一同存储。

Vigolium Audit 取代了旧版 vig-audit-agent，提供了更丰富的漏洞发现格式（YAML 前置元数据、对抗性裁决、冷验证覆盖层）以及更强大的多阶段流水线。

## 目录

- [Quick Start](#quick-start)
- [How It Works](#how-it-works)
- [Audit Modes](#audit-modes)
- [CLI](#cli)
- [API](#api)
- [Manual Import](#manual-import)
- [Configuration](#configuration)
- [Session Artifacts](#session-artifacts)
- [Finding Format](#finding-format)
- [Finding Ingestion](#finding-ingestion)
- [Architecture](#architecture)
- [Comparison with Native Scanning](#comparison-with-native-scanning)

---

## Quick Start

```bash
# Swarm 带后台 vigolium-audit（精简模式，默认）
vigolium agent swarm -t https://example.com --source ./src --audit

# Swarm 带深度 12 阶段审计
vigolium agent swarm -t https://example.com --source ./src --audit deep

# Autopilot 先运行 vigolium-audit
vigolium agent autopilot -t https://example.com --source ./src --audit balanced

# 显式禁用（覆盖配置）
vigolium agent swarm -t https://example.com --source ./src --audit off

# 导入之前运行的审计输出
vigolium import /path/to/audit-output/
```

Vigolium Audit 需要 `--source` —— 它审计的是源代码，而非网络流量。

---

## How It Works

当设置了 `--audit` 并提供了 `--source` 时：

1. Vigolium 将内置的 vigolium-audit 工具集（Agent、命令、技能）提取到 `~/.vigolium/vigolium-audit/`
2. 启动一个**独立的 Claude Code 进程**，加载审计插件，目标指向源代码目录
3. 审计 Agent 独立运行其自身的多阶段流水线
4. 审计状态和漏洞发现被复制到 Vigolium 会话目录
5. 进度记录在子 `AgenticScan` 记录中（mode=`audit`），与父运行关联
6. 审计完成后，漏洞发现被解析并导入 Vigolium 数据库
7. 在 Autopilot 中，审计输出随后被准备为稳定上下文和原生计划，然后操作员启动
8. `<source>/audit/` 目录被移除（副本保留在会话目录中）
9. 如果前台运行先被取消，审计进程会通过 SIGTERM 优雅取消（10 秒宽限期）

```
+---------------------------------------------------------------+
|                  vigolium agent swarm/autopilot                |
|                                                                |
|  +--------------+    +-------------------------------------+  |
|  |  前台进程     |    |  后台进程（独立进程）                |  |
|  |               |    |                                      |  |
|  |  Swarm/       |    |  claude --plugin-dir <audit>        |
|  |  Autopilot    |    |  /vigolium-audit:audit:{mode}         |
|  |  流水线       |    |                                      |  |
|  |               |    |  P1: 提交考古学                      |  |
|  |  normalize    |    |  P2: 补丁绕过分析                    |  |
|  |  source-      |    |  P3: 知识库 + 威胁模型               |  |
|  |   analysis    |    |  P4: 静态分析 (CodeQL+Semgrep)       |  |
|  |  code-audit   |    |  P5: 深度探测 + Bug 狩猎             |  |
|  |  discover     |    |  P6: 规范差距分析                    |  |
|  |  plan         |    |  P7: 丰富化 + 过滤                   |  |
|  |  scan         |    |  P8: 对抗性辩论室                    |  |
|  |  triage       |    |  P9: 冷验证                          |  |
|  |               |    |  P10: 变体狩猎                       |  |
|  |               |    |  P11: PoC + 报告汇编                 |  |
|  |               |    |                                      |  |
|  |               |    |  -- 每 30 秒同步状态 -->             |  |
|  |               |    |  -- 完成后导入漏洞发现 -->           |  |
|  +-------+------+    +------------------+-------------------+  |
|          |                              |                      |
|          v                              v                      |
|  +-----------------------------------------------------+      |
|  |                     数据库                             |      |
|  |  findings (source: scanner modules + audit)          |      |
|  |  http_records, agentic_scans                             |      |
|  +-----------------------------------------------------+      |
+---------------------------------------------------------------+
```

---

## Audit Modes

### Lite（3 阶段）

快速流水线，针对 CI/CD 和常规扫描优化。运行快速侦察、密钥扫描和快速 SAST。

| 阶段 | 名称 | 描述 |
|------|------|------|
| Q0 | 快速侦察 | 架构清单、依赖审计 |
| Q1 | 密钥扫描 | 凭据和密钥检测 |
| Q2 | 快速 SAST | 快速 CodeQL + Semgrep 结构扫描 |

### Balanced（9 阶段）

中级审计，包含 SAST、探测和验证。运行 9 个阶段——旧版 `scan` 值映射为 `balanced`。核心阶段如下：

| 阶段 | 名称 | 描述 |
|------|------|------|
| 1 | 情报 | CVE/GHSA/OSV 狩猎、依赖审计、架构清单 |
| 2 | 知识库 | 威胁模型、领域攻击研究、RFC 规范 |
| 3 | SAST | CodeQL 结构 + 安全扫描、Semgrep（与 P4 并行） |
| 4 | 探测 | 高风险区域定向深度分析（与 P3 并行） |
| 5 | 审查 + 误报 | 内联验证 + 误报消除 |
| 6 | PoC + 报告 | 概念验证生成和咨询式报告 |

### Deep（12 阶段）

全面审计，包含对抗性审查室。最适合发布前审计、合规或高价值目标。运行 12 个阶段（deep 在 `balanced` 基础上增加了专门的授权检查、提交考古学和补丁绕过分析）；旧版 `full` 模式映射为 `deep`。核心阶段如下：

| 阶段 | 名称 | 描述 |
|------|------|------|
| P1 | 提交考古学 | 分析 git 历史，查找静默安全修复、未公开的 CVE |
| P2 | 补丁绕过 | 测试补丁完整性，寻找替代利用路径 |
| P3 | 知识库 | 构建架构模型、信任边界、攻击面地图 |
| P4 | 静态分析 | CodeQL + Semgrep 配合自定义规则 |
| P5 | 深度探测 | 使用专用 Agent 进行多假设探测 |
| P6 | 规范差距分析 | 发现规范/文档与实现之间的差距 |
| P7 | 丰富化与过滤 | 通过可达性分析和数据流丰富 SAST 漏洞发现 |
| P8 | 对抗性辩论 | 多 Agent 辩论室验证/反驳漏洞发现 |
| P9 | 冷验证 | 独立的零上下文重新验证 |
| P10 | 变体狩猎 | 搜索已确认漏洞的变体 |
| P11 | 报告汇编 | PoC 构建和咨询式最终报告 |

---

## CLI

### 标志：`--audit`

适用于 `vigolium agent swarm` 和 `vigolium agent autopilot`。

| 值 | 行为 |
|-----|------|
| *(未设置)* | 禁用（除非在配置中启用） |
| `--audit` | Lite 模式（3 阶段快速审计） |
| `--audit lite` | Lite 模式（显式） |
| `--audit balanced` | Balanced 模式（9 阶段中级审计） |
| `--audit deep` | Deep 模式（12 阶段全面审计） |
| `--audit off` | 禁用（覆盖配置） |

### 示例

```bash
# Swarm：定向扫描 + 后台精简审计
vigolium agent swarm \
  -t https://example.com/api \
  --source ./backend \
  --audit

# Swarm：全范围扫描 + 深度审计
vigolium agent swarm \
  -t https://example.com \
  --source ./backend \
  --discover \
  --audit deep

# Autopilot：自主扫描 + 平衡模式审计
vigolium agent autopilot \
  -t https://example.com \
  --source ./backend \
  --audit balanced

# 即使配置启用了审计，也禁用
vigolium agent swarm \
  -t https://example.com \
  --source ./backend \
  --audit off
```

---

## API

`audit` 字段在 swarm 和 autopilot 运行端点上都可用。

### 字段参考

| 字段 | 类型 | 描述 |
|------|------|------|
| `audit` | string | `"lite"`、`"balanced"`、`"deep"`、`"off"`，或省略以使用配置默认值 |

### POST /api/agent/run/swarm

```bash
# Swarm 带精简 vigolium-audit
curl -s -X POST http://localhost:9002/api/agent/run/swarm \
  -H "Content-Type: application/json" \
  -d '{
    "input": "https://example.com",
    "source": "/home/user/src/my-app",
    "discover": true,
    "audit": "lite"
  }' | jq .

# Swarm 带深度 12 阶段 vigolium-audit
curl -s -X POST http://localhost:9002/api/agent/run/swarm \
  -H "Content-Type: application/json" \
  -d '{
    "input": "https://example.com",
    "source": "/home/user/src/my-app",
    "discover": true,
    "code_audit": true,
    "audit": "deep"
  }' | jq .
```

### POST /api/agent/run/autopilot

```bash
# Autopilot 带深度 vigolium-audit
curl -s -X POST http://localhost:9002/api/agent/run/autopilot \
  -H "Content-Type: application/json" \
  -d '{
    "target": "https://example.com",
    "source": "/home/user/src/my-app",
    "audit": "deep"
  }' | jq .
```

### 响应

两个端点都返回 `202 Accepted` 并附带运行 ID。Vigolium Audit 作为 Agent 运行中的后台进程运行——其进度记录在子 `AgenticScan` 记录中。漏洞发现在完成后导入数据库。

```bash
# 运行完成后查询审计漏洞发现
curl -s http://localhost:9002/api/findings?source=audit | jq .
```

---

## Manual Import

外部运行的审计输出可以直接导入，无需运行 swarm 或 autopilot：

```bash
vigolium import /path/to/audit-output-harbor/
```

文件夹必须包含 `audit-state.json` 和 `findings/`。导入过程：

1. 解析 `audit-state.json` 以获取阶段跟踪和元数据
2. 从 `findings/` 读取所有漏洞发现文件
3. 应用冷验证覆盖层（如果存在 `*.cold-verify.md` 文件）
4. 创建 `AgenticScan` 记录（mode=`audit`）
5. 保存漏洞发现并去重（按漏洞发现哈希跳过重复项）
6. 报告计数：总漏洞发现数、已保存数、跳过的重复数、严重性分布

---

## Configuration

### YAML 配置

```yaml
agent:
  audit:
    enable: false              # 默认启用（可通过 --audit off 覆盖）
    mode: lite                 # 默认模式：lite、scan 或 deep
    plugin_dir: ""             # 自定义工具集路径（默认：~/.vigolium/vigolium-audit/）
    sync_interval: 30          # 状态同步间隔（秒）
```

### 优先级

1. CLI `--audit <value>` / API `"audit": "<value>"` —— 最高优先级
2. 配置 `agent.audit.enable: true` —— 当 CLI/API 未指定时使用
3. `--audit off` / `"audit": "off"` —— 覆盖配置

### 工具集解析

vigolium-audit 工具集（Agent、命令、技能）按以下顺序解析：

1. **配置 `plugin_dir`** —— 如果设置且存在，直接使用
2. **默认路径** `~/.vigolium/vigolium-audit/` —— 接下来检查
3. **内置提取** —— 如果两者都不存在，则自动提取 Vigolium 二进制文件中捆绑的工具集。版本哈希标记检测更改以重新提取

无需手动安装——所有内容都嵌入在 Vigolium 二进制文件中。

---

## Session Artifacts

Vigolium Audit 将输出写入源目录下的 `audit/`，然后同步到会话目录：

### 源目录（临时，导入后删除）

```
<source_path>/
└── audit/
    ├── audit-state.json              # 阶段进度跟踪
    ├── findings/                     # 每个漏洞发现的 Markdown 文件
    │   ├── p7-001-open-redirect.md   # 阶段 7 漏洞发现
    │   ├── p8-001-ssrf-webhook.md    # 阶段 8 漏洞发现
    │   ├── p8-001-ssrf.cold-verify.md  # 冷验证覆盖层
    │   ├── p10-041-variant.md        # 变体漏洞发现
    │   └── ...
    ├── knowledge-base-report.md
    ├── final-audit-report.md
    ├── advisory-report.md
    ├── spec-gap-report.md
    └── attack-pattern-registry.json
```

### 会话目录（持久化）

```
~/.vigolium/agent-sessions/<uuid>/
├── vigolium-results/                   # 从源目录同步
│   ├── audit-state.json
│   ├── findings/
│   ├── final-audit-report.md
│   ├── attack-pattern-registry.json
│   └── ...
├── vigolium-audit-output.md            # 原始 Claude Code 进程输出
├── output.md                         # 主 Agent 输出
├── skills/
│   └── vigolium-scanner/
└── CLAUDE.md
```

---

## Finding Format

审计根据阶段生成两种漏洞发现格式。

### 阶段 7 漏洞发现（基于表格）

早期阶段的漏洞发现使用 Markdown 表格格式：

```markdown
# Phase 7 Enriched Finding: P7-001

## Finding Details

| 字段 | 值 |
|------|-----|
| **Finding ID** | P7-001 |
| **Title** | Open Redirect via Unvalidated postURI |
| **Severity** | HIGH |
| **Confidence** | HIGH |
| **CWE** | CWE-601 (URL Redirection to Untrusted Site) |

PoC-Status: theoretical

## Code Location

**File**: `src/core/controllers/authproxy_redirect.go`
**Lines**: 73-77

[Detailed analysis...]
```

### 阶段 8+ 漏洞发现（基于前置元数据）

后期阶段的漏洞发现使用结构化键值前置元数据，并包含对抗性裁决：

```markdown
Phase: 8
Sequence: 001
Slug: admin-db-auth-brute-force
Verdict: VALID
Severity-Original: HIGH
Severity-Final: MEDIUM
PoC-Status: pending
Adversarial-Verdict: CONFIRMED
Adversarial-Rationale: IsSuperUser forces DB auth unconditionally...

## Summary

Harbor's admin account bypasses account lockout...

## Location

- `src/core/auth/authenticator.go:142`
- `src/core/auth/lock.go:22-51`

[Full analysis with evidence...]
```

### 冷验证覆盖层

阶段 9 冷验证生成覆盖层文件（`*.cold-verify.md`），用于增强基础漏洞发现，提供独立裁决。覆盖层更新对抗性裁决和严重性，并在漏洞发现正文中附加“Cold Verification”部分。

---

## Finding Ingestion

审计完成后，漏洞发现会自动解析并存储到 Vigolium 数据库中。

### 数据库字段

| 审计字段 | 数据库字段 | 示例 |
|----------|------------|------|
| Finding ID | `module_id` | `audit:p8-001` |
| Title | `module_name` | SSRF via Webhook Job Address |
| Slug | `module_short` | `ssrf-webhook-job` |
| Severity (final) | `severity` | `high`（标准化） |
| Verdict | `confidence` | `firm`（CONFIRMED/VALID）或 `tentative` |
| CWE | `cwe_id` | `CWE-918` |
| Full analysis | `description` | Markdown 正文，包含证据 |
| First location | `source_file` | `src/jobservice/webhook_job.go` |
| All locations | `matched_at` | `src/jobservice/webhook_job.go:103-120` |
| Metadata | `tags` | `["audit", "phase-8", "valid", "poc-theoretical", "CWE-918"]` |

所有漏洞发现都存储以下字段：
- `finding_source`: `audit`
- `module_type`: `whitebox`
- `finding_hash`: MD5(auditID + moduleID + findingID) 用于去重

### 置信度映射

| 审计裁决 | 数据库置信度 |
|----------|--------------|
| CONFIRMED, VALID | `firm` |
| 其他所有（POSSIBLE, UNLIKELY 等） | `tentative` |

### 查询审计漏洞发现

```bash
# 通过 CLI
vigolium finding list --source audit

# 通过 API
GET /api/findings?source=audit
```

---

## Architecture

### 专用 Agent（共 24 个）

vigolium-audit 引擎使用一组专用 Agent，每个负责审计的特定方面：

| Agent | 阶段 | 角色 |
|-------|------|------|
| advisory-hunter | P1 | CVE/GHSA/OSV 情报收集 |
| commit-archaeologist | P1 | Git 历史分析，查找静默修复 |
| patch-bypass-checker | P2 | 已识别补丁的绕过分析 |
| knowledge-base-builder | P3 | 威胁模型 + 架构映射 |
| static-analyzer | P4 | SAST 工具协调（CodeQL, Semgrep） |
| probe-strategist | P5 | 多模型假设生成 |
| code-anatomist | P5 | 代码结构分析 |
| backward-reasoner | P5 | 逆向工程攻击路径 |
| contradiction-reasoner | P5 | 发现逻辑不一致 |
| causal-verifier | P5 | 验证因果声明 |
| evidence-harvester | P5 | 从代码证据构建证明 |
| enrichment-filter | P6-7 | 按可利用性对漏洞发现分类 |
| spec-gap-analyst | P6-7 | RFC/规范合规性差距检测 |
| chamber-synthesizer | P8 | 对抗性审查的辩论主持人 |
| attack-ideator | P8 | 利用思路头脑风暴 |
| code-tracer | P8 | 深度代码路径跟踪 |
| devils-advocate | P8 | 挑战假设 |
| variant-scout | P8 | 初始变体识别 |
| cold-verifier | P9 | 独立的零上下文验证 |
| variant-hunter | P10 | 跨代码库的深度变体分析 |
| poc-builder | P11 | 概念验证生成 |
| report-assembler | P11 | 最终报告汇编 |

### 捆绑技能

以下安全技能嵌入在 Vigolium 二进制文件中，用于 vigolium-audit：

- **audit** — 核心多阶段方法论编排器
- **codeql** — CodeQL 数据库创建和查询执行
- **semgrep** — Semgrep 规则管理和扫描
- **semgrep-rule-creator** — 自定义 Semgrep 规则生成
- **fp-check** — 误报验证方法论
- **variant-analysis** — 跨代码库漏洞变体检测
- **vuln-report** — 咨询式漏洞报告生成
- **differential-review** — 基于差异的安全审查
- **security-threat-model** — STRIDE/DREAD 威胁建模
- **sarif-parsing** — SARIF 输出解析和丰富化
- **zeroize-audit** — 内存安全分析（Rust/C）

### 对抗性审查室（阶段 8）

深度模式的对抗性辩论阶段使用结构化格式，专用 Agent 对每个漏洞发现的可利用性进行论证和反驳：

```
probe-strategist --> 生成假设
         |
         +-- attack-ideator（头脑风暴利用方法）
         +-- backward-reasoner（逆向工程路径）
         +-- evidence-harvester（构建证明）
         |
         +-- chamber-synthesizer（主持辩论）
                  |
                  +-- devils-advocate（挑战声明）
                  +-- contradiction-reasoner（发现不一致）
                  +-- causal-verifier（验证因果关系）
```

只有通过此对抗性过程的漏洞发现才会进入冷验证和最终报告。与单次分析相比，这大大减少了误报。

### 冷验证（阶段 9）

对抗性辩论之后，冷验证 Agent 对每个漏洞发现执行独立的、零上下文的重新验证。它不接收先前的裁决或理由——只接收原始代码和漏洞发现描述。冷验证覆盖层用独立的严重性评估和裁决更新基础漏洞发现，提供第二意见，捕捉辩论阶段的群体思维。

---

## Comparison with Native Scanning

| 方面 | Vigolium 原生（Swarm/Autopilot） | Audit-Audit |
|------|----------------------------------|-------------|
| **重点** | 网络漏洞（注入、XSS、SSRF 等） | 源代码漏洞（逻辑缺陷、认证漏洞、规范违规） |
| **方法** | 使用载荷进行实时 HTTP 扫描 | 静态分析 + AI 推理 + 对抗性验证 |
| **误报处理** | AI 分类阶段 | 多层：对抗性辩论室 + 冷验证 |
| **漏洞发现丰富度** | 标准严重性/置信度 | 对抗性裁决、冷验证覆盖层、CWE、PoC 状态 |
| **速度** | 分钟到小时 | 分钟（精简）到小时（深度） |
| **要求** | 目标 URL | 源代码路径 |
| **运行方式** | 前台（主流水线） | 后台（独立进程） |

两种方法互补。网络扫描发现 HTTP 响应中显现的漏洞；vigolium-audit 发现需要理解代码语义、业务逻辑和规范合规性的漏洞。同时运行两者可提供最全面的评估。