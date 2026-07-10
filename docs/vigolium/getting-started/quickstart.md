# 文档 API 参考 入门指南 安装与快速入门

**复制页面**  
几分钟内启动并运行 Vigolium：安装、验证并执行首次扫描。

---

## 第 1 步：安装 Vigolium 开源版

### 本地安装（推荐）

**npm**  
```bash
npm install -g @vigolium/vigolium
```
# 轻量启动器，作为可选依赖项拉取适用于您平台的预构建二进制文件（Node 16+）

**Bun**  
```bash
bun add -g @vigolium/vigolium
```
# 使用 Bun 全局安装 Vigolium 启动器。与 npm 安装路径相同的预构建二进制启动器；Bun 只是更快的包管理器。

**Homebrew**  
```bash
brew install vigolium/tap/vigolium
```
# 在 macOS 和 Linux 上从 Vigolium tap 安装最新版本。使用 `brew upgrade vigolium` 升级。

**Docker**  
```bash
docker pull j3ssie/vigolium:latest
docker run --rm j3ssie/vigolium:latest scan -h
```
# 容器入口点为 vigolium 二进制文件，直接传递任何子命令。

**从源码构建**  
```bash
git clone https://github.com/vigolium/vigolium.git
cd vigolium
make build
```
# 需要 Go 1.26+、Bun 1.3.11+、git 和 make。始终使用 `make build`（而非 `go build`），它会注入版本元数据。

> Vigolium 仅原生支持 Linux 和 macOS。Windows 用户应使用 WSL 或 Docker 镜像。

如果 `~/.local/bin` 尚未在您的 `PATH` 中，无需重启 shell 即可激活：
```bash
export PATH="$HOME/.local/bin:$PATH"
```

---

## 第 2 步：验证安装

```bash
vigolium version
vigolium doctor
```

`doctor` 报告任何缺失的可选依赖项（用于 SPA 爬取的浏览器、用于已知问题扫描的 nuclei 模板、用于 Agent 驱动程序的 bun/pi），并确认您的配置有效。让它自动安装任何缺失的内容：
```bash
vigolium doctor --fix                          # 自动安装/修复每个失败的检查
```
`--only` 接受以下任意值：`nuclei`、`chrome`、`bun`、`claude`、`agent-browser`、`pi`、`piolium`。

Vigolium 存储的所有内容位于 `~/.vigolium/` 下（可通过 `VIGOLIUM_HOME` 覆盖）：配置文件位于 `vigolium-configs.yaml`，扫描数据库位于 `database-vgnm.sqlite`，Agent 工件位于 `agent-sessions/` 下。运行 `vigolium init` 显式创建工作空间。

---

## 第 3 步：运行完整扫描

`vigolium scan` 默认使用平衡策略运行完整的多阶段流水线（内容发现 → 爬取 → 动态评估）：
```bash
vigolium scan -t https://example.com
```

通过策略预设调整深度/速度权衡：
```bash
vigolium scan -t https://example.com --strategy lite       # 快速，仅动态评估
vigolium scan -t https://example.com --strategy balanced
vigolium scan -t https://example.com --strategy deep       # 彻底，更多模块
```

`--intensity quick|balanced|deep` 是更高级的别名，也会调整扫描配置文件。不确定使用哪种模式？请参阅《选择扫描模式》。

---

## 第 4 步：一次性无状态扫描

对于 CI/CD 流水线、脚本编写或快速临时检查（不希望磁盘上留下任何内容），添加 `--stateless` 并使用 `-o` 导出结果。Vigolium 会启动一个临时 SQLite 数据库，运行请求的阶段，写入输出，然后在退出时删除数据库。
```bash
# 完整流水线，JSONL 输出，不持久化任何内容
vigolium scan --stateless -t https://example.com --format jsonl -o findings

# JSONL + HTML 报告，包含内容发现
vigolium scan --stateless -t https://example.com --discover --format jsonl,html -o scan

# 单个端点，无数据库
vigolium scan-url https://example.com/api/users?id=1 -j
```

多个目标各自拥有独立的临时数据库和按主机命名的文件名后缀，因此结果不会覆盖：
```bash
vigolium scan --stateless -T targets.txt --format jsonl -o results
# -> results-example.com.jsonl, results-test.example.com.jsonl, …
```

