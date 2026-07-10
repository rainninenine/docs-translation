# Architecture Overview

> **您所在位置：** Vigolium 体系结构文档的入口页面。本页概述系统全貌——操作模式、两种扫描范式以及各组件如何协同工作。同级文档深入介绍每个子系统：
>
> - [native-scan.md](native-scan.md) —— 确定性的 Go 扫描管道，端到端
> - [agentic-scan.md](agentic-scan.md) —— AI Agent 引擎、编排器及 olium 运行时
> - [data-and-storage.md](data-and-storage.md) —— 多租户隔离、数据库模型及云存储
> - [server-and-api.md](server-and-api.md) —— REST 服务器、流量数据导入及 API 接口

Vigolium 是一款用 Go 编写的高保真 Web 漏洞扫描器。它将基于模块的确定性扫描与 AI 驱动的 Agent 分析相结合，提供广泛且深入的 Web 应用安全问题覆盖。该扫描器内置 251 个模块（154 个主动模块、97 个被动模块），覆盖注入缺陷、配置错误、信息泄露、认证问题等。

Vigolium 可作为 CLI 工具执行一次性扫描，也可作为持久化 REST API 服务器接收实时流量数据导入，或作为流量转发导入客户端（`vigolium ingest`）将流量推送至运行中的服务器。所有扫描数据按项目范围隔离，支持多租户隔离。模块：`github.com/vigolium/vigolium`，需 Go 1.26+。

## Operating Modes

| 模式 | 二进制命令 | 描述 |
|------|-----------|------|
| **CLI Scanner** | `vigolium scan` | 直接从命令行对目标、输入文件（OpenAPI、Postman、Burp、cURL、HAR）或源代码路径运行扫描。 |
| **Server Mode** | `vigolium server` | 启动带有 Swagger UI 的 REST API 服务器。可导入流量、触发扫描、查询漏洞发现以及通过 HTTP 运行 Agent 会话。 |
| **Ingestor Client** | `vigolium ingest` | 轻量级客户端，用于捕获 HTTP 流量并将其转发至运行中的 Vigolium 服务器进行分析。 |

## Scanning Paradigms

### Native Scan

本地扫描管道是完全确定性的——纯 Go 实现，无 AI 参与。请求流经一组固定顺序的阶段，每个阶段处理侦察或测试的不同环节。

**阶段（按顺序）：**

```
Heuristics -> External Harvesting -> Spidering -> Discovery -> DynamicAssessment -> KnownIssueScan -> Extension
```

| 阶段 | 用途 |
|------|------|
| Heuristics | 轻量级指纹识别和技术检测 |
| External Harvesting | Wayback Machine 及其他被动源信息收集 |
| Spidering | 主动爬取、JS 分析、链接和表单提取 |
| Discovery | 通过字典进行端点和内容发现 |
| DynamicAssessment | 核心漏洞测试——注入、XSS、SSRF 等（CLI 别名：`audit`、`dast`、`assessment`） |
| KnownIssueScan | 检查已知 CVE 和常见配置错误 |
| Extension | 用户提供的 JavaScript 扫描插件 |

**策略**控制哪些阶段运行及其激进程度：

| 策略 | 行为 |
|------|------|
| Lite | 快速表面扫描；跳过大量爬取和发现 |
| Balanced | 默认。运行所有阶段，使用合理限制 |
| Deep | 穷举扫描，使用更高限制、更广泛字典和外部信息收集 |

### Agentic Scan

Agent 扫描使用 AI Agent 驱动或增强扫描过程。通过 `vigolium agent <mode>` 调用。所有 AI 调度均通过进程内 **olium** 引擎（`pkg/olium/`）运行；供应商包括 `openai-codex-oauth`、`anthropic-api-key`、`anthropic-oauth`、`openai-api-key` 和 `anthropic-cli`。

| 模式 | 命令 | 描述 |
|------|------|------|
| **Query** | `vigolium agent query` | 单次提示执行。适用于代码审查、端点发现、秘密检测。不进行网络扫描。 |
| **Autopilot** | `vigolium agent autopilot` | 一个长时间运行的 LLM 会话，拥有完整的 bash/文件/Web 工具以及 `report_finding` 和 `halt_scan` 功能。Agent 自行决定扫描内容、运行扫描、检查结果并迭代，直至停止。 |
| **Swarm** | `vigolium agent swarm` | 多阶段管道，其中本地 Go 负责繁重工作，AI 在检查点介入——规划攻击、分类结果以及生成自定义 JS 扫描器插件。 |
| **Audit** | `vigolium agent audit` | 前台多阶段 AI 源代码审计。针对源代码树驱动独立的 Claude Code / Codex 工具。 |
| **Olium** | `vigolium agent olium`（或 `vigolium ol`） | 直接交互式 TUI 访问 olium 引擎。使用 `-p` 进行非交互式单次提示。 |

