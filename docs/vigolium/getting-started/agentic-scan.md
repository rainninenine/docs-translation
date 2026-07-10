# Agentic Scan

**Agent 扫描（Agentic Scan）** 是 Vigolium 基于 AI 驱动的扫描模式。[本地扫描（Native Scan）](native-scan.md)运行一个确定性的、基于 Go 的扫描器模块流水线，而 Agent 扫描则将控制权（或部分控制权）交给 LLM，由 LLM 对目标进行推理、决定测试内容、驱动工具，并在确认漏洞发现（Finding）后进行报告。

本页是从全新安装到首次 AI 驱动扫描的最快路径。有关源代码审查和白盒审计，请参阅 [Agentic Security Audit](agentic-security-audit.md)。

## What you get

`vigolium agent` 下提供了两种 Agent 扫描模式：

| 模式 | 一句话描述 | 适用场景 |
|---|---|---|
| `autopilot` | 一次长时间的 LLM 会话，配备 shell、文件和 HTTP 工具 | 自由形式的渗透测试式探索 |
| `swarm` | LLM 规划 → 本地扫描器执行 → 可选的人工复核（Triage） | 对已知目标进行结构化、可重复的扫描 |

两者都将漏洞发现（Finding）写入与本地扫描相同的数据库，因此常规的 `vigolium finding list` / `/api/findings` 命令仍可正常使用。

## Prerequisites

Agent 扫描需要且仅需要一个已配置的 LLM 供应商（Provider）。完整的供应商矩阵和 BYOK 身份验证选项请参阅 [Setting Up the Agent](setup-agent.md) —— 简要版如下：

```bash
# 如果你已有 ChatGPT Plus/Pro/Team 订阅，这是最便宜的方案
codex login
vigolium config set agent.olium.provider openai-codex-oauth

# 验证——应输出一个模型名称
vigolium ol -p 'what model are you running'
```

只要 `vigolium ol -p` 返回了模型名称，就说明配置正确 —— `autopilot` 和 `swarm` 将复用同一个供应商。

## 1. Autopilot — hand the agent a target

`autopilot` 是最简单的 Agent 模式：一次 LLM 会话，配备 shell、文件和 HTTP 访问权限。它会自行选择策略，并在没有更多可执行的有用操作时停止。

```bash
# 纯 URL 目标
vigolium agent autopilot -t https://example.com

# 特定请求——粘贴 curl、原始 HTTP 或 Burp XML；自动检测
vigolium agent autopilot --input "curl -X POST -d 'q=test' https://example.com/api"

# 在 CI 风格运行中限制成本/时间
vigolium agent autopilot -t https://example.com --intensity quick --max-duration 5m
```

`--intensity` 预设（`quick` / `balanced` / `deep`）用于调节轮次数量、持续时间限制以及 Agent 的探索激进程度。

## 2. Swarm — let the AI direct the native scanner

`swarm` 让 AI 担任规划角色，而本地 Go 扫描器担任执行角色。Agent 选择模块，可选地编写自定义 JS 插件（Extension），确定性流水线运行，然后可选的人工复核（Triage）阶段验证漏洞发现。

```bash
# 针对单个请求的定向扫描，LLM 决定如何攻击
vigolium agent swarm -t https://example.com/api/users?id=1

# 全范围扫描——AI 规划，本地发现+扫描执行
vigolium agent swarm -t https://example.com --discover --triage
```

有用的标志：

- `--discover` —— 在扫描前运行本地内容发现
- `--triage` —— AI 审查每个漏洞发现以抑制误报
- `--vuln-type sqli` —— 专注于某一类漏洞
- `-m xss -m sqli` —— 限制到特定的扫描器模块

## 3. Add source code for a deeper read

如果你有代码库，使用 `--source` 指向它。两种模式都会获得文件系统读写权限，Agent 可以将代码与流量进行关联。

```bash
vigolium agent autopilot -t https://example.com --source ~/src/your-app
vigolium agent swarm -t https://example.com --source ~/src/your-app --code-audit --triage
```

源代码解锁了专用的审计流水线——有关完整的白盒流程，请参阅 [Agentic Security Audit](agentic-security-audit.md)。

## Choosing between autopilot and swarm

| 你希望... | 使用 |
|---|---|
| 创造性、探索性测试——让 Agent 即兴发挥 | `autopilot` |
| 结构化、可重复的扫描，附带可选验证 | `swarm` |
| 结合扫描的源代码感知代码审查 | 两者皆可，配合 `--source` |
| 从多个角度对单个请求进行深入测试 | `swarm` 配合 `--input` |
| 全范围爬取+扫描，附带 AI 规划 | `swarm --discover --triage` |

## Where results go

漏洞发现（Finding）会输出到控制台，并持久化到与本地扫描相同的 SQLite 数据库中：

```bash
vigolium finding list                           # 所有漏洞发现
vigolium finding list --agent-mode autopilot    # 按 Agent 模式筛选
vigolium agent session list                     # 历史运行记录
```

每次运行还会在 `~/.vigolium/agent-sessions/<uuid>/` 下写入一个会话目录，包含 `runtime.log`、生成的插件（Extension）和原始 Agent 输出——当扫描结果异常、你想查看 Agent 的推理过程时非常有用。

## REST API

如果你希望从自己的控制器驱动扫描，同样的模式也通过 HTTP 暴露：

```bash
vigolium server &

curl -s -X POST http://localhost:9002/api/agent/run/autopilot \
  -H "Content-Type: application/json" \
  -d '{"target": "https://example.com"}'

curl -s -X POST http://localhost:9002/api/agent/run/swarm \
  -H "Content-Type: application/json" \
  -d '{"input": "https://example.com", "discover": true, "triage": true}'
```

每个调用返回 `202 Accepted` 并附带一个 `agentic_scan_uuid`；轮询 `/api/agent/status/:id`，并在完成后从 `/api/agent/sessions/:id/...` 拉取会话产物。

## Next steps

- [Setting Up the Agent](setup-agent.md) —— 完整的供应商矩阵和 BYOK
- [Agentic Security Audit](agentic-security-audit.md) —— 白盒源代码审计
- [Autopilot](../agentic-scan/autopilot.md) —— 单循环自主扫描参考
- [Swarm](../agentic-scan/swarm.md) —— 多阶段 AI 引导扫描参考
- [Agent Mode](../agentic-scan/agent-mode.md) —— 所有 `vigolium agent` 子命令一览
