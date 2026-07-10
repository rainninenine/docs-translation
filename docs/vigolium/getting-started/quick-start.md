# Quick Start

本页将带你从全新安装到在几分钟内获得首个漏洞发现（Finding）。如果你尚未安装 Vigolium，请参阅 [Installation](installation.md)。

## 0. Verify your setup

```bash
vigolium version
vigolium doctor
```

`doctor` 会报告任何缺失的可选依赖（例如 SPA 爬虫所需的浏览器）并确认你的配置有效。

## 1. Scan a single URL

检查单个端点最快的方式。无需数据库，无需设置——直接执行主动+被动模块：

```bash
vigolium scan-url https://example.com
```

针对特定参数：

```bash
vigolium scan-url "https://example.com/search?q=test"
```

## 2. Run a full scan

`vigolium scan` 运行完整的多阶段流水线（发现 → 爬虫 → 动态评估），默认使用 **balanced** 策略：

```bash
vigolium scan -t https://example.com
```

使用策略预设调节深度/速度权衡：

```bash
vigolium scan -t https://example.com --strategy lite   # 快速，仅动态评估
vigolium scan -t https://example.com --strategy balanced
vigolium scan -t https://example.com --strategy deep   # 彻底，更多模块
```

`--intensity quick|balanced|deep` 是更高级的别名，同时也会调节扫描配置文件。

## 3. Choose what to scan

```bash
# 目标文件，每行一个 URL
vigolium scan -T targets.txt

# 从 OpenAPI / Swagger 规范
vigolium scan -i api.yaml -I openapi -t https://api.example.com

# 从标准输入管道传入 URL
cat urls.txt | vigolium scan

# 原始 HTTP 请求或 curl 命令（自动检测）
echo "curl -X POST -d 'user=admin' https://example.com/login" | vigolium scan-request
```

支持的输入模式（`-I`）：`urls`、`openapi`、`swagger`、`postman`、`curl`、`burpxml`、`nuclei`、`har`。

## 4. Pick specific modules (optional)

```bash
# 仅运行 XSS 和 SQLi 模块（按模块 ID/名称模糊匹配）
vigolium scan -t https://example.com -m xss -m sqli

# 改为按标签筛选
vigolium scan -t https://example.com --module-tag injection

# 列出所有可用模块
vigolium -M
```

## 5. Get results out

默认情况下，漏洞发现（Finding）会输出到控制台。如需文件或机器可读输出，使用 `--format` 配合 `-o`：

```bash
# JSONL 用于脚本/CI
vigolium scan -t https://example.com --format jsonl -o results

# 自包含 HTML 报告
vigolium scan -t https://example.com --format html -o report

# 同时输出多种格式
vigolium scan -t https://example.com --format jsonl,html -o scan
```

| 标志 | 效果 |
|------|--------|
| `--format console` | 人类可读的终端输出（默认） |
| `--format jsonl` / `-j` | 每行一个 JSON 对象 |
| `--format html` | 交互式 ag-grid 报告（需要 `-o`） |
| `-o, --output` | 输出文件路径（基础名称；每种格式会追加扩展名） |
| `--ci-output-format` | 仅 JSONL，无横幅或颜色——适合 CI |
| `--silent` | 抑制除漏洞发现外的所有输出 |

## 6. Run a single phase

当你只需要流水线的某个阶段时，使用 `run <phase>`（`scan --only <phase>` 的别名）：

```bash
vigolium run discovery -t https://example.com    # 仅内容发现
vigolium run spidering -t https://example.com    # 仅浏览器爬取
vigolium run dynamic-assessment -t https://example.com
```

阶段：`ingestion`、`discovery`、`external-harvest`、`spidering`、`known-issue-scan`、`dynamic-assessment`、`extension`。

## A note on persistence

`vigolium scan` 将结果写入位于 `~/.vigolium/database-vgnm.sqlite` 的持久化 SQLite 数据库，因此你可以之后浏览它们：

```bash
vigolium traffic list      # 已导入的 HTTP 记录
vigolium finding list      # 已发现的漏洞
```

对于不留痕迹的一次性运行（CI、临时检查），添加 `--stateless` 并使用 `-o` 导出。请参阅 [Native Scan & Stateless Scanning](native-scan.md) 获取完整方案集。

## Next steps

- [Native Scan & Stateless Scanning](native-scan.md) —— CLI 扫描方案
- [Scanning Strategies](../native-scan/strategies.md) —— 策略、配置文件、速率
- [Scanning Modes Overview](../native-scan/scanning-modes-overview.md) —— 比较所有模式
- [Authenticated Scanning](../native-scan/authentication.md) —— 会话和登录流程
- [Setting Up the Agent](setup-agent.md) —— AI 驱动的 autopilot 和 swarm 扫描
- [Configuration Reference](../configuration.md) —— 完整配置选项
