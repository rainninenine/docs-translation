# Quickstart

> 在 5 分钟内启动并运行 Osmedeus

## Step 1: 安装最新版 Osmedeus

**原生安装（推荐）：**
```bash
curl -fsSL https://www.osmedeus.org/install.sh | bash
```

**Homebrew：**
```bash
brew install osmedeus/tap/osmedeus
```

**Nightly 构建：**
```bash
curl -fsSL https://www.osmedeus.org/nightly-install.sh | bash
```

**从源码构建：**
```bash
git clone https://github.com/osmedeus/osmedeus.git
cd osmedeus
make build
```

> **Windows 用户：** Osmedeus 仅原生支持 Linux 和 macOS。Windows 用户请使用 WSL 或参阅 [Docker 安装](/getting-started/docker-setup)。

![安装 Osmedeus 核心引擎](https://mintcdn.com/osmedeus/rQ49bCQbs9dKARCy/images/installation/install-native.png?fit=max&auto=format&n=rQ49bCQbs9dKARCy&q=85&s=b4967ad70af87544af37471bf4fedc49)

## Step 2: 安装默认工作流

**使用默认预设工作流：**
```bash
osmedeus install base --preset
```

**从 Git 仓库安装自定义工作流：**
```bash
osmedeus install workflow https://github.com/osmedeus/osmedeus-workflow.git
```

## Step 3: 验证安装

```bash
osmedeus health
```

如果在安装过程中遇到任何问题，请查看 [FAQs and Common Errors](/others/faq)。

![安装验证](https://mintcdn.com/osmedeus/rQ49bCQbs9dKARCy/images/installation/install-validation.png?fit=max&auto=format&n=rQ49bCQbs9dKARCy&q=85&s=1d41c6f1373d0914b95a492ab42d91c5)

## Step 4: 运行第一个工作流

```bash
osmedeus run -m subdomain -t example.com
```

### 下一步

- [CLI 界面](/getting-started/cli) — 深入了解命令行工具
- [Web UI](/getting-started/web-ui) — 探索 Web 界面
