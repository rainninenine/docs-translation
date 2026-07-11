# FAQs

> 常见问题与常见错误

常见问题集锦，以及常见错误的解释和解决方法。

## 1. 通用问题

### 什么是 Osmedeus？

Osmedeus 是一个用于安全自动化的工作流引擎。它执行 YAML 定义的工作流，支持多种执行环境（主机、Docker、SSH）、调度和分布式扫描。

### 系统要求是什么？

Osmedeus 核心引擎轻量级，几乎可以在任何规格的系统上运行。<br />

    然而，如果你计划将其用于**侦察**（这是主要用例），建议使用现代 Linux、macOS 或带有 WSL 的 Windows 系统。<br />

    由于运行侦察会产生大量网络流量，还建议在云环境中运行 Osmedeus，**例如 VM、Compute Engine 或 EC2**，以获得最佳性能。

### 如何安装 Osmedeus？

```bash
    # 从源码构建
    make build

    # 安装到 $GOBIN
    make install

    # 安装安全工具
    osmedeus install binary --all
    ```

### Osmedeus 是否支持 AI/LLM 集成？能否使用 Claude Code 和 Codex 等工具？

当然可以。Osmedeus 内置了对 LLM 的支持，你可以在工作流中使用它来生成侦察报告、编写自定义脚本，甚至构建自己的代理工作流。你可以查看 [LLM 工作流示例](/advanced/llm/) 了解其工作原理。

    请注意，使用 LLM 可能需要你拥有 LLM 供应商的 API 密钥，并且可能会根据使用情况产生额外费用。在工作流中使用 LLM 时，请始终监控使用情况和成本。

    由于 Osmedeus 是一个编排框架，你可以利用它来协调自己的自定义 AI/LLM 工具，包括直接在 YAML 工作流中集成 Claude Code 或 OpenCode。例如，你可以设计一个自定义代理，在定义的管道中调用多个工具，并将其无缝插入工作流。灵活性几乎是无限的。

### Osmedeus 是否有可供代理使用的 AI 技能？

