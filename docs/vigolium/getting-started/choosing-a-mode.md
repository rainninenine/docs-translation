文档API 参考快速入门选择扫描模式复制页面Vigolium 提供多种扫描模式：本地扫描（Native Scan）、通过 Burp 的本地扫描（native-via-Burp）、审计 Agent（Audit Agent）、Autopilot 和 Swarm。本页帮助您根据任务选择合适的模式，以及控制扫描深度的强度预设（快速/均衡/深度）。复制页面如果您只记住一件事：本地扫描覆盖广度，审计 Agent 覆盖深度，Autopilot 用于无人值守的黑箱扫描，Swarm 用于手工构造的请求。从以下五种模式中选择一种，然后在本页底部调整强度。

### 概览

| 模式 | 最佳适用场景 | 黑箱/白箱 | 需要源代码？ | 需要 LLM？ |
|:-----|:-----|:-----|:-----|:-----|
| 本地扫描（Native Scan） | 最大速度的全覆盖黑箱扫描 | 黑箱 | 否 | 否 |
| 通过 Burp 插件的本地扫描（Native Scan via Burp Plugin） | 针对单个请求/URL 的精确扫描，覆盖所有参数 | 黑箱 | 否 | 否 |
| 审计 Agent（Audit Agent） | 彻底的白盒源代码审计 | 白箱 | 是 | 前沿 LLM（Claude / Codex / Pi） |
| Autopilot Agent | 无人值守的黑箱扫描，使用真实浏览器 + 即时扩展 | 黑箱（可选源代码感知） | 可选 | 是 |
| Swarm Agent | 针对单个请求生成定制化载荷 | 黑箱 | 否 | 是 |

### 1. 本地扫描（Native Scan）

可以将其视为 Burp Active Scanner + Nuclei + ffuf + Katana + Wayback Machine 的增强版，全部由单个二进制文件驱动，并行运行，共享状态。

当您希望以最快速度对目标进行最广泛的黑箱扫描时使用：外部数据收集、内容发现、浏览器驱动的爬取（支持 SPA）、已知问题扫描以及跨 313 个扫描模块的主动/被动动态评估，一次运行完成。

```bash
# 均衡全流水线
vigolium scan -t https://example.com

# 无状态一次性扫描，JSONL 输出，不留痕迹
vigolium scan --stateless -t https://example.com --format jsonl -o findings

# 调至深度模式
vigolium scan -t https://example.com --strategy deep
```

**何时选择**

- 您正在评估一个新目标，希望开启所有功能。
- 您需要可重复、确定性的输出用于 CI/CD。
- 您没有（或不想使用）LLM。

参见本地扫描与无状态扫描（Native Scan & Stateless Scanning）和策略（Strategies）。

### 2. 通过 Burp Suite 插件的本地扫描（Native Scan via Burp Suite Plugin）

相同的本地扫描器，但从 Burp Suite 标签页中调用，针对单个请求或 URL，对所有参数（头部、Cookie、请求体字段、路径段）进行模糊测试。当您已经在 Burp 中看到请求，并希望进行精确的一次性扫描，而不是爬取整个应用程序时使用。

```bash
# 等效 CLI：从剪贴板扫描原始请求或 curl 命令
vigolium scan-request -i request.txt
pbpaste | vigolium scan-request

# 或扫描单个 URL，覆盖相同表面
vigolium scan-url "https://example.com/api/users?id=1"
```

**何时选择**

- 您只关心一个请求，并希望对该请求进行最深入的参数覆盖。
- 您正在分类 Burp 的漏洞发现，并希望获得自动化的第二意见。
- 您希望在现有的 Burp 工作流中获得本地扫描输出。

该插件是 `scan-url` / `scan-request` 的轻量封装，参见本地扫描与无状态扫描（Native Scan & Stateless Scanning）。

### 3. 审计 Agent（Vigolium Audit + Piolium）

由前沿 LLM 驱动的白盒源代码审计。Vigolium 提供两个驱动程序，均可通过统一的 `vigolium agent audit` 调度器访问：

