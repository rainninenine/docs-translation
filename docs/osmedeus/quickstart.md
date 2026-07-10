> ## Documentation Index
> 获取完整文档索引，请访问：https://docs.osmedeus.org/llms.txt
> 在进一步探索之前，请使用此文件发现所有可用页面。

# Quickstart

> 在5分钟内启动并运行Osmedeus

## Step 1: Install the latest version of Osmedeus

<Tabs>
  <Tab title="Native Installation (Recommended)">
    ```bash theme={null}
    curl -fsSL https://www.osmedeus.org/install.sh | bash
    ```
  </Tab>

  <Tab title="Homebrew" icon="beer-mug">
    ```bash theme={null}
    brew install osmedeus/tap/osmedeus
    ```
  </Tab>

  <Tab title="Nightly Build " icon="bolt">
    ```bash theme={null}
    curl -fsSL https://www.osmedeus.org/nightly-install.sh | bash
    ```
  </Tab>

  <Tab title="Build from Source" icon="bolt">
    ```bash theme={null}
    git clone https://github.com/osmedeus/osmedeus.git
    cd osmedeus
    make build
    ```
  </Tab>

  <Tab title="Windows">
    <Check>Osmedeus 原生仅支持 Linux 和 macOS。Windows 用户请使用 WSL 或参阅 [Docker Setup here](/getting-started/docker-setup)</Check>
  </Tab>
</Tabs>

<Frame caption="安装 Osmedeus 核心引擎">
  <img src="https://mintcdn.com/osmedeus/rQ49bCQbs9dKARCy/images/installation/install-native.png?fit=max&auto=format&n=rQ49bCQbs9dKARCy&q=85&s=b4967ad70af87544af37471bf4fedc49" width="3490" height="1696" data-path="images/installation/install-native.png" />
</Frame>

## Step 2: Install the default workflows

<Tabs>
  <Tab title="Use default preset workflows">
    ```bash theme={null}
    osmedeus install base --preset
    ```
  </Tab>

  <Tab title="Install custom workflows from Git repository">
    ```bash theme={null}
    osmedeus install workflow https://github.com/osmedeus/osmedeus-workflow.git
    ```
  </Tab>
</Tabs>

## Step 3: Validate the installation

```bash theme={null}
osmedeus health
```

如果在安装过程中遇到任何问题，请查看 [FAQs and Common Errors](/others/faq)。

<Frame caption="安装验证">
  <img src="https://mintcdn.com/osmedeus/rQ49bCQbs9dKARCy/images/installation/install-validation.png?fit=max&auto=format&n=rQ49bCQbs9dKARCy&q=85&s=1d41c6f1373d0914b95a492ab42d91c5" width="3490" height="2450" data-path="images/installation/install-validation.png" />
</Frame>

## Step 4: Run your first workflow

```bash theme={null}
osmedeus run -m subdomain -t example.com
```

<Columns cols={2}>
  <Tile href="/getting-started/cli" title="CLI Interface">
    <img src="https://mintcdn.com/osmedeus/v2F2CQFtcKul_NUM/images/cli/cli-run-progress.png?fit=max&auto=format&n=v2F2CQFtcKul_NUM&q=85&s=a0c448d97d10f955df1589deb2251cc6" alt="CLI 运行预览" width="2704" height="1438" data-path="images/cli/cli-run-progress.png" />
  </Tile>

  <Tile href="/getting-started/web-ui" title="Web UI Interface">
    <img src="https://mintcdn.com/osmedeus/v2F2CQFtcKul_NUM/images/web-ui/web-ui-assets.png?fit=max&auto=format&n=v2F2CQFtcKul_NUM&q=85&s=0ce499f1662852c2a17527d2b49a01b7" alt="Web UI 资产预览" width="2974" height="2378" data-path="images/web-ui/web-ui-assets.png" />
  </Tile>
</Columns>