是的，我在 [github.com/osmedeus/osmedeus-skills](https://github.com/osmedeus/osmedeus-skills) 上构建了 [osmedeus-expert](https://github.com/osmedeus/osmedeus-skills) 技能，你可以在代理工具中使用它来编写 YAML 工作流、运行 CLI 命令以及配置高级功能。

***

## 2. 二进制安装

### 为什么需要安装外部二进制文件？

Osmedeus 是一个独立的 Golang 二进制文件，本身运行良好。然而，当使用 Osmedeus 运行 YAML 工作流进行安全自动化时，它通常需要调用外部工具，如 `httpx`、`nuclei`、`ffuf` 等。这些工具必须安装并在系统中可用，工作流才能正常运行。

### 安装二进制文件或运行 `osmedeus health` 时遇到错误，该怎么办？

注册表中列出的所有二进制文件并非每个工作流都需要。即使缺少某些工具，你的扫描仍可能正常运行。

### 是否需要使用 `osmedeus install binary --all --install-optional` 安装注册表中的所有二进制文件？

不需要。安装所有工具完全是可选的。注册表包含 YAML 工作流中常用的额外工具，但你只需要特定工作流所需的工具。运行基本工作流不需要安装所有工具。

### 我尝试了所有方法，但仍无法安装所需的二进制文件，该怎么办？

如上所述，注册表中列出的所有二进制文件并非每个工作流都需要。即使缺少某些工具，你的扫描仍可能正常运行。

    如果你希望获得理想设置，我建议使用 Docker 运行 Osmedeus 及其工作流。这可以确保满足所有依赖项，并消除任何兼容性问题。详情请参阅 [Docker 设置](/getting-started/docker-setup)。

***

## 3. 扫描执行与扫描结果

### 如何运行基本扫描？

```bash
    # 运行 Flow 工作流
    osmedeus run -f general -t example.com

    # 运行模块工作流
    osmedeus run -m vulnerability-scan -t example.com
    ```

### 如何扫描多个目标？

```bash
    # 从命令行
    osmedeus run -f fast -t target1.com -t target2.com

    # 从文件
    osmedeus run -f fast -T targets.txt -c 5
    ```

### 如何设置超时？

```bash
    osmedeus run -m port-scan -t example.com --timeout 2h
    ```

### 扫描结果存储在哪里？

结果存储在 `~/workspaces-osmedeus/<target>/` 的项目空间中。

### 我应该将令牌（Github、Shodan 等）放在哪里？

你只需按照 [**此指南设置令牌**](/getting-started/basic-setup/) 操作即可。

### 如何设置发送通知？

你只需按照 [**此指南设置通知**](/advanced/notification-and-cdn/) 操作即可。

### 在哪里可以获得实时支持？

你可以加入 **`https://discord.gg/mtQG2FQsYA`** 看看是否有人能提供帮助。我可能会不时回答，但不能保证回答每一个问题。

### 是否支持代理？

不支持，原生不支持代理。但由于该工具的设计是运行其他第三方工具，而其中许多工具默认不支持代理。我曾考虑过 proxychains，但它会使速度极慢并破坏许多功能。

### 为什么我的扫描卡在端口扫描阶段？

它会卡在那里，因为出现了 sudo 密码提示。某些特殊工具（如 **nmap**）需要 *root* 权限才能运行。请确保允许 **nmap** 在没有 sudo 密码提示的情况下运行。

### 为什么我的扫描（如漏洞扫描、端口扫描或内容发现）耗时很长？

这可能是因为你输入的目标非常大。想象一下，尝试对 **2000 个不同的主机** 运行内容发现。这就是为什么需要很长时间。

### 为什么即使我扫描了故意存在漏洞的应用，Osmedeus 也没有发现任何漏洞？

再次强调，这很大程度上取决于你的目标。Osmedeus 在大范围目标上表现出色，而不是单个故意存在漏洞的 Web 应用。只需扫描一些随机的 VDP，你就会看到结果。
    它无法在故意存在漏洞的应用上发现漏洞的原因是 **vulnscan** 模块不支持它。但你完全可以自定义工作流来实现这一点。

### 我发现了一个很酷的新工具，你能把它添加到 Osmedeus 中吗？

可以，只需按照 [**此指南**](/advanced/writing-your-first-workflow/) 将其添加到你的工作流中。

### X 扫描是否运行工具 Y？

1. 阅读 flow 和 module 文件，确定步骤实际运行的内容。
    2. 说真的，请阅读 flow 和 module 文件。
    3. 记住，你已经两次被警告要阅读 flow 和 module 文件。
    4. 在工作流文件夹中搜索工具命令，以确认是否使用（例如：`rg -F 'nuclei' ~/osmedeus-base/workflows/`）。

### 在哪里可以找到 Web UI 的密码？

请参考 [**此页面**](/getting-started/web-ui/) 启动 Web 服务器并获取凭据。你可能需要运行命令 `osmedeus config view server.password`。

### 如何让扫描或 Web UI 在后台持续运行？

最简单的方法是使用 `https://tmuxcheatsheet.com/` 运行进程。此外，你可以设置一个服务，将 osmedeus Web 服务器作为后台进程运行。

### 如果 Osmedeus 发现了漏洞 X，我该怎么办？

1. 阅读漏洞 X 的描述。
    2. 说真的，请阅读漏洞 X 的描述。
    3. 记住，你已经两次被警告要阅读漏洞 X 的描述。
    4. 搜索漏洞 X 的名称。
    5. 手动验证漏洞 X。
    6. 仍然没有结果？也许 `https://letmegooglethat.com/?q=what+is+a+vulnerability+X` 可以帮到你。

### Osmedeus 发现了一些易受攻击的子域名，但我无法访问它们？

通常情况下，扫描期间发现的子域名的可用性可能与手动验证时不同。这取决于目标，并且可能有所不同。

### 当我使用 `--debug` 标志运行时，我注意到某些命令返回了退出状态码 128、255 或 -1 的错误。这是预期的吗？

是的，某些命令显示预期的退出状态码是正常的，因为它们可能在特定条件下成功。但是，如果你确信原始 bash 命令应该成功但失败了，请尝试复制原始 bash 命令并调查其失败原因。

### 如何确定针对我的目标应该运行哪个工作流？

你可以运行 `osmedeus workflow ls` 或 `osmedeus workflow show <workflow-name> --verbose` 查看描述，然后选择适合扫描的工作流。

### 扫描执行成功，但 UI 没有显示任何资产？

这很可能是因为你执行的工作流没有生成任何资产。你可以通过检查位于 `~/workspaces-osmedeus/<target>/` 的项目空间目录来确认是否有文件被创建。

    也可能是因为工作流没有使用任何数据库工具函数将资产保存到数据库中。你可以检查工作流文件，看是否使用了类似 `db_import_asset` 的数据库工具函数。你还可以通过 `osmedeus func ls db --example` 查看所有与数据库相关的函数列表。

### 为什么即使我设置了通知，也没有收到任何通知？

这很可能是因为你执行的工作流没有生成任何资产。你可以通过检查位于 `~/workspaces-osmedeus/<target>/` 的项目空间目录来确认是否有文件被创建。

    也可能是因为工作流没有使用任何通知工具函数将资产保存到通知中。你可以检查工作流文件，看是否使用了类似 `notify_telegram` 的通知工具函数。你还可以通过 `osmedeus func ls noti --example` 查看所有与通知相关的函数列表。

***

## 4. 工作流

### Flow 和 Module 有什么区别？

* **Module**：包含按顺序执行的步骤的单个工作流单元。
    * **Flow**：编排多个模块，允许模块之间并行执行和依赖关系。

### 工作流存储在哪里？

工作流存储在 `~/osmedeus-base/workflows/` 中：

    * `flows/` - Flow 工作流
    * `modules/` - Module 工作流

### 如何创建自定义工作流？

在工作流目录中创建一个 YAML 文件：

    ```yaml
    name: my-workflow
    kind: module
    description: My custom workflow
    params:
      - name: target
        required: true
    steps:
      - name: scan-target
        type: bash
        command: nmap {{target}}
    ```

### 有哪些步骤类型可用？

| 类型               | 描述                             |
    | ------------------ | -------------------------------- |
    | `bash`             | 执行 shell 命令                  |
    | `function`         | 执行 JavaScript 工具函数         |
    | `parallel-steps`   | 并发运行步骤                     |
    | `foreach`          | 遍历项目                         |
    | `remote-bash`      | 在 Docker 中或通过 SSH 执行      |
    | `http`             | 发起 HTTP 请求                   |
    | `llm`              | 执行 LLM API 调用                |
    | `agent`            | 带工具调用的代理型 LLM 执行      |

***

## 5. API 与服务器

### 如何启动 API 服务器？

```bash
    osmedeus server
    ```

    服务器默认在端口 8002 上启动。

### 我遇到了 `failed to run database migrations` 错误，该怎么办？

```bash
    $ osmedeus server
    2026-02-16T22:59:11+07:00 ERROR Failed to create server {"error": "failed to run database migrations: failed to create index: SQL logic error: no such column: webhook_uuid (1)"}
    Error: failed to run database migrations: failed to create index: SQL logic error: no such column: ... (1)
    ```

    此类错误意味着数据库架构已过时，服务器无法启动。要修复此问题，你可以运行 `osmedeus db clean --force` 清理数据库，然后重新启动服务器。这将重置你的数据库，因此请确保在运行命令之前备份任何重要数据。

### 如何通过 API 进行身份验证？

```bash
    # 获取 JWT 令牌
    curl -X POST http://localhost:8002/osm/api/login \
      -H "Content-Type: application/json" \
      -d '{"username": "osmedeus", "password": "admin"}'

    # 使用令牌
    curl http://localhost:8002/osm/api/workflows \
      -H "Authorization: Bearer <token>"
    ```

### 如何禁用身份验证？

```bash
    osmedeus server --no-auth
    ```

### 能否使用 API 密钥代替 JWT？

可以，在服务器配置中启用 API 密钥身份验证。然后使用 `X-API-Key` 标头代替 `Authorization: Bearer`。

***

## 6. 调度

### 如何安排定期扫描？

```bash
    # 通过 CLI（创建 cron 调度）
    osmedeus run -f subdomain-enum -t example.com --schedule "0 2 * * *"

    # 通过 API
    curl -X POST http://localhost:8002/osm/api/schedules \
      -H "Authorization: Bearer $TOKEN" \
      -H "Content-Type: application/json" \
      -d '{
        "name": "daily-scan",
        "workflow_name": "subdomain-enum",
        "target": "example.com",
        "schedule": "0 2 * * *"
      }'
    ```

### 使用什么 cron 格式？

标准 5 字段 cron：`分钟 小时 日 月 星期`

    示例：

    * `0 2 * * *` - 每天凌晨 2 点
    * `0 0 * * 0` - 每周日
    * `*/30 * * * *` - 每 30 分钟

***

## 7. Runner

### 有哪些 Runner 可用？

| Runner   | 描述                            |
    | -------- | ------------------------------- |
    | `host`   | 在本地机器上执行（默认）        |
    | `docker` | 在 Docker 容器中执行            |
    | `ssh`    | 通过 SSH 在远程机器上执行       |

### 如何在 Docker 中运行扫描？

```bash
    osmedeus run -m port-scan -t example.com --runner docker --docker-image osmedeus/osmedeus:latest
    ```

### 如何在远程主机上运行扫描？

```bash
    osmedeus run -m port-scan -t example.com --runner ssh --ssh-host worker.example.com
    ```

***

## 8. 分布式模式

### 如何设置分布式扫描？

启动主节点：

    ```bash
    osmedeus server --master
    ```

    加入工作节点：

    ```bash
    osmedeus worker join --master http://master:8002
    ```

### 如何向分布式池提交任务？

```bash
    curl -X POST http://localhost:8002/osm/api/tasks \
      -H "Authorization: Bearer $TOKEN" \
      -H "Content-Type: application/json" \
      -d '{
        "workflow_name": "subdomain-enum",
        "target": "example.com"
      }'
    ```

***

## 9. 故障排除

### 如何检查工具是否已安装？

```bash
    osmedeus install binary --all --check
    ```

### 如何安装缺失的工具？

```bash
    # 安装特定工具
    osmedeus install binary --name nuclei --name httpx

    # 安装所有工具
    osmedeus install binary --all
    ```

### 如何查看扫描日志？

日志存储在项目空间中：

    ```bash
    cat ~/osmedeus-base/workspaces/<target>/log/execution.log
    ```

### 如何导出项目空间以供共享？

```bash
    osmedeus snapshot export <workspace>
    ```

### 如何导入共享的项目空间？

```bash
    osmedeus snapshot import snapshot.zip
    ```

***

## 10. 设置

### 配置文件在哪里？

`~/osmedeus-base/osm-settings.yaml`

### 如何更改默认端口？

编辑 `osm-settings.yaml`：

    ```yaml
    server:
      port: 9000
    ```

    或使用 `--port` 标志：

    ```bash
    osmedeus server --port 9000
    ```

### 如何配置数据库设置？

编辑 `osm-settings.yaml`：

    ```yaml
    database:
      db_engine: sqlite3  # 或 postgres
      host: localhost
      port: 5432
      name: osmedeus
      username: user
      password: pass
    ```

## 11. 清理与卸载

### 如何清理项目空间？

运行以下命令清理项目空间和数据库，并生成默认的 osmedeus 配置：

    ```bash
    rm -rf ~/osmedeus-base ~/workspaces-osmedeus
    osmedeus install base --preset
    ```

### 如何清理项目空间？

运行以下命令清理项目空间和数据库，并生成默认的 osmedeus 配置：

    ```bash
    rm -rf ~/workspaces-osmedeus
    osmedeus db clean --force
    ```

### 如何卸载 osmedeus？

运行以下命令：

    ```bash
    rm -rf ~/.osmedeus ~/osmedeus-base ~/workspaces-osmedeus
    rm -rf $(which osmedeus)
    ```