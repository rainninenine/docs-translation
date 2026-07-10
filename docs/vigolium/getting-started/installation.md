# Installation

Vigolium 以单个静态链接的 Go 二进制文件形式发布，无运行时依赖。选择适合你环境的安装方式——所有方式都提供相同的 `vigolium` 二进制文件。

## Quick install (recommended)

```bash
curl -fsSL https://vigolium.com/install.sh | bash
```

安装程序会：

- 从 CDN 解析最新版本，下载匹配的二进制文件，并在安装前**验证其 SHA-256 校验和**
- 安装到 `~/.local/bin/vigolium`
- 将 `~/.local/bin` 添加到相应 shell 配置文件（`.zshrc`、`.bashrc`/`.bash_profile` 或 `config.fish`）的 `PATH` 中

如果 `~/.local/bin` 尚未在你的 `PATH` 中，无需重启 shell 即可激活：

```bash
export PATH="$HOME/.local/bin:$PATH"
vigolium doctor
```

## npm

npm 包是一个轻量启动器，会为你的平台拉取正确的预构建二进制文件作为可选依赖（Node 16+）。

```bash
# 全局安装
npm install -g @vigolium/vigolium

# 或无需安装直接运行一次
npx @vigolium/vigolium scan -h
```

## Build from source

需要 **Go 1.26+**、`git` 和 `make`。无需 C 工具链。

```bash
git clone https://github.com/vigolium/vigolium.git
cd vigolium
make build
```

`make build` 输出 `bin/vigolium` 并将二进制文件安装到 `$GOPATH/bin`。

> **不要**直接运行 `go build`。`make build` 会注入版本元数据并保证构建的清洁性；临时性的 `go build` 会产生一个报告错误版本的二进制文件，并可能在工作树中留下过时产物。

要显式地将二进制文件放入 `PATH`（假设 `$GOPATH/bin` 已在 PATH 中）：

```bash
make install
```

其他有用的目标：
`make build-all`（交叉编译 Linux/macOS/Windows）、`make test-unit`（快速测试）、`make lint`。运行 `make help` 或查看 `Makefile` 获取完整列表。

## Docker

```bash
docker pull j3ssie/vigolium:latest

# 运行任意命令——入口点为 vigolium 二进制文件
docker run --rm j3ssie/vigolium:latest scan -h

# 扫描目标并将报告写入主机
docker run --rm -v "$PWD:/out" j3ssie/vigolium:latest \
  scan --stateless -t https://example.com --format jsonl -o /out/results
```

## Verify the installation

```bash
vigolium version     # 打印版本、构建信息和提交哈希
vigolium doctor      # 验证环境、配置和可选依赖
```

`vigolium doctor` 是确认一切配置正确的最快方式——在任何安装或升级后运行它。

### Auto-install missing dependencies

Vigolium 的大部分核心扫描功能没有外部依赖，但某些可选功能需要额外的工具（SPA 爬虫需要浏览器，已知问题扫描需要 nuclei 模板，某些 Agent 驱动需要 `bun`/`pi`）。如果 `doctor` 报告了可修复的项目，让它为你安装：

```bash
vigolium doctor --fix              # 自动安装/修复所有失败的检查项
vigolium doctor --fix --only chrome,nuclei   # 仅修复特定项目
```

`--fix` 打印报告，安装缺失项，然后重新检查并显示更新后的状态。`--only` 接受以下任意值：
`nuclei`、`chrome`、`bun`、`claude`、`agent-browser`、`pi`、`piolium`
（不与 `--fix` 配合使用时无效）。

## Updating

`vigolium update` 重新运行官方安装程序以获取最新版本，并刷新已知问题扫描使用的本地 nuclei-templates 检出：

```bash
vigolium update                  # 更新二进制文件 + nuclei 模板
vigolium update --skip-templates # 仅重新安装二进制文件
vigolium update --skip-binary    # 仅刷新 nuclei 模板
vigolium update -F               # 跳过确认提示
```

二进制文件更新始终安装到 `~/.local/bin/vigolium`。如果你的运行中二进制文件位于其他位置（例如 `make install` 构建在 `$GOPATH/bin` 或 Docker 镜像中），`update` 会打印警告——请确保 `~/.local/bin` 在 `PATH` 中优先于旧位置，或手动升级该副本。

> npm 和 Docker 安装通过各自的工具进行升级（`npm update -g @vigolium/vigolium` / `docker pull`），而非 `vigolium update`。

## Where Vigolium stores data

所有数据存放在 `~/.vigolium/` 下（可通过 `VIGOLIUM_HOME` 环境变量覆盖）：

| 路径 | 内容 |
|------|----------|
| `~/.vigolium/vigolium-configs.yaml` | 主配置文件 |
| `~/.vigolium/database-vgnm.sqlite` | 默认 SQLite 扫描数据库 |
| `~/.vigolium/agent-sessions/` | Agent 扫描会话产物 |
| `~/.vigolium/prompts/` | 用户提示模板 |

显式初始化工作区和初始配置：

```bash
vigolium init
```

## Uninstall

```bash
rm -f ~/.local/bin/vigolium      # 或：$GOPATH/bin/vigolium（源码安装）
rm -rf ~/.vigolium               # 可选：删除配置、数据库、会话
```

npm：`npm uninstall -g @vigolium/vigolium`

## Next steps

- [Quick Start](quick-start.md) —— 在一分钟内运行你的首次扫描
- [Native Scan & Stateless Scanning](native-scan.md) —— CLI 扫描方案
- [Setting Up the Agent](setup-agent.md) —— 配置 AI 驱动扫描
- [Configuration Reference](../configuration.md) —— 所有配置项
- [Troubleshooting](../troubleshooting.md) —— 常见问题修复