- **Vigolium Audit（内嵌）**：内置于 `vigolium` 二进制文件中，驱动 `claude` 或 `codex` CLI。深度模式下最多 12 个阶段。无需额外安装。
- **Piolium**：Pi 编码 Agent 扩展。深度模式下最多 17 个阶段。需要 `pi` 运行时 + `pi install piolium`。支持 Pi 支持的任何供应商，包括本地模型。

```bash
# 自动（默认）：内嵌审计，仅在审计不可用时回退到 piolium
vigolium agent audit --source ~/src/your-app --mode deep

# 强制使用内嵌审计驱动程序
vigolium agent audit --driver audit --source ~/src/your-app --mode deep

# Piolium，最彻底的审计
vigolium agent audit --driver=piolium --source ~/src/your-app --mode deep

# 同时运行两者，在同一父扫描下，项目级去重
vigolium agent audit --driver both --source ~/src/your-app
```

**何时选择**

- 您拥有源代码，并希望获得最深入的漏洞覆盖。
- 您希望漏洞发现关联到具体的文件/行范围，而不仅仅是 URL。
- 您愿意支付前沿模型的 Token 成本，或者如果预算有限，可以运行 Pi + Piolium 针对本地模型。

审计 Agent 仅在前沿模型（Claude Opus、GPT-5.x 等）上提供最佳结果。Piolium 路径是通过 Pi 自身的供应商配置驱动本地模型进行审计的唯一方式。

参见设置 Agent（Setting Up the Agent）和 Agent 安全审计（Agentic Security Audit）。

> Autopilot 和 Swarm 模式仍处于早期阶段。我们非常感谢您对任何误报或 Bug 的反馈。

### 4. Autopilot Agent

无人值守的黑箱扫描，其中 `olium` 运行时驱动真实的 Chromium 浏览器，即时生成自定义 JavaScript 扫描器扩展，并自行决定运行哪些 CLI 子命令和模块。您也可以提供源代码路径，Autopilot 将具备源代码感知能力，并利用代码上下文指导其扫描。

```bash
# 纯黑箱
vigolium agent autopilot -t https://example.com --intensity balanced

# 源代码感知——将黑箱运行时检查与白盒代码阅读相结合
vigolium agent autopilot -t https://example.com --source ~/src/your-app

# 在后台加入 vigolium-audit
vigolium agent autopilot -t https://example.com --source ~/src/your-app --audit=balanced
```

**何时选择**

- 您希望将目标交给扫描器然后离开。
- 目标包含大量 JavaScript / 需要身份验证，真实浏览器是唯一可达方式。
- 您希望 Agent 为应用程序特定的特性编写自己的扫描器扩展。

参见 Autopilot。

### 5. Swarm Agent

引导式多阶段扫描，Agent 的任务是针对特定请求生成定制化载荷。当您从 Burp（或其他地方）获得一个已知有效的请求，并希望进行定制模糊测试，而不是通用的主动扫描时效果最佳。

```bash
# 针对单个 URL 的 Swarm
vigolium agent swarm -i "https://example.com/api/users?id=1"

# 针对原始 HTTP 请求文件
vigolium agent swarm -i ./request.txt --triage

# 添加发现 + 分类以形成更完整的流水线
vigolium agent swarm -i https://example.com --discover --triage
```

**何时选择**

- 您有一个单个请求，并希望 LLM 专门为其设计载荷。
- 本地扫描的默认载荷未命中，但您怀疑存在漏洞。
- 您希望使用 AI 检查点（规划 → 分类 → JS 扩展生成），而不给予 Agent 完全自主权。

参见 Swarm。

### 强度矩阵

`--intensity quick|balanced|deep` 是跨模式的控制旋钮，决定每种模式的运行深度。对于本地扫描，它也是 `--strategy` 的别名。对于 Agent 模式，它映射到每个驱动程序的阶段计数。