`--stateless` 和 `--db` 互斥。请参阅《本地扫描与无状态扫描》获取完整配方手册。

---

## 第 5 步：选择扫描内容

```bash
# 目标文件，每行一个 URL
vigolium scan -T targets.txt

# 从 OpenAPI / Swagger 规范
vigolium scan -i api.yaml -I openapi -t https://api.example.com

# 从标准输入管道输入 URL
cat urls.txt | vigolium scan

# 原始 HTTP 请求或 curl 命令（自动检测）
echo "curl -X POST -d 'user=admin' https://example.com/login" | vigolium scan-request
```

支持的输入模式（`-I`）：`urls`、`openapi`、`swagger`、`postman`、`curl`、`burpxml`、`nuclei`、`har`。

---

## 第 6 步：选择特定模块（可选）

```bash
# 仅运行 XSS 和 SQLi 模块（模糊匹配模块 ID/名称）
vigolium scan -t https://example.com -m xss -m sqli

# 按标签过滤
vigolium scan -t https://example.com --module-tag injection

# 列出所有可用模块
vigolium -M
```

---

## 第 7 步：获取结果

默认情况下，漏洞发现会流式输出到控制台。对于文件或机器可读输出，使用 `--format` 和 `-o`：
```bash
# JSONL 用于脚本/CI
vigolium scan -t https://example.com --format jsonl -o results

# 自包含 HTML 报告
vigolium scan -t https://example.com --format html -o report

# 同时输出多种格式
vigolium scan -t https://example.com --format jsonl,html -o scan
```

| 标志 | 效果 |
|:-----|:-----|
| `--format console` | 人类可读的终端输出（默认） |
| `--format jsonl` / `-j` | 每行一个 JSON 对象 |
| `--format html` | 交互式 ag-grid 报告（需要 `-o`） |
| `-o, --output` | 输出文件路径（基本名称；每种格式自动添加扩展名） |
| `--ci-output-format` | 仅 JSONL，无横幅或颜色，适合 CI |
| `--silent` | 抑制除漏洞发现外的所有内容 |

---

## 第 8 步：运行单个阶段

当您只需要流水线的某个阶段时，使用 `run <phase>`（`scan --only <phase>` 的别名）：
```bash
vigolium run discovery -t https://example.com    # 仅内容发现
vigolium run spidering -t https://example.com    # 仅浏览器爬取
vigolium run dynamic-assessment -t https://example.com
```

阶段：`ingestion`、`discovery`、`external-harvest`、`spidering`、`known-issue-scan`、`dynamic-assessment`、`extension`。

---

## 关于持久化的说明

`vigolium scan` 将结果写入 `~/.vigolium/database-vgnm.sqlite` 中的持久化 SQLite 数据库，因此您可以稍后浏览它们：
```bash
vigolium traffic list      # 已导入的 HTTP 记录
vigolium finding list      # 已发现的漏洞
```

对于一次性运行（不留任何内容，如 CI、临时检查），添加 `--stateless` 并使用 `-o` 导出。请参阅《本地扫描与无状态扫描》获取完整配方集。

---

## 更新与卸载

```bash
vigolium update                  # 更新二进制文件 + nuclei 模板
vigolium update --skip-templates # 仅重新安装二进制文件
vigolium update -F               # 跳过确认提示
```

Homebrew、npm 和 Docker 安装通过各自的工具升级（`brew upgrade vigolium` / `npm update -g @vigolium/vigolium` / `docker pull`），而非 `vigolium update`。

---

## 下一步

- **选择扫描模式**：为您的任务选择正确的模式。
- **本地扫描与无状态扫描**：CLI 扫描配方。
- **扫描策略**：策略、配置文件、节奏。
- **认证扫描**：会话和登录流程。
- **设置 Agent**：AI 驱动的自动导航和群组扫描。
- **配置参考**：完整配置选项。

---

**上一页**：选择扫描模式  
Vigolium 提供多种扫描模式：本地扫描、通过 Burp 的本地扫描、审计 Agent、自动导航和群组扫描。本页帮助您为任务选择正确的模式，以及控制扫描深度的强度预设（快速/平衡/深度）。

**下一页**：⌘I

---

*网站 | Twitter | GitHub | Discord | X | LinkedIn*  
*本文档使用 Mintlify 构建和托管，一个开发者文档平台*