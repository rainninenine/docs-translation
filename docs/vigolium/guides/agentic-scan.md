# Agentic Scan

Agentic Scan 使用 AI agent 来理解您的应用程序并发现漏洞。与依赖预定义载荷的本地扫描不同，Agentic Scan 可以推理代码、理解业务逻辑，并生成针对性的攻击。

## 何时使用 Agentic Scan

| 场景 | 推荐方式 |
|------|----------|
| 您有源代码 | Agentic Scan（Swarm、Autopilot、Audit） |
| 您需要深度业务逻辑测试 | Agentic Scan（Autopilot） |
| 您想要结构化的、可重现的 AI 扫描 | Agentic Scan（Swarm） |
| 您需要快速的代码审计 | Agentic Scan（Audit） |
| 您只有 URL，没有源代码 | 本地扫描 |
| 您需要快速检查已知漏洞 | 本地扫描（KnownIssueScan） |

## 扫描模式

### Swarm — 结构化 AI 扫描

Swarm 模式使用多个专门的 AI agent 协作扫描目标。它最适合结构化的、可重现的扫描。

```bash
# 基本 Swarm 扫描
vigolium agent swarm -t https://example.com

# 带源代码的 Swarm 扫描
vigolium agent swarm -t https://example.com --source ./src

# 仅发现端点
vigolium agent swarm --discover -t https://example.com

# 完整流水线：发现 → 扫描 → 审计
vigolium agent swarm --discover -t https://example.com --source ./src
```

### Autopilot — 探索性 AI 扫描

Autopilot 模式使用一个 agent 自主探索目标并发现漏洞。它最适合探索性、创造性的扫描。

```bash
# 基本 Autopilot 扫描
vigolium agent autopilot -t https://example.com

# 聚焦特定漏洞类别
vigolium agent autopilot -t https://example.com --focus "injection"

# 带源代码的 Autopilot
vigolium agent autopilot -t https://example.com --source ./src

# 带审计的 Autopilot
vigolium agent autopilot -t https://example.com --audit
```

### Audit — 代码审计

Audit 模式对源代码进行深入的、AI 驱动的安全审计。

```bash
# 基本代码审计
vigolium agent audit --source ./src

# 快速审计
vigolium agent audit --source ./src --mode lite

# 深度审计
vigolium agent audit --source ./src --mode deep

# 使用自定义提示模板
vigolium agent query --prompt-template security-code-review --source ./src
```

## 通用选项

```bash
# 指定 AI 供应商
vigolium agent swarm -t https://example.com --provider anthropic-api-key

# 指定模型
vigolium agent swarm -t https://example.com --model claude-opus-4-7

# 预览提示而不运行（节省 token）
vigolium agent swarm -t https://example.com --dry-run

# 限制运行轮次
vigolium agent swarm -t https://example.com --max-turns 10
```

## 配置

```yaml
# vigolium-configs.yaml
agent:
  olium:
    provider: openai-codex-oauth    # AI 供应商
    model: gpt-5.5                  # AI 模型
    max_turns: 32                   # 最大 agent 轮次
    max_concurrent: 4               # 最大并发 agent 数
    call_timeout_sec: 600           # API 调用超时

  llm:
    provider: anthropic             # JS 扩展用 LLM
    model: claude-sonnet-4-6
    api_key_env: ANTHROPIC_API_KEY
```

## 输出

Agentic Scan 生成：

1. **漏洞发现** — 保存到数据库，可通过 `vigolium finding` 查询
2. **HTTP 记录** — 扫描期间发出的所有请求
3. **会话日志** — 完整的 agent 交互日志，可通过 `vigolium log <uuid>` 查看
4. **报告** — 可通过 `vigolium export --format html` 导出

## 延伸阅读

- [Swarm](swarm.md) — Swarm 模式详情
- [Autopilot](autopilot.md) — Autopilot 模式详情
- [Olium Agent](olium-agent.md) — AI agent 引擎
- [Vigolium Audit](vigolium-audit.md) — 代码审计详情
- [Piolium Audit](piolium-audit.md) — Pi 驱动的审计