| 模式 | quick | balanced（默认） | deep |
|:-----|:-----|:-----|:-----|
| 本地扫描（Native Scan） | 仅动态评估，无发现、无爬取、无已知问题扫描。最适合无状态/CI 门控。 | 完整流水线：发现 + 爬取 + 已知问题 + 动态评估。所有内容的合理默认值。 | 添加外部收集、递归发现、额外模块、更长的阶段持续时间。全力出击。 |
| 通过 Burp 的本地扫描（Native via Burp） | 一次性 `scan-url` / `scan-request`，仅主动模块。 | 主动 + 被动，所有插入点。 | 主动 + 被动 + 重度变异策略和更长的超时时间。 |
| Vigolium Audit | 3 个阶段：侦察、分类、快速扫描。适合 CI。 | 9 个阶段：添加深度挖掘、漏洞利用设计、验证。 | 12 个阶段：完整审计，包括二次审查和跨文件分析。 |
| Piolium Audit | 4 个阶段：快速分类。 | 9 个阶段：标准审计。 | 17 个阶段：详尽：侦察、主要、远射、重新审视、确认、合并、差异。 |
| Autopilot | 较短的最大持续时间上限，保守的模块预算，更少的轮次。冒烟测试范围。 | 默认轮次/持续时间预算。启用真实浏览器引导模式。 | 更大的轮次预算，启用 JS 扩展生成，如果提供了 `--source`，则在后台运行 `vigolium-audit`。 |
| Swarm | 单次载荷生成，无发现，无分类。 | 生成 + 分类载荷；一轮反馈。 | 跨所有插入点的完整多轮生成/分类/优化。 |

### 经验法则

- **CI / 合并前门控 → quick**。足够快以阻止 PR；捕获明显问题。
- **每日回归 / 定时扫描 → balanced**。默认值是有原因的。
- **渗透测试 / 发布前审计 → deep**。时间预算为数小时，而非数分钟。

### 结合强度与策略

对于本地扫描，您也可以直接使用 `--strategy lite|balanced|deep`，具有相同的阶段切换效果，但更精细地控制哪些阶段运行。`--intensity` 是更高级的别名，还会调整扫描配置文件（速度、模块预算、变异攻击性）。

```bash
# 本地扫描，深度模式
vigolium scan -t https://example.com --intensity deep

# 或一次调整一个旋钮
vigolium scan -t https://example.com --strategy deep --profile aggressive
```

### 决策捷径

- “我现在就想扫描一个 URL。” → 本地扫描（Native Scan），`vigolium scan-url`。
- “我手头有一个 Burp 请求。” → 通过 Burp 插件的本地扫描（Native via Burp Plugin）或 `vigolium scan-request`。
- “我有源代码并且有时间。” → 审计 Agent（Audit Agent）。
- “我希望扫描器整夜自行运行。” → Autopilot Agent。
- “我希望 LLM 针对这个端点定制载荷。” → Swarm Agent。

### 下一步

- 快速入门（Quickstart），在一分钟内运行您的第一次扫描。
- 本地扫描与无状态扫描（Native Scan & Stateless Scanning），所有 CLI 扫描配方。
- 设置 Agent（Setting Up the Agent），在使用 Agent 模式前配置供应商。
- 策略（Strategies），完整的策略/速度/配置文件参考。
- 扫描模式概述（Scanning Modes Overview），详细比较所有本地扫描命令。

上一页本地扫描与无状态扫描（Native Scan & Stateless Scanning）本地扫描是 Vigolium 的确定性、基于 Go 的扫描流水线，快速、模块化且无 AI。本页是 CLI 运行本地扫描的实践指南，重点介绍无状态扫描。下一页⌘IwebsitetwittergithubdiscordxlinkedinPowered by本文档基于 Mintlify 构建和托管，一个开发者文档平台本页内容概览1. 本地扫描2. 通过 Burp Suite 插件的本地扫描3. 审计 Agent（Vigolium Audit + Piolium）4. Autopilot Agent5. Swarm Agent强度矩阵经验法则结合强度与策略决策捷径下一步