所有 Agent 模式均支持 `--source` 进行源代码感知分析，并将会话产物（计划、插件、输出）存储在可配置的会话目录中。

## Architecture at a Glance

```
                          +------------------+
                          |    输入源         |
                          | curl/OpenAPI/Burp |
                          |  HAR/Postman/URL  |
                          +--------+---------+
                                   |
                    +--------------+--------------+
                    |                             |
              vigolium scan                 vigolium server
                    |                             |
                    v                             v
            +---------------+           +-----------------+
            |  范围过滤器    |           | REST API (Fiber)|
            +-------+-------+           +--------+--------+
                    |                             |
                    +-------------+---------------+
                                  |
                    +-------------+-------------+
                    |                           |
              本地扫描                    Agent 扫描
                    |                           |
         +----------+----------+      +---------+---------+
         |  执行器（工作线程）   |      |   Agent 引擎      |
         |  速率限制器          |      |   提示模板         |
         +----------+----------+      |                   |
                    |                 +---------+---------+
         +----------+----------+                |
         |  模块注册表          |      +---------+---------+
         | 152 个主动模块       |      | Olium 供应商      |
         |  93 个被动模块       |      | openai-codex-oauth /     |
         +----------+----------+      | anthropic-api-key |
                    |                 | anthropic-oauth /    |
                    |                 | openai-api-key /  |
                    |                 | anthropic-cli   |
                    |                 +---------+---------+
                    +-------------+---------------+
                                  |
                    +-------------+-------------+
                    |        结果存储            |
                    |  SQLite / PostgreSQL      |
                    |  HTML / JSONL / 控制台    |
                    +---------------------------+
```

## Architecture Documents

每个子系统的深入文档与本页并列：

| 子系统 | 文档 | 覆盖内容 |
|--------|------|----------|
| 本地扫描管道 | [native-scan.md](native-scan.md) | CLI 入口 → 输入解析 → 执行器 → 模块 → 结果 → 数据库，全部 12 个阶段 |
| Agent 扫描引擎 | [agentic-scan.md](agentic-scan.md) | 子命令、编排器、引擎接缝、olium 运行时、供应商 |
| 数据与持久化 | [data-and-storage.md](data-and-storage.md) | `project_uuid` 多租户隔离、仓库模式、数据模型、云存储 |
| 服务器与 API | [server-and-api.md](server-and-api.md) | Fiber 服务器、流量数据导入、REST 接口、Agent 运行 API |

## Where to Go Next (task docs)

| 我想... | 请前往 |
|---------|--------|
| 快速上手 | [../getting-started.md](../getting-started.md) |
| 选择扫描策略 | [../native-scan/strategies.md](../native-scan/strategies.md) |
| 了解各个扫描阶段 | [../native-scan/phases/](../native-scan/phases/)（discovery、spidering、dynamic-assessment、extension、known-issue-scan） |
| 探索 Agent 扫描 | [../agentic-scan/agent-mode.md](../agentic-scan/agent-mode.md) |
| 使用 Autopilot / Swarm 模式 | [../agentic-scan/autopilot.md](../agentic-scan/autopilot.md) · [../agentic-scan/swarm.md](../agentic-scan/swarm.md) |
| 直接使用 olium 引擎（TUI / 无头模式） | [../agentic-scan/olium-agent.md](../agentic-scan/olium-agent.md) |
| 以服务器模式运行 Vigolium | [../server-mode/](../server-mode/) |
| 配置扫描和设置 | [../configuration.md](../configuration.md) |
| 格式化和导出结果 | [../output-and-reporting.md](../output-and-reporting.md) |
| 编写自定义 JS 插件 | [../customization/writing-extensions.md](../customization/writing-extensions.md) |
| 浏览 REST API | [../api-references/](../api-references/) |
| 管理项目（多租户隔离） | [../projects.md](../projects.md) |
| 使用云存储（gs:// URL、捆绑包、上传） | [../storage.md](../storage.md) |
| 调试问题 | [../troubleshooting.md](../troubleshooting.md) |
