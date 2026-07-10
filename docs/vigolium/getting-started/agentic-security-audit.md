# Agentic Security Audit

**Agent 安全审计（Agentic Security Audit）** 是 Vigolium 的白盒、源代码驱动的审计方案，与 [Agent 扫描（Agentic Scan）](agentic-scan.md)相辅相成。Agent 扫描发送流量并探测响应，而审计则读取代码：它遍历 git 历史、构建威胁模型、运行 SAST、搜索已确认漏洞的变体，并在将每个候选漏洞发现（Finding）纳入报告之前，通过对抗性辩论（Adversarial Debate）进行验证。

它专为**彻底的**代码审查而设计——适用于发布前审计、合规性工作、漏洞奖励式的深度挖掘——而非仅仅是 lint 式的简单检查。

## Two harnesses, same output

Vigolium 提供两种审计驱动（Harness），它们共享相同的磁盘模式、漏洞发现格式以及应用的数据库标签。它们的区别仅在于**驱动审计的模型系列**不同。

| 驱动 | 驱动方式 | 默认 Agent | 独立 npm 包 |
|---|---|---|---|
| `vigolium agent vigolium-audit` | 内嵌——无需额外安装 | Claude（Opus 系列） | [`@vigolium/vigolium-audit`](https://www.npmjs.com/package/@vigolium/vigolium-audit) |
| `vigolium agent audit --driver=piolium` | Pi 运行时 + Pi 插件 | 供应商无关（GPT、Gemini、Claude……） | [`@vigolium/piolium`](https://www.npmjs.com/package/@vigolium/piolium) |

两者产生相同的漏洞发现模式，都导入到 Vigolium 数据库中，并且都可以**在 Vigolium 之外独立运行**，作为各自的 npm CLI——当你没有安装 Vigolium，但拥有 Node/Bun 和 API 密钥时非常有用。

经验法则：

- **有 Claude Opus 访问权限？** 使用 `vigolium-audit`。它的提示编排是针对 Opus 调优的，在该模型系列上能产生最高质量的漏洞发现。
- **使用 OpenAI、Gemini 或其他非 Claude 模型？** 使用 `piolium`。Pi 的适配器层抽象了供应商，使相同的审计流水线能够在不同模型系列上工作。
- **不确定/两者都想要？** 使用 `vigolium agent audit`——统一驱动会先运行 `vigolium-audit`，如果失败则回退到 `piolium`，或者使用 `--driver both` 在同一个父扫描下顺序运行两者，并进行项目级去重。

## What "thorough" means here

两种驱动都运行一个多阶段流水线。深度模式如下所示：

| 阶段 | 功能 |
|---|---|
| Commit Archaeology | 遍历 git 历史，寻找静默安全修复和未披露的 CVE |
| Patch Bypass | 测试之前的补丁是否真正完整 |
| Knowledge Base | 构建架构模型、信任边界、攻击面 |
| Static Analysis | 使用项目定制规则的 CodeQL + Semgrep |
| Deep Probe | 使用专用 Agent 进行多假设探索 |
| Spec Gap Analysis | 查找文档/RFC 与实现之间的差异 |
| Enrichment & Filtering | 对 SAST 漏洞发现进行可达性+数据流分析 |
| Adversarial Debate | Agent 对每个候选漏洞发现进行正反辩论 |
| Cold Verification | 独立的零上下文重新验证 |
| Variant Hunting | 在代码库中搜索已确认漏洞的变体 |
| Report Assembly | PoC 构建和咨询式最终报告 |

轻量模式（`lite`、`balanced`）运行子集，适合 CI 环境快速周转。`deep` 是完整流水线；`confirm` 根据实时行为重新验证现有审计；`longshot`（仅 piolium）是按文件的全面扫描。

## Prerequisites

- 一个已配置的 LLM 供应商。请参阅 [Setting Up the Agent](setup-agent.md)。`vigolium-audit` 驱动与 Claude 通信（通过 `claude` 或通过 Codex 的 `codex`）；`piolium` 与你为 `pi` 配置的任何供应商通信。
- 你想要审计的源代码——本地路径或 git URL。Vigolium 会将远程 URL 克隆到临时目录。

对于 `piolium`，你还需要在主机上安装一次 Pi 运行时和 piolium 插件：

```bash
bun install -g @earendil-works/pi-coding-agent
pi install git:git@github.com:vigolium/piolium.git
pi login
```

`vigolium-audit` 是**内嵌的**——无需额外安装。Vigolium 在首次运行时提取驱动。

## 1. Quick run

```bash
# 内嵌审计（Claude/Codex），balanced 模式
vigolium agent vigolium-audit --source ~/src/your-app

# Piolium（Pi 驱动，供应商无关），默认 balanced 模式
vigolium agent audit --driver=piolium --source ~/src/your-app

# 统一模式——让 Vigolium 自动选择。失败时从 vigolium-audit 回退到 piolium
vigolium agent audit --source ~/src/your-app
```

漏洞发现（Finding）会输出到控制台，并导入到数据库中，标记上产生它们的驱动。像查询其他漏洞发现一样查询它们：

```bash
vigolium finding list --source vigolium-audit
vigolium finding list --source piolium
```

## 2. Pick a depth

`--mode` 标志选择要运行的阶段链；`--intensity` 是更高级的别名。

```bash
# 快速分类——几分钟，适合 CI
vigolium agent vigolium-audit --source ./src --mode lite
vigolium agent audit --driver=piolium       --source ./src --intensity quick

# 默认 balanced——在合理时间内提供良好覆盖率
vigolium agent vigolium-audit --source ./src --mode balanced

# 完整深度审计——数小时，对抗性辩论+冷验证
vigolium agent vigolium-audit --source ./src --mode deep
vigolium agent audit --driver=piolium       --source ./src --intensity deep   # 17 个阶段
```

你也可以使用 `--modes a,b,c` 将多个模式串联执行——例如 `--modes deep,confirm` 先运行完整审计，然后重新验证漏洞发现。`--intensity deep` 会自动展开为该链。

值得了解的仅 piolium 模式：

- `revisit` —— 使用新的假设重新遍历先前的审计
- `longshot` —— 按文件的全面扫描，在完整审计后使用
- `diff` —— 仅审计自某个基准提交以来的变更

## 3. Pair an audit with a live scan

审计会导入到与网络扫描相同的数据库中，因此你可以将代码级别的漏洞发现与同一目标的流量级别漏洞发现进行关联。最简单的方式是让 `swarm` 或 `autopilot` 为你调度审计：

```bash
# Swarm——AI 规划扫描并在后台运行 vigolium-audit
vigolium agent swarm \
  -t https://example.com \
  --source ./src \
  --audit deep \
  --triage

# Autopilot——先运行 vigolium-audit，Agent 将其漏洞发现作为上下文使用
vigolium agent autopilot \
  -t https://example.com \
  --source ./src \
  --audit balanced
```

当本地安装了 `pi` + `piolium` 时，Vigolium 会自动选择 piolium 用于流水线内审计；传递 `--audit <mode>` 可强制使用 `vigolium-audit`。

## 4. Standalone CLIs (no Vigolium needed)

两种驱动都以独立的 npm 包形式发布。它们产生与内嵌版本相同的漏洞发现格式和磁盘模式——当你希望在没有 Vigolium 二进制文件的机器上运行审计，或在 CI 中锁定特定驱动版本时非常有用。

```bash
# Vigolium Audit — https://www.npmjs.com/package/@vigolium/vigolium-audit
npm install -g @vigolium/vigolium-audit
vigolium-audit --source ~/src/your-app --mode balanced

# Piolium — https://www.npmjs.com/package/@vigolium/piolium
npm install -g @vigolium/piolium
piolium --source ~/src/your-app --mode deep
```

漏洞发现（Finding）以 markdown 文件形式存放在 `<source>/vigolium-audit/`（或 `<source>/piolium/`）目录下。如果你之后希望将它们导入 Vigolium 数据库：

```bash
vigolium import /path/to/audit-output/
```

> **提示——不要污染工作树。** 审计漏洞发现默认写入源代码目录下。为保持仓库整洁，请针对工作树外的检出副本运行，或确保 `findings-draft/`、`longshot/` 和 `attack-surface/` 已加入 `.gitignore`。

## Where results go

每次审计都会产生：

- **数据库漏洞发现** —— `vigolium finding list --source vigolium-audit`、`vigolium finding list --source piolium`
- **会话产物** 位于 `~/.vigolium/agent-sessions/<uuid>/` 下——`audit-state.json`、每个漏洞发现的 markdown 文件、最终报告、CodeQL/Semgrep 输出、对抗性辩论记录、原始 Agent 日志
- **一份最终咨询式报告**，适合与代码所有者团队共享

## Next steps

- [Setting Up the Agent](setup-agent.md) —— 供应商设置、BYOK 身份验证、验证检查
- [Agentic Scan](agentic-scan.md) —— 网络端对应方案
- [Vigolium Audit](../agentic-scan/vigolium-audit.md) —— 内嵌驱动的完整参考、模式、漏洞发现格式
- [Piolium Audit](../agentic-scan/piolium-audit.md) —— Pi 驱动驱动的完整参考
- [Audit BYOK matrix](../agentic-scan/audit-byok.md) —— `--provider` / `--agent` / API 密钥标志的解析方式
